# 📚 Arquitetura de um Agente de IA Corporativo com RAG no Azure

> **Livro técnico em Markdown** sobre como projetar, construir, operar e governar
> um agente de IA corporativo (chatbot) que responde sobre políticas internas,
> usando **Azure**, **RAG**, **multi-agentes**, **DevOps/IAOps/DataOps/SRE**,
> **EU AI Act** e **LGPD**.

---

## 🔎 Visão Geral

Este repositório descreve, em formato de **livro técnico**, todas as fases
(da **Fase 0 à Fase 14**) de um projeto realista:

- Agente de IA (chatbot) que responde sobre **políticas internas da empresa**  
- Utilizando **Azure** (Blob, AI Search, AI Document Intelligence, Azure OpenAI)  
- Com **RAG** bem projetado (ingestão + indexação + metadados + cache)  
- Orquestrado por um **Multi-Agent Pattern**:
  - `OrchestratorAgent`
  - `PolicyRAGAgent`
  - `GuardAgent`
- Rodando em uma arquitetura **Hexagonal / Ports & Adapters**, com
  **DevOps + IAOps + DataOps + SRE + FinOps**  
- Governado segundo **EU AI Act** e **LGPD**, com versionamento completo de:
  - código
  - prompts
  - modelos
  - dados (índices, embeddings)
  - infraestrutura

---

## 🧱 Design Pattern & Arquitetura (resumo)

Antes de navegar pelas fases, alguns conceitos que aparecem em todo o livro:

### 🧩 Padrão de Arquitetura de Software

- **Hexagonal Architecture / Ports & Adapters**
  - Domínios principais:
    - `policies-data` (DataOps & RAG)
    - `chat-assistant` (agentes e regras de negócio)
    - `channels` (Web, Teams)
    - `observability` (logs, métricas, tracing)
    - `governance` (versionamento, compliance, FinOps)
  - Ports:
    - Canal de chat (HTTP/WebSocket/Bot Framework)
    - Serviço de RAG (Azure AI Search)
    - LLM (Azure OpenAI e afins)
    - Storage (Blob / DB)
    - Avaliação (frameworks de IAOps/Eval)

### 🤖 Padrão de Agentes de IA

- **Multi-Agent Orchestration Pattern**:
  - `OrchestratorAgent`  
    Decide intenção, escolhe agente/tool, coordena fluxo.
  - `PolicyRAGAgent`  
    Faz busca semântica + montagem de resposta com citações (RAG).
  - `GuardAgent`  
    Aplica regras de segurança, content safety, anti-jailbreak/prompt injection.

- **Tools Pattern** (LLM chamando ferramentas):
  - `search_policies`, `get_snippet`, `log_interaction`, `get_policy_by_id`, etc.

- **RAG Pattern (Retriever-Augmented Generation)**:
  - Pipelines de ingestão (Fase 2) → Azure AI Search (híbrido) → contexto → resposta.

> Os detalhes completos de arquitetura e padrões estão aprofundados principalmente em:
> - [`./bloco-a/fase1.md`](./bloco-a/fase1.md) – Arquitetura & Design  
> - [`./bloco-a/fase3.md`](./bloco-a/fase3.md) – Backend de Agentes (IA + API)

---

## 🧭 Como este “livro” está organizado

O conteúdo técnico está dividido em **3 grandes blocos**, cada um com várias fases.
Cada **fase** é um capítulo em um arquivo `.md` dentro da pasta do bloco.

---

## 🟩 BLOCO A – CONSTRUIR & PREPARAR  
> **Foco principal:** ambientes **não produtivos** (dev, test, staging)  
> **Objetivo:** definir, projetar, implementar e deixar pronto para subir.

📂 Pasta: [`./bloco-a/`](./bloco-a/)

1. ### [Fase 0 – Descoberta & Requisitos](./bloco-a/fase0.md)
   - Mapeamento de stakeholders (RH, Jurídico, Segurança, TI, Compliance).
   - Inventário de fontes de políticas (SharePoint, OneDrive, etc.).
   - Escopo do MVP (ex.: primeiro RH, depois Segurança da Informação).
   - Requisitos funcionais e não funcionais (latência, disponibilidade).
   - Métricas de sucesso (negócio + técnicas).
   - QA na Fase 0 (cobertura de requisitos, owners e critérios de aceitação).

