# FluxDesk — Especificação técnica e guia de implementação

> **Versão:** 1.0 (proposta técnica inicial)  
> **Data:** 17/10/2026  
> **Estado:** arquitetura proposta; decisões assinaladas como *pendentes* exigem validação  
> **Público:** desenvolvedores, mantenedores e agentes de IA que implementam o FluxDesk  
> **Documento funcional de referência:** `FluxDesk_Definicao_Detalhada.md`, versão 1.1  
> **Natureza:** contrato técnico de referência, não código-fonte pronto e não substitui decisões futuras de produto.

## 0. Como usar este documento

Este guia traduz as regras funcionais do FluxDesk em uma proposta executável de **stack, arquitetura, limites de módulos, dados, API, tempo real, autenticação, UX, testes e entregas**. O propósito é permitir que um agente de IA desenvolva o produto de maneira incremental e consistente.

**Hierarquia para decidir conflitos:** (1) requisitos funcionais aprovados; (2) regras de negócio e segurança deste guia; (3) contratos de API e dados deste guia; (4) convenções locais do repositório; (5) escolhas de implementação documentadas. Se dois requisitos aprovados entrarem em conflito, **não fazer uma mudança silenciosa**: registrar a dúvida e sugerir uma decisão.

Legendas: **[DECIDIDO]** faz parte desta proposta para implementação; **[PROPOSTA]** é um padrão técnico recomendado que ainda pode ser revisto; **[PENDENTE]** exige confirmação humana antes de sua fixação como identidade ou regra definitiva; **[FUTURO]** está fora do MVP.

O MVP deve ser **um monólito modular** (API única com responsabilidades bem separadas), um painel web e um widget incorporável. Não introduzir microsserviços, brokers complexos, IA ou vários canais de atendimento antes de haver necessidade real.

---

## 1. Produto e invariantes de domínio

**FluxDesk** é uma plataforma open source de help desk auto-hospedável. Um visitante inicia uma conversa por meio de um chat incorporado no site de uma organização. Um analista pode resolver a demanda no chat **sem criar ticket** ou abrir **manualmente** um ticket para acompanhar a solicitação. O analista encerra a conversa registrando o resultado; o ciclo de vida do ticket é independente.

### 1.1 Regras que nenhuma implementação pode violar

1. **Conversa (`Conversation`) e ticket (`Ticket`) são entidades independentes.** Nunca criar ticket automaticamente ao receber mensagem.
2. Só um analista principal pode ser atribuído à mesma conversa em determinado instante; a atribuição concorrente deve ser atômica.
3. O encerramento de uma conversa **exige resultado explícito**: `resolved`, `unresolved` ou `follow_up`.
4. `resolved` significa **sucesso declarado pelo analista**, e não validação do cliente nem satisfação/CSAT.
5. Encerrar conversa não resolve ticket; resolver ticket não encerra conversa.
6. A reabertura de conversa remove o resultado *corrente* enquanto ela estiver aberta, preservando o histórico imutável dos encerramentos anteriores.
7. Notas internas e eventos internos nunca são entregues pelo widget público.
8. Todo dado de negócio deve ser isolado por organização; verificar autorização no servidor, **não apenas na interface**.
9. O visitante só lê e escreve mensagens das conversas autorizadas por **sua sessão de visitante**; nome ou e-mail informados não provam identidade.
10. Mensagens e eventos precisam ser persistidos antes de notificar clientes conectados.
11. O projeto deve ser modular e documentado: regras de negócio não ficam nas rotas, nem nos componentes React, nem em modelos ORM.

### 1.2 Fluxo canônico

```mermaid
flowchart TD
    A[Visitante abre widget] --> B[Inicia conversa]
    B --> C[Caixa de entrada compartilhada]
    C --> D[Analista assume ou recebe atribuição]
    D --> E{Exige acompanhamento?}
    E -- Não --> F[Atender pelo chat]
    E -- Sim --> G[Criar ticket manualmente]
    G --> H[Ticket no Kanban]
    F --> I[Encerrar conversa e escolher resultado]
    H --> I
    H --> J[Resolver ou reabrir ticket em fluxo independente]
    I --> K[Métricas de conversas]
    J --> L[Métricas de tickets]
```

O caminho pelo Kanban pode continuar **depois** de o atendimento ter sido encerrado como `follow_up`.

### 1.3 Escopo

| MVP | Fora do MVP / evolução |
|---|---|
| Workspaces, usuários, papéis e contatos | SSO corporativo, autenticação social |
| Widget web com conversa e mensagens | WhatsApp, e-mail e redes sociais |
| Caixa de entrada, atribuição, transferência | Roteamento automático sofisticado |
| Tickets, Kanban, prioridade, prazo | SLA avançado e automações |
| Notas internas, histórico, notificações básicas | Arquivos/anexos e base de conhecimento |
| Encerramento com resultado e métricas essenciais | CSAT, IA, análise semântica e relatórios avançados |
| API documentada, testes, instalação local | SDK público completo e marketplace |

---

## 2. Stack e responsabilidades

### 2.1 Backend

| Componente | Escolha | Responsabilidade |
|---|---|---|
| Linguagem | **Python 3.12+** [PROPOSTA] | Sintaxe tipada e recursos modernos; travar a versão exata no repositório |
| Web/API | **FastAPI** | REST, dependências, validações de entrada, OpenAPI e WebSockets |
| Schemas | **Pydantic v2** | Contratos de entrada/saída e validação |
| Configuração | **pydantic-settings** | Variáveis de ambiente tipadas |
| ORM | **SQLAlchemy 2.x**, API assíncrona | Persistência relacional e mapeamentos |
| Migrações | **Alembic** | Evolução versionada do banco |
| Banco de dados | **PostgreSQL** | Dados transacionais, índices, restrições e concorrência |
| Driver | **asyncpg** [PROPOSTA] | Acesso assíncrono ao PostgreSQL pelo SQLAlchemy |
| Servidor ASGI | **Uvicorn** | Execução HTTP e WebSocket |
| Hash de senhas | **pwdlib[argon2]** [PROPOSTA] | Hash seguro e verificação de credenciais |
| HTTP de integração/testes | **httpx** | Cliente assíncrono e testes de API |
| Testes | **pytest**, `pytest-asyncio`, `pytest-cov` | Testes de unidade, integração e cobertura |
| Qualidade | **Ruff**, `mypy` ou Pyright [PROPOSTA] | Lint, formato e verificação estática |
| Ambiente/gerência | **uv** + `pyproject.toml` [PROPOSTA] | Dependências reproduzíveis e tarefas |
| Observabilidade | `logging` estruturado; OpenTelemetry [FUTURO] | Diagnóstico sem vazar informações privadas |
| Cache/eventos entre instâncias | **Redis** [ADICIONAR QUANDO NECESSÁRIO] | Pub/sub para múltiplos processos e eventual cache |

**Orientações:** usar `Mapped[...]` e `mapped_column()` do SQLAlchemy 2.x; `async_sessionmaker` com **uma `AsyncSession` por unidade de trabalho**; nunca compartilhar a mesma sessão entre tarefas concorrentes. Evitar IO implícito durante a serialização de respostas. A camada de serviços define a transação; o repositório **não efetua `commit()`**.

### 2.2 Frontend / widget

| Componente | Escolha | Responsabilidade |
|---|---|---|
| Interface | **React + TypeScript** | Painel web com tipagem estática |
| Build | **Vite** | Desenvolvimento e bundle de produção |
| CSS | **Tailwind CSS v4** | Tokens e utilitários de estilo |
| Componentes | **shadcn/ui** (Radix primitives) [PROPOSTA] | Componentes acessíveis e personalizáveis |
| Ícones | **lucide-react** | Ícones consistentes |
| Navegação | **React Router** | Rotas protegidas, layout e navegação |
| Dados remotos | **TanStack Query** | Consultas, cache, invalidação e mutações |
| Formulários | **React Hook Form + Zod** | Formulários e validação no cliente |
| Datas | **date-fns** [PROPOSTA] | Formatação e intervalos |
| Kanban | **@dnd-kit** [PROPOSTA] | Arrastar cartões com fallback acessível por menu/botão |
| Testes | **Vitest + Testing Library + Playwright** | Unidade, componentes e E2E |
| Widget | **React + Vite, bundle independente** | Chat incorporável isolado do CSS da aplicação hospedeira |

