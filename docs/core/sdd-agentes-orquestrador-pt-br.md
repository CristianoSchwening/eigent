# SDD — Agentes necessários (Orquestrador) — Eigent (Cursor)

Este documento descreve, em estilo **SDD (Software Design Description)**, os **agentes** e **componentes** necessários para executar o “agente orquestrador” do Eigent no ambiente local (incluindo execução via Cursor, com streaming de eventos para a UI).

## 1. Escopo e objetivos

### 1.1 Objetivo

Especificar, de forma implementável, **quais agentes** existem, **quais são obrigatórios**, como são **instanciados**, quais **toolkits** dependem, como os eventos são **orquestrados** e quais **configurações/variáveis de ambiente** são necessárias para o sistema funcionar.

### 1.2 Fora de escopo

- Implementação/alteração de UI (frontend).
- Estratégia de prompts além do que já está codificado em `backend/app/agent/prompt.py`.
- Provisionamento de credenciais (é responsabilidade do operador).

## 2. Referências de código (fonte de verdade)

- **Loop de orquestração + SSE + lifecycle**: `backend/app/service/chat_service.py` (`step_solve()`)
- **Workforce (decomposição, atribuição, execução, métricas, cleanup)**: `backend/app/utils/workforce.py`
- **Worker executor (processamento de subtarefas)**: `backend/app/utils/single_agent_worker.py`
- **Agente instrumentado (eventos activate/deactivate + toolkits)**: `backend/app/agent/listen_chat_agent.py`
- **Factory de agente base (modelo + toolkits + tracking)**: `backend/app/agent/agent_model.py` (`agent_model()`)
- **Factories específicas por tipo**: `backend/app/agent/factory/*.py`
- **Prompts de sistema**: `backend/app/agent/prompt.py`
- **Loader de toolkits e MCP tools**: `backend/app/agent/tools.py`
- **Contratos de eventos (Action*) e TaskLock**: `backend/app/service/task.py`

## 3. Visão geral da arquitetura

### 3.1 Conceitos principais

- **Projeto / api_task_id**: identificador do “projeto” (o mesmo id usado para `TaskLock` e para a UI).
- **TaskLock**: estrutura por projeto com fila (`queue`) de eventos `Action*` e estado compartilhado.
- **Orquestrador**: o par:
  - `step_solve()` (dispatcher de eventos + SSE) e
  - `Workforce` (agendamento/execução de subtarefas).
- **Agentes**:
  - Agentes “core” de Workforce: `coordinator_agent`, `task_agent`, `new_worker_agent`.
  - Agentes workers (capabilidades): `developer`, `browser`, `document`, `multi_modal` e opcionalmente outros.
  - Agentes auxiliares: `question_confirm_agent`, `task_summary_agent`, `mcp_agent`.

### 3.2 Sequência nominal (alto nível)

1. Entrada HTTP coloca ações em `TaskLock.queue`.
2. `step_solve()` consome a fila e decide:
   - **Pergunta simples** → resposta direta com `question_confirm_agent` (sem Workforce).
   - **Tarefa complexa** → cria Workforce, decompõe em subtarefas, aguarda `Action.start`.
3. Em `Action.start`, `Workforce.eigent_start()` executa subtarefas:
   - atribui um worker por subtask,
   - executa via `SingleAgentWorker` / `ListenChatAgent`,
   - emite eventos de execução (`assign_task`, `activate_agent`, toolkits, `task_state`, `end`).
4. `step_solve()` transforma ações em **SSE** para a UI.
5. No final, Workforce é encerrado e faz cleanup (incl. recursos de browser/CDP).

## 4. Inventário de agentes (todos os necessários)

> Regra prática: para “rodar aqui no Cursor” com a experiência completa (planejar subtarefas + executar ferramentas + streaming), o conjunto mínimo é:
> `question_confirm_agent`, `coordinator_agent`, `task_agent`, `new_worker_agent`, `developer_agent`, `browser_agent`, `document_agent`, `multi_modal_agent`.
> `mcp_agent` é **recomendado** (habilita MCP search/instalação).

### 4.1 Agentes core do Workforce (obrigatórios no modo complexo)

#### 4.1.1 `coordinator_agent` (obrigatório)

