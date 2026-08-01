# ADR-006 — Reuso dos padrões existentes do projeto no módulo de webhooks

- **Status:** Aceito
- **Data:** Quinta-feira, reunião técnica de definição (ver `TRANSCRICAO.md`)
- **Decisores:** Larissa (Tech Lead), Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Sofia (Eng. Segurança), Marcos (PM). O bloco de padrões de código foi conduzido por Bruno e fechado por Larissa.
- **Contexto técnico:** API Node.js + TypeScript + Express + Prisma + MySQL (`order-management-api`). Módulo novo `src/modules/webhooks/` (a criar) e processo worker (a criar), convivendo com `src/modules/orders/`, `src/shared/errors/`, `src/middlewares/` e `prisma/schema.prisma`.

## Contexto

O sistema em produção tem uma convenção estabelecida e uniforme. Cada domínio é uma pasta em `src/modules/` com `*.controller.ts`, `*.service.ts`, `*.repository.ts`, `*.routes.ts` e `*.schemas.ts` — é assim em `orders`, `products`, `customers` e `users`. Erros herdam de `AppError` (`src/shared/errors/app-error.ts`) e carregam um `errorCode` em SCREAMING_SNAKE_CASE; `src/shared/errors/http-errors.ts` já traz os precedentes `INVALID_STATUS_TRANSITION` e `INSUFFICIENT_STOCK`. O middleware `errorMiddleware` (`src/middlewares/error.middleware.ts`) serializa qualquer `AppError` em `{ error: { code, message, details? } }` e já trata `ZodError` e erros conhecidos do Prisma. Validação de entrada passa por `validate({ body, query, params })` (`src/middlewares/validate.middleware.ts`). Log é Pino, instanciado uma vez em `src/shared/logger/index.ts`. Toda entidade do `prisma/schema.prisma` tem chave primária `@id @default(uuid()) @db.Char(36)` — a única exceção é `OrderNumberSequence`, tabela de contador com `id Int @id @default(1)` — e toda tabela usa `@@map` em snake_case.

A força que obriga uma decisão é que o módulo de webhooks é o primeiro pedaço do sistema que **não cabe inteiro dentro do ciclo request/response HTTP**. Ele traz um processo Node separado ([ADR-002](./ADR-002-worker-separado-com-polling.md)), tabelas próprias (`webhook_outbox` e `webhook_dead_letter`, a criar), segredo por endpoint ([ADR-004](./ADR-004-hmac-sha256-secret-por-endpoint.md)) e um caminho de erro que acontece longe de qualquer `res.status()`. Nada disso tem precedente na codebase — o `errorMiddleware`, por exemplo, só existe dentro do processo HTTP montado em `src/app.ts`.

Somam-se as restrições de calendário e de time. A Atlas Comercial ameaçou migrar para o concorrente se a feature não sair até o fim do trimestre (`[09:00] Marcos:`), prazo depois fixado em fim de novembro (`[09:45] Marcos:`); Larissa estimou três sprints incluindo a revisão de segurança (`[09:46] Larissa:`), e Sofia pediu que se reservassem pelo menos dois dias úteis para ela revisar HMAC e geração de secret antes do deploy (`[09:46] Sofia:`). Diego já havia registrado que "a gente é um time pequeno" ao rejeitar infra nova (`[09:07] Diego:`).

O dilema: um módulo com essas diferenças estruturais nasce dentro das convenções atuais — herdando junto o que elas não resolvem fora do HTTP — ou ganha estrutura própria porque é diferente dos outros cinco módulos de `src/modules/` — os quatro que seguem o conjunto completo de cinco arquivos (`orders`, `products`, `customers`, `users`) e o `auth`, que dispensa repository próprio? Bruno colocou a pergunta na mesa de forma explícita: "Webhook vai seguir igual. Vou propor uma pasta `src/modules/webhooks` com toda a estrutura. Faz sentido?" (`[09:27] Bruno:`).