**Decisão de frontend:** TypeScript será usado mesmo que o requisito funcional mencione apenas React. Não misturar lógica de HTTP em componentes visuais; TanStack Query representa *server state*, e estado local/contexto fica restrito a UI e sessão visual. Uma biblioteca global de estado, como Zustand, só será introduzida se houver caso concreto não atendido por React e Query.

### 2.3 Integração e execução

- [DECIDIDO] REST JSON para comandos, listagens e recuperação histórica.
- [DECIDIDO] WebSocket para **notificações e eventos de atualização**. Para o MVP, o envio de mensagens é **HTTP POST**, e a confirmação/distribuição é **evento WebSocket**, facilitando idempotência e tratamento de erros.
- [PROPOSTA] Docker Compose para execução local da API, frontend e PostgreSQL; serviços podem rodar também diretamente sem Docker.
- [PROPOSTA] GitHub Actions para lint, testes, build e validação de migrações.
- [PROPOSTA] OpenAPI gerado automaticamente e cliente TypeScript gerado a partir do schema **quando o contrato estabilizar** (evitar tipos duplicados manualmente).

---

## 3. Arquitetura geral

### 3.1 Visão de componentes

```mermaid
flowchart LR
    W[Widget no site do cliente] -->|REST + WS| API[FastAPI — monólito modular]
    P[Painel React] -->|REST + WS| API
    API --> PG[(PostgreSQL)]
    API -. eventos entre instâncias .-> R[(Redis — futuro)]
```

O painel é uma aplicação autenticada para analistas; o widget possui uma identidade de **visitante**, de menor privilégio e escopo limitado. As duas interfaces não compartilham autorização.

### 3.2 Backend: arquitetura em camadas (*Layered Architecture*)

```text
HTTP/WebSocket
     │
     ▼
Router / Controller (FastAPI)        ← autenticação, schemas, status HTTP
     │
     ▼
Service / Application               ← casos de uso e regras de negócio
     │
     ▼
Repository / Persistence            ← consultas e escrita SQLAlchemy
     │
     ▼
ORM Model / PostgreSQL              ← dados, constraints e migrações
```

**Camada de apresentação (`router.py`):** declara endpoint, `Depends`, tipos Pydantic, status e código de erro; nunca concentra regras do ticket/conversa.  
**Camada de serviço (`service.py`):** recebe contexto autorizado e dependências, aplica invariantes, controla unidade de trabalho, chama repositórios e agenda publicação *após commit*.  
**Camada de repositório (`repository.py`):** consultas tipadas, filtros obrigatórios por workspace e persistência; não decide permissões ou workflow.  
**Camada de domínio/ORM (`models.py`, `enums.py`):** identidade, atributos, relações e invariantes estruturais; sem dependência de HTTP.  
**Schemas (`schemas.py`):** contratos Pydantic separados em `Create`, `Update`, `Read`, `List`, `Command`; não retornar ORM bruto.  
**Infraestrutura compartilhada (`core/`, `db/`):** config, autenticação, contexto, exceções, sessões, broadcast, logs e tipos comuns.

Usar classes apenas onde agregam encapsulamento ou injeção útil. Repositório base e serviço base genéricos **não são obrigatórios**: abstrações devem eliminar duplicações reais, sem transformar o sistema em uma arquitetura excessivamente indireta.

### 3.3 Estrutura recomendada do repositório

```text
fluxdesk/
├── backend/
│   ├── pyproject.toml
│   ├── uv.lock
│   ├── alembic.ini
│   ├── migrations/
│   ├── src/fluxdesk/
│   │   ├── main.py                 # criação da aplicação e ciclo de vida
│   │   ├── api/router.py           # composição de routers versionados
│   │   ├── core/
│   │   │   ├── config.py           # pydantic-settings
│   │   │   ├── security.py         # identidade, senhas e tokens/sessões
│   │   │   ├── permissions.py      # RBAC e resolução de workspace
│   │   │   ├── exceptions.py       # exceções do domínio e mapeamento HTTP
│   │   │   ├── pagination.py       # cursor e respostas padronizadas
│   │   │   └── events.py           # publicação após commit
│   │   ├── db/
│   │   │   ├── base.py             # DeclarativeBase, mixins, metadata
│   │   │   ├── session.py          # async engine e sessionmaker
│   │   │   └── model_registry.py   # registro central para Alembic
│   │   ├── modules/
│   │   │   ├── auth/
│   │   │   ├── organizations/
│   │   │   ├── users/
│   │   │   ├── contacts/
│   │   │   ├── widget/
│   │   │   ├── conversations/
│   │   │   ├── messages/
│   │   │   ├── tickets/
│   │   │   ├── notes/
│   │   │   ├── notifications/
│   │   │   └── metrics/
│   │   │       # cada módulo: router.py, service.py,
│   │   │       # repository.py, models.py, schemas.py
│   │   │       # criar apenas arquivos que o módulo realmente usa
│   │   └── realtime/
│   │       ├── router.py
│   │       ├── manager.py
│   │       └── schemas.py
│   └── tests/
│       ├── unit/
│       ├── integration/
│       ├── api/
│       └── conftest.py
├── frontend/
│   ├── package.json
│   ├── src/
│   │   ├── app/                   # providers, layout, router e bootstrap
│   │   ├── pages/                 # composição das páginas
│   │   ├── features/              # módulos por funcionalidade
│   │   │   ├── auth/
│   │   │   ├── inbox/
│   │   │   ├── conversations/
│   │   │   ├── tickets/
│   │   │   ├── members/
│   │   │   ├── widget-settings/
│   │   │   └── metrics/
│   │   ├── components/ui/         # shadcn/ui, sem regras do domínio
│   │   ├── components/layout/     # shell e navegação
│   │   ├── shared/
│   │   │   ├── api/                # cliente HTTP + tratamento de falhas
│   │   │   ├── realtime/           # conexão WS e reconciliação
│   │   │   ├── lib/                # utilitários puros
│   │   │   ├── hooks/
│   │   │   └── types/              # tipos globais mínimos
│   │   ├── styles/
│   │   └── main.tsx
│   └── tests/
├── widget/
│   ├── package.json
│   └── src/
│       ├── launcher/              # script pequeno incorporável
│       ├── app/                   # iframe e UI do chat
│       ├── features/chat/
│       └── shared/api/
├── infra/
│   └── compose.yaml
├── docs/
│   ├── adr/                      # decisões arquiteturais
│   ├── architecture/
│   ├── api/
│   └── operations/
├── .github/workflows/
├── .env.example
├── CONTRIBUTING.md
├── LICENSE
└── README.md
```

Dentro de um módulo do backend, preferir dependência `router → service → repository`. Módulos se comunicam por **serviços públicos de outros módulos**, não acessando arbitrariamente repositórios internos. Dependências circulares devem ser resolvidas por extração de lógica compartilhada ou ajuste dos limites, não por imports dinâmicos.

### 3.4 Exemplo de responsabilidade de cada camada

Caso de uso: **assumir uma conversa**.

1. `POST .../conversations/{id}/claim` resolve a sessão do analista e o `workspace_id` válido.
2. `ConversationService.claim(...)` verifica papel/permissões e estado elegível.
3. `ConversationRepository.claim_if_unassigned(...)` executa **UPDATE condicional atômico** (`assigned_agent_id IS NULL` e `status != closed`), sempre filtrando `workspace_id`.
4. Se 0 linhas forem atualizadas, serviço devolve conflito (`409`) quando houver disputa de atribuição, ou `404/409` conforme o estado.
5. O serviço registra evento de auditoria na mesma transação, executa commit e só então publica `conversation.assigned`.
6. O frontend confirma a alteração e sincroniza listas/cache. Em `409`, informa que outro analista já assumiu.

**Pseudocódigo ilustrativo — não copiar como implementação integral:**

```python
class ConversationService:
    def __init__(self, repository, audit_repository, unit_of_work, publisher):
        self.repository = repository
        self.audit_repository = audit_repository
        self.uow = unit_of_work
        self.publisher = publisher

    async def claim(self, *, workspace_id, conversation_id, actor):
        await self.uow.begin()
        updated = await self.repository.claim_if_unassigned(
            workspace_id=workspace_id,
            conversation_id=conversation_id,
            agent_id=actor.id,
        )
        if not updated:
            raise ConversationAlreadyAssignedError()
        await self.audit_repository.record_assignment(...)
        await self.uow.commit()
        await self.publisher.publish_assignment(...)
        return updated
```