- **Papel**: coordenação durante decomposição/planejamento macro.
- **Criação**: em `construct_workforce()` via `agent_model()` com prompt inline (contém OS/arquitetura/diretório/data).
- **Tools**: lista vazia (no código atual, é criado com `[]`).
- **Entradas típicas**: “contexto do histórico de conversa” + tarefa atual.
- **Saídas**: influência no plano e na decomposição (via CAMEL).

#### 4.1.2 `task_agent` (obrigatório)

- **Papel**: decompõe tarefas e participa do planejamento de execução.
- **Criação**: em `construct_workforce()` via `agent_model()` com prompt inline.
- **Tools**: lista vazia (no código atual, é criado com `[]`).
- **Requisito especial**: em `Workforce.__init__`, há `self.task_agent.stream_accumulate = True` para suportar streaming de decomposição.

#### 4.1.3 `new_worker_agent` (obrigatório para padrões CAMEL / criação dinâmica)

- **Papel**: worker genérico para criar/adaptar workers (CAMEL workforce).
- **Criação**: em `construct_workforce()` via `agent_model()`.
- **Tools** (fixas no código):
  - `HumanToolkit` (mensagens/ask)
  - `NoteTakingToolkit` (via `ToolkitMessageIntegration`)
  - `SkillToolkit`
- **Observação**: este agente entra como `new_worker_agent` na criação do Workforce.

### 4.2 Agentes workers (executores de subtarefas)

Esses agentes são adicionados ao Workforce via `workforce.add_single_agent_worker(...)` em `construct_workforce()`.

#### 4.2.1 `developer_agent` (obrigatório na prática)

- **Factory**: `backend/app/agent/factory/developer.py`
- **Prompt base**: `DEVELOPER_SYS_PROMPT` em `backend/app/agent/prompt.py`
- **Toolkits** (sempre adicionados na factory):
  - `HumanToolkit`
  - `NoteTakingToolkit`
  - `WebDeployToolkit`
  - `TerminalToolkit` (safe_mode=True, clone_current_env=True)
  - `ScreenshotToolkit`
  - `SkillToolkit`
  - `SearchToolkit` (se disponível)
- **Uso típico**:
  - escrever/rodar código, manipular arquivos, instalar dependências, automatizar tarefas técnicas.
- **Observações operacionais**:
  - O `TerminalToolkit` é o núcleo para “rodar no Cursor” (execução de comandos e automações).

#### 4.2.2 `browser_agent` (obrigatório para pesquisa/navegação e coleta web)

- **Factory**: `backend/app/agent/factory/browser.py`
- **Prompt base**: `BROWSER_SYS_PROMPT`
- **Toolkits**:
  - `HybridBrowserToolkit` (CDP) com ferramentas habilitadas: click/type/back/forward/select/console/switch_tab/enter/visit/scroll/sheet_read/sheet_input/get_snapshot/open/upload/download
  - `TerminalToolkit` (apenas `shell_exec` registrado como função)
  - `NoteTakingToolkit`
  - `ScreenshotToolkit`
  - `SearchToolkit` (se disponível)
  - `SkillToolkit`
  - `HumanToolkit`
- **Requisito especial (CDP Browser Pool)**:
  - suporta múltiplas instâncias de browser por porta e “pool manager” (`CdpBrowserPoolManager`).
  - em clone, usa callbacks (`_cdp_acquire_callback` / `_cdp_release_callback`) para não conflitar portas.
- **Config**:
  - se `options.cdp_browsers` existir: tenta alocar porta livre.
  - senão: usa `env("browser_port","9222")`.

#### 4.2.3 `document_agent` (obrigatório quando há geração/edição de documentos)

- **Factory**: `backend/app/agent/factory/document.py`
- **Prompt base**: `DOCUMENT_SYS_PROMPT`
- **Toolkits**:
  - `FileToolkit` (escrita/edição de arquivos)
  - `PPTXToolkit`
  - `MarkItDownToolkit` (conversões/leitura)
  - `ExcelToolkit`
  - `NoteTakingToolkit`
  - `TerminalToolkit` (safe_mode=True, clone_current_env=True)
  - `ScreenshotToolkit`
  - `GoogleDriveMCPToolkit` (via bun env; pode ser vazio conforme credenciais)
  - `SkillToolkit`
  - `SearchToolkit` (se disponível)
  - `HumanToolkit`