## Decisão

Usamos os padrões que já existem no projeto, sem introduzir stack, biblioteca ou convenção nova para o módulo de webhooks.

Concretamente:

- **Estrutura de módulo.** O código vive em `src/modules/webhooks/` (a criar), com o mesmo conjunto de arquivos dos módulos existentes — controller, service, repository, routes e schemas — espelhando `src/modules/orders/`. A lógica de processamento dos eventos fica num arquivo dentro do próprio módulo, `webhook.worker.ts` ou `webhook.processor.ts` (a criar); a entry-point do processo separado é `src/worker.ts` (a criar), espelhando a convenção de `src/server.ts` — o processo em si é assunto do [ADR-002](./ADR-002-worker-separado-com-polling.md).
- **Erros.** Toda exceção do módulo herda de `AppError` (`src/shared/errors/app-error.ts`) ou das subclasses de `src/shared/errors/http-errors.ts`, seguindo o molde de `InsufficientStockError` e `InvalidStatusTransitionError`. Os códigos de erro levam o prefixo literal `WEBHOOK_` — `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED` e os demais do módulo (`[09:28] Bruno:`, `[09:29] Larissa:`).
- **Middleware de erro.** `errorMiddleware` (`src/middlewares/error.middleware.ts`) é reaproveitado sem alteração: como ele já ramifica em `err instanceof AppError`, os erros `WEBHOOK_*` saem no mesmo envelope `{ error: { code, message, details? } }` dos demais módulos sem uma linha de mudança.
- **Validação.** Os schemas Zod do módulo entram via `validate({ body, query, params })` (`src/middlewares/validate.middleware.ts`), como em `src/modules/orders/order.routes.ts`.
- **Autenticação.** `authenticate` e `requireRole(...roles)` (`src/middlewares/auth.middleware.ts`) são usados como estão, sem nova camada de autorização. A política de quem pode chamar o quê é o [ADR-008](./ADR-008-controle-de-acesso-dos-endpoints.md).
- **Log.** Pino, via o `logger` já exportado por `src/shared/logger/index.ts`. Nenhuma biblioteca de log nova entra no projeto (`[09:29] Bruno:`).
- **Persistência.** As tabelas novas usam UUID como chave primária, por decisão explícita de `[09:51] Larissa:` ("UUID, segue o padrão do resto do projeto"). A forma concreta vem da convenção já existente no código, não da reunião: `@id @default(uuid()) @db.Char(36)` e `@@map` com nome de tabela em snake_case, como em todas as entidades de `prisma/schema.prisma`.
- **Conexão de banco.** O worker reaproveita o helper `createPrismaClient()` de `src/config/database.ts`, que já existe, em vez de montar configuração própria de client. A decisão de instanciar um `PrismaClient` por processo, e não compartilhar o singleton `prisma` de `src/config/database.ts:10`, é do [ADR-002](./ADR-002-worker-separado-com-polling.md) e não se repete aqui.

Larissa fechou o bloco: "reuso máximo do que já existe. AppError, Pino, error middleware, padrão de módulos, padrão de schemas Zod, padrão de códigos de erro. Webhook fica como módulo igual aos outros" (`[09:30] Larissa:`).

## Alternativas consideradas

**Registro honesto:** a decisão de reuso em si não teve alternativa concorrente na reunião. Bruno apresentou a proposta em `[09:27]` pedindo validação e Diego respondeu "Faz." em `[09:28]`; ninguém defendeu módulo autônomo com stack própria nem biblioteca de terceiros de webhooks. Foi consenso imediato — provavelmente porque as restrições de time pequeno e três sprints já estavam explícitas na mesa. A alternativa abaixo é a que foi de fato levantada e derrubada dentro deste bloco. A outra levantada no mesmo trecho — `PrismaClient` compartilhado entre API e worker (`[09:29] Diego:`, `[09:30] Bruno:`) — está registrada no [ADR-002](./ADR-002-worker-separado-com-polling.md) e não se repete aqui.