O exemplo define **fronteiras de responsabilidade**, não uma API obrigatória para Unit of Work. Se a publicação falhar após o commit, registrar a falha e permitir reconciliação pelo REST; não sinalizar ao usuário que a transação foi revertida quando os dados já foram persistidos. Eventos críticos que não podem se perder devem usar *transactional outbox* quando confiabilidade entre instâncias passar a ser necessária; não presumir que publicação depois de commit é entrega garantida.

---

## 4. Modelo de dados relacional

### 4.1 Convenções para todas as entidades

- PK `id UUID`; timestamps `created_at` / `updated_at` como `TIMESTAMP WITH TIME ZONE`, sempre UTC no armazenamento.
- Nomear tabelas com `snake_case` e plural consistente; enums do domínio com valores estáveis em inglês.
- Entidades pertencentes a workspace incluem `organization_id UUID NOT NULL` e índices adequados.
- Todas as FKs, índices e `UNIQUE` críticos criados por **migração Alembic**, nunca pelo `create_all()` de produção.
- Exclusão em cascata de dados operacionais **não deve ser presumida**. Definir política de desativação/retenção e processos de privacidade antes de permitir exclusão de organizações.
- Para relacionamentos com recursos de outras organizações, **não basta** uma FK simples para ID: validar pertença no serviço e, onde praticável, reforçar com FKs compostas `organization_id + resource_id`/restrições únicas correspondentes.

### 4.2 Entidades e campos mínimos propostos

| Entidade | Campos relevantes | Restrições / observações |
|---|---|---|
| `organizations` | `id`, `name`, `slug`, `created_at` | `slug` único; isolamento lógico por workspace |
| `users` | `id`, `name`, `email`, `password_hash`, `is_active` | `email` único por identidade de login, normalizado |
| `auth_sessions` | `id`, `user_id`, `token_hash`, `expires_at`, `revoked_at?`, `last_seen_at?` | Sessão administrativa revogável; armazenar somente hash do token opaco |
| `memberships` | `id`, `organization_id`, `user_id`, `role`, `created_at` | `UNIQUE(organization_id, user_id)`; role `admin`/`agent` |
| `contacts` | `id`, `organization_id`, `name?`, `email?`, `created_at` | E-mail digitado sem verificação não autentica o contato |
| `widget_configs` | `id`, `organization_id`, `public_key`, `allowed_origins`, `enabled` | `public_key` identificador público, **não é segredo**; origens validadas |
| `visitor_sessions` | `id`, `organization_id`, `widget_config_id`, `contact_id`, `token_hash`, `expires_at`, `revoked_at?` | Token opaco aleatório, guardar apenas hash; acesso limitado à sessão |
| `conversations` | `id`, `organization_id`, `contact_id`, `visitor_session_id`, `assigned_agent_id?`, `status`, `outcome?`, `closed_at?`, `outcome_recorded_by?`, `outcome_recorded_at?`, `last_message_sequence` | Status `waiting/active/closed`; resultado corrente só se `closed` |
| `messages` | `id`, `organization_id`, `conversation_id`, `sender_type`, `sender_key`, `body`, `sequence`, `client_message_id`, `created_at` | `sender_key` derivado pelo backend (`agent:<id>`/`visitor:<session_id>`); `UNIQUE(conversation_id, sequence)` e `UNIQUE(conversation_id, sender_key, client_message_id)` |
| `conversation_outcome_events` | `id`, `organization_id`, `conversation_id`, `actor_id`, `action`, `old_outcome?`, `new_outcome?`, `created_at` | Registro append-only de encerramento, correção e reabertura |
| `tickets` | `id`, `organization_id`, `number`, `source_conversation_id?`, `contact_id?`, `title`, `description`, `status`, `priority`, `category?`, `assignee_id?`, `due_at?`, `resolved_at?` | `UNIQUE(organization_id, number)`; origem opcional |
| `ticket_events` | `id`, `organization_id`, `ticket_id`, `actor_id`, `action`, `changes_json`, `created_at` | Mudanças importantes auditáveis; não confiar no cliente para preencher `actor_id` |
| `internal_notes` | `id`, `organization_id`, `ticket_id`, `author_id`, `body`, `created_at` | Sem endpoint de leitura pelo visitante |
| `notifications` | `id`, `organization_id`, `recipient_id`, `type`, `resource_type`, `resource_id`, `read_at?`, `created_at` | Índice por destinatário e não lidos |

**Campos padrão adicionais:** `updated_at` em entidades mutáveis, e relações/FKs associadas. `sender_key` é um identificador interno derivado do ator autenticado, não enviado pelo cliente; evita que valores `NULL` invalidem a restrição de deduplicação. `auth_sessions` é infraestrutura de autenticação, não um recurso de domínio por organização: membership continua sendo verificada para cada workspace. Categorias e tags **configuráveis** e tabelas de junção são evolução posterior, mas a primeira versão pode suportar categorias fixas validadas pelo backend.

**Modelo de vínculo no MVP:** `tickets.source_conversation_id` opcional. A regra operacional do MVP será **no máximo um ticket de origem por conversa**, aplicada com índice único parcial no PostgreSQL para valores não nulos: `UNIQUE(organization_id, source_conversation_id) WHERE source_conversation_id IS NOT NULL`. Este é um limite deliberado do MVP; uma migração futura poderá liberar 1:N. Tickets criados diretamente não têm conversa de origem.

### 4.3 Status e regras de transição

**Conversa:**

| Origem | Destino | Condição |
|---|---|---|
| — | `waiting` | Visitante inicia conversa, sem responsável |
| `waiting` | `active` | Analista assume/é atribuído |
| `active` | `closed` | Analista autorizado fornece resultado obrigatório |
| `waiting` | `closed` | Encerramento administrativo autorizado, com resultado explícito |
| `closed` | `active` | Reabertura por agente autorizado; apagar apenas resultado corrente, não o histórico |

**Resultado da conversa (`outcome`):** `resolved` (resolvido no atendimento), `unresolved` (encerrado sem resolução), `follow_up` (encaminhado para acompanhamento). **Não é status de ticket.** O comando de encerramento nunca aceita `outcome=null`.

**Ticket:** `open` → `in_progress` → `waiting_customer` ou `resolved`; `waiting_customer` pode retornar a `in_progress`; `resolved` pode reabrir em `open` ou `in_progress`; `cancelled` é um encerramento administrativo explícito. Toda resolução exige `resolution_note` não vazia. Não usar a posição no Kanban como única fonte de verdade: **estado de domínio é persistido no banco**.

**Prioridade:** `low`, `medium`, `high`, `urgent`. `due_at` é opcional no MVP. Data vencida não altera automaticamente a prioridade.

### 4.4 Concorrência e integridade

- Para `claim`, usar `UPDATE ... WHERE assigned_agent_id IS NULL ... RETURNING` ou bloqueio de linha adequado; nunca fazer apenas *read-then-write* sem proteção.
- Para incremento `last_message_sequence`, operar dentro de transação com atualização atômica/`SELECT FOR UPDATE`; mensagem e incremento de sequência entram no mesmo commit.
- Para número humano de ticket, utilizar contador por organização com bloqueio ou sequência segura; nunca derivar de `COUNT(*) + 1`.
- Garantir integridade na criação de ticket vinculando `organization_id` consistente à conversa e ao contato.
- Ao fechar/reabrir, gravar atualização do agregado e evento histórico na mesma transação.
- Criar índices iniciais para `(organization_id, status, updated_at)`, `(organization_id, assigned_agent_id, status)`, `(conversation_id, sequence)` e `(organization_id, due_at)` conforme consultas existentes; validar plano de execução antes de adicionar índices indiscriminadamente.

### 4.5 Diagrama simplificado