#### 4.2.4 `multi_modal_agent` (obrigatório para mídia: imagem/áudio/vídeo)

- **Factory**: `backend/app/agent/factory/multi_modal.py`
- **Prompt base**: `MULTI_MODAL_SYS_PROMPT`
- **Toolkits**:
  - `VideoDownloaderToolkit`
  - `ScreenshotToolkit`
  - `TerminalToolkit` (safe_mode=True, clone_current_env=True)
  - `NoteTakingToolkit`
  - `SkillToolkit`
  - `SearchToolkit` (se disponível)
  - `HumanToolkit`
  - Condicionais:
    - `OpenAIImageToolkit` **somente** se `options.is_cloud() == True`
    - `AudioAnalysisToolkit` **somente** se `model_platform == OPENAI`

#### 4.2.5 `social_media_agent` (opcional / não entra no Workforce por padrão)

- **Factory**: `backend/app/agent/factory/social_media.py`
- **Prompt base**: `SOCIAL_MEDIA_SYS_PROMPT`
- **Toolkits**:
  - `WhatsAppToolkit`, `TwitterToolkit`, `LinkedInToolkit`, `RedditToolkit`
  - `NotionMCPToolkit`
  - `GoogleGmailMCPToolkit`, `GoogleCalendarToolkit`
  - `TerminalToolkit`, `NoteTakingToolkit`, `SkillToolkit`, `SearchToolkit`, `HumanToolkit`
- **Status**:
  - existe no codebase, mas **não é adicionado** ao Workforce em `construct_workforce()` (pode ser adicionado via `options.new_agents` / endpoint de “add-agent”).

### 4.3 Agentes auxiliares (controle/infra)

#### 4.3.1 `question_confirm_agent` (obrigatório para roteamento simples vs complexo)

- **Factory**: `backend/app/agent/factory/question_confirm.py`
- **Prompt base**: `QUESTION_CONFIRM_SYS_PROMPT`
- **Função**: responder apenas “yes/no” para decidir se usa Workforce.
- **Uso**: `question_confirm(...)` em `chat_service.py` chama `agent.step(...)`.

#### 4.3.2 `task_summary_agent` (recomendado; UX)

- **Factory**: `backend/app/agent/factory/task_summary.py`
- **Prompt base**: `TASK_SUMMARY_SYS_PROMPT`
- **Função**:
  - gerar “Task Name|Summary” após decomposição;
  - também pode sumarizar resultados de múltiplas subtarefas.

#### 4.3.3 `mcp_agent` (recomendado; extensibilidade)

- **Factory**: `backend/app/agent/factory/mcp.py`
- **Prompt base**: `MCP_SYS_PROMPT`
- **Tools**:
  - `McpSearchToolkit` (sempre)
  - MCP tools instalados de `options.installed_mcp` (via `get_mcp_tools`)
- **Observação**:
  - cria um `ListenChatAgent` manualmente e também emite `ActionCreateAgentData` para a UI.
  - habilita o fluxo `install_mcp(...)` em `chat_service.py`.

## 5. Como o orquestrador executa (design detalhado)

### 5.1 Dispatcher principal (`step_solve`)

Responsabilidades:

- Manter um loop consumindo `TaskLock.queue`.
- Transformar `Action*` em eventos SSE (`sse_json(...)`).
- Controlar lifecycle: criar/reusar/parar `Workforce` e limpar `TaskLock`.
- Implementar **multi-turn**: pausar workforce, confirmar complexidade, decompor novamente, retomar.

### 5.2 Workforce (`Workforce`)

Responsabilidades:

- **Decomposição**: `eigent_make_sub_tasks` / `handle_decompose_append_task`.
- **Execução**: `eigent_start` → `start()` do CAMEL.
- **Atribuição**: override `_find_assignee` (emite `ActionAssignTaskData` com estados `waiting`).
- **Início real**: override `_post_task` (emite `ActionAssignTaskData` com estado `running`).
- **Conclusão**: `_handle_completed_task` emite `ActionTaskStateData` e sincroniza results para parent subtasks.
- **Falha / retries**: `_handle_failed_task` mantém retry/replan e só emite erro final ao esgotar.
- **Timeout**: `_get_returned_task` com `asyncio.wait_for` e `ActionTimeoutData`.
- **Cleanup**:
  - encerra workforce e emite `ActionEndData` em `stop()`;
  - libera recursos CDP de browser (inclusive clones/pool).

