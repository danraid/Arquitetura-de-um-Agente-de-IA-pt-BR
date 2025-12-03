# Fase 3 — Implementação do Backend, Agentes e API

> Objetivo: construir, em código, todo o "miolo" da solução de **assistente de políticas internas**, com backend HTTP, agentes de IA, camada de segurança, observabilidade e testes — pronto para ser containerizado e plugado na esteira de CI/CD (Fase 4).

---

## Índice

1. [3.0 Visão geral e objetivos](#30-visão-geral-e-objetivos)
2. [3.1 Estrutura do repositório e módulos](#31-estrutura-do-repositório-e-módulos)
3. [3.2 Configuração e gestão de ambiente](#32-configuração-e-gestão-de-ambiente)
4. [3.3 Modelagem de domínio em código](#33-modelagem-de-domínio-em-código)
5. [3.4 Adapters de infraestrutura](#34-adapters-de-infraestrutura)
6. [3.5 Implementação dos agentes](#35-implementação-dos-agentes)
7. [3.6 Gestão de prompts (PromptStore)](#36-gestão-de-prompts-promptstore)
8. [3.7 API HTTP (FastAPI)](#37-api-http-fastapi)
9. [3.8 Sessão e contexto de conversa](#38-sessão-e-contexto-de-conversa)
10. [3.9 Erros, timeouts e fallbacks](#39-erros-timeouts-e-fallbacks)
11. [3.10 Segurança na aplicação](#310-segurança-na-aplicação)
12. [3.11 Observabilidade & Telemetria (IAOps-ready)](#311-observabilidade--telemetria-iaops-ready)
13. [3.12 Testes (unit, integração, mini-eval)](#312-testes-unit-integração-mini-eval)
14. [3.13 Artefatos da Fase 3 & Gate de saída](#313-artefatos-da-fase-3--gate-de-saída)
15. [3.14 Diagramas de arquitetura (C4)](#314-diagramas-de-arquitetura-c4)
16. [3.15 Link para Fase 4](#315-link-para-fase-4)

---

## 3.0 Visão geral e objetivos

A Fase 3 pega tudo o que foi preparado nas Fases 1 e 2 e transforma em **software executável**:

* Backend HTTP (FastAPI) para o chatbot de políticas internas.
* Implementação dos agentes:

  * `OrchestratorAgent`
  * `PolicyRAGAgent`
  * `GuardAgent`
  * Agentes auxiliares: `IntentClassifierAgent`, `EvalAgent`.
* Adapters de infraestrutura para Azure OpenAI, Azure AI Search, Redis, Storage, Telemetria.
* Gestão de prompts, sessões e contexto de conversa.
* Camada de segurança (auth, autorização, filtros, clearance).
* Tratamento de erros, timeouts e fallbacks.
* Testes unitários e de integração desde o primeiro sprint.

**Saída esperada:** backend executável, versionado, com agentes funcionando ponta-a-ponta em DEV/STAGING, pronto para containerização (Fase 4).

---

## 3.1 Estrutura do repositório e módulos

### 3.1.1 Estrutura de alto nível (monorepo orientado a domínio/IA)

```text
/ai-policy-assistant/
  backend/
    app/
      api/
      agents/
      core/
      domain/
      infra/
      services/
      prompts/
      tests/
    pyproject.toml
    README.md
  infra/
    bicep-or-terraform/
    pipelines/
  docs/
    phase0-....md
    phase1-....md
    phase2-....md
    phase3-....md
```

### 3.1.2 Organização do backend (`backend/app`)

* `main.py` — ponto de entrada FastAPI.
* `config.py` — leitura de env vars e configs.
* `api/`

  * `routes_chat.py` — `/api/chat`.
  * `routes_feedback.py` — `/api/feedback`.
  * `routes_health.py` — `/health`.
  * `routes_admin.py` — `/api/admin/*`.
* `domain/`

  * `models.py` — `UserContext`, `Question`, `Answer`, `PolicyChunkRef`, `Interaction` etc.
  * `value_objects.py` — enums, tipos específicos.
* `infra/`

  * `azure_openai_client.py`
  * `azure_search_client.py`
  * `redis_client.py`
  * `telemetry.py` — logging, métricas, tracing.
  * `auth.py` — integração com Entra ID / JWT.
* `services/`

  * `chat_service.py` — orquestra chamada dos agentes.
  * `rag_service.py` — wrapper de chamada RAG + Search.
  * `guard_service.py` — wrapper do `GuardAgent` + Content Safety.
  * `feedback_service.py` — persistência de feedback.
* `agents/`

  * `base_agent.py` — contratos abstratos.
  * `orchestrator_agent.py`
  * `policy_rag_agent.py`
  * `guard_agent.py`
  * `intent_classifier_agent.py`
  * `eval_agent.py`.
* `prompts/`

  * `orchestrator/system_v1.txt`, `examples_v1.json`
  * `policy_rag/system_v1.txt`, `examples_v1.json`
  * `guard/system_v1.txt`, ...
  * `registry.yaml` — mapeia agente → versões → arquivos.
* `tests/`

  * `unit/`, `integration/`, `contract/`.

**Benefício:** Cada peça de domínio (agentes, infra, API, prompts) evolui de forma isolada, mantendo rastreabilidade para versionamento futuro.

---

## 3.2 Configuração e gestão de ambiente

### 3.2.1 Configuração via `config.py`

Todas as configurações devem vir por variáveis de ambiente (12-Factor):

* `APP_ENV` (`dev`, `staging`, `prod`).
* `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_KEY`, `AZURE_OPENAI_MODEL_CHAT`, `AZURE_OPENAI_MODEL_EMBEDDING`.
* `AZURE_SEARCH_ENDPOINT`, `AZURE_SEARCH_KEY`, `AZURE_SEARCH_INDEX_NAME`.
* `REDIS_URL`.
* `TELEMETRY_CONNECTION_STRING`.
* `JWT_ISSUER`, `JWT_AUDIENCE`.
* `GUARD_POLICY_LEVEL`.
* Parâmetros de RAG: `RAG_TOP_K`, `RAG_SCORE_THRESHOLD`.

Arquivos `.env.dev`, `.env.staging`, `.env.prod` (sem segredos commitados) podem auxiliar no desenvolvimento.

### 3.2.2 Princípios 12-Factor + 12-Factor Agent

* App **stateless** (estado de sessão → Redis / Storage).
* Config externa, nunca hardcoded.
* Logs em `stdout` em formato JSON, coletados por AppInsights / Log Analytics.
* Agentes como workers stateless:

  * Recebem input completo.
  * Devolvem output + telemetria da execução.

---

## 3.3 Modelagem de domínio em código

### 3.3.1 Modelos essenciais (`domain/models.py`)

Sugestão em Pydantic:

```python
class UserContext(BaseModel):
    user_id: str
    email: str | None = None
    department: str | None = None
    roles: list[str] = []
    clearance_level: str  # INTERNAL, INTERNAL_RESTRICTED, CONFIDENTIAL
    locale: str = "pt-BR"

class Question(BaseModel):
    text: str
    timestamp: datetime
    channel: str  # web, teams, etc.
    session_id: str | None = None

class PolicyChunkRef(BaseModel):
    chunk_id: str
    policy_id: str
    policy_title: str
    section_title: str
    source_url: str
    confidence: float

class Answer(BaseModel):
    text: str
    sources: list[PolicyChunkRef]
    meta: dict[str, Any] = {}

class InteractionMetrics(BaseModel):
    latency_ms: int
    tokens_input: int
    tokens_output: int
    model_name: str
    cache_hit: bool = False

class Interaction(BaseModel):
    interaction_id: str
    user_context: UserContext
    question: Question
    answer: Answer | None = None
    created_at: datetime
    agent_path: list[str] = []
    metrics: InteractionMetrics | None = None
```

### 3.3.2 Contrato genérico de agente (`agents/base_agent.py`)

```python
class AgentInput(BaseModel):
    interaction: Interaction
    payload: dict[str, Any] = {}

class AgentOutput(BaseModel):
    interaction: Interaction
    logs: list[dict[str, Any]] = []

class Agent(Protocol):
    name: str
    version: str

    async def run(self, input: AgentInput) -> AgentOutput: ...
```

Cada agente pode especializar `AgentInput`/`AgentOutput`, mas o contrato básico facilita telemetria genérica.

---

## 3.4 Adapters de infraestrutura

### 3.4.1 Azure OpenAI Client (`infra/azure_openai_client.py`)

Responsabilidades:

* Encapsular chamadas de **chat completions** e **embeddings**.
* Implementar timeouts, retries com backoff, tratamento de erros específicos.

```python
class AzureOpenAIClient:
    async def chat(self, messages: list[dict], *, model: str | None = None,
                   temperature: float = 0.2, max_tokens: int | None = None,
                   **kwargs) -> ChatResponse:
        ...

    async def embed(self, texts: list[str], *, model: str | None = None) -> list[list[float]]:
        ...
```

### 3.4.2 Azure Search Client (`infra/azure_search_client.py`)

```python
class AzureSearchClient:
    async def search_policies(self, query: str, user_context: UserContext,
                              top_k: int, score_threshold: float
                              ) -> list[PolicyChunkRef]:
        ...
```

* Aplica filtros por:

  * `sensitivity_level` ≤ `clearance_level` do usuário.
  * `effective_from` ≤ hoje ≤ `effective_to`.
  * `area` (se desejar restringir por departamento).
* Ajusta tipo de busca (lexical, semantic, vector ou híbrido).

### 3.4.3 Redis Client (`infra/redis_client.py`)

* Cache de resultados de RAG.
* Armazenamento de histórico curto de conversa.

```python
class SessionStore:
    async def get_history(self, session_id: str) -> list[dict]: ...
    async def append_message(self, session_id: str, role: str, content: str): ...
```

### 3.4.4 Telemetry & Logging (`infra/telemetry.py`)

* Logging estruturado.
* Emissão de métricas e traces.

```python
class Telemetry:
    def log_interaction(self, interaction: Interaction): ...
    def log_agent_run(self, agent_name: str, input, output, metrics): ...
    def track_metric(self, name: str, value: float, properties: dict | None = None): ...
```

---

## 3.5 Implementação dos agentes

### 3.5.1 OrchestratorAgent

Responsabilidades:

* Receber `UserContext`, `Question` e histórico de sessão.
* Determinar intenção (policy vs outro assunto) via `IntentClassifierAgent`.
* Chamar ou não `PolicyRAGAgent`.
* Passar resposta pelo `GuardAgent`.

Pseudo-código:

```python
class OrchestratorAgent(Agent):
    name = "orchestrator"
    version = "1.0.0"

    async def run(self, input: OrchestratorInput) -> OrchestratorOutput:
        intent = await self.intent_classifier.classify(
            question=input.question.text,
            user_context=input.user_context,
        )
        path = ["Orchestrator"]

        if intent != "POLICY_QUERY":
            answer = self._build_out_of_scope_answer()
            return OrchestratorOutput(answer=answer, agent_path=path)

        policy_answer = await self.policy_rag_agent.run(...)
        path.append("PolicyRAG")

        guard_result = await self.guard_agent.run(...)
        path.append("Guard")

        final_answer = self._merge_policy_answer_and_guard(policy_answer, guard_result)
        return OrchestratorOutput(answer=final_answer, agent_path=path)
```

### 3.5.2 IntentClassifierAgent

* Combinação de regras simples (regex/keywords) + LLM leve.
* Classes: `POLICY_QUERY`, `SMALL_TALK`, `TECHNICAL_SUPPORT`, `OUT_OF_SCOPE`.

```python
class IntentClassifierAgent(Agent):
    async def run(self, input: IntentInput) -> IntentOutput:
        ...  # retorna intent + confidence
```

### 3.5.3 PolicyRAGAgent

Fluxo:

1. Recebe `question` + `UserContext` (+ histórico curto opcional).
2. Chama `RAGService.search` para obter `List[PolicyChunkRef]`.
3. Monta prompt usando:

   * `policy_rag/system_v1.txt`.
   * `policy_rag/examples_v1.json`.
   * Contexto: chunks numerados (id, seção, conteúdo).
4. Chama `AzureOpenAIClient.chat(...)`.
5. Pede retorno em JSON: `{"answer": "...", "chunks_used": ["id1", ...]}`.
6. Mapeia `chunks_used` → `PolicyChunkRef`.
7. Constrói `Answer` e devolve.

Trecho conceitual de system prompt:

> Você é o assistente oficial de políticas internas da empresa.
> Suas respostas devem seguir rigorosamente as políticas fornecidas como contexto.
> NUNCA invente regras. Se a política não cobrir algo, diga que não há informação suficiente.
> Sempre cite o nome da política e a seção usada como base.

### 3.5.4 GuardAgent

Entradas:

* Texto da resposta proposta.
* Lista de `PolicyChunkRef` usados.
* `UserContext`.

Passos:

1. Validar `sensitivity_level` dos chunks vs `clearance_level` do usuário.
2. Opcional: chamar Azure AI Content Safety.
3. Opcional: chamar LLM de auditor para checar violações de regras internas.
4. Se inseguro:

   * Tentar sanitizar pedindo nova resposta sem os trechos problemáticos.
   * Se falhar, bloquear e retornar mensagem neutra.

Saída:

```python
class GuardResult(BaseModel):
    allowed: bool
    sanitized_answer: str | None
    reason: str | None
```

### 3.5.5 EvalAgent (IA Evaluation)

* Usado em testes/evaluation, não no fluxo online principal.
* Dado pergunta, resposta gerada e resposta esperada, devolve scores (0–1) de:

  * Correção.
  * Clareza.
  * Aderência às políticas.

---

## 3.6 Gestão de prompts (PromptStore)

### 3.6.1 Organização em disco

`prompts/registry.yaml`:

```yaml
orchestrator:
  current_version: "1.0.0"
  versions:
    "1.0.0":
      system: "orchestrator/system_v1.txt"
      examples: "orchestrator/examples_v1.json"

policy_rag:
  current_version: "1.0.0"
  versions:
    "1.0.0":
      system: "policy_rag/system_v1.txt"
      examples: "policy_rag/examples_v1.json"
```

### 3.6.2 PromptStore

```python
class PromptConfig(BaseModel):
    system_text: str
    examples: list[dict] = []
    version: str

class PromptStore:
    def __init__(self, registry_path: str):
        ...

    def get_prompt(self, agent_name: str, version: str | None = None) -> PromptConfig:
        ...
```

Uso:

* Cada agente recebe um `PromptStore` via injeção de dependência.
* Carrega `PromptConfig` no startup ou sob demanda com cache.

### 3.6.3 QA de prompts

* Testes unitários garantindo:

  * Arquivos existem.
  * Não estão vazios.
  * Placeholders são resolvidos corretamente.

---

## 3.7 API HTTP (FastAPI)

### 3.7.1 Endpoint `/api/chat`

**Request model:**

```python
class ChatRequest(BaseModel):
    message: str
    session_id: str | None = None
```

**Response model:**

```python
class PolicyChunkRefModel(BaseModel):
    chunk_id: str
    policy_id: str
    policy_title: str
    section_title: str
    source_url: str
    confidence: float

class ChatResponse(BaseModel):
    answer: str
    sources: list[PolicyChunkRefModel]
    metrics: dict[str, Any]
    interaction_id: str
```

**Fluxo interno:**

1. Autenticar e extrair claims → `UserContext`.
2. Construir `Question`.
3. Chamar `chat_service.handle_question(...)`.
4. `chat_service`:

   * Cria `Interaction`.
   * Chama `OrchestratorAgent`.
   * Recebe `Answer`.
   * Popula métricas.
   * Loga interação.
5. Retorna `ChatResponse`.

### 3.7.2 Endpoint `/api/feedback`

```python
class FeedbackRequest(BaseModel):
    interaction_id: str
    rating: int  # -1, 0, 1
    comment: str | None = None
```

* Service persiste feedback e emite evento para uso em IA evaluation / ajuste de prompts.

### 3.7.3 Endpoint `/health`

* Checa:

  * Conexão com Azure Search.
  * Conexão com Azure OpenAI (ping leve opcional).
  * Conexão com Redis.

### 3.7.4 Endpoints admin `/api/admin/*`

* `/api/admin/reindex_policy` — dispara pipelines de DataOps ou eventos.
* `/api/admin/toggle_agent` — ativa/desativa versões de agente.
* Protegidos por roles específicas (ex.: `PolicyAdmin`).

---

## 3.8 Sessão e contexto de conversa

### 3.8.1 Modelo de histórico (Redis)

* Guardar últimas **N** mensagens (ex.: 5 pares user/assistant):

```json
[
  {"role": "user", "content": "..."},
  {"role": "assistant", "content": "..."}
]
```

### 3.8.2 Uso no RAG

* Orchestrator/PolicyRAG podem incluir histórico reduzido no prompt para perguntas encadeadas.
* Controlar:

  * Limite de tokens do histórico.
  * Tamanho máximo por sessão.

### 3.8.3 QA

Cenário de teste:

1. Q1: "Quais meus direitos de férias?".
2. Q2: "E posso dividir em 3 períodos?".

* Verificar que Q2 é respondida considerando o contexto de Q1.

---

## 3.9 Erros, timeouts e fallbacks

### 3.9.1 Casos mapeados

* Timeout do LLM.
* Erro do Azure Search.
* Falha de Redis.
* Exceções internas.

### 3.9.2 Estratégia

* Middleware global de erro:

  * Captura exceções.
  * Loga com `request_id` e `interaction_id`.
  * Retorna mensagem genérica ao cliente.
* Timeout LLM (ex.: 10s):

  * Abort a chamada.
  * Logar evento.
  * Responder fallback.
* Erro Search:

  * Tentar fallback lexical.
  * Se ainda falhar, informar indisponibilidade temporária.

### 3.9.3 QA

* Testes simulando:

  * Timeout (mock client LLM).
  * Erro 500 Search.
  * Falha de conexão Redis.

---

## 3.10 Segurança na aplicação

### 3.10.1 Autenticação (JWT / Entra ID)

* Middleware/dependency do FastAPI:

  * Lê `Authorization: Bearer <token>`.
  * Valida assinatura com chave pública.
  * Verifica `audience` e `issuer`.
  * Em caso de falha → `HTTP 401`.

### 3.10.2 Autorização

* Mapeia claims → `UserContext.clearance_level`.
* Exemplos:

  * Grupo `Security-Team` → `CONFIDENTIAL`.
  * Qualquer funcionário → `INTERNAL`.
* Helpers:

  * `require_role("PolicyAdmin")` para endpoints admin.
  * `require_clearance("INTERNAL")` para usar o bot.

### 3.10.3 Sanitização de logs

* Evitar logar texto completo de perguntas/respostas se forem sensíveis.
* Opções:

  * Logar apenas hash.
  * Logar apenas trecho truncado.
* Respeitar políticas internas de privacidade/compliance.

---

## 3.11 Observabilidade & Telemetria (IAOps-ready)

### 3.11.1 Logging de `AgentRun`

Formato sugerido:

```json
{
  "type": "agent_run",
  "agent_name": "PolicyRAGAgent",
  "agent_version": "1.0.0",
  "interaction_id": "uuid",
  "latency_ms": 850,
  "model_name": "gpt-4.x",
  "tokens_input": 900,
  "tokens_output": 180,
  "cache_hit": false,
  "success": true
}
```

### 3.11.2 Métricas principais

* `agent_latency_ms{agent_name}`.
* `agent_error_count{agent_name}`.
* `rag_chunks_used_count`.
* `guard_blocked_count`.
* `guard_sanitized_count`.

### 3.11.3 Tracing (OpenTelemetry)

* Span raiz: `chat_request`.
* Subspans:

  * `intent_classification`.
  * `rag_search`.
  * `llm_policy_answer`.
  * `guard_check`.

Usado depois em testes de carga e tuning de custo/performance.

---

## 3.12 Testes (unit, integração, mini-eval)

### 3.12.1 Testes unitários

* `tests/unit/test_orchestrator_agent.py`

  * Mock de `IntentClassifierAgent`, `PolicyRAGAgent`, `GuardAgent`.
  * Casos:

    * Intenção ≠ policy → resposta "fora de escopo".
    * Intenção = policy → chama RAG + Guard.

* `tests/unit/test_policy_rag_agent.py`

  * Mock de `AzureSearchClient`, `AzureOpenAIClient`.
  * Valida formatação de prompt, uso de `top_k`.

* `tests/unit/test_guard_agent.py`

  * Casos:

    * Chunk confidencial para usuário sem clearance → `allowed=False`.
    * Resposta segura → `allowed=True`.

* `tests/unit/test_auth.py`

  * Parsing de token, roles, clearance.

### 3.12.2 Testes de integração

Ambiente de dev com:

* Azure Search apontando para índice de dev.
* Azure OpenAI real ou mock.
* Redis (container/local).

Cenários:

1. Pergunta real de férias → resposta com referência à política correta.
2. Política inexistente → resposta "não tenho informação".
3. Tentativa de jailbreak → Guard impede violação de política.

### 3.12.3 Mini-bateria de IA Evaluation

* 20–50 perguntas de teste com resposta esperada (extraída manualmente da política).
* Script que:

  * Chama `/api/chat` para cada pergunta.
  * Armazena respostas.
  * Opcional: usa `EvalAgent` para gerar score.
* Métrica: `policy_answer_accuracy` ≈ % de respostas com score ≥ threshold.

---

## 3.13 Artefatos da Fase 3 & Gate de saída

### 3.13.1 Artefatos esperados

* Código do backend em `backend/app`:

  * API, domain, infra, services, agents, prompts.
* `requirements.txt` ou `pyproject.toml` definido.
* Documentos em `docs/`:

  * `31-backend-architecture-implementation.md` — módulos e dependências.
  * `32-agents-implementation-details.md` — contratos, fluxos, pseudo-códigos.
  * `33-prompts-registry-and-usage.md`.
  * `34-api-contracts.md` — schemas `/chat`, `/feedback`, etc.
  * `35-error-handling-and-fallbacks.md`.
  * `36-security-implementation.md`.
  * `37-observability-implementation.md`.
  * `38-test-strategy-phase3.md`.

### 3.13.2 Critérios GO/NO-GO

**GO se:**

* Endpoints principais (`/api/chat`, `/api/feedback`, `/health`) estão:

  * Implementados.
  * Testados (unit + integração).
* Agentes principais (`Orchestrator`, `PolicyRAG`, `Guard`, `IntentClassifier`) estão implementados e integrados.
* Chamadas a Azure Search, Azure OpenAI e Redis funcionam em DEV/STAGING.
* Testes automatizados com cobertura mínima em módulos críticos.
* Mini-bateria de IA evaluation atinge acurácia mínima (ex.: ≥ 70% no golden set).
* Segurança básica implementada:

  * Auth + autorização.
  * Guard bloqueando acesso indevido a políticas confidenciais.

**NO-GO se:**

* `/api/chat` ainda depende de mocks para RAG/LLM.
* `GuardAgent` não implementado ou bypassado.
* Logs/telemetria insuficientes para reconstruir uma interação.
* Testes automatizados quase inexistentes.

---

## 3.14 Diagramas de arquitetura (C4)

### 3.14.1 C4 — Nível 1 (Contexto)

* **Sistema:** AI Policy Assistant.
* **Atores externos:**

  * Colaborador (usuário final via Web/Teams).
  * Admin de Políticas.
* **Sistemas externos:**

  * Azure OpenAI.
  * Azure AI Search.
  * Azure Storage / Data Lake.
  * Azure Monitor / AppInsights.
  * Entra ID (IdP).

> Diagrama (descrição):
> O colaborador interage com o AI Policy Assistant (backend FastAPI).
> O backend usa Azure AI Search para recuperar chunks de políticas (baseado na Fase 2) e Azure OpenAI para gerar respostas, com logs e métricas indo para Azure Monitor/AppInsights. Entra ID fornece tokens JWT para autenticação.

### 3.14.2 C4 — Nível 2 (Containers)

Containers principais:

* `Web/API Backend (FastAPI)`
* `Redis (cache/sessão)`
* `Azure AI Search (policy-index)`
* `Azure OpenAI (chat + embeddings)`
* `Data Lake (raw/extracted/processed)`
* `Monitoring/Logging` (AppInsights/Log Analytics)

Relações:

* Backend ↔ Redis (sessão/cache).
* Backend ↔ Azure Search (RAG).
* Backend ↔ Azure OpenAI (LLM/embeddings).
* Backend → Monitoring (logs/metrics/traces).

### 3.14.3 C4 — Nível 3 (Componentes do Backend)

Componentes do backend:

* `API Layer` (routes_*).
* `ChatService`.
* `RAGService`.
* `GuardService`.
* `Agents` (orchestrator, policy_rag, guard, intent_classifier, eval).
* `Infra Adapters` (openai_client, search_client, redis_client, telemetry, auth).
* `Domain Models`.
* `PromptStore`.

Fluxo típico:

1. Request HTTP chega em `routes_chat`.
2. `ChatService` cria `Interaction` e chama `OrchestratorAgent`.
3. `OrchestratorAgent` chama `PolicyRAGAgent`, que usa `RAGService`.
4. `RAGService` usa `AzureSearchClient` e `AzureOpenAIClient`.
5. `GuardAgent` valida saída.
6. `ChatService` retorna resposta e loga via `Telemetry`.

---

## 3.15 Link para Fase 4

Próximo documento:

➡️ [Fase 4 — Containerização & CI/CD (DevOps puro)](Fase4.md)