```mermaid
erDiagram
    ORGANIZATION ||--o{ MEMBERSHIP : has
    USER ||--o{ MEMBERSHIP : belongs
    ORGANIZATION ||--o{ CONTACT : owns
    ORGANIZATION ||--o{ WIDGET_CONFIG : configures
    CONTACT ||--o{ CONVERSATION : starts
    WIDGET_CONFIG ||--o{ VISITOR_SESSION : issues
    VISITOR_SESSION ||--o{ CONVERSATION : authorizes
    CONVERSATION ||--o{ MESSAGE : contains
    CONVERSATION ||--o{ CONVERSATION_OUTCOME_EVENT : records
    CONVERSATION o|--o| TICKET : originates_MVP
    TICKET ||--o{ TICKET_EVENT : has
    TICKET ||--o{ INTERNAL_NOTE : has
    USER ||--o{ NOTIFICATION : receives
```

O diagrama é **conceitual**, não substitui a DDL: os vínculos com usuários responsáveis/atores foram omitidos para legibilidade.

---

## 5. API REST v1 — contratos iniciais

### 5.1 Convenções gerais

- Base: `/api/v1`.
- JSON UTF-8, campos em `snake_case` no contrato REST **[PROPOSTA]**. No TypeScript, mapear ou gerar tipos que preservem esses campos sem renomeações inconsistentes.
- IDs são UUID; datas são strings ISO 8601 UTC, por exemplo `2026-10-17T14:00:00Z`.
- `POST` cria recurso com `201 Created`; operações sem corpo retornam `204 No Content`; consultas `200 OK`.
- Consultas paginadas retornam `{ "data": [...], "page": { "next_cursor": null, "has_more": false } }`; aceitar `limit` (padrão 25, máximo 100) e `cursor` opaco. Ordenação estável, com `(created_at, id)` ou `sequence` conforme recurso.
- Campos editáveis são *allowlisted* por schema; `organization_id`, `actor_id` e outras informações sensíveis derivam do contexto autenticado, **nunca do JSON fornecido pelo browser**.
- Filtragem por workspace é mandatória até para recursos obtidos por ID; IDs válidos de outro workspace devem aparecer como `404` para quem não tem autorização.
- Esquema oficial de erros: **Problem Details**, inspirado na RFC 9457, com `application/problem+json` e `request_id`. FastAPI possui formatos padrão próprios, portanto configurar handlers personalizados para uniformizar (inclusive erros de validação).

**Erro de exemplo:**

```json
{
  "type": "https://docs.fluxdesk.dev/problems/conversation-already-assigned",
  "title": "Conversation already assigned",
  "status": 409,
  "detail": "Esta conversa já está sob responsabilidade de outro analista.",
  "code": "CONVERSATION_ALREADY_ASSIGNED",
  "request_id": "req_01example"
}
```

`type` é identificador documental previsto, **não afirmar que a URL já existe**; até a documentação pública estar disponível, usar URI estável `about:blank` e preservar `code`.

**Erros usuais:** `400` JSON inválido/regra de entrada malformada, `401` não autenticado, `403` não permitido, `404` recurso inexistente/não visível, `409` disputa de estado/duplicidade, `422` validação semântica do schema, `429` limite excedido, `500` erro interno sem detalhes sensíveis.

### 5.2 Sessão administrativa e organização

| Método | Rota | Quem usa / resultado |
|---|---|---|
| `POST` | `/auth/login` | Público; autentica credenciais e configura sessão segura |
| `POST` | `/auth/logout` | Membro; encerra sessão |
| `GET` | `/auth/me` | Membro; identidade + workspaces disponíveis |
| `GET` | `/workspaces` | Membro; workspaces autorizados |
| `POST` | `/workspaces` | Usuário autorizado; cria organização e membership `admin` |
| `GET` | `/workspaces/{w}/members` | Admin; lista equipe |
| `POST` | `/workspaces/{w}/members` | Admin; cria/associa integrante por fluxo seguro |
| `PATCH` | `/workspaces/{w}/members/{membership_id}` | Admin; altera papel ou estado |
| `GET`/`PATCH` | `/workspaces/{w}/widget-config` | Admin; consulta/edita configuração do widget |

**Provisionamento inicial self-hosted:** disponibilizar comando CLI administrativo para criar o primeiro usuário, documentando seu uso. Não expor um endpoint público de `bootstrap` persistentemente acessível. No MVP, a criação/associação de membros deve exigir confirmação de identidade existente ou convite seguro; **não permitir adicionar terceiro por e-mail e supor posse da conta**.

### 5.3 Conversas e mensagens para analistas

| Método | Rota | Efeito |
|---|---|---|
| `GET` | `/workspaces/{w}/conversations` | Lista por `status`, `assignee_id`, `unread`, cursor |
| `GET` | `/workspaces/{w}/conversations/{id}` | Detalhe, contato, responsável e resultado atual |
| `POST` | `/workspaces/{w}/conversations/{id}/claim` | Assume se não houver responsável; conflito `409` se disputa |
| `POST` | `/workspaces/{w}/conversations/{id}/assign` | Transfere responsável; verifica permissão |
| `POST` | `/workspaces/{w}/conversations/{id}/close` | Fecha com resultado e grava evento |
| `POST` | `/workspaces/{w}/conversations/{id}/reopen` | Reabre, zera resultado corrente e registra evento |
| `GET` | `/workspaces/{w}/conversations/{id}/messages` | Histórico paginado, com cursor/`after_sequence` |
| `POST` | `/workspaces/{w}/conversations/{id}/messages` | Envia mensagem do analista e retorna versão persistida |
| `GET` | `/workspaces/{w}/conversations/{id}/events` | Auditoria e desfechos visíveis à equipe |

**Exemplo de fechamento:**

```http
POST /api/v1/workspaces/8f3a.../conversations/c64b.../close
Content-Type: application/json
```

```json
{
  "outcome": "follow_up",
  "note": "Problema encaminhado para análise técnica; ticket continua aberto."
}
```

```json
{
  "id": "c64b0000-0000-4000-8000-000000000001",
  "status": "closed",
  "outcome": "follow_up",
  "closed_at": "2026-10-17T14:00:00Z",
  "outcome_recorded_by": "a74e0000-0000-4000-8000-000000000001"
}
```

**Contrato de fechamento:** `outcome` é obrigatório e enum fechado; `note` opcional com comprimento máximo definido no schema. Se o comando for repetido para uma conversa já encerrada, retornar `409` em vez de registrar um segundo encerramento. Correção posterior usa comando administrativo próprio e evento de auditoria, em vez de sobrescrever silenciosamente.

**Exemplo de mensagem:**

```http
POST /api/v1/workspaces/{w}/conversations/{id}/messages
```

```json
{
  "client_message_id": "32ec8d6b-1899-4df5-a08f-2a46ca6f79b0",
  "body": "Vou verificar o problema e retorno por aqui."
}
```

```json
{
  "id": "57a10000-0000-4000-8000-000000000001",
  "conversation_id": "c64b0000-0000-4000-8000-000000000001",
  "sequence": 18,
  "client_message_id": "32ec8d6b-1899-4df5-a08f-2a46ca6f79b0",
  "sender_type": "agent",
  "body": "Vou verificar o problema e retorno por aqui.",
  "created_at": "2026-10-17T14:01:00Z"
}
```

`client_message_id` implementa deduplicação: repetir a mesma mensagem com o mesmo ID de cliente e mesma identidade retorna a mensagem já criada, sem duplicação; o servidor nunca confia em `sender_type` recebido do cliente. Reutilizar o mesmo ID com conteúdo diferente deve retornar `409`.

### 5.4 Tickets, Kanban, notas e notificações

| Método | Rota | Efeito |
|---|---|---|
| `GET`/`POST` | `/workspaces/{w}/tickets` | Busca/cria ticket (inclusive sem conversa) |
| `GET`/`PATCH` | `/workspaces/{w}/tickets/{id}` | Detalhe e edição de campos permitidos |
| `POST` | `/workspaces/{w}/tickets/{id}/status` | Transição de estado validada |
| `GET` | `/workspaces/{w}/tickets/{id}/events` | Histórico de atividade |
| `GET`/`POST` | `/workspaces/{w}/tickets/{id}/notes` | Notas internas (membros autorizados) |
| `GET` | `/workspaces/{w}/notifications` | Notificações do próprio usuário |
| `POST` | `/workspaces/{w}/notifications/{id}/read` | Marca como lida |
| `GET` | `/workspaces/{w}/metrics/overview` | Contadores essenciais por período |

**Criar ticket a partir de conversa (sem criar automaticamente):**