### 5.3 Execução de subtarefas (SingleAgentWorker)

- Cada subtask vira um prompt `PROCESS_TASK_PROMPT` (CAMEL).
- `SingleAgentWorker._process_task`:
  - pega `worker_agent` via pool/clone,
  - seta `worker_agent.process_task_id = task.id`,
  - chama `worker_agent.astep(...)` e acumula streaming se necessário,
  - grava `task.result` e retorna `TaskState.DONE/FAILED`.

### 5.4 Instrumentação de eventos (ListenChatAgent)

O `ListenChatAgent` garante “telemetria funcional” para a UI:

- Antes de `step/astep`: enfileira `ActionActivateAgentData`.
- Após completar/erro: enfileira `ActionDeactivateAgentData` com tokens e mensagem.
- Ao executar tools:
  - ativa toolkit (`ActionActivateToolkitData`)
  - executa a função mantendo `ContextVar` de `process_task_id`
  - desativa toolkit (`ActionDeactivateToolkitData`)
- Streaming:
  - wrappers `_stream_chunks` / `_astream_chunks` acumulam conteúdo e disparam deactivate no fim do stream.

## 6. Contratos de eventos (SSE / fila)

### 6.1 Tipos principais (Action)

Definidos em `backend/app/service/task.py`:

- **Roteamento e lifecycle**: `improve`, `start`, `stop`, `pause`, `resume`, `end`, `timeout`
- **Workforce**:
  - `decompose_progress` (emissão final de subtasks)
  - `decompose_text` (streaming bruto da decomposição)
  - `assign_task` (waiting/running)
  - `task_state` (DONE/FAILED por subtask)
  - `new_task_state` (multi-turn)
- **Agentes**:
  - `create_agent`
  - `activate_agent` / `deactivate_agent`
- **Toolkits**:
  - `activate_toolkit` / `deactivate_toolkit`
  - `write_file`, `terminal`, `notice`, `ask`
- **MCP**:
  - `search_mcp`, `install_mcp`

### 6.2 Requisitos mínimos do frontend/consumidor SSE

Para “rodar no Cursor”, o consumidor SSE deve:

- Manter sessão por `project_id` e renderizar eventos na ordem recebida.
- Tratar `assign_task` com `state` `"waiting"` e `"running"`.
- Tratar `activate_agent/deactivate_agent` vinculando ao `process_task_id`.
- Tratar `activate_toolkit/deactivate_toolkit` para mostrar tool calls.
- Tratar `decompose_text` para exibir texto incremental e `decompose_progress`/`to_sub_tasks` para a árvore.
- Tratar `end` como encerramento lógico de tarefa (mas o loop aceita multi-turn).

## 7. Configuração necessária para rodar localmente (Cursor)

### 7.1 Config de modelo (obrigatória)

O sistema usa `options` (tipo `Chat`) com os campos:

- `model_platform` (ex.: `openai`, `anthropic`, `openrouter`, `aws-bedrock-converse`, etc.)
- `model_type`
- `api_key`
- `api_url`
- `extra_params` (timeout, retries, region, etc.)

O `agent_model()` em `backend/app/agent/agent_model.py` também injeta:

- caching de prompt:
  - Anthropic / Bedrock: `cache_control = "5m"`
  - OpenAI: `prompt_cache_key = project_id`
- Ajustes específicos:
  - `browser_agent`: desliga `parallel_tool_calls` em certas plataformas
  - `task_agent`: `stream = True`

### 7.2 Browser/CDP (recomendado para o browser_agent)

Opções:

- **Simples**: setar `browser_port` no env (padrão `9222`) e ter um Chrome/Chromium com `--remote-debugging-port=9222`.
- **Avançado**: preencher `options.cdp_browsers` com uma lista de instâncias (portas) para paralelismo.

Variável:

- `browser_port` (fallback usado em `browser.py`)

### 7.3 MCP (opcional, mas recomendado)

Para persistir autenticação do `mcp-remote` entre execuções, `get_mcp_tools()` injeta:

- `MCP_REMOTE_CONFIG_DIR` (default: `~/.mcp-auth`), pode ser definido por env.