2. ### [Fase 1 – Arquitetura & Design (software + IA)](./bloco-a/fase1.md)
   - System design de alto nível: componentes, fluxos, integrações.
   - Modelo de agentes (Orchestrator, PolicyRAG, Guard).
   - Desenho de prompts (system prompts, contexto, formato de resposta).
   - Caching (tokens, contexto, embeddings) e controle de custos.
   - Design de segurança (authz, masking, PII).
   - QA na Fase 1 (cobertura de requisitos, riscos mapeados).

3. ### [Fase 2 – DataOps: Ingestão & Indexação](./bloco-a/fase2.md)
   - Modelagem de dados de políticas (metadados, seções, chunking).
   - Pipelines de ingestão (Data Factory / Synapse / Airflow / n8n).
   - Extração com Azure AI Document Intelligence.
   - Indexação em Azure AI Search (texto + vetor).
   - Atualizações incrementais, invalidação de cache.
   - QA na Fase 2 (qualidade do OCR, chunks, metadados, latência de indexação).

4. ### [Fase 3 – Backend de Agentes (IA + API)](./bloco-a/fase3.md)
   - Estrutura do backend (FastAPI/Node, pastas, módulos).
   - Implementação de tools (`search_policies`, `log_interaction`, etc.).
   - Implementação de `PolicyRAGAgent` e `GuardAgent`.
   - Implementação de `OrchestratorAgent` e fluxo de decisão.
   - API REST (`/chat`, `/feedback`, `/health`).
   - QA na Fase 3 (testes unitários, integração, contratos de API).

5. ### [Fase 4 – Frontend & Canais](./bloco-a/fase4.md)
   - UI de chat (Web) – layout, UX, exibição de fontes.
   - Integração com Azure Entra ID (SSO).
   - Integração com a API `/chat`.
   - Integração opcional com Microsoft Teams (Bot Service).
   - QA na Fase 4 (E2E, acessibilidade, segurança no frontend).

6. ### [Fase 5 – DevOps + IAOps + DataOps (CI/CD & Eval)](./bloco-a/fase5.md)
   - Pipelines de CI (build, lint, testes unitários, API schema).
   - IAOps – datasets de avaliação, LLM-as-a-judge, gates de qualidade.
   - DataOps – monitorar pipelines de ingestão.
   - CD – dev → staging → prod (com approvals).
   - QA na Fase 5 (pipelines quebram quando qualidade cai).

7. ### [Fase 8 – Estratégia de Versionamento](./bloco-a/fase8.md)
   - Versionamento de código (Git, tags, releases).
   - Versionamento de prompts (Prompt Registry, versões semânticas).
   - Versionamento de dados de RAG (índices, embeddings, versões de políticas).
   - Versionamento de modelos (model_name + model_version).
   - Versionamento de infraestrutura (IaC).
   - Como as atualizações são usadas e como fazer rollback “cirúrgico”.

---

## 🟨 BLOCO B – LIBERAR COM SEGURANÇA  
> **Foco:** transição para produção (piloto, rollout, operação inicial, compliance).  
> **Ambiente:** staging + produção controlada.

📂 Pasta: [`./bloco-b/`](./bloco-b/)

8. ### [Fase 6 – Segurança & Governança Técnica](./bloco-b/fase6.md)
   - Guardrails de entrada e saída (filtros, classificadores).
   - Proteção de dados sensíveis (classificação, masking, filtros no RAG).
   - Logs e auditoria de acessos.
   - Testes de prompt injection / jailbreak / pentest em API & Web.
   - QA na Fase 6 (métricas de segurança, cobertura de cenários críticos).

9. ### [Fase 7 – Piloto, Rollout & Operação Contínua (Inicial)](./bloco-b/fase7.md)
   - Piloto com grupo controlado (30–100 usuários).
   - Coleta de feedback e ajustes de prompt/RAG.
   - Rollout geral, comunicação interna, treinamento dos usuários.
   - Operação contínua (run) inicial.
   - QA na Fase 7 (satisfação, redução de tickets, latência e custo real).

10. ### [Fase 10 – Operação Contínua, SRE, Playbooks & Runbooks](./bloco-b/fase10.md)
    - Definição de SLIs/SLOs/SLAs (latência, disponibilidade, erro).
    - Criação de playbooks de incidentes e runbooks de rotina.
    - Observabilidade avançada (logs, metrics, traces).
    - Gestão de incidentes, post-mortems, gestão de capacidade.
    - Integração com FinOps, segurança, compliance.

11. ### [Fase 12 – EU AI Act & LGPD Compliance](./bloco-b/fase12.md)
    - Escopo regulatório e classificação de risco (EU AI Act).
    - Mapeamento de dados e bases legais (LGPD).
    - DPIA / Relatório de Impacto.
    - Políticas internas de IA, privacidade, logs, retenção.
    - Controles técnicos: transparência, logging, supervisão humana.
    - Integração com DevOps/IAOps/DataOps/SRE (gates de compliance).