```json
{
  "source_conversation_id": "c64b0000-0000-4000-8000-000000000001",
  "title": "Erro ao concluir pagamento",
  "description": "O cliente recebeu mensagem de erro na finalização.",
  "priority": "high",
  "category": "bug",
  "assignee_id": null,
  "due_at": null
}
```

Resposta `201` inclui `id`, `number`, `organization_id`, `status: "open"`, `created_at`, demais campos públicos ao analista. O backend deriva ou valida o `contact_id` do atendimento de origem e verifica permissão e vínculo ao workspace. `source_conversation_id` é opcional; não permitir dois tickets com a mesma conversa de origem no MVP (`409`).

**Transição de status:**

```json
{
  "status": "resolved",
  "resolution_note": "Ajustada a integração; comportamento validado com o solicitante."
}
```

Ao resolver, persistir `resolved_at` e evento atômico. Ao reabrir, limpar `resolved_at` corrente e manter eventos anteriores; o cálculo histórico distingue ciclos de resolução conforme a evolução do relatório. Os campos de quem alterou e quando são preenchidos no backend.

### 5.5 API pública do widget

Prefixo: `/api/v1/public/widget`; não expor rotas administrativas nem permitir informar arbitrariamente `organization_id`.

| Método | Rota | Efeito |
|---|---|---|
| `POST` | `/sessions` | Troca `widget_key` público por sessão de visitante de curta duração |
| `POST` | `/conversations` | Cria conversa vinculada à sessão autorizada |
| `GET` | `/conversations` | Lista **somente** as conversas autorizadas para a sessão atual |
| `GET` | `/conversations/{id}/messages` | Histórico limitado da conversa acessível |
| `POST` | `/conversations/{id}/messages` | Envia mensagem de visitante com deduplicação |
| `GET` | `/config/{widget_key}` | Configuração pública mínima (nome, disponibilidade e aparência, quando suportada) |

`POST /sessions` recebe `widget_key` e identificação opcional (`name`, `email`); retorna um **token opaco aleatório** e prazo de expiração. Armazenar apenas hash no servidor e proteger o token contra exposição. Um e-mail não verificado **não permite importar histórico de atendimentos de outras sessões**. Recuperação de identidade e identificação forte do usuário logado no produto integrador ficam para fase própria, com token assinado pelo servidor do cliente e verificação no FluxDesk.

**Nota sobre origin/domínio:** `widget_key` é pública e não concede privilégios de administração; verificar `Origin`/lista de origens autorizadas em browsers, aplicar rate limit e tokens limitados. `Origin` não é autenticação forte para clientes fora do navegador. Cada requisição privada do visitante deve verificar o token de sessão, organização, conversa e validade.

### 5.6 Métricas: definição reprodutível

`GET /api/v1/workspaces/{w}/metrics/overview?from=2026-10-01&to=2026-10-31`

**Convenção [PROPOSTA]:** intervalo por `closed_at` para métricas de atendimento encerrado; `from` inclusivo e `to` inclusivo em dias do timezone selecionado pelo workspace, convertidos para intervalo UTC `[início, próximo_dia)`. Definir timezone padrão configurável (inicialmente `UTC` até ser escolhido pelo workspace) e exibir na UI.

- `closed_conversations`: quantidade atualmente `closed`, com `closed_at` no período e resultado não nulo.
- `resolved_conversations`: subconjunto dessas conversas com `outcome=resolved`.
- `success_rate_pct`: `resolved_conversations / closed_conversations * 100`; `null` se denominador zero (não `0%`).
- `unresolved_conversations`: resultado `unresolved`.
- `follow_up_conversations`: resultado `follow_up`.
- `tickets_open`, `tickets_in_progress`, `tickets_waiting_customer`, `tickets_resolved`: estado atual; o painel deve deixar explícito que são **estoque atual**, não entradas no período.
- `first_response_seconds_avg`: média entre criação da conversa e primeira mensagem humana (excluir mensagens do visitante e automações).

O resultado é **declarado pela equipe**; não rotular como satisfação do cliente. Correções e reaberturas podem mudar métricas calculadas sobre o estado *corrente*; os eventos históricos permanecem para auditoria. Futuramente disponibilizar métricas históricas baseadas em eventos de cada período.

---

## 6. Tempo real: contrato de eventos WebSocket

### 6.1 Estratégia

- Endpoint WebSocket `/api/v1/realtime` no mesmo host lógico da API.
- **Painel:** conexão autenticada por sessão administrativa válida (cookie seguro, quando mesma origem/site) e verificação do cabeçalho `Origin`.
- **Widget:** conexão inicialmente não assinada; exige mensagem de autenticação com token de visitante **como primeiro frame em até 5 segundos**, antes de aceitar subscriptions. O servidor aplica timeout e fecha conexões inválidas com código de política (`1008`). **Não incluir tokens em query strings**.
- Autorizações verificadas **em cada assinatura** de conversa/canal; um widget nunca assina workspace inteiro.
- Mensagens são transmitidas por `POST REST`; WebSocket recebe eventos de atualização. O cliente reconcilia estado pelo REST quando reconectar ou detectar salto de sequência.
- MVP **um processo API** com `ConnectionManager` em memória. Antes de escalar para múltiplos workers/instâncias, implementar broker/pub-sub (ex.: Redis) e reconciliação; a memória de um processo não entrega mensagens ao outro.

### 6.2 Envelope versionado

```json
{
  "v": 1,
  "event_id": "8db1d64f-3197-43db-b444-22ea827fe290",
  "type": "message.created",
  "occurred_at": "2026-10-17T14:01:00Z",
  "resource": { "type": "conversation", "id": "c64b0000-0000-4000-8000-000000000001" },
  "payload": {
    "message_id": "57a10000-0000-4000-8000-000000000001",
    "sequence": 18,
    "sender_type": "agent"
  }
}
```

Evento não substitui o endpoint de mensagens: usar `message_id`/`sequence` para reconsultar ou inserir a mensagem se houver payload autorizado. Não incluir PII desnecessária no broadcast.

**Eventos mínimos:** `conversation.created`, `conversation.assigned`, `conversation.closed`, `conversation.reopened`, `message.created`, `ticket.created`, `ticket.updated`, `ticket.status_changed`, `notification.created`.

**Mensagens cliente → WebSocket:**

```json
{ "type": "auth", "token": "VISITOR_SESSION_TOKEN" }
```

```json
{ "type": "subscribe", "resource": "conversation", "id": "c64b0000-0000-4000-8000-000000000001" }
```

O exemplo `auth` vale para o **widget**; a sessão administrativa é validada na abertura. Para ambos, `subscribe` exige autorização derivada da identidade da conexão. Responder `subscribed` ou `error` com código sem informações sensíveis; limitar o número de subscriptions por conexão.

### 6.3 Entrega e reconexão

- Eventos são **melhor esforço** no MVP, não fila durável.
- O frontend persiste `last_sequence` das mensagens que já confirmou, descarta duplicatas por `message_id` e consulta `GET .../messages?after_sequence=N` após reconectar.
- Ao receber `message.created` com `sequence > last_sequence + 1`, buscar intervalo faltante no REST.
- Para atualizações de tickets e listas, invalidar/refazer consultas TanStack Query após reconexão.
- Aplicar heartbeat/ping e *exponential backoff* com variação aleatória para reconexão; encerrar listeners corretamente ao desmontar componente.
- Separar conexão/subscrição de apresentação React: `shared/realtime` é o único ponto responsável pelo socket no painel.

---

## 7. Identidade, RBAC e segurança

### 7.1 Perfis e matriz básica

| Ação | Admin | Analista | Visitante |
|---|:---:|:---:|:---:|
| Gerenciar membros e configurações | Sim | Não | Não |
| Listar conversas do workspace | Sim | Sim | Não |
| Assumir/responder/transferir conversa | Sim | Sim* | Não |
| Criar e operar tickets | Sim | Sim* | Não |
| Registrar encerramento de atendimento | Sim | Sim* | Não |
| Corrigir desfecho já registrado | Sim | Não (MVP) | Não |
| Ver notas internas | Sim | Sim | Não |
| Acessar somente a própria conversa | — | — | Sim |
| Consultar métricas administrativas | Sim | Sim (leitura) | Não |

`*` Sujeito às regras de atribuição e ao estado do recurso. O servidor é a autoridade final. O `workspace_id` é sempre validado contra membership ativa, não apenas contra um header ou query parameter.