### `id` auto incremental nas tabelas do módulo — descartada

Levantada por Diego já no fim da call, depois da saída de Marcos e Sofia: "Quando a gente for modelar a outbox, prefere id auto incremental ou UUID?" (`[09:51] Diego:`).

Larissa derrubou pela consistência do schema: "UUID, segue o padrão do resto do projeto. Tudo é uuid" (`[09:51] Larissa:`). O argumento é verificável no `prisma/schema.prisma`: as seis entidades (`User`, `Customer`, `Product`, `Order`, `OrderItem`, `OrderStatusHistory`) usam `@default(uuid()) @db.Char(36)`; a sétima model, `OrderNumberSequence`, é uma linha de contador único com `id Int @id @default(1)` (`prisma/schema.prisma:133-138`), não uma entidade. Auto incremental numa tabela de alto volume traria vantagem de índice, mas quebraria a convenção em troca de um ganho que ninguém mediu.

## Consequências

**Positivas**

- Nenhuma dependência nova em `package.json`. Pino 9.5.0, Zod 3.23.8 e uuid 11.0.3 já estão declarados; o módulo não adiciona superfície de supply chain nem trabalho de aprovação.
- Zero alteração no caminho de erro HTTP: `errorMiddleware` já ramifica em `AppError`, `ZodError` e `Prisma.PrismaClientKnownRequestError`. Os erros `WEBHOOK_*` entram no contrato de resposta existente sem tocar o middleware — como Bruno afirmou, "vai pegar nossos erros sem precisar mudar nada" (`[09:29] Bruno:`).
- Revisão de código mais barata: os pelo menos dois dias úteis que Sofia pediu para auditar segurança (`[09:46] Sofia:`) são gastos em HMAC e geração de secret, não numa stack inteira desconhecida.
- Quem já mexe em `orders` consegue navegar `webhooks` sem contexto novo — os cinco arquivos têm os mesmos nomes e as mesmas responsabilidades.
- A estimativa de três sprints (`[09:46] Larissa:`) só fecha porque nenhum tempo é gasto montando fundação.

**Negativas**

- **O módulo herda os limites do `AppError` fora do HTTP.** O construtor é `AppError(message, statusCode, errorCode, details)` — `statusCode` é obrigatório. Falha de entrega dentro do worker não tem status HTTP próprio: o worker vai preencher um número que ninguém lê, porque não existe `res` do outro lado. É semântica emprestada, e ela fica no código.
- **O `errorMiddleware` não cobre o worker.** Ele é registrado em `src/app.ts` e só existe no processo HTTP. O worker precisa do próprio tratamento de erro em cima do Pino, o que significa dois caminhos de erro no mesmo repositório — reuso de classes, não de tratamento.
- **O `redact` atual do logger não conhece `secret`.** `src/shared/logger/index.ts` redige `req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.passwordHash`, `*.token` e `*.accessToken`. Reaproveitar "sem botar nada novo" significa que a secret por endpoint introduzida pelo [ADR-004](./ADR-004-hmac-sha256-secret-por-endpoint.md) não está coberta por essa lista hoje. Diego lembrou na própria reunião que "a gente já teve cliente que vazou secret em log de aplicação dele" (`[09:22] Diego:`).
- **Chave `Char(36)` não sequencial numa tabela de escrita quente.** A outbox recebe uma linha por mudança de status; UUID aleatório em `@db.Char(36)` é um índice mais largo e com inserção menos localizada do que um auto incremental. É o preço consciente da consistência de schema escolhida em `[09:51]`.
- **Wiring manual em arquivos compartilhados.** Não há container de injeção de dependência: `buildControllers()` em `src/app.ts` instancia repository → service → controller na mão, e `src/routes/index.ts` declara o tipo `Controllers` e monta cada rota. Registrar `webhooks` obriga a editar esses dois arquivos comuns, com o conflito de merge que isso implica durante três sprints de trabalho paralelo.
- **Divergir depois custa refatoração, com preço já identificável.** Sair do `AppError` para um erro de worker sem `statusCode` obriga a revisar todo `throw` do módulo e o ramo `err instanceof AppError` de `src/middlewares/error.middleware.ts:15`. Trocar a chave da `webhook_outbox` de `@db.Char(36)` para auto incremental depois do primeiro deploy vira migração de dados numa tabela em escrita contínua, não alteração de schema. E o wiring manual espalha o módulo por `src/app.ts` e `src/routes/index.ts`: qualquer mudança estrutural mexe nesses dois arquivos compartilhados.