---

## 🟦 BLOCO C – OPERAR & EVOLUIR (PLATAFORMA VIVA)  
> **Foco:** ambientes **produtivos**, com otimização, resiliência, experimentação e governança.  
> **Ambiente:** produção + “sombra” (shadow) + ambientes de teste dedicados.

📂 Pasta: [`./bloco-c/`](./bloco-c/)

12. ### [Fase 9 – FinOps, Cost Engineering & Token Optimization](./bloco-c/fase9.md)
    - Métricas de custo (por modelo, por agente, por usuário).
    - Estratégias de otimização: caching, prompts, modelos menores, truncamento.
    - Alertas de custo, limites orçamentários, revisões mensais.
    - Relatórios de FinOps para diretoria.

13. ### [Fase 11 – Resiliência & Chaos Engineering](./bloco-c/fase11.md)
    - Cenários de falha: indisponibilidade de LLM, RAG, Storage, auth.
    - Testes destrutivos em ambientes controlados (e eventualmente em prod).
    - Automação de recuperação, fallback modes, degraded modes.
    - Métricas de resiliência (MTTR, MTBF, sucesso de recovery).

14. ### [Fase 13 – Experimentação Contínua (Model Loop)](./bloco-c/fase13.md)
    - Experimentation Engine (registry, runner, validator).
    - Tipos de experimento: modelo, prompt, RAG, guardrails, ingestão.
    - Datasets de avaliação (reais, sintéticos, ataques, regulatórios).
    - Batch eval, shadow mode, A/B testing, canary release.
    - Métricas de experimentação (qualidade, custo, segurança, RAG).
    - Integração com CI/CD (gates de experimento).

15. ### [Fase 14 – Governança de IA & Ciclo de Vida](./bloco-c/fase14.md)
    - Princípios de governança de IA (propósito, accountability, transparência).
    - Operating model: comitê de IA, papéis (PO, Tech Lead, DPO, etc.), RACI.
    - Ciclo de vida completo (ideação → deploy → operação → descontinuação).
    - Políticas e normas específicas de IA (uso, desenvolvimento, segurança, dados).
    - Gestão de riscos (risk register de IA).
    - Modelo de maturidade de IA.
    - KPIs & OKRs de governança de IA.

---

## 🧩 Referência de Arquitetura & Código

Os capítulos (fases) descrevem **conceitos, arquitetura e processos**.  
O código em si deve seguir uma estrutura semelhante a:

- `backend/` – API + Agentes (Orchestrator, PolicyRAG, Guard, tools)  
- `frontend/` – UI Web + integração com Teams  
- `data-pipelines/` – ingestão, extração, indexação  
- `infra/` – Terraform/Bicep (Azure infra)  
- `eval/` – scripts e datasets de IAOps  
- `prompts/` – templates versionados de prompts  
- `governance/` – documentação de políticas, riscos, RACI, etc.

> ⚠️ O livro descreve a **arquitetura e os patterns de código** de forma clara.
> A implementação concreta (código-fonte) pode ser adicionada depois, seguindo
> as recomendações de estrutura presentes nos capítulos.

---

## 🚀 Como usar este livro

- Se você é **Arquiteto/Tech Lead**: comece pelo **Bloco A** (Fases 0–3) e depois
  pule para Fases 5, 8, 13 e 14.
- Se você é **Dev/Engenheiro de Dados/ML**: foque em Fases 2, 3, 5, 9, 11 e 13.
- Se você é de **RH/Jurídico/Compliance**: veja Fases 0, 6, 7, 10, 12 e 14.
- Se você é **gestor/diretoria**: leia a introdução de cada fase + Fases 7, 9, 12 e 14.

---

## 🔒 Direitos & Permissões

Este repositório contém **conteúdo privado e proprietário**, desenvolvido
exclusivamente para uso interno do autor e/ou de sua organização.

**⚠️ Atenção: Não é concedida nenhuma licença.**

Isso significa que:

- ❌ Não é permitido copiar, redistribuir ou publicar qualquer parte deste conteúdo  
- ❌ Não é permitido usar este material em projetos externos  
- ❌ Não é permitido modificar e redistribuir  
- ❌ Não é permitido utilizá-lo em treinamentos, cursos, livros ou produtos  
- ✔️ Apenas o autor pode utilizá-lo internamente ou compartilhá-lo manualmente

Qualquer uso externo **exige autorização por escrito** do autor.