### 7.2 Sessão de analista [PROPOSTA]

- Sessão opaca gerida pelo servidor, com cookie `HttpOnly`, `Secure` em produção e `SameSite=Lax` quando painel e API forem servidos sob mesmo site.
- Endpoint `login` com proteção contra brute force; usar Argon2 para senhas.
- Sessão revogável no logout e na desativação da conta.
- Proteger métodos que alteram dados com estratégia anti-CSRF apropriada ao uso de cookies; **SameSite isoladamente não deve ser a única defesa**. Segregar origens de desenvolvimento e configurar CORS estritamente.
- Preferir servir painel e API por **mesma origem** através de proxy reverso (`/api`), simplificando cookies e WebSocket.
- Nunca armazenar sessão administrativa em `localStorage` se usar cookie HttpOnly.

### 7.3 Sessão de visitante [PROPOSTA]

- Token aleatório de alta entropia, de escopo restrito; hash no banco, validade e revogação explícitas.
- Validação de associação `session → organization → conversation` em toda leitura/escrita.
- O launcher deve incorporar um **iframe isolado** do domínio do FluxDesk; o token fica no contexto desse iframe, e não é transmitido ao JavaScript da página hospedeira. Definir estratégia de persistência compatível com navegadores que particionam armazenamento de terceiros; aceitar perda de histórico ao limpar sessão, em vez de usar e-mail para recuperar dados indevidamente.
- `postMessage` somente para eventos estritamente necessários (abrir, fechar, redimensionar), verificando `event.origin` e `event.source`; nunca expor tokens pelo canal para a página hospedeira.
- Restringir destinos de scripts/iframes via CSP adequada; checar domínios autorizados também no handshake.

### 7.4 Outras salvaguardas

- Rate limit em login, criação de sessão/conversa e envio de mensagens; limites configuráveis por IP **e** identificadores seguros quando aplicáveis.
- Limites para tamanho de mensagens (ex.: 5.000 caracteres, configurável), tamanho de payload, conexões e inscrições WebSocket.
- Renderizar mensagens como texto por padrão; não inserir HTML não sanitizado, prevenindo XSS.
- Segredos em variáveis de ambiente, nunca em repositório ou `VITE_*` (variáveis de build do frontend são públicas).
- Limitar exposição de PII em logs e eventos; incluir `request_id`, tipo de evento e IDs internos somente quando apropriado.
- Implementar exportação/exclusão/retencão de dados conforme política e LGPD antes de ofertar publicamente produção real.
- HTTPS/WSS obrigatório em produção; headers de segurança e reverse proxy bem configurados.
- Fazer testes explícitos de acesso cruzado entre organizações, dados públicos do widget e notas internas.

---

## 8. Frontend modular e experiência do usuário

### 8.1 Padrão de módulos React (*feature-based architecture*)

Cada funcionalidade possui seus próprios componentes, hooks, schemas, serviços de API e testes. Exemplo:

```text
frontend/src/features/conversations/
├── api/
│   ├── conversations.api.ts
│   └── conversations.queries.ts
├── components/
│   ├── ConversationList.tsx
│   ├── ConversationThread.tsx
│   ├── ConversationDetails.tsx
│   └── CloseConversationDialog.tsx
├── hooks/
│   └── useConversationRealtime.ts
├── schemas/
│   └── conversation.schema.ts
├── types/
│   └── conversation.types.ts
└── index.ts                   # exports públicos do módulo
```

**Importações permitidas:** `pages` importa exports públicos de `features`; features podem importar `shared` e `components/ui`; evitar imports profundos entre duas features. Para comunicação entre inbox e tickets, passar IDs e callbacks por composição, ou usar interfaces públicas do módulo. Não recriar componentes de UI de base em cada feature.

**Separação:**

- `*.api.ts`: transporte HTTP e mapeamento de resposta; sem UI.
- `*.queries.ts`: query keys, hooks de busca, mutations e invalidação.
- `*.schema.ts`: Zod de formulários quando necessário; API é validada pelo backend.
- Componentes: composição visual, interações e acessibilidade; sem regra de autorização confiada ao cliente.
- `shared/realtime`: infraestrutura WebSocket e sincronização genérica.
- `app`: roteamento, providers (`QueryClientProvider` etc.), sessão, temas e layouts.

### 8.2 Mapa de telas

| Tela | Objetivo | Componentes essenciais |
|---|---|---|
| Login | Acesso seguro | Formulário, feedback de erro, estado de envio |
| Visão geral | Leitura rápida da operação | Indicadores essenciais e atalhos |
| Caixa de entrada | Atender sem alternar telas | Lista/filtragem, mensagens, contato, assumir/transferir/encerrar |
| Ticket/Kanban | Acompanhamento estruturado | Colunas e cartões, status, prioridade, responsável, filtros |
| Detalhe do ticket | Investigação e colaboração | Descrição, histórico, notas internas, vínculo à conversa |
| Equipe | Gestão de colaboradores | Lista, papéis e estados (admin) |
| Configuração do widget | Incorporação e segurança | Script de instalação, allowlist de domínios, preview |
| Métricas | Desempenho de suporte | Indicadores com denominadores e período explícitos |

**Rotas exemplificativas:** `/app/:workspaceSlug/overview`, `/inbox`, `/inbox/:conversationId`, `/tickets`, `/tickets/:ticketId`, `/team`, `/settings/widget`, `/metrics`.

### 8.3 Layout e interação

**Desktop — inbox:** navegação lateral compacta, lista de conversas com busca/filtros, painel central com histórico e composer, painel lateral retrátil com contato, tickets vinculados e ações. Enfatizar **estado do atendimento, responsável, tempo de espera e próxima ação**.

**Mobile:** navegação progressiva em uma coluna (lista → conversa → detalhes), sem tentar comprimir três painéis. Manter ações críticas alcançáveis e evitar menus sobrepostos no teclado virtual.

**Kanban:** cartões mostram título, número, prioridade, responsável e prazo se houver. Arrastar é opcional na interação: **sempre oferecer ação de alterar status por teclado/menu**. Confirmar transições inválidas com mensagem clara; tratar `409` de concorrência refazendo dados.

**Encerrar atendimento:** abrir diálogo com 3 opções mutuamente exclusivas e descrições curtas. O analista escolhe explicitamente entre **Resolvido com sucesso**, **Encerrado sem resolução** e **Encaminhado para acompanhamento**. Nunca deixar `resolved` pré-selecionado. Exibir aviso quando há ticket pendente, sem inferir que o ticket foi resolvido. Mostrar sucesso apenas após confirmação HTTP.

**Feedback obrigatório de UI:** skeletons para carregamento, estado vazio útil, erros recuperáveis, retry e toast acessível, validação inline, status de conexão em tempo real e indicador de mensagens não enviadas/repetidas.

### 8.4 Sistema visual — cores e tipografia

**Fonte definida:** **Inter**, com fallback `ui-sans-serif, system-ui, sans-serif`. Garantir subset/licença e carregamento adequado em projetos open source.

**Cores fornecidas pelo responsável do produto:** `#e8e8e` e `#00123`. **[PENDENTE]** Ambas têm **5 dígitos hexadecimais**, portanto **não são valores CSS válidos** no formato `#RGB`, `#RGBA`, `#RRGGBB` ou `#RRGGBBAA`. Não colocá-las literalmente em CSS de produção, não afirmar que já foram aprovadas outras variantes.

**Sugestão estritamente provisória**, a confirmar antes do design visual final:

- Superfície clara: `#E8E8E8` (hipótese para `#e8e8e`).
- Cor escura principal: `#001233` (hipótese para `#00123`).

A paleta técnica precisa também de **superfícies, texto, bordas, foco, perigo, alerta e sucesso**, contrastes AA/WCAG e eventuais tokens do modo escuro. Derivar tons sem alterar semanticamente a identidade. **Esses valores não são uma aprovação da paleta.**

CSS de exemplo com **valores provisórios** e tokens centralizados (Tailwind v4, CSS-first):

```css
@import "tailwindcss";

:root {
  --fd-surface: #e8e8e8; /* provisório: confirmar */
  --fd-ink: #001233;     /* provisório: confirmar */
  --fd-font: "Inter", ui-sans-serif, system-ui, sans-serif;
}

@theme inline {
  --font-sans: var(--fd-font);
  --color-brand-surface: var(--fd-surface);
  --color-brand-ink: var(--fd-ink);
}
```