**Neutras / limitações conhecidas**

- O nome do arquivo de processamento ficou em aberto entre `webhook.worker.ts` e `webhook.processor.ts` (`[09:28] Bruno:`) — Diego respondeu apenas "Beleza." (`[09:28] Diego:`). Não é decisão arquitetural: o FDD, seção 10.1, fechou em `webhook.processor.ts`.
- A cobertura de `redact` do Pino permanece como está. **Gatilho de reabertura:** no momento em que a secret circular dentro de um objeto entregue ao logger — no serviço de rotação, no worker ou em log de falha de entrega — a lista `redactPaths` de `src/shared/logger/index.ts` precisa ganhar uma entrada, e aí a promessa de "nada novo" é revista para esse ponto específico.
- Duas instâncias de `PrismaClient` contra o mesmo MySQL. **Gatilho de reabertura:** saturação de conexões do banco, ou o cenário de múltiplos workers em paralelo que o [ADR-002](./ADR-002-worker-separado-com-polling.md) registra como adiado — qualquer um dos dois obriga a configurar limite de pool explicitamente em `src/config/database.ts`.
- O padrão de módulo assume um domínio servido por HTTP. **Gatilho de reabertura:** um segundo processo em background no projeto. Com um só, `src/modules/webhooks/` com o processador dentro se sustenta; com dois ou mais, vale discutir uma pasta própria para código de background.
- A convenção de UUID `@db.Char(36)`. **Gatilho de reabertura:** degradação medida de escrita ou de tamanho de índice na `webhook_outbox` em produção. Sem medição, a consistência de schema vence.

## Rastreabilidade

**Transcrição** (`TRANSCRICAO.md`)

- `[09:27] Bruno:` — padrão de módulo em `src/modules` com controller, service, repository, routes e schemas; propõe `src/modules/webhooks`
- `[09:28] Diego:` — "Faz. E o worker fica onde?" (aceite imediato, sem alternativa)
- `[09:28] Bruno:` — `src/worker.ts` como entry separada; lógica de processamento dentro do módulo (`webhook.worker.ts` ou `webhook.processor.ts`)
- `[09:28] Bruno:` — erros seguem `AppError`, `InsufficientStockError`, `InvalidStatusTransitionError`; códigos `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`
- `[09:29] Larissa:` — "Prefixo WEBHOOK_ pra tudo do módulo."
- `[09:29] Bruno:` — Pino já está no projeto inteiro, nada novo; middleware de erro já trata AppError, Zod e Prisma
- `[09:29] Diego:` — pergunta se o worker abre o mesmo `PrismaClient` ou um separado
- `[09:30] Bruno:` — separado; `PrismaClient` é por processo, mesma `DATABASE_URL`
- `[09:30] Larissa:` — fecha: reuso máximo do que já existe
- `[09:51] Diego:` — id auto incremental ou UUID?
- `[09:51] Larissa:` — UUID, segue o padrão do resto do projeto
- `[09:22] Diego:` — cliente já vazou secret em log de aplicação (sustenta a consequência sobre `redact`)
- `[09:07] Diego:` — time pequeno (restrição de contexto)
- `[09:00] Marcos:` — ameaça de migração da Atlas se não sair até o fim do trimestre
- `[09:45] Marcos:` — prazo alvo: fim de novembro
- `[09:46] Larissa:` — estimativa de três sprints incluindo a revisão da Sofia
- `[09:46] Sofia:` — "pelo menos dois dias úteis" para revisar HMAC e geração de secret antes do deploy

