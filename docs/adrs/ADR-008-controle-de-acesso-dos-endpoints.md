# ADR-008 — Replay de DLQ exige role ADMIN; CRUD de webhooks exige apenas autenticação

- **Status:** Aceito
- **Data:** Quinta-feira, reunião técnica de definição (ver `TRANSCRICAO.md`)
- **Decisores:** Larissa (Tech Lead), Sofia (Engenheira de Segurança), Marcos (Product Manager), Diego (Engenheiro Sênior, Plataforma), Bruno (Engenheiro Pleno, Pedidos)
- **Contexto técnico:** Módulo `src/modules/webhooks/` (a criar) do Order Management System — camada de rotas e middlewares de autorização; reaproveita `src/middlewares/auth.middleware.ts`

## Contexto

O módulo de webhooks introduz dois grupos de endpoints com naturezas de risco muito diferentes.

O primeiro grupo é o CRUD de configuração que Marcos listou em `[09:31] Marcos:` — cadastro de webhook por POST, com `url`, lista de status de interesse e o segredo gerado por nós e devolvido na criação — mais `PATCH`, `DELETE` e `GET` de listagem descritos em `[09:33] Bruno:`, e o histórico de entregas `GET /webhooks/:id/deliveries` de `[09:34] Marcos:`.

O segundo é o endpoint operacional que o Diego propôs em `[09:18] Diego:` e repetiu em `[09:35] Diego:`: `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca na outbox um evento já declarado permanentemente falho. Ele não lê estado: ele reinjeta tráfego para fora da nossa infra.

A restrição que forçou a discussão veio do modelo de identidade que já roda. Bruno levantou em `[09:32] Bruno:` que o JWT atual é do usuário operador, não do cliente. Marcos respondeu em `[09:32] Marcos:` que o cliente cadastra pela nossa API direto, autenticado com JWT do nosso sistema, e que existem usuários que representam o cliente. Larissa fechou o desenho em `[09:32] Larissa:`: o endpoint é autenticado normal e o `customer_id` vai no body ou no path — não vem do JWT. Ou seja, o token não carrega vínculo com cliente algum, e o sistema só conhece dois papéis: `src/middlewares/auth.middleware.ts:9` define `role: 'ADMIN' | 'OPERATOR'`. Não há papel intermediário nem noção de posse de recurso.

O dilema, então: com um modelo de papéis binário e sem vínculo usuário↔cliente no token, qual o nível de autorização de cada grupo de endpoints — sabendo que apertar o CRUD exigiria construir uma noção de posse que hoje não existe, dentro de um prazo de três sprints estimado em `[09:46] Larissa:` para a feature inteira?

## Decisão

`POST /admin/webhooks/dead-letter/:id/replay` exige role `ADMIN`. A verificação usa o `requireRole` que já existe em `src/middlewares/auth.middleware.ts:49`, encadeado depois do `authenticate`, exatamente como o precedente de `src/modules/users/user.routes.ts:15` (`requireRole('ADMIN')`). Nada de middleware novo. A negativa devolve `403 FORBIDDEN` pela `ForbiddenError` de `src/shared/errors/http-errors.ts:21`, já formatada pelo middleware de erro central.

O replay registra quem o executou, para auditoria, conforme exigido em `[09:36] Sofia:`.

Os demais endpoints do módulo — criação, edição, remoção, listagem de webhooks e consulta de deliveries — exigem apenas autenticação, sem restrição de papel. O padrão é o de `src/modules/orders/order.routes.ts:14`, onde o router aplica `router.use(authenticate)` e nenhuma rota adiciona `requireRole`. Qualquer usuário autenticado, `ADMIN` ou `OPERATOR`, opera o CRUD, e o `customer_id` alvo continua vindo do body ou do path, não do token (`[09:32] Larissa:`).

Larissa consolidou os dois lados no resumo de fechamento em `[09:48] Larissa:`: "Endpoints CRUD de configuração autenticados normal, endpoint de replay de DLQ exige role ADMIN."

## Alternativas consideradas

### Exigir `ADMIN` em todo o módulo de webhooks — descartada por ora

A alternativa nunca foi proposta por ninguém: ela só aparece por negação, quando Marcos pergunta em `[09:36] Marcos:` — "O resto do CRUD de configuração de webhook pode ser qualquer role autenticada?" —, deixando implícito o cenário oposto, de estender o `ADMIN` também ao CRUD. Sofia respondeu em `[09:37] Sofia:`: "Por enquanto sim. Mais pra frente a gente pode endurecer." Ninguém na reunião verbalizou o motivo além desse "por enquanto sim". O trade-off registrado a seguir é reconstrução dos autores deste ADR a partir de `[09:32]`, não fala da reunião: exigir `ADMIN` no CRUD quebraria o caso de uso central levantado em `[09:32] Marcos:`, o cliente cadastrando o próprio webhook pela nossa API com usuário que o representa — e esse usuário não é `ADMIN`. Endurecer de verdade exigiria vincular `customer_id` ao usuário do JWT, o oposto do que foi decidido em `[09:32] Larissa:`.

### Deixar o replay de DLQ acessível a qualquer papel autenticado — descartada

Larissa deixou a opção em aberto ao perguntar em `[09:35] Larissa:` "Quem é admin? Tem que ser role ADMIN do JWT?". Sofia derrubou em `[09:36] Sofia:` com o argumento de separação de responsabilidade operacional: "Mexer em fila de entrega de notificação não é coisa de operador." Ninguém defendeu a posição contrária; Larissa fechou no turno seguinte, ainda em `[09:36] Larissa:`, decidindo pelo `requireRole` existente.

### Registro honesto sobre o mecanismo

Não houve alternativa de **mecanismo** discutida na reunião. Escopos por token, ACL por `customer_id`, API key por cliente ou qualquer outro esquema de autorização não foram cogitados por ninguém — reaproveitar o `requireRole` que já existe foi fechado de imediato pela Larissa em `[09:36] Larissa:`, sem que ninguém propusesse ou contestasse alternativa. Quem for reabrir este ADR deve saber que essas opções nunca foram avaliadas e descartadas: elas simplesmente não entraram na pauta.

## Consequências

**Positivas**

- Zero código de autorização novo no módulo. `authenticate` e `requireRole` já existem e já têm precedente de uso com `ADMIN` em `src/modules/users/user.routes.ts:15`. Ressalva: a suíte atual (`tests/auth.test.ts`, `tests/orders.test.ts`) não exercita `requireRole` — não há nenhuma asserção de `403` no repositório, então o caminho de negativa entra sem cobertura.
- O erro de autorização já é padronizado: `ForbiddenError` → `403` / código `FORBIDDEN`, tratado pelo middleware de erro central sem nenhuma mudança.
- O caminho do cliente cadastrar o próprio webhook pela API fica destravado sem depender de um modelo de identidade novo, o que mantém a feature dentro das três sprints estimadas em `[09:46] Larissa:`.
- A operação destrutiva/reinjetora fica atrás de um papel restrito, cumprindo a separação pedida por Sofia.

**Negativas**

- **Qualquer usuário autenticado pode criar, alterar ou apagar o webhook de qualquer cliente.** Como o `customer_id` vem do body ou do path e não do JWT (`[09:32] Larissa:`), nada no código impede um `OPERATOR` de apontar o webhook da Atlas Comercial para uma URL que ele controla e passar a receber os eventos de pedido daquele cliente. É vazamento de dados entre clientes viabilizado por desenho, aceito conscientemente nesta fase.
- **Quem cria webhook obtém segredo válido.** O segredo é gerado por nós e devolvido na criação (`[09:31] Marcos:`) — a reunião não definiu se o `PATCH` devolve segredo. Ainda assim, sem restrição de papel qualquer autenticado cria um endpoint apontando para um `customer_id` de terceiro e recebe, na resposta, um segredo válido para ele. O controle de acesso frouxo aqui corrói, na prática, parte da garantia que a assinatura pretende dar.
- **A auditoria do replay nasce genérica.** `src/middlewares/request-logger.middleware.ts:21` já emite `userId: req.user?.id` em cada requisição HTTP via Pino, e isso atende ao "logar quem fez o replay" de `[09:36] Sofia:` no sentido literal. Mas é log de aplicação, não trilha de auditoria: o `path: req.originalUrl` de `src/middlewares/request-logger.middleware.ts:18` até carrega o id da DLQ embutido na URL, porém como texto de rota, sem retenção garantida e sem imutabilidade. Se alguém precisar provar, meses depois, quem reprocessou o quê, provavelmente não vai conseguir — a reunião não definiu nenhuma janela de retenção.
- **O modelo de papéis binário limita o próprio endurecimento futuro.** `src/middlewares/auth.middleware.ts:9` só conhece `'ADMIN' | 'OPERATOR'`. Não existe papel intermediário: apertar o CRUD hoje significa saltar direto de "qualquer autenticado" para "só ADMIN", quebrando o caso de uso do cliente. Qualquer meio-termo exige criar um papel novo ou um modelo de posse — trabalho que não está orçado.

**Neutras / limitações conhecidas**

- O endurecimento do RBAC no CRUD fica como questão em aberto declarada por Sofia em `[09:37] Sofia:` ("mais pra frente a gente pode endurecer"). **Gatilho de reabertura:** o primeiro incidente de webhook apontado para o cliente errado, ou a entrada de um cliente com exigência contratual de isolamento. A reabertura implica revisitar `[09:32] Larissa:` e derivar o `customer_id` do próprio token, em vez de aceitá-lo do body/path.
- A decisão pode ser revista antes do go-live: Sofia reservou em `[09:46] Sofia:` pelo menos dois dias úteis de revisão de segurança, escopo que ela mesma delimitou como "HMAC e geração de secret" — controle de acesso não foi mencionado. **Gatilho:** achado dessa revisão, caso ela seja estendida ao RBAC dos endpoints.
- A troca do log genérico por trilha de auditoria dedicada do replay fica pendente. **Gatilho:** exigência de retenção ou de compliance sobre operações administrativas.
- A rota `/admin/webhooks/dead-letter/:id/replay` introduz o primeiro prefixo `admin` do projeto — hoje `src/routes/index.ts:21` monta apenas os routers de domínio (`/auth`, `/users`, `/customers`, `/products`, `/orders`) sob `/api/v1`. A convenção de montagem do prefixo administrativo é detalhe de implementação e fica para o FDD.

## Rastreabilidade

**Transcrição**

- `[09:18] Diego:` — replay manual via endpoint admin, `POST /admin/webhooks/dead-letter/:id/replay`, recoloca na outbox como pendente
- `[09:31] Marcos:` — CRUD de cadastro; o segredo é gerado por nós e devolvido na criação
- `[09:32] Bruno:` — o JWT atual é do usuário operador, não do cliente
- `[09:32] Marcos:` — cliente cadastra pela nossa API direto, com JWT do nosso sistema; há usuários que representam o cliente
- `[09:32] Larissa:` — endpoint autenticado normal; `customer_id` vai no body ou no path, não vem do JWT
- `[09:33] Bruno:` — PATCH, DELETE e GET de listagem
- `[09:34] Marcos:` — `GET /webhooks/:id/deliveries`
- `[09:35] Larissa:` — "Quem é admin? Tem que ser role ADMIN do JWT?"
- `[09:35] Diego:` — confirma a rota do replay
- `[09:36] Sofia:` — tem que ser ADMIN; mexer em fila de entrega não é coisa de operador; o endpoint tem que logar quem fez o replay, para auditoria
- `[09:36] Larissa:` — decidido: role ADMIN obrigatório no replay, reaproveitando o `requireRole` que já existe
- `[09:36] Marcos:` — pergunta se o resto do CRUD pode ser qualquer role autenticada
- `[09:37] Sofia:` — "Por enquanto sim. Mais pra frente a gente pode endurecer."
- `[09:46] Larissa:` — estimativa de três sprints
- `[09:46] Sofia:` — dois dias úteis reservados para revisão de segurança antes do deploy, com escopo declarado de "HMAC e geração de secret"
- `[09:48] Larissa:` — resumo de fechamento com os dois níveis de acesso

**Código**

- `src/middlewares/auth.middleware.ts:49` — `requireRole(...roles)`, devolve `ForbiddenError('Insufficient permissions')` quando o papel não bate
- `src/middlewares/auth.middleware.ts:9` — `role: 'ADMIN' | 'OPERATOR'`, os únicos papéis existentes
- `src/middlewares/auth.middleware.ts:27` — `authenticate`, valida o Bearer e popula `req.user`
- `src/modules/users/user.routes.ts:15` — precedente real de `requireRole('ADMIN')` encadeado após `authenticate`
- `src/modules/orders/order.routes.ts:14` — precedente de `router.use(authenticate)` sem restrição de papel, molde do CRUD de webhooks
- `src/shared/errors/http-errors.ts:21` — `ForbiddenError` → `403` / `FORBIDDEN`
- `src/middlewares/request-logger.middleware.ts:18` e `:21` — log Pino já inclui `path: req.originalUrl` e `userId: req.user?.id` em toda requisição
- `src/routes/index.ts:21` — `buildApiRouter`, ponto de montagem dos routers sob `/api/v1`
- `src/modules/webhooks/webhook.routes.ts` — **a criar**, onde a política deste ADR é aplicada

**Relacionado**

- [ADR-003](./ADR-003-retry-com-backoff-e-dlq.md) — retry com backoff e DLQ em tabela separada (define o que o replay reprocessa; a mecânica de retry e DLQ não é escopo deste ADR)
- [ADR-004](./ADR-004-hmac-sha256-secret-por-endpoint.md) — HMAC-SHA256 com segredo por endpoint e rotação (define o segredo cujo acesso este ADR deixa desprotegido no CRUD)
- [ADR-006](./ADR-006-reuso-dos-padroes-do-projeto.md) — reuso dos padrões existentes do projeto no módulo de webhooks (contexto geral de reaproveitamento de middlewares)