O `shadcn/ui` pode oferecer componentes de base, mas não define a identidade visual. Criar tokens semânticos para `background`, `foreground`, `primary`, `muted`, `border`, `destructive`, `success` e `focus-ring` após validação. Evitar cores arbitrárias repetidas em cada componente.

### 8.5 Qualidade de experiência e acessibilidade

- Contraste e estados de foco visíveis (WCAG 2.2 AA como meta); suporte a teclado para ações do Kanban e chat.
- Botões com rótulos compreensíveis, `aria-live` para novas mensagens/erros, controle de foco ao abrir/fechar diálogos.
- Tempo e data no fuso do workspace/usuário; persistir UTC na API.
- Estados de texto curtos, consistentes e localizados em português brasileiro inicialmente.
- Responsividade a partir de telas estreitas; validar widget em celulares reais e navegadores comuns.
- Performance: paginação de mensagens, lazy loading de módulos/rotas, virtualização **apenas quando necessário**.
- Manter design responsivo mesmo antes das métricas e funcionalidades avançadas estarem prontas.

### 8.6 Widget incorporável: isolamento por iframe

O site cliente adiciona **um único `<script>`** assíncrono, com chave pública de widget. O launcher cria um botão e um iframe do FluxDesk ao abrir; o React do widget executa dentro do iframe, evitando conflito de versões de React/Tailwind e vazamento de CSS. O launcher deve ser minúsculo, sem depender do React da aplicação hospedeira.

```html
<!-- Exemplo de API de incorporação proposta, não endpoint já publicado -->
<script
  async
  src="https://SUPPORT_HOST/widget/loader.js"
  data-fluxdesk-key="WIDGET_PUBLIC_KEY">
</script>
```

`SUPPORT_HOST` é domínio configurável da instalação self-hosted, não um serviço já existente. Controlar a origem permitida e estabelecer CSP/frame-ancestors conforme os domínios declarados na configuração. `postMessage` com checagem de origem. A tela pública deve funcionar sem exigir login administrativo.

---

## 9. Estratégia de testes e qualidade

### 9.1 Backend

**Unitários:** serviços de negócio com repositórios simulados, transições de estado, cálculo de resultados, permissões, exceções do domínio.  
**Integração:** PostgreSQL real de testes, migrações, constraints, concorrência de `claim`, número de ticket e deduplicação de mensagens.  
**API:** autenticação, validação de requests, códigos HTTP, paginação, erros Problem Details, isolamento entre organizações e autorização do visitante.  
**WebSocket:** autenticação, subscrição indevida, broadcast após commit, reconexão e recuperação de mensagens perdidas.

### 9.2 Frontend

**Unidade/componentes:** estado da lista, diálogo de encerramento, formulário de ticket, ações por teclado, feedback de conflito.  
**Integração:** query invalidation, reconexão, transição de tela responsiva e mensagens de erro.  
**E2E:** fluxo visitante → analista → ticket opcional → encerramento/resultado → métricas.

### 9.3 Casos de aceitação obrigatórios

1. Visitante abre widget e inicia conversa **sem gerar ticket**.
2. Dois agentes tentam assumir ao mesmo tempo; um vence, outro recebe conflito sem dupla atribuição.
3. Uma conversa sem ticket é encerrada como `resolved` e altera corretamente as métricas.
4. Uma conversa com ticket pendente é encerrada como `follow_up`; o ticket permanece aberto.
5. Resolver um ticket não encerra nem altera resultado da conversa vinculada.
6. Reabrir conversa remove resultado corrente e conserva registro anterior para auditoria.
7. Reenviar um `POST /messages` com mesmo `client_message_id` não duplica a mensagem.
8. Widget de uma organização não enxerga conversa nem ticket de outra.
9. Nota interna não aparece em endpoints públicos nem nos eventos destinados ao widget.
10. Analista sem permissão não altera membros/configurações.
11. Usuário reconecta WebSocket e recupera via REST mensagens que perdeu.
12. Interface do Kanban permite mudar status sem usar mouse/drag-and-drop.
13. Campos de métricas não rotulam confirmação interna como CSAT.
14. Migrações passam em banco vazio; build, lint, type check e suíte de testes passam.

### 9.4 Critério de pronto para cada entrega (Definition of Done)

- Requisito funcional e regras de negócio mapeados.
- Código tipado, modular, formatado e com nomes explícitos; docstrings para lógica não trivial e métodos públicos importantes.
- Migração para alteração de esquema, com reversibilidade avaliada e índice/constraint necessário.
- Testes adequados às novas regras (inclusive cenários de falha/concorrência/tenant).
- OpenAPI e tipos de frontend atualizados quando contrato mudar.
- Sem segredo hardcoded, vazamento de PII ou autorização apenas na interface.
- UI responsiva e acessível para a mudança correspondente.
- README, ADR ou docs atualizados se uma decisão técnica/requisito relevante mudar.
- Pull request pequeno e revisável; não misturar refatorações grandes com feature pequena sem justificativa.

---

## 10. Padrões de código e colaboração com agentes de IA

### 10.1 Regras mandatórias para agentes

**Antes de implementar:**

1. Ler `FluxDesk_Definicao_Detalhada.md` (produto), este documento (arquitetura), `README.md`, ADRs e módulos relacionados.
2. Resumir o comportamento que será entregue, os endpoints/modelos afetados, as restrições de segurança e os testes a criar.
3. Planejar a menor alteração que cumpre o requisito; não criar funcionalidades futuras por iniciativa própria.
4. Se uma decisão estiver marcada **[PENDENTE]**, não a apresentar como aprovada; propor alternativa reversível ou solicitar decisão quando bloquear a feature.

**Durante a implementação:**

5. Respeitar a direção `router → service → repository → ORM`, com regras de domínio no serviço.
6. Toda query de entidade tenant-owned recebe `organization_id` resolvido de contexto confiável e não vazado do payload.
7. Não chamar `commit()` dentro de repositório; transação pertence ao serviço/unidade de trabalho.
8. Definir schema de entrada e saída; não retornar modelo SQLAlchemy diretamente; validar transições e permissões no backend.
9. Preservar contrato HTTP e WS existente; se precisar alterá-lo, ajustar testes e documentação juntos.
10. Não expor segredos, tokens, e-mails desnecessários nem notas internas em logs, eventos ou frontend público.
11. Preferir componentes pequenos, composição e custom hooks com objetivo claro; não duplicar `fetch()` em componentes.
12. Preservar estados de loading/error/empty, navegação por teclado e layout móvel.
13. Escrever testes que provem regras e falhas; não mascarar erro de teste, desligar verificação ou substituir teste por mock irrelevante.
14. Não introduzir arquitetura, biblioteca ou serviço novo sem motivação e registro em ADR curto.

**Ao finalizar:**

15. Informar arquivos alterados, migrations adicionadas, endpoints/contratos modificados e testes executados.
16. Informar explicitamente o que está pendente ou não pôde ser verificado — sem declarar implementado o que não foi testado.
17. Para cada feature, incluir um exemplo mínimo reproduzível de uso e uma nota de segurança quando aplicável.

### 10.2 Estilo e nomenclatura

**Python:** nomes `snake_case` para funções/atributos; `PascalCase` para classes; tipagem das APIs públicas; docstrings de intenção e invariantes, não comentários óbvios. Usar `Enum`/`StrEnum` para estados. Erros do domínio (`ConversationAlreadyClosedError`) traduzidos para HTTP no limite de apresentação. Não capturar `Exception` indiscriminadamente.

**TypeScript:** componentes `PascalCase`, hooks `use*`, utilitários `camelCase`, arquivos por convenção adotada no projeto; `strict: true`. Evitar `any` sem justificativa. Tipos derivados de contratos em vez de copiar interfaces incompatíveis.

**SQL:** migrations pequenas e legíveis; índices nomeados; FK e constraints explícitas. Nunca editar migration já aplicada em ambientes compartilhados para esconder divergência de histórico.

**Commits/PRs [PROPOSTA]:** Conventional Commits e escopo por módulo (`feat(conversations): ...`, `fix(widget): ...`).

### 10.3 Exemplo de tarefa para um agente