**Código** (verificado no repositório)

- `src/shared/errors/app-error.ts` — `AppError(message, statusCode, errorCode, details)`; `statusCode` obrigatório
- `src/shared/errors/http-errors.ts` — hierarquia; `InvalidStatusTransitionError` → `INVALID_STATUS_TRANSITION`, `InsufficientStockError` → `INSUFFICIENT_STOCK`
- `src/shared/errors/index.ts` — barrel de exportação dos erros
- `src/middlewares/error.middleware.ts` — `errorMiddleware`; ramo `err instanceof AppError` responde `{ error: { code, message, details? } }`; trata `ZodError` e `Prisma.PrismaClientKnownRequestError` (P2002, P2025)
- `src/middlewares/validate.middleware.ts` — `validate({ body, query, params })`, converte `ZodError` em `ValidationError` com `details`
- `src/middlewares/auth.middleware.ts` — `authenticate`; `requireRole(...roles)` com papéis `ADMIN` | `OPERATOR`
- `src/shared/logger/index.ts` — Pino com `redact.paths` = `req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.passwordHash`, `*.token`, `*.accessToken` (sem entrada para `secret`)
- `src/modules/orders/` — molde do módulo: `order.controller.ts`, `order.service.ts`, `order.repository.ts`, `order.routes.ts`, `order.schemas.ts`
- `src/modules/orders/order.routes.ts` — uso combinado de `authenticate` + `validate(...)` por rota
- `src/modules/orders/order.schemas.ts` — convenção de schemas Zod exportando os tipos inferidos
- `src/modules/users/user.routes.ts` — precedente real de `requireRole('ADMIN')`
- `src/app.ts` — `buildControllers()` com wiring manual repository → service → controller; registro de `errorMiddleware` só no processo HTTP
- `src/routes/index.ts` — tipo `Controllers` e montagem das rotas sob `/api/v1`
- `src/config/database.ts` — `createPrismaClient()` e o singleton `prisma`, sem configuração de limite de pool
- `src/server.ts` — molde de entry-point que `src/worker.ts` (a criar) espelha
- `prisma/schema.prisma` — `@id @default(uuid()) @db.Char(36)` em todas as entidades; `@@map` snake_case
- `package.json` — dependências já presentes: `pino` 9.5.0, `zod` 3.23.8, `uuid` 11.0.3, `@prisma/client` 5.22.0

**A criar pela feature:** `src/modules/webhooks/` (controller, service, repository, routes, schemas), `src/modules/webhooks/webhook.worker.ts` ou `webhook.processor.ts`, `src/worker.ts`, models `webhook_outbox` e `webhook_dead_letter` em `prisma/schema.prisma`.

**Relacionado:** [ADR-001](./ADR-001-outbox-no-mysql.md) (outbox no MySQL), [ADR-002](./ADR-002-worker-separado-com-polling.md) (worker em processo separado com polling), [ADR-003](./ADR-003-retry-com-backoff-e-dlq.md) (retry e DLQ — as duas tabelas novas que seguem as convenções de schema fixadas aqui), [ADR-004](./ADR-004-hmac-sha256-secret-por-endpoint.md) (HMAC-SHA256 e secret por endpoint), [ADR-005](./ADR-005-at-least-once-com-x-event-id.md) (`X-Event-Id` em formato UUID, mesma convenção), [ADR-007](./ADR-007-snapshot-do-payload-na-insercao.md) (snapshot do payload gravado pelo módulo), [ADR-008](./ADR-008-controle-de-acesso-dos-endpoints.md) (controle de acesso dos endpoints).