### 7.4 Credenciais por toolkit (opcionais, por agente)

Cada toolkit pode exigir chaves/creds próprias (ex.: WhatsApp, Twitter, Google, Notion). O design do sistema prevê que:

- toolkits “sem creds” continuem operando (ou retornem erro controlado),
- o operador configure as creds via `.env`/ambiente conforme necessário.

## 8. Como adicionar um novo agente (extensão)

### 8.1 Caminho suportado pelo produto

O fluxo oficial para adicionar agentes extras ao Workforce é via:

- `options.new_agents` (criados no `construct_workforce()` após o Workforce ser instanciado), e/ou
- endpoint `POST /task/{id}/add-agent` que enfileira `Action.new_agent`.

O `step_solve()` trata `Action.new_agent` assim:

- pausa workforce,
- adiciona worker via `workforce.add_single_agent_worker(...)`,
- resume workforce.

### 8.2 Checklist de design para um novo agente

- Criar factory em `backend/app/agent/factory/<nome>.py`.
- Definir sys prompt em `backend/app/agent/prompt.py` (ou inline).
- Montar lista de toolkits:
  - `HumanToolkit` é recomendado para interrupções/perguntas.
  - `TerminalToolkit` é recomendado para “rodar no Cursor”.
  - `NoteTakingToolkit` e `SkillToolkit` seguem o padrão dos demais agentes.
- Se o toolkit aloca recursos (ex.: browser/CDP), garantir:
  - callbacks de acquire/release em clone,
  - cleanup em `Workforce._cleanup_all_agents`.

## 9. Matriz “Agente → Toolkits” (resumo)

- **Developer**: Human, Terminal, NoteTaking, WebDeploy, Screenshot, Skill, (Search)
- **Browser**: HybridBrowser(CDP), Terminal(shell_exec), NoteTaking, Screenshot, Skill, (Search), Human
- **Document**: FileWrite, PPTX, MarkItDown, Excel, Terminal, NoteTaking, Screenshot, GoogleDriveMCP, Skill, (Search), Human
- **Multi-Modal**: VideoDownload, Screenshot, Terminal, NoteTaking, Skill, (Search), Human, (+ OpenAIImage se cloud), (+ AudioAnalysis se OpenAI)
- **Social Media (opcional)**: WhatsApp/Twitter/LinkedIn/Reddit/NotionMCP/GoogleGmailMCP/Calendar + Terminal + Note + Skill + (Search) + Human
- **MCP**: McpSearch + MCP tools instalados
- **Question Confirm**: sem toolkits
- **Task Summary**: sem toolkits
- **Coordinator/Task Agent**: sem toolkits (no código atual)

## 10. Critérios de pronto (“done”) para rodar no Cursor

Você considera este SDD atendido quando:

- O backend consegue:
  - receber uma pergunta,
  - classificar simples vs complexa,
  - para complexa: decompor subtarefas, emitir streaming de decomposição e gerar “to_sub_tasks”,
  - iniciar execução com `Action.start`, atribuir workers e emitir eventos `assign_task`, `activate_agent`, toolkits e `task_state`,
  - finalizar com `end` e limpar recursos (incl. CDP pool).
- Pelo menos **um** worker com `TerminalToolkit` funciona (normalmente `developer_agent`).
- Se `browser_agent` estiver habilitado, existe um CDP browser configurado ou `browser_port` definido.

## 11. Apêndice — Lista de toolkits disponíveis (codebase)

Toolkits cadastrados em `backend/app/agent/tools.py` (pode haver dependências externas por toolkit):

`audio_analysis_toolkit`, `openai_image_toolkit`, `excel_toolkit`, `file_write_toolkit`, `github_toolkit`,
`google_calendar_toolkit`, `google_drive_mcp_toolkit`, `google_gmail_mcp_toolkit`, `linkedin_toolkit`,
`lark_toolkit`, `mcp_search_toolkit`, `notion_mcp_toolkit`, `pptx_toolkit`, `rag_toolkit`, `reddit_toolkit`,
`search_toolkit`, `slack_toolkit`, `terminal_toolkit`, `twitter_toolkit`, `video_analysis_toolkit`,
`video_download_toolkit`, `whatsapp_toolkit`.