```text
Objetivo: implementar POST /api/v1/workspaces/{w}/conversations/{id}/close.
Fontes: documento funcional v1.1, seções 1, 5, 7, 9 e 10 deste guia.
Obrigatório:
- Autorizar membership no workspace e agente com acesso à conversa.
- Aceitar apenas resolved | unresolved | follow_up.
- Persistir status closed, outcome, closed_at, actor e evento de histórico na mesma transação.
- Retornar 409 se a conversa já estiver closed.
- Não alterar tickets vinculados.
- Publicar conversation.closed após commit.
- Criar testes positivos, validação 422, permissão 403/404,
  workspace cruzado, duplicidade e ticket pendente.
- Atualizar schema OpenAPI e documentação.
Não implementar: CSAT, resolução automática do ticket, automação ou IA.
Entregar: arquivos, testes executados e limitações.
```

---

## 11. Configuração, ambientes e deploy

### 11.1 Variáveis propostas

```dotenv
# Backend; valores ilustrativos e nunca usar em produção como estão
APP_ENV=development
DATABASE_URL=postgresql+asyncpg://USER:PASSWORD@localhost:5432/fluxdesk
SESSION_SECRET=GENERATE_A_STRONG_RANDOM_SECRET
PUBLIC_BASE_URL=http://localhost:8000
APP_ORIGIN=http://localhost:5173
LOG_LEVEL=INFO
WIDGET_SESSION_TTL_SECONDS=86400
MAX_MESSAGE_LENGTH=5000
# REDIS_URL=redis://localhost:6379/0  # somente ao escalar
```

Em produção, segredos devem vir de um gerenciador seguro ou variáveis de runtime; restringir CORS/Origins, HTTPS e logs. Não disponibilizar `SESSION_SECRET` ao frontend.

### 11.2 Infraestrutura do MVP

- Um PostgreSQL com volume persistente; serviço FastAPI (1 worker inicialmente); build do painel React servido como estático por reverse proxy; assets do widget no mesmo host lógico apropriado.
- `health/live` verifica processo; `health/ready` verifica dependências necessárias sem expor segredos.
- Migrações Alembic executadas como etapa controlada de deploy, **não simultaneamente por múltiplos workers**.
- Backup testado do banco; caminho documentado de restauração e atualização; configuração de retenção a definir.
- CI valida lint, tipos, testes, build e migrações em banco descartável.

### 11.3 Fases para escalar

**Primeiro:** otimizar índices e consultas, paginação, tamanho de payload e número de conexões.  
**Depois:** múltiplas instâncias da API com Redis pub/sub para eventos entre processos, escalabilidade de WS, health checks e reconciliação REST.  
**Quando necessário:** transactional outbox + worker para eventos indispensáveis; fila separada para tarefas custosas, anexos ou notificações externas.  
Não adotar Redis/filas desde o início somente porque o chat usa WebSocket.

---

## 12. Roadmap técnico de implementação

| Etapa | Entrega técnica | Critério de aceite |
|---|---|---|
| **0 — Fundação** | Monorepo, FastAPI, React/Vite, Tailwind v4, PostgreSQL, Alembic, CI, lint e testes | API, painel e banco sobem localmente; migração inicial funciona |
| **1 — Auth/tenancy** | Usuários, workspaces, memberships, sessão admin, RBAC | Um workspace não acessa dados de outro; auth testada |
| **2 — Tickets** | ORM, CRUD, status, eventos, notas, Kanban | Criar/mover/resolver/reabrir e auditar, inclusive sem conversa |
| **3 — Conversas REST** | Contatos, config do widget, sessão de visitante, conversas, inbox, atribuição | Analistas assumem sem concorrência indevida; visitante isolado |
| **4 — Chat em tempo real** | Mensagens persistidas, HTTP send, WS broadcast, reconexão, widget iframe | Mensagens chegam, não duplicam e recuperam histórico após desconexão |
| **5 — Fluxo completo** | Vinculação manual ticket↔conversa, diálogo de encerramento, outcome histórico | Conversa encerra sem ticket ou com ticket pendente; regras preservadas |
| **6 — Métricas e lançamento** | Contadores, testes E2E, hardening, documentação, build e deploy | Indicadores corretos, UX responsiva e instalação documentada |

Cada etapa pode ser dividida em issues pequenas: um módulo do backend, seus testes, seus contratos e a tela correspondente. Não começar pela dashboard de métricas antes dos eventos e resultados estarem corretamente persistidos.

---

## 13. Decisões pendentes e ADRs recomendados

**Não atribuir certeza a estes itens:**

| Decisão | Direção recomendada | Status |
|---|---|---|
| Códigos de cor de marca | Confirmar variantes válidas de `#e8e8e` e `#00123` | **PENDENTE** |
| Licença open source | Comparar MIT, Apache-2.0 e AGPLv3 segundo objetivos | **PENDENTE** |
| Cadastro inicial e convites | CLI para admin inicial + convites seguros | Proposta |
| Sessão administrativa | Sessão opaca em cookie HttpOnly, com CSRF | Proposta |
| Sessão do visitante | Token opaco por iframe, sem recuperação por e-mail não verificado | Proposta |
| Política de reabertura pelo visitante | No MVP, apenas agente pode reabrir | Proposta |
| Cardinalidade origem conversa → ticket | No MVP, no máximo 1 ticket originado por conversa | Derivado do documento funcional v1.1 |
| Timezone de workspace | UTC inicial, configuração posteriormente | Proposta |
| Política de retenção/anonimização | Definir antes de produção com dados reais | **PENDENTE** |
| Compatibilidade browser e suporte | Definir matriz mínima e metas de acessibilidade | Proposta |
| Deploy oficial | Self-hosted com Docker Compose como primeira via | Proposta |

**ADRs iniciais sugeridos:** `ADR-001-monolito-modular.md`, `ADR-002-multi-tenancy.md`, `ADR-003-sessoes-e-permissoes.md`, `ADR-004-rest-e-websocket.md`, `ADR-005-widget-em-iframe.md`, `ADR-006-front-modular.md`, `ADR-007-semantica-de-metricas.md`. Cada ADR deve registrar contexto, opções, decisão, consequências e condições de revisão.

---

## 14. Referências técnicas (documentação oficial)

As escolhas de bibliotecas foram conferidas com referências oficiais. Os links não substituem versionamento e testes no repositório:

- FastAPI — WebSockets: <https://fastapi.tiangolo.com/advanced/websockets/>
- FastAPI — OAuth2/JWT e hash Argon2 com pwdlib: <https://fastapi.tiangolo.com/tutorial/security/oauth2-jwt/>
- SQLAlchemy 2.x — AsyncIO / `async_sessionmaker`: <https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html>
- Tailwind CSS — integração Vite: <https://tailwindcss.com/docs/installation/using-vite>
- shadcn/ui — instalação com Vite: <https://ui.shadcn.com/docs/installation/vite>
- TanStack Query — server state: <https://tanstack.com/query/latest/docs/framework/react/overview>
- React Router — instalação declarativa: <https://reactrouter.com/start/declarative/installation>
- RFC 9457 — Problem Details: <https://www.rfc-editor.org/rfc/rfc9457>

---

## 15. Checklist para início do projeto

- [ ] Confirmar os dois hexadecimais da identidade visual.
- [ ] Colocar a definição funcional v1.1 e esta especificação em `docs/`.
- [ ] Criar repositório e esqueleto `backend/`, `frontend/`, `widget/`, `infra/`.
- [ ] Fixar versões efetivas de Python, Node e bibliotecas no lockfile.
- [ ] Inicializar FastAPI, configuração via env, handlers de erro e OpenAPI.
- [ ] Configurar PostgreSQL, SQLAlchemy assíncrono e Alembic.
- [ ] Criar modelos e migração inicial de workspaces/usuários/memberships.
- [ ] Inicializar React + Vite + TypeScript + Tailwind v4 + shadcn/ui.
- [ ] Definir tokens de design, layout geral e padrão de módulos.
- [ ] Criar testes de isolamento de workspace **antes** dos recursos públicos.
- [ ] Implementar as etapas do roadmap, com E2E de ponta a ponta.
- [ ] Definir licença, privacidade, documentação e release antes da publicação.

**Princípio arquitetural:** *manter o FluxDesk simples na infraestrutura, rigoroso nas fronteiras de domínio, seguro por workspace e coerente do banco de dados até a experiência de atendimento.*
