# FDD — Sistema de Webhooks de Notificação de Pedidos

- **Altitude:** implementação. Este documento responde *como construir*, não *por que decidimos assim*.
- **Documento pai:** [RFC](./RFC.md). Decisões fechadas: [ADRs 001–008](./adrs/README.md).
- **Fonte factual:** [`TRANSCRICAO.md`](../TRANSCRICAO.md) (reunião técnica de definição, quinta-feira 09:00) e o código verificado em `src/`, `prisma/` e `tests/`.
- **Stack:** Node.js >= 20 (`package.json`, `engines`), TypeScript, Express 4.21.1, Prisma 5.22.0, MySQL 8.0 (`docker-compose.yml`).

## Convenções de leitura deste documento

| Marca | Significado |
| --- | --- |
| `[hh:mm] Nome:` | fala literal da transcrição que sustenta a afirmação |
| **(a criar)** | arquivo, tabela ou script que ainda **não existe** no repositório |
| **(proposta do FDD)** | nome, dimensionamento ou regra que a reunião **não** fixou e que este documento especifica para destravar a implementação |
| **(em aberto)** | ponto que um ADR registrou como não decidido; a implementação depende de ratificação |

Números que aparecem sem uma dessas marcas vêm da reunião e estão conferidos contra o arquivo.

---

## 1. Contexto e motivação técnica

Três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — fazem hoje polling em `GET /orders` para descobrir mudança de status dos próprios pedidos (`[09:00] Marcos:`). O primeiro deles **já existe na massa de dados do repositório**: `prisma/seed.ts:76` cadastra o customer `Logística Atlas Comercial Ltda` (`contato@atlascomercial.com.br`). Não é cliente hipotético — é a linha que `npm run db:seed` insere, e é o registro sobre o qual os testes de aceitação da seção 12 podem ser escritos sem inventar fixture. A feature substitui esse polling por notificação **outbound** (`[09:02] Marcos:`, `[09:03] Sofia:`), com orçamento de latência de 10 segundos ponta a ponta (`[09:02] Marcos:`). O detalhamento do problema e das alternativas está no [RFC](./RFC.md); este documento não o renarra.

A restrição técnica que molda todo o desenho já está no código: `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126-179`) é uma transação única que valida a transição, debita ou repõe estoque, atualiza `orders` e insere em `order_status_history`. Chamada HTTP não entra ali (`[09:04] Bruno:`). O evento é gravado numa outbox dentro dessa mesma transação ([ADR-001](./adrs/ADR-001-outbox-no-mysql.md)) e entregue por um processo Node separado ([ADR-002](./adrs/ADR-002-worker-separado-com-polling.md)).

As oito decisões abaixo são **premissas fechadas** deste documento. Nenhuma seção as reabre:

| Premissa | ADR |
| --- | --- |
| Evento gravado em `webhook_outbox` dentro da transação de `changeStatus` | [ADR-001](./adrs/ADR-001-outbox-no-mysql.md) |
| Worker em processo próprio, polling de 2 segundos, instância única | [ADR-002](./adrs/ADR-002-worker-separado-com-polling.md) |
| Retry com backoff `1m/5m/30m/2h/12h` e DLQ em tabela separada | [ADR-003](./adrs/ADR-003-retry-com-backoff-e-dlq.md) |
| HMAC-SHA256 em `X-Signature`, segredo por endpoint, rotação com 24h de grace period | [ADR-004](./adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) |
| Entrega at-least-once, dedup delegada ao cliente via `X-Event-Id` | [ADR-005](./adrs/ADR-005-at-least-once-com-x-event-id.md) |
| Módulo em `src/modules/webhooks/`, `AppError`, prefixo `WEBHOOK_`, Pino, Zod, UUID `@db.Char(36)` | [ADR-006](./adrs/ADR-006-reuso-dos-padroes-do-projeto.md) |
| Payload renderizado como snapshot na inserção; filtragem por status na origem | [ADR-007](./adrs/ADR-007-snapshot-do-payload-na-insercao.md) |
| Replay de DLQ exige `ADMIN`; CRUD exige apenas autenticação | [ADR-008](./adrs/ADR-008-controle-de-acesso-dos-endpoints.md) |

---

## 2. Objetivos técnicos

1. **Emitir o evento sem janela de inconsistência.** Commit de `changeStatus` implica linha na `webhook_outbox`; rollback não deixa evento órfão. Falha na inserção derruba a mudança de status (`[09:40] Bruno:` — "Não pode ter caso de status mudar e evento não sair").
2. **Entregar dentro do orçamento de 10 segundos** (`[09:02] Marcos:`), com o ciclo de polling de 2 segundos como maior parcela fixa do atraso (`[09:09] Diego:`).
3. **Não acoplar a disponibilidade do pedido à do cliente.** Nenhuma chamada HTTP de saída dentro de `prisma.$transaction`.
4. **Tornar cada entrega auditável sem acesso ao banco:** `GET /webhooks/:id/deliveries` devolve sucesso/falha, payload, response e tempo de resposta (`[09:34] Marcos:`).
5. **Provar autoria e integridade do evento** com HMAC-SHA256 sobre o corpo exato enviado, com segredo por cadastro e rotação sem downtime (`[09:22] Sofia:`).
6. **Não perder evento por falha do outro lado:** 5 tentativas de entrega com backoff ([ADR-003](./adrs/ADR-003-retry-com-backoff-e-dlq.md), `[09:15] Diego:` / `[09:17] Larissa:`) e, esgotadas, persistência em `webhook_dead_letter` com payload e motivo (`[09:18] Diego:`).
7. **Não introduzir dependência nova em `package.json`** — HMAC via `node:crypto`, HTTP via `fetch` global (ver seção 11).
8. **Não alterar o contrato dos endpoints já publicados** de `/api/v1` além do que a seção 11 declara explicitamente.

---

## 3. Escopo e exclusões

### 3.1 Dentro do escopo

- Tabela de configuração de webhook por cliente, com CRUD autenticado e rotação de segredo.
- `webhook_outbox`, `webhook_dead_letter` e registro de entregas.
- Processo worker (`src/worker.ts`, **a criar**) com polling, assinatura, envio, retry e movimentação para DLQ.
- Endpoint administrativo de replay de DLQ.
- Alteração pontual em `OrderService.changeStatus` para publicar o evento dentro da transação.

### 3.2 Exclusões técnicas — o que o módulo deliberadamente **não** faz

Esta lista existe para que ninguém implemente por conta própria:

| O módulo não faz | Origem |
| --- | --- |
| **Rate limiting de saída.** O worker dispara uma chamada por evento pendente, sem teto por cliente por janela de tempo. 50 mudanças de status geram 50 chamadas. | `[09:39] Larissa:` — "observar e decidir depois" |
| **Arquivamento ou expurgo** de linhas já entregues da outbox. A tabela cresce monotonicamente. | `[09:08] Diego:` — fora do escopo desta feature |
| **Recepção de webhooks (inbound).** Não há rota que aceite evento de terceiro. | `[09:02] Marcos:`, `[09:03] Sofia:` |
| **Notificação ao cliente quando o webhook dele falha** (e-mail ou qualquer canal). O único aviso é o próprio cliente consultar `GET /webhooks/:id/deliveries`. | `[09:37] Larissa:` — próxima fase |
| **Painel ou dashboard.** Só endpoints HTTP. | `[09:40] Larissa:` — projeto separado do time de frontend |
| **Deduplicação do nosso lado.** Nenhuma tabela de ids processados, nenhum lock. | [ADR-005](./adrs/ADR-005-at-least-once-com-x-event-id.md) |
| **Ordenação global entre pedidos** e **execução de mais de um worker**. Acrescente-se que **nem a ordenação por `order_id` é garantida**: qualquer retry a quebra, mesmo com worker único (análise deste FDD, ver 5.2.2). | `[09:12] Diego:`, `[09:13] Diego:`, `[09:13] Larissa:` — a reunião condicionou a limitação ao single-worker; a quebra por retry é extensão deste FDD |
| **Restrição de papel no CRUD.** Qualquer usuário autenticado opera o cadastro de qualquer `customerId`. | `[09:37] Sofia:` — "Por enquanto sim" |
| **Classificação de `4xx` como falha permanente.** Qualquer resposta fora de `2xx` é retentável; a única falha definitiva é o esgotamento das tentativas. Criar classe de falha por status seria decisão nova — não foi tomada. | `[09:15] Diego:` (teto de tentativas é o único critério) |
| **Replay automático ou em lote a partir da DLQ.** Um evento por chamada, disparado por humano com role `ADMIN`. | `[09:18] Diego:`, [ADR-003](./adrs/ADR-003-retry-com-backoff-e-dlq.md) |
| **Cifra do segredo em repouso.** A coluna guarda o valor utilizável. | [ADR-004](./adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) — **(em aberto)**, cai na revisão de `[09:46] Sofia:` |
| **Versionamento do payload.** Evento já enfileirado sai no formato com que foi gravado. | [ADR-007](./adrs/ADR-007-snapshot-do-payload-na-insercao.md) |
| **Emissão de evento na criação do pedido.** `OrderService.create` (`src/modules/orders/order.service.ts:50`) grava `history` com `toStatus = PENDING` fora de `changeStatus`; o gatilho do webhook é exclusivamente a mudança de status. | `[09:06] Diego:` — "quando o status do pedido muda" |

### 3.3 Pontos herdados como **(em aberto)**

Dois pontos chegam ao FDD sem decisão fechada, e ambos **bloqueiam** trechos específicos da implementação:

1. **Qual segredo assina durante o grace period de 24h** — o novo, o antigo, ou ambos em headers distintos ([ADR-004](./adrs/ADR-004-hmac-sha256-secret-por-endpoint.md), seção Decisão). O modelo de dados da seção 4.1 (`secret` + `previousSecret` + `previousSecretExpiresAt`) suporta as três opções; o passo 5 do fluxo B não pode ser codificado antes da ratificação. Encaminhar para a revisão de segurança de `[09:46] Sofia:`.
2. **Tamanho e codificação do segredo gerado** — mesma revisão. A coluna está dimensionada em `VARCHAR(255)` **(proposta do FDD)**, folgada para qualquer escolha.

---

## 4. Modelo de dados

**Convenções seguidas — explicitamente as do schema existente**, verificadas em `prisma/schema.prisma` e em `prisma/migrations/20260519182739_init/migration.sql`:

- Chave primária `String @id @default(uuid()) @db.Char(36)`, como em `User:26`, `Customer:41`, `Product:57`, `Order:75`, `OrderItem:100` e `OrderStatusHistory:117`. Decisão explícita de `[09:51] Larissa:` — "UUID, segue o padrão do resto do projeto".
- Nome da tabela em snake_case via `@@map`; **campos em camelCase**, inclusive no SQL gerado (`migration.sql:52` — `` `orderNumber` ``, `:60` — `` `createdAt` ``).
- Timestamps `DateTime` → `DATETIME(3)`, com `@default(now())` e `@updatedAt` (`migration.sql:8-9`).
- Tabela `DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci` (`migration.sql:13`), herdado do `docker-compose.yml` (`--character-set-server=utf8mb4`).
- Coluna `Json` quando o conteúdo é estrutura variável — precedente real: `Customer.address Json` (`prisma/schema.prisma:46` → `JSON NOT NULL` em `migration.sql:22`).
- Enums em SCREAMING_SNAKE_CASE de valores, como `OrderStatus` e `UserRole` (`prisma/schema.prisma:11-23`).

Tamanhos de coluna e nomes de índice são **(proposta do FDD)** — a reunião não os discutiu. Nomes de tabela citados na reunião estão marcados como tal.

### 4.0 Enum novo

```prisma
enum WebhookOutboxStatus {
  PENDING     // aguardando a primeira tentativa
  PROCESSING  // reservado pelo worker; chamada HTTP em voo
  FAILED      // tentativa falhou, aguardando nextAttemptAt
  DELIVERED   // 2xx recebido; linha permanece como registro
}
```

Os quatro estados são os da reunião: `[09:08] Diego:` — "pendente, processando, falhou, entregue".

### 4.1 `webhook_endpoints` — configuração *(nome: **proposta do FDD**; a reunião disse apenas "tabela de configuração de webhook", `[09:21] Bruno:`)*

```prisma
model WebhookEndpoint {
  id                      String   @id @default(uuid()) @db.Char(36)
  customerId              String   @db.Char(36)
  url                     String   @db.VarChar(500)
  secret                  String   @db.VarChar(255)
  previousSecret          String?  @db.VarChar(255)
  previousSecretExpiresAt DateTime?
  secretRotatedAt         DateTime?
  subscribedStatuses      Json
  active                  Boolean  @default(true)
  createdAt               DateTime @default(now())
  updatedAt               DateTime @updatedAt

  customer     Customer          @relation(fields: [customerId], references: [id])
  outboxEvents WebhookOutbox[]
  deliveries   WebhookDelivery[]
  deadLetters  WebhookDeadLetter[]

  @@index([customerId, active])
  @@map("webhook_endpoints")
}
```

| Coluna | Tipo MySQL | Origem |
| --- | --- | --- |
| `id` | `CHAR(36)` | convenção do schema; `[09:51] Larissa:` |
| `customerId` | `CHAR(36)`, FK → `customers(id)` | `[09:21] Bruno:` — "url + secret + customer_id + estado ativo"; `[09:32] Larissa:` — vem do body ou do path, **não do JWT** |
| `url` | `VARCHAR(500)` **(proposta)** | `[09:21] Bruno:`; esquema `https` obrigatório (`[09:23] Sofia:`) |
| `secret` | `VARCHAR(255)` **(proposta)** | segredo único por endpoint, `[09:21] Sofia:` |
| `previousSecret` | `VARCHAR(255)` NULL | segredo anterior durante o grace period, `[09:21] Sofia:` |
| `previousSecretExpiresAt` | `DATETIME(3)` NULL | vencimento das 24h, `[09:21] Sofia:` |
| `secretRotatedAt` | `DATETIME(3)` NULL **(proposta)** | instante da última rotação; `NULL` enquanto o cadastro usar o segredo original. Serializável — não é material secreto |
| `subscribedStatuses` | `JSON` | lista de status que o cadastro escuta, `[09:33] Marcos:`; formato `["SHIPPED","DELIVERED"]` |
| `active` | `BOOLEAN` default `true` | "estado ativo", `[09:21] Bruno:` |

**Sobre `secret`: o único padrão criptográfico do projeto não se aplica aqui.** Vale registrar, porque a tentação de "seguir o que já existe" ([ADR-006](./adrs/ADR-006-reuso-dos-padroes-do-projeto.md)) leva ao lugar errado. O projeto guarda credencial em um único ponto — `src/modules/users/user.service.ts:16`, `BCRYPT_ROUNDS = 10`, aplicado à senha do usuário. **Esse padrão é inaplicável ao segredo de webhook:** bcrypt é hash unidirecional, útil porque para validar senha basta comparar hashes; o HMAC de saída ([ADR-004](./adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)) exige o **segredo em claro** para recomputar a assinatura a cada envio, e de um hash bcrypt não se recupera valor nenhum. Guardar o segredo como o projeto guarda senha simplesmente impede a feature de funcionar.

Consequência prática: a coluna guarda **valor utilizável**, e a única escolha real é entre texto claro e cifra reversível em repouso. Este FDD segue o [ADR-004](./adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) e mantém texto claro (exclusão declarada em 3.2), mas isso deixa de ser detalhe de implementação e passa a ser **questão de segurança com decisão pendente** — vai à revisão de `[09:46] Sofia:` junto com geração e tamanho do segredo, e é o que dá peso ao risco R-4. **A comparação com o `BCRYPT_ROUNDS` é análise deste FDD sobre o código; a reunião não discutiu armazenamento do segredo.**

**Índices e justificativa**

| Índice | Por que existe |
| --- | --- |
| `PRIMARY KEY (id)` | acesso por id em `GET/PATCH/DELETE /webhooks/:id`, `/deliveries` e `/secret/rotate` |
| `@@index([customerId, active])` | é **a consulta do caminho quente**: dentro da transação de `changeStatus`, `WHERE customerId = ? AND active = true` (fluxo A, passo 4). Composto porque os dois predicados são aplicados juntos e sempre. O mesmo índice serve `GET /webhooks?customerId=` (`[09:33] Bruno:`) usando só o prefixo `customerId`. |

**Índice que deliberadamente não existe:** nenhum sobre `url` — não há consulta por URL em nenhum fluxo desta feature, e a reunião não estabeleceu unicidade de URL por cliente.

### 4.2 `webhook_outbox` — fila de trabalho *(nome fixado em `[09:06] Diego:`)*

```prisma
model WebhookOutbox {
  id                String              @id @default(uuid()) @db.Char(36)
  webhookEndpointId String              @db.Char(36)
  orderId           String              @db.Char(36)
  eventType         String              @db.VarChar(50)
  payload           Json
  status            WebhookOutboxStatus @default(PENDING)
  attemptCount      Int                 @default(0)
  nextAttemptAt     DateTime            @default(now())
  lastError         String?             @db.VarChar(500)
  requestId         String?             @db.VarChar(64)
  createdAt         DateTime            @default(now())
  updatedAt         DateTime            @updatedAt

  endpoint WebhookEndpoint @relation(fields: [webhookEndpointId], references: [id], onDelete: Cascade)
  order    Order           @relation(fields: [orderId], references: [id], onDelete: Cascade)

  @@index([status, nextAttemptAt])
  @@index([createdAt])
  @@index([orderId])
  @@index([webhookEndpointId])
  @@map("webhook_outbox")
}
```

**`id` é o `X-Event-Id`.** A chave primária da linha é o UUID gerado na inserção e é exatamente o valor enviado no header (`[09:25] Diego:` — "um UUID gerado quando o evento entra na outbox"). Não existe segunda coluna de identificador de evento; é isso que garante a estabilidade do valor entre as reentregas exigida pelo [ADR-005](./adrs/ADR-005-at-least-once-com-x-event-id.md).

**Uma linha por (mudança de status × cadastro que escuta aquele status).** Se a Atlas tem dois cadastros e ambos escutam `SHIPPED`, uma transição para `SHIPPED` cria duas linhas, com dois `X-Event-Id` distintos.

> **Esta é uma lacuna que a reunião não fechou, e este FDD a resolve.** A ata tem as duas leituras convivendo sem que ninguém as confronte: a filtragem na inserção (`[09:34] Bruno:`) e o snapshot único sugerem *uma linha por evento*, enquanto o `X-Webhook-Id` (`[09:44] Sofia:`) só faz sentido se o envio souber a qual cadastro pertence — isto é, *fan-out por endpoint*. Ninguém enunciou o caso de um cliente com dois cadastros escutando o mesmo status. **A identificação da lacuna e a escolha do modelo são análise deste FDD, não fala de participante.**

**Por que uma linha por par é a única modelagem coerente.** As três colunas que governam a entrega — `attemptCount`, `nextAttemptAt` e o destino em `webhook_dead_letter` — são estado **por destino**, não por evento. Com uma única linha para dois endpoints, um destino que responde `200` e outro que devolve `503` teriam de compartilhar o mesmo contador de tentativas e o mesmo agendamento de backoff: reentregar para o que já recebeu, ou parar de tentar no que ainda não recebeu. E a ida para a DLQ (fluxo 5.4), que **remove a linha da outbox**, apagaria a fila do destino que ainda estava saudável. Some-se a isso que cada linha é assinada com o segredo de **um** cadastro ([ADR-004](./adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)), e o desdobramento por destinatário deixa de ser preferência e passa a ser exigência do retry e da DLQ já fechados no [ADR-003](./adrs/ADR-003-retry-com-backoff-e-dlq.md).

| Coluna | Origem |
| --- | --- |
| `webhookEndpointId` | destinatário resolvido na inserção (filtragem na origem, `[09:34] Bruno:`) |
| `orderId` | chave de ordenação lógica por pedido, `[09:12] Diego:` |
| `eventType` | `"order.status_changed"`, `[09:43] Diego:` |
| `payload` | snapshot JSON renderizado na inserção, `[09:52] Larissa:` / [ADR-007](./adrs/ADR-007-snapshot-do-payload-na-insercao.md) |
| `status` | os quatro estados de `[09:08] Diego:` |
| `attemptCount` | chamadas HTTP já realizadas; teto na seção 8.2 |
| `nextAttemptAt` | momento a partir do qual o worker pode tentar; recebe a progressão de `[09:17] Diego:` |
| `lastError` **(proposta)** | motivo da última falha, insumo do `failureReason` da DLQ (`[09:18] Diego:`) |
| `requestId` **(proposta)** | valor de `req.id` (`src/middlewares/request-logger.middleware.ts:6-7`) da requisição que originou o evento — é o elo de correlação da seção 9.3 |

**Índices e justificativa**

| Índice | Por que existe |
| --- | --- |
| `@@index([status, nextAttemptAt])` | consulta única do laço do worker, executada a cada 2s para sempre: `WHERE status IN ('PENDING','FAILED') AND nextAttemptAt <= NOW(3)`. O composto cobre os dois predicados. Atende ao "índice no campo de status" pedido em `[09:08] Diego:` na forma que a política de backoff exige. |
| `@@index([createdAt])` | ordem de consumo — `ORDER BY createdAt ASC` (`[09:09] Diego:` "os eventos pendentes mais antigos"; `[09:12] Diego:` "processa em ordem de created_at"). Pedido nominalmente em `[09:08] Diego:`. |
| `@@index([orderId])` | suporte e diagnóstico: "quais eventos este pedido gerou e em que estado estão". É a consulta que torna o caminho do evento inspecionável com um `SELECT`, conforme a consequência registrada no [ADR-002](./adrs/ADR-002-worker-separado-com-polling.md). Também é o índice exigido pela FK. |
| `@@index([webhookEndpointId])` | exigido pela FK e usado pelo `ON DELETE CASCADE` do `DELETE /webhooks/:id` (fluxo detalhado na seção 6.4). |

### 4.3 `webhook_deliveries` — histórico de entregas *(nome: **proposta do FDD**; o requisito é de `[09:34] Marcos:`)*

```prisma
model WebhookDelivery {
  id                 String   @id @default(uuid()) @db.Char(36)
  webhookEndpointId  String   @db.Char(36)
  eventId            String   @db.Char(36)
  attemptNumber      Int
  success            Boolean
  requestPayload     Json
  responseStatusCode Int?
  responseBody       String?  @db.Text
  errorMessage       String?  @db.VarChar(500)
  durationMs         Int
  createdAt          DateTime @default(now())

  endpoint WebhookEndpoint @relation(fields: [webhookEndpointId], references: [id], onDelete: Cascade)

  @@index([webhookEndpointId, createdAt])
  @@index([eventId])
  @@map("webhook_deliveries")
}
```

Os quatro dados que `[09:34] Marcos:` exigiu mapeiam um a um: **sucesso/falha** → `success`; **payload** → `requestPayload`; **response** → `responseStatusCode` + `responseBody`; **tempo de resposta** → `durationMs`.

`eventId` **não** é FK para `webhook_outbox`: a linha da outbox é removida quando o evento vai para a DLQ (seção 6.4), e o histórico precisa sobreviver a isso. É referência solta, e está dito de propósito.

`responseBody` é `TEXT` e é truncado na gravação — o corpo vem de sistema de terceiro e não tem tamanho controlado por nós. O limite é `WEBHOOK_RESPONSE_BODY_MAX_CHARS` **(proposta do FDD)**, default **2048 caracteres**, no mesmo estilo das demais variáveis da seção 11.1; a reunião não fixou valor. O truncamento é por caracteres e acrescenta o sufixo `…[truncated]`, para que quem lê o histórico saiba que o corpo não é o completo. Um corpo maior que isso não tem valor de diagnóstico e só infla a tabela que o [ADR-007](./adrs/ADR-007-snapshot-do-payload-na-insercao.md) já registra como duplicadora de dado (risco R-7).

**Índices e justificativa**

| Índice | Por que existe |
| --- | --- |
| `@@index([webhookEndpointId, createdAt])` | é literalmente a consulta de `GET /webhooks/:id/deliveries`: filtra por cadastro e ordena por data decrescente para devolver os últimos ~100 (`[09:34] Marcos:`). O composto resolve filtro e ordenação no mesmo índice, sem filesort. |
| `@@index([eventId])` | reunir as tentativas de um mesmo evento para responder "por que este evento foi parar na DLQ" — a consulta de diagnóstico que a DLQ manual (`[09:18] Diego:`) exige. |

**Índice que deliberadamente não existe:** nenhum sobre `success`. Cardinalidade dois e nenhuma consulta dos fluxos filtra só por ele; a triagem por cliente já vem do índice composto.

### 4.4 `webhook_dead_letter` — falhas permanentes *(nome fixado em `[09:18] Diego:`)*

```prisma
model WebhookDeadLetter {
  id                String    @id @default(uuid()) @db.Char(36)
  eventId           String    @unique @db.Char(36)
  webhookEndpointId String    @db.Char(36)
  orderId           String    @db.Char(36)
  eventType         String    @db.VarChar(50)
  payload           Json
  failureReason     String    @db.VarChar(500)
  attemptCount      Int
  requestId         String?   @db.VarChar(64)
  failedAt          DateTime  @default(now())
  replayedAt        DateTime?
  replayedById      String?   @db.Char(36)

  endpoint WebhookEndpoint @relation(fields: [webhookEndpointId], references: [id], onDelete: Cascade)

  @@index([webhookEndpointId])
  @@index([failedAt])
  @@map("webhook_dead_letter")
}
```

Os três campos que `[09:18] Diego:` exigiu: **payload** → `payload`; **motivo da falha** → `failureReason`; **timestamp** → `failedAt`.

`replayedById` guarda o `req.user.id` de quem executou o replay — é o registro de auditoria pedido em `[09:36] Sofia:` ("o endpoint de admin tem que logar quem fez o replay"), persistido em vez de deixado só no log de aplicação.

**Índices e justificativa**

| Índice | Por que existe |
| --- | --- |
| `@@unique([eventId])` | torna a movimentação para a DLQ idempotente: se o worker morrer entre gravar a DLQ e apagar a linha da outbox, a segunda tentativa colide em vez de duplicar a evidência. Também é o índice usado ao verificar se o `X-Event-Id` preservado no replay já existe. |
| `@@index([webhookEndpointId])` | triagem operacional por cliente — "quais eventos deste cadastro desistimos de entregar". Exigido também pela FK. |
| `@@index([failedAt])` | a DLQ é operada por humano (`[09:18] Diego:`, [ADR-003](./adrs/ADR-003-retry-com-backoff-e-dlq.md)); a varredura é sempre cronológica ("o que caiu desde ontem"). |

**Índice que deliberadamente não existe:** nenhum sobre `replayedAt`, apesar de `WHERE replayedAt IS NULL` ser a consulta de triagem mais frequente. A tabela é pequena no regime normal — chega evento que falhou todas as tentativas ao longo de horas — e o filtro por `failedAt` já reduz o conjunto. A exceção é a desativação de cadastro com fila acumulada (5.2.2, passo 3.2), que insere em lote. Se isso virar rotina, ou se a DLQ crescer a ponto de a varredura pesar, o índice entra com medição; sem medição, seria cargo cult.

### 4.5 Ordem de dependência das FKs

`webhook_endpoints` → `customers`. `webhook_outbox` → `webhook_endpoints`, `orders`. `webhook_deliveries` → `webhook_endpoints`. `webhook_dead_letter` → `webhook_endpoints`. Nenhuma coluna de tabela existente muda; a migration é **puramente aditiva** (ver seção 11.2). A consequência para os testes está na seção 10.

---

## 5. Fluxos detalhados

### 5.1 Fluxo A — criação do evento dentro da transação de `changeStatus`

**Gatilho:** `PATCH /api/v1/orders/:id/status` (`src/modules/orders/order.routes.ts:19-23`).

**Onde a chamada entra.** `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126-179`) executa tudo dentro de um único `this.prisma.$transaction` (`:131`). A sequência real, conferida linha a linha no arquivo, é esta — e é ela que vale para a implementação:

| Passo | Linhas | Operação |
| --- | --- | --- |
| 1 | `:132-136` | `tx.order.findUnique` com `include: { items: true }`; `NotFoundError('Order')` se não existir |
| 2 | `:138-149` | valida `from === to` e `canTransition(from, to)` |
| 3 | `:151-156` | **estoque** — `debitStock` / `replenishStock` |
| 4 | `:158` | `tx.order.update({ status: to })` |
| 5 | `:159-167` | `tx.orderStatusHistory.create` |
| **6** | **(novo)** | **`publishWebhookEvent` — inserção na `webhook_outbox`** |
| 7 | `:169-177` | refetch com `include` e `return refreshed!` |

> **Divergência entre a ata e o código — o código prevalece.** `[09:40] Bruno:` descreve a transação como *"update na order, insere no history e atualiza estoque"*. **Essa ordem não é a do código:** o estoque é movimentado **antes** do `update` da order (`:151-156` vem antes de `:158`), não depois. A inversão não muda a garantia transacional discutida na reunião — tudo commita junto —, mas muda onde a publicação encaixa e quais exceções já ocorreram quando ela é alcançada. Quem implementar deve abrir `src/modules/orders/order.service.ts:126-179` e seguir a tabela acima, não a frase da ata. **Esta observação é análise deste FDD sobre o código; não foi levantada na reunião.**

A publicação entra **entre a linha 167 e a linha 169** — depois do `orderStatusHistory.create`, antes do refetch. Razão de posição: o evento só é montado quando a mudança de estado já está escrita na transação, e uma falha na publicação aborta antes de gastar o refetch. Como o estoque (passo 3) já foi movimentado nesse ponto, um rollback provocado pela publicação também desfaz o débito ou a reposição — o que é o comportamento desejado, e é consequência da ordem **real**, não da ordem descrita na ata.

```ts
// src/modules/orders/order.service.ts — dentro do $transaction, após :167
const enqueuedEventIds = await publishWebhookEvent(tx, order, from, to, requestId);
```

**Passo a passo (tudo abaixo está DENTRO da transação):**

1. `changeStatus` chega ao ponto de publicação com `order` (carregada em `:132`, contendo `id`, `orderNumber`, `customerId`, `totalCents`), `from` e `to`.
2. `publishWebhookEvent(tx, order, fromStatus, toStatus, requestId)` é invocada com o `tx` client corrente — função recebendo o `tx`, sem injetar repository no `OrderService` (`[09:41] Bruno:`, `[09:41] Diego:`).
3. **Consulta dos destinatários:** `tx.webhookEndpoint.findMany({ where: { customerId: order.customerId, active: true } })`. Usa `@@index([customerId, active])`.
4. **Filtragem por status de interesse:** mantém apenas os cadastros cujo `subscribedStatuses` contém `toStatus`. A filtragem é em memória sobre o resultado do passo 3, porque `subscribedStatuses` é coluna `JSON`.
5. **Caminho de exceção — nenhum destinatário:** a função retorna lista vazia **sem inserir nada**. "Se nenhum webhook do customer quer aquele status, nem insere" (`[09:34] Bruno:`). A transação segue normalmente e a mudança de status acontece.
6. **Renderização do snapshot**, uma vez, reaproveitada para todos os destinatários: objeto JSON no formato da seção 6.8, com os campos em snake_case.
7. **Verificação do teto de tamanho:** `Buffer.byteLength(JSON.stringify(payload), 'utf8')` contra 65536 bytes (64 KB, `[09:24] Diego:` / `[09:24] Larissa:`).
   - **Caminho de exceção:** lança `WebhookPayloadTooLargeError` (`WEBHOOK_PAYLOAD_TOO_LARGE`, 422). Como está dentro do `$transaction`, **a mudança de status sofre rollback** e o `PATCH /orders/:id/status` responde 422. **Esta consequência é inferência deste FDD (proposta do FDD)**, não fala da reunião: `[09:23] Sofia:` ("Eu sou a favor de erra") tratava do **envio** do webhook, e `[09:40] Bruno:` ("Se a outbox falhar de inserir, rollback") tratava de **falha de inserção**. Derrubar a mudança de status por tamanho de payload é a leitura conservadora da garantia do [ADR-001](./adrs/ADR-001-outbox-no-mysql.md), e é inalcançável com o corpo fechado em nove campos escalares (6.8.3) — o teto existe como rede contra enriquecimento futuro do payload, não como caminho esperado. Ratificar com Bruno e Larissa no kickoff; a alternativa (registrar o evento truncado e não derrubar o pedido) exige decisão nova.
8. **Inserção:** um `tx.webhookOutbox.create` por destinatário, com `status = PENDING`, `attemptCount = 0`, `nextAttemptAt = now()`, `payload` = snapshot, `requestId` = `req.id` propagado.
9. `publishWebhookEvent` devolve os ids criados (que são os `X-Event-Id`).
10. A transação prossegue para o refetch (`:169-176`) e commita.

**Fora da transação:** apenas o log. `changeStatus` guarda os ids em variável do escopo externo ao callback e emite `webhook_event_enqueued` **depois** que `this.prisma.$transaction` resolve — logar dentro do callback anunciaria evento que ainda pode sofrer rollback.

**Caminhos de exceção herdados, que continuam funcionando sem mudança:** `NotFoundError('Order')` (`:136`), `ConflictError('INVALID_STATUS_TRANSITION')` (`:141-145`), `InvalidStatusTransitionError` (`:148`) e `InsufficientStockError` (lançado por `debitStock`, `:223`) ocorrem **antes** do ponto de publicação; nenhum evento é criado nesses casos, por construção da ordem das operações.

**Garantia resultante:** commit implica evento na outbox; rollback implica nenhum evento. Não existe estado intermediário observável.

### 5.2 Fluxo B — processamento pelo worker em polling

**Processo:** `src/worker.ts` **(a criar)**, no molde de `src/server.ts`, iniciado por `npm run worker` **(a criar)** (`[09:11] Larissa:`).

#### 5.2.1 Bootstrap

1. Carrega `env` (`src/config/env.ts`, com as variáveis novas da seção 11.1).
2. Instancia `PrismaClient` **próprio** via `createPrismaClient()` (`src/config/database.ts:4`) — mesma `DATABASE_URL`, instância nova porque é outro processo Node (`[09:30] Bruno:`).
3. Cria o logger filho: `logger.child({ component: 'webhook-worker' })`, sobre o singleton de `src/shared/logger/index.ts:32`.
4. **Reclamação de linhas órfãs:** `updateMany` de `status: 'PROCESSING'` para `status: 'PENDING'`. Como o worker é único (`[09:12] Diego:`), nenhuma outra instância pode estar processando essas linhas — elas só existem porque um processo anterior morreu no meio. Emite `webhook_outbox_reclaimed` (nível `warn`) com a contagem.
5. **Expiração de segredos anteriores:** `updateMany` em `webhook_endpoints` com `where: { previousSecretExpiresAt: { lt: new Date() } }` e `data: { previousSecret: null, previousSecretExpiresAt: null }`. É o descarte que o [ADR-004](./adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) exige ("Passadas as 24 horas, o segredo antigo é descartado do nosso lado"). Repetido a cada N ciclos de polling — `WEBHOOK_SECRET_SWEEP_EVERY_CYCLES` **(proposta do FDD)**, default 300 (10 minutos a 2s/ciclo) — porque o worker pode ficar dias no ar. Emite `webhook_secret_expired` (nível `info`) com a contagem.
6. Registra `SIGINT`/`SIGTERM` no molde de `src/server.ts:20-21`: marca o laço para parar, aguarda o ciclo corrente terminar, `prisma.$disconnect()`, `process.exit(0)`.
7. Emite `webhook_worker_started` e entra no laço.

#### 5.2.2 Ciclo de polling (a cada 2 segundos, `[09:09] Diego:`)

1. Consulta o lote:
   ```sql
   SELECT * FROM webhook_outbox
   WHERE status IN ('PENDING','FAILED') AND nextAttemptAt <= NOW(3)
   ORDER BY createdAt ASC
   LIMIT :batchSize
   ```
   `FAILED` entra junto porque é o estado de "falhou e está aguardando a próxima tentativa". O tamanho do lote é "batch pequeno" (`[09:08] Diego:`) — a reunião **não deu número**, então é variável de ambiente obrigatória, sem default (seção 11.1).
2. **Caminho de exceção — lote vazio:** dorme 2 segundos e reinicia. Nada em nível `info` (só `debug`), para não inundar o log com ciclos ociosos.
3. Para cada linha, **sequencialmente** — uma chamada HTTP por vez, sem paralelismo (é o que mantém a vazão previsível com worker único, `[09:12] Diego:`):
   1. **Reserva:** `update` para `status = PROCESSING`. Escrita isolada, não transacional com o envio.
   2. **Carrega o cadastro** por `webhookEndpointId`.
      - **Caminho de exceção — cadastro inativo (`active = false`):** **(proposta do FDD)** — a reunião não definiu o destino de evento já enfileirado quando o cadastro é desativado, e o [ADR-003](./adrs/ADR-003-retry-com-backoff-e-dlq.md) só previa a DLQ para esgotamento de tentativas. O evento **não é enviado**; vai direto para a DLQ com `failureReason = WEBHOOK_ENDPOINT_INACTIVE` (fluxo 5.4), sem contabilizar tentativa. Preserva a evidência e permite recuperação por replay após reativação, em vez de descartar silenciosamente. **Consequência a conhecer:** desativar um cadastro com fila acumulada move todas as linhas pendentes dele para a DLQ de uma vez, e o replay é unitário (6.7) — é o cenário que o [ADR-003](./adrs/ADR-003-retry-com-backoff-e-dlq.md) registra como gatilho para revisar o replay unitário.
      - **Não há caminho de exceção "sem segredo":** `secret` é `NOT NULL` (4.1) e é preenchido na criação do cadastro, porque a plataforma o gera (`[09:31] Marcos:`). Um cadastro carregado sempre tem segredo utilizável — ver 7.1.1.
   3. **Serializa o corpo uma única vez** (`const body = JSON.stringify(payload)`) e usa **exatamente essa string** para assinar e para enviar. Assinar uma serialização e enviar outra é o erro clássico que quebra a verificação do cliente.
   4. **Assina:** `createHmac('sha256', secret).update(body).digest('hex')`, de `node:crypto` ([ADR-004](./adrs/ADR-004-hmac-sha256-secret-por-endpoint.md); nenhuma dependência nova — seção 11).
      - **(em aberto)** durante o grace period de 24h existem dois segredos válidos e a reunião não definiu qual assina (`[09:21] Sofia:`, [ADR-004](./adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)). Este passo depende da ratificação da seção 3.3.
   5. **Envia:** `fetch(url, { method: 'POST', headers, body, signal: AbortSignal.timeout(10_000) })` — 10 segundos (`[09:42] Diego:`). `fetch` global do Node >= 20 (`package.json`, campo `engines`).
   6. **Mede a duração** com `process.hrtime.bigint()`, no mesmo padrão de `src/middlewares/request-logger.middleware.ts:10,13`.
   7. **Classifica o resultado:**

      | Resultado | Classificação |
      | --- | --- |
      | HTTP `2xx` | sucesso |
      | HTTP fora de `2xx` (inclui `4xx` e `5xx`) | falha **retentável** — `WEBHOOK_DELIVERY_FAILED` |
      | Timeout de 10s (`AbortError`) | falha **retentável** — `WEBHOOK_DELIVERY_TIMEOUT` (`[09:42] Diego:`) |
      | Erro de rede, DNS ou TLS | falha **retentável** — `WEBHOOK_DELIVERY_FAILED` |

      Não há falha permanente por status HTTP — ver exclusão na seção 3.2.
   8. **Sucesso — transação curta do worker** (`prisma.$transaction`, distinta e independente da transação do fluxo A):
      - `webhookDelivery.create` com `success = true`, `responseStatusCode`, `responseBody` truncado, `durationMs`, `attemptNumber = attemptCount + 1`;
      - `webhookOutbox.update` para `status = DELIVERED` e `attemptCount + 1`.

      **Fora dessa transação:** a chamada HTTP (já concluída) e o log `webhook_delivery_attempt`.
   9. **Falha:** segue para o fluxo 5.3.

**Limite da ordenação — o retry a quebra, e a limitação real é mais forte que a registrada pelo time.**

`[09:13] Larissa:` fechou a limitação como *"Não é garantia de ordering global, só por `order_id` e enquanto for single-worker"* — isto é, condicionada à concorrência. **A análise do desenho detalhado mostra que a condição não é essa: o worker único não preserva a ordem por `order_id`, e não é preciso nenhum segundo worker para quebrá-la.** Basta um retry:

1. `PAID` do pedido X é enviado, o cliente devolve `503`, a linha vai para `FAILED` com `nextAttemptAt = agora + 1 min` (fluxo 5.3).
2. Nos ciclos seguintes, o filtro do passo 1 (`nextAttemptAt <= NOW(3)`) **pula** essa linha.
3. Dentro desse minuto, o pedido X muda para `PROCESSING`; a linha nova entra como `PENDING` com `nextAttemptAt = now()` e é elegível imediatamente.
4. O cliente recebe `PROCESSING` e, até um minuto depois, recebe `PAID`.

Isso acontece com **uma única instância do worker, sem paralelismo algum e sem nenhuma falha do nosso lado** — a inversão é produzida pela própria política de backoff do [ADR-003](./adrs/ADR-003-retry-com-backoff-e-dlq.md), e a janela de inversão chega a 12 horas nos últimos intervalos da progressão (8.2). Dentro de um mesmo lote os eventos saem em ordem de `createdAt`; entre lotes, não há ordem.

A mitigação existente é de contrato, não de mecanismo: o cliente tem `from_status`/`to_status` no corpo (6.8.3) e reconstrói a sequência por eles, e o portal do desenvolvedor precisa dizer isso com todas as letras (`[09:26] Marcos:`) — "os eventos podem chegar fora de ordem" é mais honesto do que "a ordem é garantida por pedido". Ordenação real exigiria bloquear a fila de um `order_id` enquanto houver evento anterior dele pendente, o que converte um cliente com falha em fila parada — decisão nova, que ninguém tomou.

> **Esta é uma extensão de análise deste FDD, não fala da reunião.** A reunião registrou a limitação sob a condição de concorrência (`[09:12] Diego:`, `[09:13] Diego:`, `[09:13] Larissa:`); ninguém examinou a interação entre o backoff e a ordem de consumo. O [ADR-002](./adrs/ADR-002-worker-separado-com-polling.md) e a questão em aberto de ordenação do [RFC](./RFC.md) precisam refletir a formulação mais forte.

#### 5.2.3 Caminho de exceção crítico — crash do worker entre enviar e marcar

É o caso que justifica a garantia at-least-once ([ADR-005](./adrs/ADR-005-at-least-once-com-x-event-id.md)) e ele **não tem solução dentro do processo**:

1. O worker reserva a linha (`PROCESSING`).
2. Faz o `POST`; o cliente recebe, processa e responde `200`.
3. O processo morre — OOM, deploy, `SIGKILL` — antes de executar o passo 8 do 5.2.2.
4. A linha permanece `PROCESSING` indefinidamente; nenhum ciclo a seleciona, porque o filtro é `PENDING`/`FAILED`.
5. No próximo bootstrap, o passo 4 do 5.2.1 devolve a linha para `PENDING`.
6. O evento é reenviado. O cliente recebe **o mesmo `X-Event-Id`**, porque ele é a chave primária da linha e não muda entre tentativas — e deduplica do lado dele.

Sem o passo 5, a linha ficaria presa em `PROCESSING` para sempre e o evento se perderia em silêncio. A regra "toda linha `PROCESSING` no bootstrap volta a `PENDING`" só é correta porque o worker é **único** (`[09:12] Diego:`); com múltiplos workers em paralelo — adiado em `[09:13] Diego:` — ela devolveria à fila linhas que outra instância está processando naquele instante, e precisaria virar lease com expiração.

### 5.3 Fluxo C — retry com backoff

Disparado por qualquer falha classificada como retentável no passo 7 do 5.2.2.

1. Calcula `attemptNumber = attemptCount + 1`.
2. Consulta a progressão `[1m, 5m, 30m, 2h, 12h]` (`[09:17] Diego:`) pelo índice `attemptNumber - 1`, onde `attemptNumber` é o valor calculado no passo 1 (a chamada HTTP que acabou de falhar, contada a partir de 1). Primeira falha → `attemptNumber = 1` → índice `0` → 1 minuto. Se o índice não existir no array, a progressão acabou: vai para o fluxo 5.4. (Aritmética completa na seção 8.2.)
3. **Ainda há intervalo disponível — transação curta do worker:**
   - `webhookDelivery.create` com `success = false`, `responseStatusCode` (quando houve resposta), `errorMessage`, `durationMs`;
   - `webhookOutbox.update` para `status = FAILED`, `attemptCount = attemptNumber`, `nextAttemptAt = now() + intervalo`, `lastError`.

   **Fora da transação:** log `webhook_delivery_failed` (nível `warn`) com `nextAttemptAt`.
4. **A progressão acabou:** vai para o fluxo 5.4.
5. **Caminho de exceção — falha na própria gravação do resultado** (banco indisponível no meio): a transação do worker faz rollback, a linha continua `PROCESSING` e cai no caso 5.2.3 no próximo restart. Nenhuma entrega é perdida; no máximo é repetida.

### 5.4 Fluxo D — movimentação para a DLQ

1. Ocorre quando o último intervalo da progressão foi consumido e a tentativa falhou, ou quando o cadastro está inativo (5.2.2, passo 3.2).
2. **Transação curta do worker**, tudo ou nada:
   - `webhookDelivery.create` da tentativa que falhou (quando houve tentativa);
   - `webhookDeadLetter.create` copiando da linha da outbox: `eventId` = `id` da linha, `webhookEndpointId`, `orderId`, `eventType`, `payload`, `attemptCount`, `requestId`; mais `failureReason` e `failedAt = now()`;
   - `webhookOutbox.delete` da linha.
3. A linha **sai** da outbox. É o que mantém a fila de trabalho enxuta — "mais limpa a leitura da outbox principal" (`[09:18] Diego:`).
4. **Fora da transação:** log `webhook_event_dead_lettered` (nível `error`) e a métrica `webhook_dead_letter_total`.
5. **Caminho de exceção — crash entre o `create` da DLQ e o `delete` da outbox:** não deixa estado inconsistente, porque os dois estão na mesma transação. Se o crash for antes do commit, a linha volta a ser `PROCESSING` órfã e é reclamada no bootstrap; se a segunda passagem tentar inserir a DLQ de novo, colide no `@@unique([eventId])` (seção 4.4), e o worker trata a colisão como "já está na DLQ", apenas removendo a linha da outbox.
6. **Nada é notificado a ninguém automaticamente.** Não há e-mail (`[09:37] Larissa:`); o sinal operacional é a métrica e o log da seção 9.

### 5.5 Fluxo E — replay manual da DLQ

**Gatilho:** `POST /api/v1/admin/webhooks/dead-letter/:id/replay`, com role `ADMIN` ([ADR-008](./adrs/ADR-008-controle-de-acesso-dos-endpoints.md)).

1. `authenticate` valida o Bearer e popula `req.user` (`src/middlewares/auth.middleware.ts:27-47`).
   - **Exceção:** ausência de token ou token inválido → `401 UNAUTHORIZED`.
2. `requireRole('ADMIN')` (`src/middlewares/auth.middleware.ts:49-61`).
   - **Exceção:** `OPERATOR` → `403 FORBIDDEN` via `ForbiddenError('Insufficient permissions')`.
3. `validate({ params })` com `z.object({ id: z.string().uuid() })`.
   - **Exceção:** id não-UUID → `400 VALIDATION_ERROR` com `details: [{ path, message }]`.
4. Carrega a linha da DLQ por `id`.
   - **Exceção:** inexistente → `404 WEBHOOK_DEAD_LETTER_NOT_FOUND`.
5. **Exceção — já reprocessada** (`replayedAt` preenchido) → `409 WEBHOOK_ALREADY_REPLAYED`. Evita reinjetar o mesmo evento duas vezes por clique duplo.
6. Carrega o cadastro de destino.
   - **Exceção:** cadastro inativo → `409 WEBHOOK_ENDPOINT_INACTIVE`. Reinjetar evento para cadastro desativado só produziria uma segunda ida à DLQ.
7. **Transação (esta é uma `prisma.$transaction` da API, não do worker):**
   - `webhookOutbox.create` com **`id` = `eventId` original** — o `X-Event-Id` é preservado —, mais `webhookEndpointId`, `orderId`, `eventType`, `payload` e `requestId` copiados da linha da DLQ, e `status = PENDING`, `attemptCount = 0`, `nextAttemptAt = now()`, `lastError = null` ("Recoloca na outbox como pendente", `[09:18] Diego:`);
   - `webhookDeadLetter.update` com `replayedAt = now()` e `replayedById = req.user.id` (auditoria de `[09:36] Sofia:`).
   - **Exceção:** se já existir linha na outbox com aquele id, o Prisma devolve `P2002` e `errorMiddleware` (`src/middlewares/error.middleware.ts:37-47`) responde `409 CONFLICT` — código genérico, não `WEBHOOK_*`, porque vem do middleware compartilhado. É salvaguarda; o caminho normal é barrado no passo 5.
8. Responde `202 Accepted` — o evento foi **reenfileirado**, não entregue. A entrega acontece no ciclo seguinte do worker, em até 2 segundos.
9. Emite `webhook_dlq_replayed` (nível `info`) com `userId`, `deadLetterId` e `eventId`.

**Preservação do `X-Event-Id` no replay:** o [ADR-005](./adrs/ADR-005-at-least-once-com-x-event-id.md) registra este ponto como **(em aberto)** — "não discutiu se o `X-Event-Id` original é preservado". Este FDD especifica que **é preservado**, por ser a única leitura compatível com a garantia central do próprio ADR-005: a dedup do cliente só funciona se o identificador for estável entre tentativas. Gerar id novo faria o cliente que deduplica corretamente reprocessar assim mesmo. Se a decisão for revista, é ADR novo, não alteração deste documento.

**O que o replay reenvia:** o snapshot gravado na inserção original ([ADR-007](./adrs/ADR-007-snapshot-do-payload-na-insercao.md)). Um evento reprocessado dias depois entrega o estado do pedido **no momento em que o status mudou**, não o estado atual. Quem opera o replay precisa saber disso.

---

## 6. Contratos públicos

**Prefixo real da API: `/api/v1`** (`src/app.ts:67` — `app.use('/api/v1', buildApiRouter(controllers))`). Os caminhos citados na reunião (`POST /webhooks`, `GET /webhooks/:id/deliveries`, `POST /admin/webhooks/dead-letter/:id/replay`) são relativos a esse prefixo.

**Envelope de erro — idêntico ao do resto da API**, produzido por `errorMiddleware` (`src/middlewares/error.middleware.ts:15-24`) sem nenhuma alteração:

```json
{ "error": { "code": "WEBHOOK_NOT_FOUND", "message": "Webhook not found" } }
```

O campo `details` só aparece quando o erro o carrega (`:20`).

**Autenticação.** Todos os endpoints exigem `Authorization: Bearer <jwt>`. O JWT carrega `{ sub, email, role }` e é verificado em `src/middlewares/auth.middleware.ts:41`. Apenas o replay exige `role = ADMIN` ([ADR-008](./adrs/ADR-008-controle-de-acesso-dos-endpoints.md)).

**Ordem dos middlewares** — segue o precedente real de `src/modules/users/user.routes.ts:12-18`: `authenticate` → `requireRole(...)` → `validate({...})` → handler do controller.

**Convenção de caixa:** as respostas da nossa API usam **camelCase** (como todo o resto do projeto — `orderNumber`, `createdAt`); o corpo do evento entregue ao cliente usa **snake_case** (seção 6.8). É mapeamento deliberado de contrato externo, não inconsistência — ver 6.8.

**Correlação:** toda resposta traz `X-Request-Id`, gerado ou refletido por `requestLogger` (`src/middlewares/request-logger.middleware.ts:6-8`). Nada a implementar no módulo.

Os exemplos abaixo são coerentes entre si: o mesmo cadastro (`3f2b8c14-…`) do mesmo cliente Atlas Comercial (`7c9e6679-…`), o mesmo pedido (`e4b8f2a1-…`) e o mesmo evento (`9b1f4c7a-…`) atravessam todos os endpoints.

---

### 6.1 `POST /api/v1/webhooks` — cadastrar webhook

`[09:31] Marcos:` — "O cliente precisa cadastrar webhook. Endpoint POST. Campos: url, secret é gerada pela gente e devolvida na criação. Lista de status que ele quer receber. Customer_id implícito do JWT." — **a última frase foi revertida na própria reunião**: `[09:32] Bruno:` apontou que o JWT é do usuário operador, não do cliente, e `[09:32] Larissa:` fechou que o `customer_id` vai no body ou no path.

- **Autorização:** autenticado, qualquer role (`ADMIN` ou `OPERATOR`) — [ADR-008](./adrs/ADR-008-controle-de-acesso-dos-endpoints.md).
- **`customerId` vai no body, não vem do JWT** (`[09:32] Larissa:`).

**Schema de request (Zod, `webhook.schemas.ts` a criar):**

```ts
export const createWebhookSchema = z.object({
  customerId: z.string().uuid(),
  url: z.string().url().max(500),
  subscribedStatuses: z.array(z.nativeEnum(OrderStatus)).min(1),
});
```

O esquema `https` **não** é validado aqui: a checagem é feita no service para poder lançar `WEBHOOK_INVALID_URL` (`[09:28] Bruno:`) em vez do `VALIDATION_ERROR` genérico do `validate` — ver a regra de repartição na seção 7.2.

**Request**

```http
POST /api/v1/webhooks HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json
```

```json
{
  "customerId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "url": "https://integracoes.atlascomercial.com.br/hooks/oms/orders",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"]
}
```

**Response `201 Created`** — é a **única** resposta, junto com a de rotação (6.6), que devolve o campo `secret` (`[09:31] Marcos:`):

```json
{
  "id": "3f2b8c14-9d5e-4a71-b0c3-5e8f1a2d6b90",
  "customerId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "url": "https://integracoes.atlascomercial.com.br/hooks/oms/orders",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "4f8a1c93e6b52d70a4c1f8e39b6d0527a1c4e79b3f6d8025c1a4e79b3f6d8025",
  "createdAt": "2026-06-11T13:45:22.318Z",
  "updatedAt": "2026-06-11T13:45:22.318Z"
}
```

> O valor do segredo no exemplo é ilustrativo. **Tamanho e codificação do segredo gerado são (em aberto)** e ficam na revisão de segurança de `[09:46] Sofia:` ("HMAC e geração de secret eu quero olhar com calma").

**Códigos de status**

| Código | Semântica |
| --- | --- |
| `201` | cadastro criado; corpo inclui o segredo, exibido **uma única vez** nesta resposta |
| `400 VALIDATION_ERROR` | corpo fora do schema Zod (campo ausente, `customerId` não-UUID, `subscribedStatuses` vazio) |
| `400 WEBHOOK_INVALID_URL` | URL sintaticamente válida mas com esquema diferente de `https` (`[09:23] Sofia:`) |
| `401 UNAUTHORIZED` | token ausente, malformado ou expirado |
| `404 WEBHOOK_CUSTOMER_NOT_FOUND` | `customerId` não existe em `customers` |
| `500 INTERNAL_SERVER_ERROR` | falha não mapeada; `errorMiddleware:56-64` já loga com `requestId` |

---

### 6.2 `GET /api/v1/webhooks` — listar os webhooks de um cliente

`[09:33] Bruno:` — "GET pra listar os webhooks de um customer".

- **Autorização:** autenticado, qualquer role.
- `customerId` é **query param obrigatório** — não vem do JWT (`[09:32] Larissa:`).
- Paginação no formato do projeto: `paginated(data, page, pageSize, total)` de `src/shared/http/response.ts:22`, com os mesmos limites já praticados em `src/modules/orders/order.schemas.ts:24-25` (`page` default 1, `pageSize` default 20, máximo 100).

**Schema de request**

```ts
export const listWebhooksQuerySchema = z.object({
  customerId: z.string().uuid(),
  page: z.coerce.number().int().min(1).default(1),
  pageSize: z.coerce.number().int().min(1).max(100).default(20),
  active: z.enum(['true', 'false']).transform((v) => v === 'true').optional(),
});
```

**Request**

```http
GET /api/v1/webhooks?customerId=7c9e6679-7425-40de-944b-e07fc1f90ae7&page=1&pageSize=20 HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Response `200 OK`** — **sem o campo `secret`**:

```json
{
  "data": [
    {
      "id": "3f2b8c14-9d5e-4a71-b0c3-5e8f1a2d6b90",
      "customerId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
      "url": "https://integracoes.atlascomercial.com.br/hooks/oms/orders",
      "subscribedStatuses": ["SHIPPED", "DELIVERED"],
      "active": true,
      "secretRotatedAt": null,
      "createdAt": "2026-06-11T13:45:22.318Z",
      "updatedAt": "2026-06-11T13:45:22.318Z"
    },
    {
      "id": "a1d4e2c8-6b39-4f57-9c02-8d7e3f1b4a56",
      "customerId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
      "url": "https://erp.atlascomercial.com.br/webhooks/pedidos",
      "subscribedStatuses": ["PAID", "CANCELLED"],
      "active": false,
      "secretRotatedAt": null,
      "createdAt": "2026-06-11T14:02:10.771Z",
      "updatedAt": "2026-06-12T08:19:44.502Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 2, "totalPages": 1 }
}
```

**Regra de serialização, válida para 6.2, 6.2.1, 6.3, 6.4 e 6.5:** as colunas `secret`, `previousSecret` e `previousSecretExpiresAt` **nunca** são serializadas. A projeção acontece no repository, com `select` explícito — não com `delete obj.secret` no controller, que é a forma que vaza quando alguém acrescenta um caminho novo.

**Códigos de status**

| Código | Semântica |
| --- | --- |
| `200` | lista devolvida; lista vazia é `200` com `data: []`, não `404` |
| `400 VALIDATION_ERROR` | `customerId` ausente ou não-UUID, `pageSize` acima de 100 |
| `400 VALIDATION_ERROR` | `active` com valor diferente de `true`/`false` |
| `401 UNAUTHORIZED` | token ausente ou inválido |

---

### 6.2.1 `GET /api/v1/webhooks/:id` — consultar um cadastro

> **Endpoint não citado na reunião — (proposta do FDD).** `[09:33] Bruno:` listou PATCH, DELETE e GET de listagem. A leitura unitária entra porque 6.3, 6.4 e 6.6 operam por id e o cliente precisa conferir o estado do cadastro sem paginar a listagem.

- **Autorização:** autenticado, qualquer role.

**Request**

```http
GET /api/v1/webhooks/3f2b8c14-9d5e-4a71-b0c3-5e8f1a2d6b90 HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Response `200 OK`** — **sem o campo `secret`**, mesma projeção de 6.2:

```json
{
  "id": "3f2b8c14-9d5e-4a71-b0c3-5e8f1a2d6b90",
  "customerId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "url": "https://integracoes.atlascomercial.com.br/hooks/oms/orders",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secretRotatedAt": null,
  "createdAt": "2026-06-11T13:45:22.318Z",
  "updatedAt": "2026-06-11T13:45:22.318Z"
}
```

**Códigos de status**

| Código | Semântica |
| --- | --- |
| `200` | cadastro devolvido |
| `400 VALIDATION_ERROR` | `id` de path não-UUID |
| `401 UNAUTHORIZED` | token ausente ou inválido |
| `404 WEBHOOK_NOT_FOUND` | cadastro inexistente |

---

### 6.3 `PATCH /api/v1/webhooks/:id` — editar cadastro

`[09:33] Bruno:` — "PATCH pra editar".

- **Autorização:** autenticado, qualquer role.
- Campos editáveis: `url`, `subscribedStatuses`, `active`. **`customerId` não é editável** (mover cadastro entre clientes muda o destinatário dos eventos e não foi discutido). **`secret` não é editável por aqui** — rotação tem endpoint próprio (6.6), e esta resposta **não** devolve segredo (o [ADR-008](./adrs/ADR-008-controle-de-acesso-dos-endpoints.md) registra que a reunião não definiu isso; este FDD fecha em "não devolve").

**Schema de request**

```ts
export const updateWebhookSchema = z.object({
  url: z.string().url().max(500).optional(),
  subscribedStatuses: z.array(z.nativeEnum(OrderStatus)).min(1).optional(),
  active: z.boolean().optional(),
}).refine((v) => Object.keys(v).length > 0, 'at least one field must be provided');
```

**Request**

```http
PATCH /api/v1/webhooks/3f2b8c14-9d5e-4a71-b0c3-5e8f1a2d6b90 HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json
```

```json
{ "subscribedStatuses": ["PAID", "PROCESSING", "SHIPPED", "DELIVERED"] }
```

**Response `200 OK`**

```json
{
  "id": "3f2b8c14-9d5e-4a71-b0c3-5e8f1a2d6b90",
  "customerId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "url": "https://integracoes.atlascomercial.com.br/hooks/oms/orders",
  "subscribedStatuses": ["PAID", "PROCESSING", "SHIPPED", "DELIVERED"],
  "active": true,
  "secretRotatedAt": null,
  "createdAt": "2026-06-11T13:45:22.318Z",
  "updatedAt": "2026-06-12T10:07:31.940Z"
}
```

**Efeito sobre eventos já enfileirados: nenhum.** A filtragem por status acontece na inserção ([ADR-007](./adrs/ADR-007-snapshot-do-payload-na-insercao.md), `[09:34] Bruno:`); alterar `subscribedStatuses` só afeta mudanças de status **futuras**. Desativar (`active: false`) afeta o que já está na fila — o worker manda o evento para a DLQ em vez de enviar (fluxo 5.2.2, passo 3.2).

**Códigos de status**

| Código | Semântica |
| --- | --- |
| `200` | cadastro atualizado |
| `400 VALIDATION_ERROR` | corpo vazio ou fora do schema; `id` de path não-UUID |
| `400 WEBHOOK_INVALID_URL` | nova URL com esquema diferente de `https` |
| `401 UNAUTHORIZED` | token ausente ou inválido |
| `404 WEBHOOK_NOT_FOUND` | cadastro inexistente (`[09:28] Bruno:`) |

---

### 6.4 `DELETE /api/v1/webhooks/:id` — remover cadastro

`[09:33] Bruno:` — "DELETE pra remover".

- **Autorização:** autenticado, qualquer role.
- **Semântica: remoção física.** As FKs de `webhook_outbox`, `webhook_deliveries` e `webhook_dead_letter` são `onDelete: Cascade` (seção 4), então a remoção leva junto os eventos pendentes, o histórico de entregas e as linhas de DLQ daquele cadastro.
- **Quem quer só parar de receber deve usar `PATCH { "active": false }`** (6.3), que preserva o histórico. A distinção precisa estar na documentação do portal do desenvolvedor — ver risco R-6 na seção 13.

**Request**

```http
DELETE /api/v1/webhooks/a1d4e2c8-6b39-4f57-9c02-8d7e3f1b4a56 HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Response `204 No Content`** — sem corpo, no mesmo padrão de `OrderController.delete` (`src/modules/orders/order.controller.ts:51`).

**Códigos de status**

| Código | Semântica |
| --- | --- |
| `204` | cadastro removido, junto com outbox pendente, deliveries e DLQ associados |
| `400 VALIDATION_ERROR` | `id` de path não-UUID |
| `401 UNAUTHORIZED` | token ausente ou inválido |
| `404 WEBHOOK_NOT_FOUND` | cadastro inexistente |

---

### 6.5 `GET /api/v1/webhooks/:id/deliveries` — histórico de entregas

`[09:34] Marcos:` — "esses são os últimos 100 webhooks que vocês mandaram pra mim, sucesso/falha, payload, response, tempo de resposta".

- **Autorização:** autenticado, qualquer role.
- Ordenação `createdAt DESC`, servida pelo índice `@@index([webhookEndpointId, createdAt])`.
- Os "últimos ~100" de `[09:34] Marcos:` são a primeira página no `pageSize` máximo já praticado pelo projeto (100, `src/modules/orders/order.schemas.ts:25`). Não inventamos limite novo.

**Schema de request**

```ts
export const listDeliveriesQuerySchema = z.object({
  page: z.coerce.number().int().min(1).default(1),
  pageSize: z.coerce.number().int().min(1).max(100).default(100),
  success: z.enum(['true', 'false']).transform((v) => v === 'true').optional(),
});
```

**Request**

```http
GET /api/v1/webhooks/3f2b8c14-9d5e-4a71-b0c3-5e8f1a2d6b90/deliveries?pageSize=100 HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Response `200 OK`**

```json
{
  "data": [
    {
      "id": "5d3a9e17-8c42-4b6f-92e0-1a7c4f8b3d65",
      "eventId": "9b1f4c7a-2e58-4d36-a0c9-6f3b8e2d5147",
      "attemptNumber": 2,
      "success": true,
      "responseStatusCode": 200,
      "durationMs": 342,
      "errorMessage": null,
      "requestPayload": {
        "event_id": "9b1f4c7a-2e58-4d36-a0c9-6f3b8e2d5147",
        "event_type": "order.status_changed",
        "timestamp": "2026-06-12T09:14:03.127Z",
        "order_id": "e4b8f2a1-7c63-4d09-8b5e-2f9a1c3d7e60",
        "order_number": "ORD-004821",
        "from_status": "PROCESSING",
        "to_status": "SHIPPED",
        "customer_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
        "total_cents": 1289000
      },
      "responseBody": "{\"received\":true}",
      "createdAt": "2026-06-12T09:15:06.884Z"
    },
    {
      "id": "b8f14c62-0d37-49a5-8e1b-3c6a2f9d047e",
      "eventId": "9b1f4c7a-2e58-4d36-a0c9-6f3b8e2d5147",
      "attemptNumber": 1,
      "success": false,
      "responseStatusCode": 503,
      "durationMs": 1187,
      "errorMessage": "WEBHOOK_DELIVERY_FAILED: unexpected status 503",
      "requestPayload": {
        "event_id": "9b1f4c7a-2e58-4d36-a0c9-6f3b8e2d5147",
        "event_type": "order.status_changed",
        "timestamp": "2026-06-12T09:14:03.127Z",
        "order_id": "e4b8f2a1-7c63-4d09-8b5e-2f9a1c3d7e60",
        "order_number": "ORD-004821",
        "from_status": "PROCESSING",
        "to_status": "SHIPPED",
        "customer_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
        "total_cents": 1289000
      },
      "responseBody": "Service Unavailable",
      "createdAt": "2026-06-12T09:14:05.204Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 100, "total": 2, "totalPages": 1 }
}
```

Note que as duas linhas compartilham o mesmo `eventId` — são a tentativa 1 (falha, `503`) e a tentativa 2 (sucesso, um minuto depois, conforme o primeiro intervalo do backoff). O `X-Event-Id` entregue ao cliente foi o mesmo nas duas.

**Códigos de status**

| Código | Semântica |
| --- | --- |
| `200` | histórico devolvido; cadastro sem entregas retorna `data: []` |
| `400 VALIDATION_ERROR` | `id` de path não-UUID, `pageSize` acima de 100 |
| `400 VALIDATION_ERROR` | `success` com valor diferente de `true`/`false` |
| `401 UNAUTHORIZED` | token ausente ou inválido |
| `404 WEBHOOK_NOT_FOUND` | cadastro inexistente — verificado **antes** de consultar as entregas |

---

### 6.6 `POST /api/v1/webhooks/:id/secret/rotate` — rotacionar segredo

`[09:21] Sofia:` — "a secret tem que ser rotacionável. Endpoint pro cliente conseguir pedir nova secret pela API. Quando ele rotaciona, a antiga fica válida por 24 horas em paralelo".

> **O caminho exato deste endpoint não foi dito na reunião — `/webhooks/:id/secret/rotate` é (proposta do FDD).** A existência do endpoint e o grace period de 24h, esses sim, são decisão fechada ([ADR-004](./adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)).

- **Autorização:** autenticado, qualquer role — **(proposta do FDD)**, por simetria com o CRUD. O [ADR-008](./adrs/ADR-008-controle-de-acesso-dos-endpoints.md) enumera criação, edição, remoção, listagem e deliveries; **rotação não foi discutida na reunião e não está coberta por ele**. Como este endpoint devolve segredo utilizável e invalida o segredo em uso pelo cliente, é candidato natural a `requireRole('ADMIN')`. Levar à revisão de segurança de `[09:46] Sofia:` junto com a geração de segredo.
- **Sem corpo de request.** O segredo é gerado por nós (`[09:31] Marcos:`); o cliente não escolhe valor.

**Efeito no modelo de dados (transação única):**

1. `previousSecret := secret` (o valor atual)
2. `previousSecretExpiresAt := now() + 24h` (`[09:21] Sofia:`)
3. `secret := <novo valor gerado>`
4. `secretRotatedAt := now()`

**Request**

```http
POST /api/v1/webhooks/3f2b8c14-9d5e-4a71-b0c3-5e8f1a2d6b90/secret/rotate HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Response `200 OK`**

```json
{
  "id": "3f2b8c14-9d5e-4a71-b0c3-5e8f1a2d6b90",
  "secret": "b7d2e40f9a13c8562e4f0b9d7a3c165e8f2b4d09a7c3e615b8d0f2a47c9e3b16",
  "secretRotatedAt": "2026-06-13T10:00:00.000Z",
  "previousSecretExpiresAt": "2026-06-14T10:00:00.000Z"
}
```

**Códigos de status**

| Código | Semântica |
| --- | --- |
| `200` | segredo rotacionado; o novo valor é exibido **uma única vez**, aqui |
| `400 VALIDATION_ERROR` | `id` de path não-UUID |
| `401 UNAUTHORIZED` | token ausente ou inválido |
| `404 WEBHOOK_NOT_FOUND` | cadastro inexistente |
| `409 WEBHOOK_SECRET_ROTATION_IN_PROGRESS` | existe `previousSecret` com `previousSecretExpiresAt > now()`. Há **uma** coluna de segredo anterior; rotacionar de novo antes do vencimento descartaria um segredo que o cliente ainda pode estar usando. Passado o vencimento, a varredura de 5.2.1 zera as colunas e a rotação volta a ser aceita. **(proposta do FDD)** — consequência direta do modelo de dados, não regra da reunião |

---

### 6.7 `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — replay manual

`[09:18] Diego:` e `[09:35] Diego:` — caminho literal da reunião. Montagem: `router.use('/admin/webhooks', buildAdminWebhookRouter(...))` em `src/routes/index.ts`, que hoje monta só routers de domínio (`:24-28`). É o primeiro prefixo `admin` do projeto.

- **Autorização:** `authenticate` + `requireRole('ADMIN')` (`src/middlewares/auth.middleware.ts:49`), exatamente o encadeamento de `src/modules/users/user.routes.ts:14-15`. `[09:36] Sofia:` — "Mexer em fila de entrega de notificação não é coisa de operador."
- **Registra quem executou** em `webhook_dead_letter.replayedById` e no log `webhook_dlq_replayed` (`[09:36] Sofia:`).

**Request**

```http
POST /api/v1/admin/webhooks/dead-letter/c6e0b3d9-4f18-42a7-8b5c-0d9e6a1f7238/replay HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Response `202 Accepted`** — reenfileirado, não entregue:

```json
{
  "deadLetterId": "c6e0b3d9-4f18-42a7-8b5c-0d9e6a1f7238",
  "eventId": "9b1f4c7a-2e58-4d36-a0c9-6f3b8e2d5147",
  "webhookEndpointId": "3f2b8c14-9d5e-4a71-b0c3-5e8f1a2d6b90",
  "status": "PENDING",
  "replayedAt": "2026-06-13T11:02:48.006Z",
  "replayedById": "2a7d5f60-3b91-4e28-9c14-8f6b0a3e5d72"
}
```

**Response de erro `403`** (usuário `OPERATOR`), produzida sem uma linha de código novo — `ForbiddenError` (`src/shared/errors/http-errors.ts:21`) serializada por `errorMiddleware:15-24`:

```json
{ "error": { "code": "FORBIDDEN", "message": "Insufficient permissions" } }
```

**Códigos de status**

| Código | Semântica |
| --- | --- |
| `202` | evento recolocado na outbox como `PENDING`; será enviado no próximo ciclo de 2s |
| `400 VALIDATION_ERROR` | `id` de path não-UUID |
| `401 UNAUTHORIZED` | token ausente ou inválido |
| `403 FORBIDDEN` | role `OPERATOR` — [ADR-008](./adrs/ADR-008-controle-de-acesso-dos-endpoints.md) |
| `404 WEBHOOK_DEAD_LETTER_NOT_FOUND` | id de DLQ inexistente |
| `409 WEBHOOK_ALREADY_REPLAYED` | `replayedAt` já preenchido |
| `409 WEBHOOK_ENDPOINT_INACTIVE` | cadastro de destino está com `active = false` |
| `409 CONFLICT` | salvaguarda de unicidade: já existe linha na outbox com o `X-Event-Id` preservado (`P2002`, `errorMiddleware:37-47`) |

---

### 6.8 Contrato de **saída** — o que o consumidor recebe

Este é o contrato que os três clientes B2B implementam do lado deles. Ele é o produto da feature; os endpoints de 6.1 a 6.7 são a administração dele.

#### 6.8.1 Requisição que enviamos

```http
POST /hooks/oms/orders HTTP/1.1
Host: integracoes.atlascomercial.com.br
Content-Type: application/json
X-Event-Id: 9b1f4c7a-2e58-4d36-a0c9-6f3b8e2d5147
X-Webhook-Id: 3f2b8c14-9d5e-4a71-b0c3-5e8f1a2d6b90
X-Timestamp: 2026-06-12T09:14:05.041Z
X-Signature: 5e1c0b9a7d38f42615c8e0b3d97a4f251e8c6b0d3f9a72c48e15b0d6f3a29c47
```

```json
{
  "event_id": "9b1f4c7a-2e58-4d36-a0c9-6f3b8e2d5147",
  "event_type": "order.status_changed",
  "timestamp": "2026-06-12T09:14:03.127Z",
  "order_id": "e4b8f2a1-7c63-4d09-8b5e-2f9a1c3d7e60",
  "order_number": "ORD-004821",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "total_cents": 1289000
}
```

#### 6.8.2 Headers — nome exato e finalidade

| Header | Valor | Finalidade | Fonte |
| --- | --- | --- | --- |
| `Content-Type` | `application/json` | tipo do corpo | `[09:44] Diego:` |
| `X-Event-Id` | UUID da linha da outbox | **deduplicação do lado do cliente**; estável entre todas as reentregas do mesmo evento | `[09:25] Diego:`, `[09:44] Diego:` |
| `X-Signature` | HMAC-SHA256 do corpo, em hex minúsculo **(proposta do FDD)** | cliente recalcula com o segredo dele e compara antes de confiar no evento | `[09:20] Sofia:` (algoritmo), `[09:44] Diego:` (nome do header) |
| `X-Timestamp` | instante **do envio**, ISO 8601 **(formato: proposta do FDD)** | frescor da requisição; ver a ressalva abaixo antes de tratar como defesa contra replay | `[09:44] Diego:` |
| `X-Webhook-Id` | id do cadastro | cliente com vários cadastros descobre qual caiu naquele envio | `[09:44] Sofia:` |

Detalhes que a implementação precisa acertar:

- **A codificação da assinatura é (proposta do FDD).** A reunião fixou o algoritmo (`[09:20] Sofia:`) e o nome do header (`[09:44] Diego:`), mas não disse hex nem base64. Este FDD fecha em **hex minúsculo, 64 caracteres, sem prefixo de esquema** — saída direta de `createHmac('sha256', secret).update(body).digest('hex')`. É contrato externo: mudar depois quebra os três clientes. Confirmar na revisão de segurança de `[09:46] Sofia:`.
- **`X-Timestamp` muda a cada tentativa; `timestamp` do corpo não.** O header é o instante do envio; o campo do corpo é o instante da mudança de status, congelado no snapshot ([ADR-007](./adrs/ADR-007-snapshot-do-payload-na-insercao.md)). Uma reentrega duas horas depois — ou um replay dias depois — carrega `X-Timestamp` de agora e `timestamp` de quando o status mudou. Formato ISO 8601 para o header é **(proposta do FDD)**, por coerência com o corpo (`[09:43] Diego:`); a reunião não fixou o formato do header.
- **A assinatura cobre o corpo, não os headers — e isso limita o `X-Timestamp`.** Quem captura a requisição íntegra pode reenviá-la com assinatura ainda válida, e pode **reescrever o `X-Timestamp`** para o instante atual sem invalidar nada, porque o header está fora do escopo assinado. Logo, a janela de frescor do lado do cliente barra reenvio ingênuo (o mesmo request repetido tal e qual), não um atacante ativo. A defesa real contra reprocessamento é o `X-Event-Id` ([ADR-005](./adrs/ADR-005-at-least-once-com-x-event-id.md)), que o cliente deduplica de qualquer forma. Levar ao `[09:46] Sofia:` a opção de incluir o timestamp no escopo assinado — está registrada como gatilho de reabertura no [ADR-004](./adrs/ADR-004-hmac-sha256-secret-por-endpoint.md).
- **Nenhum header de correlação nosso vai no request de saída.** O conjunto acima é fechado. A correlação com a requisição HTTP original é interna, pela coluna `webhook_outbox.requestId` (seção 9.3) — não expomos `X-Request-Id` ao cliente.

#### 6.8.3 Corpo do evento — campo a campo

| Campo | Tipo | Origem no nosso modelo |
| --- | --- | --- |
| `event_id` | UUID | `webhook_outbox.id` (mesmo valor de `X-Event-Id`) |
| `event_type` | string | literal `"order.status_changed"` (`[09:43] Diego:`) |
| `timestamp` | ISO 8601 | instante da mudança de status |
| `order_id` | UUID | `orders.id` |
| `order_number` | string | `orders.orderNumber`, formato `ORD-` + 6 dígitos (`src/modules/orders/order.service.ts:253`) |
| `from_status` | `OrderStatus` | status anterior |
| `to_status` | `OrderStatus` | status novo |
| `customer_id` | UUID | `orders.customerId` |
| `total_cents` | inteiro | `orders.totalCents` |

**Não enviamos `items`** — "Não manda items pra não inflar. Se o cliente quiser detalhes, ele bate no GET /orders/:id depois" (`[09:43] Diego:`).

**Sobre "campos básicos da order tipo total_cents"** (`[09:43] Diego:`): a reunião citou `total_cents` como exemplo. Este FDD **fecha o conjunto em `total_cents`**. Acrescentar `subtotal_cents`, `discount_cents` ou qualquer outro campo é mudança de contrato externo e exige decisão nova — não é escolha do PR de implementação.

**Mapeamento de caixa é deliberado.** As colunas do MySQL são camelCase (`migration.sql:52-61`: `orderNumber`, `customerId`, `totalCents`) e o payload é snake_case. A tradução acontece um único lugar: na renderização do snapshot dentro de `publishWebhookEvent` (fluxo 5.1, passo 6). Isso isola o contrato externo do nome das nossas colunas — renomear uma coluna não vaza para o cliente.

**Valores possíveis de `to_status`.** A máquina de estados (`src/modules/orders/order.status.ts:3-10`) não tem nenhuma transição que chegue em `PENDING`, e a criação de pedido não passa por `changeStatus`. Logo, `to_status` só assume `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED` ou `CANCELLED`. Um cadastro que inclua `PENDING` em `subscribedStatuses` é aceito pelo schema e simplesmente nunca dispara.

#### 6.8.4 O que esperamos de volta

| Resposta do cliente | Nossa interpretação | Efeito |
| --- | --- | --- |
| **`2xx`** (qualquer) | **sucesso** | linha vai para `DELIVERED`; nenhuma reentrega |
| **`3xx`, `4xx`, `5xx`** | **falha retentável** | `WEBHOOK_DELIVERY_FAILED`; agenda a próxima tentativa pelo backoff |
| **Sem resposta em 10 segundos** | **falha retentável** | `WEBHOOK_DELIVERY_TIMEOUT` (`[09:42] Diego:`) |
| **Erro de conexão, DNS ou TLS** | **falha retentável** | `WEBHOOK_DELIVERY_FAILED` |
| **Falha após esgotar a progressão de backoff** | **falha definitiva** | evento sai da outbox e vai para `webhook_dead_letter`; só volta por replay manual (6.7) |

Corolários que o cliente precisa saber, e que o portal do desenvolvedor tem de destacar (`[09:26] Marcos:`):

- **O corpo da resposta dele é ignorado** para efeito de decisão. É persistido em `webhook_deliveries.responseBody` para diagnóstico, nada mais.
- **Um `4xx` não interrompe as reentregas.** Rejeitar um evento com `400` não faz o evento parar de chegar; ele volta segundo a progressão até esgotar.
- **Duplicidade é comportamento previsto, não incidente** ([ADR-005](./adrs/ADR-005-at-least-once-com-x-event-id.md)). Cliente que não deduplica por `X-Event-Id` vai processar o mesmo evento mais de uma vez.
- **Responder rápido importa.** O cliente deve confirmar com `2xx` e processar em background: 10 segundos é o teto e ele é contado como falha (`[09:42] Diego:`).

---

## 7. Matriz de erros

`[09:29] Larissa:` — "Prefixo `WEBHOOK_` pra tudo do módulo." Os três exemplos nominais vieram de `[09:28] Bruno:` (`WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`); os dois primeiros abrem a matriz, e o terceiro **não entra nela** — ver 7.1.1.

**Regra desta seção:** toda linha da matriz precisa ser alcançável por um passo numerado das seções 5.1 a 5.5 ou por um endpoint da seção 6. Código sem caminho de ocorrência é código que ninguém consegue testar e que induz tratamento defensivo inútil no cliente.

Convenção do projeto seguida sem variação: `SCREAMING_SNAKE_CASE`, classes herdando de `AppError` (`src/shared/errors/app-error.ts:3`), no molde de `InvalidStatusTransitionError` e `InsufficientStockError` (`src/shared/errors/http-errors.ts:45-63`).

### 7.1 Matriz

| Código | HTTP | Onde ocorre | Quando ocorre | Mensagem |
| --- | --- | --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | `404` | API | `GET`/`PATCH`/`DELETE /webhooks/:id`, `GET /webhooks/:id/deliveries` ou `POST /webhooks/:id/secret/rotate` com id que não existe em `webhook_endpoints` | `Webhook not found` |
| `WEBHOOK_INVALID_URL` | `400` | API | `POST /webhooks` ou `PATCH /webhooks/:id` com URL cujo esquema não é `https` (`[09:23] Sofia:`) | `Webhook url must use https` |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | `404` | API | `POST /webhooks` com `customerId` inexistente em `customers` | `Customer not found for webhook` |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | `422` | API, dentro de `PATCH /orders/:id/status` | o snapshot serializado passa de 64 KB na inserção da outbox (`[09:24] Diego:`, `[09:24] Larissa:`). **Provoca rollback da mudança de status** (fluxo 5.1, passo 7) | `Webhook event payload exceeds the 64KB limit` |
| `WEBHOOK_ENDPOINT_INACTIVE` | `409` | API (replay) e **worker** | replay para cadastro com `active = false` (fluxo 5.5, passo 6); ou o worker encontra o cadastro desativado na hora de enviar e manda o evento para a DLQ (fluxo 5.2.2, passo 3.2) | `Webhook endpoint is inactive` |
| `WEBHOOK_SECRET_ROTATION_IN_PROGRESS` | `409` | API | `POST /webhooks/:id/secret/rotate` enquanto o grace period de 24h do segredo anterior ainda está vigente | `A secret rotation is already in progress for this webhook` |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | `404` | API | `POST /admin/webhooks/dead-letter/:id/replay` com id inexistente em `webhook_dead_letter` | `Dead letter entry not found` |
| `WEBHOOK_ALREADY_REPLAYED` | `409` | API | replay de uma linha de DLQ com `replayedAt` já preenchido | `Dead letter entry has already been replayed` |
| `WEBHOOK_DELIVERY_TIMEOUT` | `504` **(proposta do FDD)** — declarado, não trafega | **worker** | a chamada HTTP não respondeu em 10 segundos (`[09:42] Diego:`) | `Webhook delivery timed out after 10000ms` |
| `WEBHOOK_DELIVERY_FAILED` | `502` **(proposta do FDD)** — declarado, não trafega | **worker** | resposta fora de `2xx`, ou erro de conexão, DNS ou TLS | `Webhook delivery failed` |

**Erros do worker e o `statusCode` que ninguém lê.** As **duas** linhas marcadas como "declarado, não trafega" ocorrem fora do ciclo request/response: não há `res` para responder, e por isso os valores `504` e `502` são escolha deste FDD, não da reunião — qualquer outro número funcionaria igual. O `statusCode` existe só porque o construtor de `AppError` o exige (`src/shared/errors/app-error.ts:8`) — é exatamente a consequência negativa que o [ADR-006](./adrs/ADR-006-reuso-dos-padroes-do-projeto.md) registrou ao decidir reusar `AppError` fora do HTTP. O valor real desses erros é o `errorCode`, que vai para `webhook_deliveries.errorMessage`, `webhook_outbox.lastError`, `webhook_dead_letter.failureReason` e para o label `reason` das métricas da seção 9.1.

### 7.1.1 Código descartado — `WEBHOOK_SECRET_REQUIRED`

`[09:28] Bruno:` citou `WEBHOOK_SECRET_REQUIRED` como um dos exemplos de código do módulo. **Ele não entra na matriz, porque não tem caminho de ocorrência.** A verificação é direta:

| Onde poderia ocorrer | Por que não ocorre |
| --- | --- |
| No cadastro (`POST /webhooks`, 6.1) | o cliente **nunca envia** segredo: "secret é gerada pela gente e devolvida na criação" (`[09:31] Marcos:`). O campo não existe no `createWebhookSchema`, então não há como faltar |
| Na rotação (`POST /webhooks/:id/secret/rotate`, 6.6) | o endpoint **não tem corpo de request** — o valor novo é gerado por nós. Nada pode ser omitido |
| No worker, ao assinar (5.2.2, passo 3.4) | `WebhookEndpoint.secret` é `VARCHAR(255)` **NOT NULL** (4.1) e é preenchido na criação. Um cadastro carregado sempre tem segredo utilizável; a coluna anulável é `previousSecret`, que é opcional por desenho |

Duas falas da reunião se contradizem aqui, e a segunda é a que fecha: o código do `[09:28] Bruno:` foi listado em bloco, três minutos antes de `[09:31] Marcos:` decidir que a plataforma gera o segredo. **A identificação dessa contradição é análise deste FDD; ela não foi percebida na reunião.**

**Quando o código voltaria a fazer sentido:** se a rotação passar a aceitar segredo informado pelo cliente — hipótese não decidida, que teria de sair da revisão de `[09:46] Sofia:`. Nesse caso `WEBHOOK_SECRET_REQUIRED` volta como erro `400` da API, no corpo de 6.6, e não como erro de worker. Enquanto isso não acontecer, declarar a classe seria criar código morto que nenhum teste alcança.

**Consequência para o worker:** não existe caminho "sem segredo utilizável" no passo de assinatura, e por isso ele **não** aparece entre as falhas retentáveis de 8.3. Se o segredo estiver ausente em produção, é corrupção de dado — cai no `webhook_worker_cycle_error` genérico (8.5), não em um código de erro previsto.

### 7.2 Repartição entre `VALIDATION_ERROR` e `WEBHOOK_*`

Há uma tensão real entre duas falas: `[09:23] Sofia:` disse que a exigência de `https` "nem é decisão arquitetural, é só uma validação no schema Zod", e `[09:28] Bruno:` listou `WEBHOOK_INVALID_URL` como código do módulo. O `validate` existente converte todo `ZodError` em `ValidationError` com código fixo `VALIDATION_ERROR` (`src/middlewares/validate.middleware.ts:26-32`), então não é possível ter as duas coisas no mesmo lugar. A regra:

| Natureza do erro | Onde é detectado | Código |
| --- | --- | --- |
| **Forma** — campo ausente, tipo errado, `id` não-UUID, `pageSize` fora do intervalo, URL sintaticamente inválida | `validate({...})` no router | `VALIDATION_ERROR` (com `details: [{ path, message }]`) |
| **Regra do módulo** — esquema não-`https`, cliente inexistente, cadastro inativo, rotação em curso | service do módulo | `WEBHOOK_*` |

O prefixo `WEBHOOK_` de `[09:29] Larissa:` vale para os erros **do módulo**; os erros de infraestrutura compartilhada continuam com os códigos do projeto.

### 7.3 Erros do projeto reaproveitados sem alteração

| Código | HTTP | Origem | Alcançado por |
| --- | --- | --- | --- |
| `VALIDATION_ERROR` | `400` | `ValidationError` (`http-errors.ts:9`) via `validate` | qualquer endpoint da seção 6 com corpo, query ou param fora do schema |
| `UNAUTHORIZED` | `401` | `UnauthorizedError` (`http-errors.ts:15`) via `authenticate` | todos os endpoints da seção 6 |
| `FORBIDDEN` | `403` | `ForbiddenError` (`http-errors.ts:21`) via `requireRole('ADMIN')` | apenas `POST /admin/webhooks/dead-letter/:id/replay` |
| `CONFLICT` | `409` | `Prisma` `P2002` tratado em `error.middleware.ts:38-47` | salvaguarda de unicidade no replay (`@@unique([eventId])` / PK da outbox) |
| `NOT_FOUND` | `404` | `error.middleware.ts:48-53` (`P2025`) e handler de rota (`src/app.ts:69-71`) | rota inexistente sob `/api/v1` |
| `INTERNAL_SERVER_ERROR` | `500` | fallback de `error.middleware.ts:56-64` | falha não mapeada; já loga `err`, `requestId`, `method`, `path` |

### 7.4 Classes a criar — `src/modules/webhooks/webhook.errors.ts` **(a criar)**

```ts
export class WebhookNotFoundError extends AppError {
  constructor() { super('Webhook not found', 404, 'WEBHOOK_NOT_FOUND'); }
}
export class WebhookInvalidUrlError extends BadRequestError {
  constructor() { super('Webhook url must use https', 'WEBHOOK_INVALID_URL'); }
}
export class WebhookEndpointInactiveError extends ConflictError {
  constructor(id: string) { super('Webhook endpoint is inactive', 'WEBHOOK_ENDPOINT_INACTIVE', { webhookEndpointId: id }); }
}
```

**Detalhe que economiza uma hora de quem for codar:** `NotFoundError` **não pode ser reaproveitada** para `WEBHOOK_NOT_FOUND` nem para `WEBHOOK_CUSTOMER_NOT_FOUND`. O construtor dela fixa o código em `'NOT_FOUND'` (`src/shared/errors/http-errors.ts:27-31`) e não aceita override. As duas herdam direto de `AppError` com `404`. Já `BadRequestError` (`:3-7`), `ConflictError` (`:33-37`) e `UnprocessableEntityError` (`:39-43`) **aceitam** código customizado no segundo parâmetro e são as bases corretas para o restante da matriz — mesmo mecanismo que `InvalidStatusTransitionError` usa em `:45-53`.

**Consistência da matriz (checagem de fechamento):** todo código da tabela 7.1 é lançado por um passo numerado das seções 5.1 a 5.5 ou por um endpoint da seção 6, e todo erro citado naqueles fluxos aparece na tabela. As duas listas foram conferidas nos dois sentidos.

---

## 8. Estratégias de resiliência

### 8.1 Timeouts

| O quê | Valor | Onde | Fonte |
| --- | --- | --- | --- |
| Chamada HTTP ao endpoint do cliente | **10 segundos** | `AbortSignal.timeout(10_000)` no `fetch` do worker | `[09:42] Diego:` — "Cliente lento que não responde em 10s a gente trata como falha e marca pra retry" |
| Ciclo de polling | **2 segundos** entre ciclos | laço do worker | `[09:09] Diego:` |

**O que explicitamente não tem timeout novo:** as transações do Prisma seguem o default do client configurado em `src/config/database.ts:5-7`, que não define `transactionOptions`. Este FDD não altera isso — mas a transação de `changeStatus` fica mais longa (fluxo 5.1) e o comportamento sob carga é o risco R-1 da seção 13.

#### 8.1.1 O orçamento de latência não fecha no limite — aritmética

> **Análise deste FDD, não conclusão da reunião.** Os três números da tabela acima e o alvo de produto foram fixados separadamente e nunca foram somados na reunião.

| Parcela | Valor | Fonte |
| --- | --- | --- |
| Alvo de notificação ponta a ponta | **< 10 s** | `[09:02] Marcos:` (PRD-RNF-01) |
| Espera até o worker pegar a linha | até **2 s** | `[09:09] Diego:` (PRD-RNF-02) |
| Espera pela resposta do destino | até **10 s** antes de virar falha | `[09:42] Diego:` (PRD-RNF-03) |

O timeout sozinho iguala o orçamento inteiro. Um destino que responde `200` em 9 segundos entrega em **≈ 11 segundos** ponta a ponta (2 s de polling + 9 s de resposta) — **acima do alvo, sem que nada tenha falhado e sem violar nenhuma decisão do pacote**. O caso não é patológico: é o limite que o próprio desenho admite.

Reduzir `WEBHOOK_HTTP_TIMEOUT_MS` não resolve — empurraria destinos legitimamente lentos para o backoff da 8.2, onde o alvo é perdido por margem muito maior. É trade-off, não ajuste, e mexeria em `[09:42] Diego:`.

**Consequência para a implementação:** o compromisso de latência **não pode ser verificado em valor absoluto**, apenas em percentil sobre a primeira tentativa bem-sucedida — que é exatamente como o [PRD OBJ-1](./PRD.md) o formula (95% abaixo de 10 s). É por isso que `webhook_event_lag_ms` é **histograma e não gauge** (9.1) e que `webhook_delivery_duration_ms` é rotulada por `webhookEndpointId`: p95 por endpoint próximo do timeout é conversa com aquele cliente, não defeito nosso.

### 8.2 Política de retry — progressão exata

Progressão fechada em `[09:17] Diego:` e reafirmada em `[09:48] Larissa:`: **1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas**.

| Chamada HTTP | Acontece em (contado a partir da falha da entrega inicial) | Após a falha, espera |
| --- | --- | --- |
| envio inicial | até 2s depois do commit de `changeStatus` | 1 min |
| tentativa 1 | +1 min | 5 min |
| tentativa 2 | +6 min | 30 min |
| tentativa 3 | +36 min | 2 h |
| tentativa 4 | +2 h 36 min | 12 h |
| tentativa 5 | +14 h 36 min | — falha definitiva, vai para a DLQ |

> **Nota de leitura dos números.** São **5 tentativas de entrega**, conforme a Decisão do [ADR-003](./adrs/ADR-003-retry-com-backoff-e-dlq.md) ("no máximo 5 tentativas de entrega por evento"), premissa fechada da seção 1. A contagem que `[09:17] Diego:` usa é explícita — "entre primeira falha e última tentativa" —, e a primeira falha é a da **entrega inicial**, em `t = 0`. Os cinco intervalos vêm depois dela, cada um antes de uma tentativa: mapeamento 1:1, nenhum intervalo sem uso. A última tentativa cai em **+14 h 36 min**, que são as "quase 15 horas" anunciadas por Diego e a mesma faixa de "até 12 ou 24 horas" de `[09:15] Diego:`. É por isso que `BACKOFF_INTERVALS_MS` tem os cinco intervalos e todos são percorridos.

**Implementação:** constante `BACKOFF_INTERVALS_MS = [60_000, 300_000, 1_800_000, 7_200_000, 43_200_000]` em `src/modules/webhooks/webhook.backoff.ts` **(a criar)**. O índice é `attemptNumber - 1`, com `attemptNumber` = número da chamada HTTP que falhou, contado a partir de 1 — o mesmo valor gravado em `webhook_deliveries.attemptNumber` e em `webhook_outbox.attemptCount` após o update. Nunca indexar pelo `attemptCount` lido antes do incremento. Progressão em tabela, não em fórmula: os intervalos da reunião não seguem uma razão constante (×5, ×6, ×4, ×6) e derivá-los de uma fórmula introduziria números que ninguém decidiu.

**Sem jitter.** A reunião não mencionou aleatorização e o volume (três clientes B2B, `[09:00] Marcos:`) não produz thundering herd. Acrescentar jitter seria inventar comportamento.

### 8.3 Falha retentável versus permanente

| Classe | O que é | Efeito |
| --- | --- | --- |
| **Retentável** | resposta fora de `2xx`, timeout de 10s, erro de rede/DNS/TLS | `status = FAILED`, `nextAttemptAt` agendado, volta para a fila |
| **Permanente** | **exclusivamente** o esgotamento da progressão de backoff | sai da outbox, entra em `webhook_dead_letter` |
| **Descarte com evidência** | cadastro desativado entre a inserção e o envio | vai direto para a DLQ com `failureReason = WEBHOOK_ENDPOINT_INACTIVE`, sem consumir tentativa |

**Não existe falha permanente por status HTTP.** Um `410 Gone` ou `404` do cliente é tratado igual a um `503`. É consequência direta de a reunião ter definido o teto de tentativas como único critério (`[09:15] Diego:`); criar uma classe de falha por status seria decisão nova (exclusão declarada em 3.2).

### 8.4 Comportamento na falha definitiva

O preço da progressão decidida: entre a falha da entrega inicial e a entrada na DLQ passam-se **~14 h 36 min** (8.2). Até lá a falha só existe no log e no histórico de entregas — um evento pode levar quase 15 horas para ser declarado perdido. Foi o custo aceito conscientemente em `[09:17] Marcos:` — "Se um cliente meu cair por 15 horas, ele já tá com problema sério dele. Acho aceitável."

1. O evento **não some**: `webhook_dead_letter` guarda payload, motivo e timestamp (`[09:18] Diego:`).
2. **Ninguém é avisado.** Sem e-mail (`[09:37] Larissa:`), sem alerta automático. O sinal é a métrica `webhook_dead_letter_total` e o log `webhook_event_dead_lettered` (seção 9).
3. A recuperação é **manual e unitária**, via `POST /admin/webhooks/dead-letter/:id/replay` com role `ADMIN` (6.7).
4. **Nada esvazia a DLQ sozinho.** Se ninguém olhar, o evento nunca chega — consequência assumida no [ADR-003](./adrs/ADR-003-retry-com-backoff-e-dlq.md).

### 8.5 O que acontece se o próprio worker cair

| Situação | O que acontece | Como o sistema se recupera |
| --- | --- | --- |
| Worker morre **entre reservar e enviar** | linha presa em `PROCESSING`; nada foi enviado | bootstrap devolve para `PENDING` (5.2.1, passo 4) e envia normalmente |
| Worker morre **entre enviar e marcar** | cliente **recebeu** o evento; nosso lado não sabe | bootstrap devolve para `PENDING`; o evento é **reenviado com o mesmo `X-Event-Id`**. É a origem concreta da duplicidade que a garantia at-least-once assume ([ADR-005](./adrs/ADR-005-at-least-once-com-x-event-id.md)) |
| Worker fica **fora do ar por horas** | a API continua aceitando mudanças de status e inserindo na outbox; **nada falha e nenhum cliente recebe nada** | ao voltar, o worker drena a fila em ordem de `createdAt`. Eventos antigos são entregues com o snapshot de quando o status mudou |
| Banco indisponível **durante o ciclo** | a consulta do lote ou a transação de resultado falha | o erro é capturado no laço, logado como `webhook_worker_cycle_error` (nível `error`), e o worker dorme os 2 segundos e tenta de novo. **O processo não morre por erro de ciclo** — morrer transformaria indisponibilidade momentânea do banco em parada total da entrega |
| Worker **não é reiniciado** | acúmulo silencioso; nenhuma requisição HTTP falha | é o ponto único de falha registrado no [ADR-002](./adrs/ADR-002-worker-separado-com-polling.md). A detecção depende de supervisão de processo e do gauge `webhook_outbox_pending_total` (9.1) — não há mecanismo interno que perceba a própria ausência |

**Deploy do worker é `SIGTERM` limpo:** o handler para de aceitar novo lote, aguarda o item em voo terminar (no máximo os 10s do timeout), fecha o Prisma e sai — mesma estrutura de `src/server.ts:13-21`. Item interrompido à força cai no caso 5.2.3.

---

## 9. Observabilidade

O projeto **não tem cliente de métricas nem SDK de tracing** — `package.json` não traz `prom-client`, OpenTelemetry ou equivalente, e a decisão de não acrescentar dependência vale aqui (`[09:29] Bruno:` — "o logger, que é Pino, já tá no projeto inteiro. Não vamos botar nada novo"). Portanto: **as métricas são emitidas como eventos estruturados do Pino**, com o nome no campo `metric`, e agregadas na stack de logs. Os nomes e labels abaixo ficam fixados desde já, de modo que a troca por um exporter dedicado no futuro seja substituição de emissor, não redesenho.

### 9.1 Métricas

| Nome | Tipo | Labels | Pergunta operacional que responde |
| --- | --- | --- | --- |
| `webhook_outbox_pending_total` | gauge | — | **A fila está drenando?** Emitida uma vez por ciclo de polling. Crescimento monotônico é o único sintoma de worker morto ([ADR-002](./adrs/ADR-002-worker-separado-com-polling.md)) ou de cliente derrubando a vazão |
| `webhook_event_lag_ms` | histograma | — | **Estamos dentro dos 10 segundos prometidos?** Mede `createdAt` da linha até a primeira chamada HTTP. É a métrica que valida o requisito de `[09:02] Marcos:` e a aposta de `[09:09] Diego:` no polling de 2s. **Histograma e não gauge por causa de 8.1.1:** o compromisso só é verificável em percentil |
| `webhook_delivery_attempts_total` | contador | `outcome` (`success`/`failure`), `webhookEndpointId`, `httpStatus` | **Qual cliente está falhando e com que resposta?** Separa "nosso problema" de "problema do cliente" sem abrir o banco |
| `webhook_delivery_duration_ms` | histograma | `webhookEndpointId` | **Quem está perto de estourar o timeout de 10s?** (`[09:42] Diego:`) Identifica o cliente lento antes de ele virar falha |
| `webhook_dead_letter_total` | contador | `webhookEndpointId`, `reason` | **De quem desistimos de entregar, e por quê?** É o substituto operacional do alerta por e-mail que ficou para a próxima fase (`[09:37] Larissa:`) |
| `webhook_outbox_reclaimed_total` | contador | — | **O worker está morrendo no meio de entregas?** Valor diferente de zero no bootstrap significa que houve crash com entrega em voo — e portanto duplicata provável no cliente |
| `webhook_retry_scheduled_total` | contador | `attemptNumber` | **A progressão de backoff está sendo percorrida ou os eventos morrem cedo?** A distribuição por `attemptNumber` mostra se as reentregas resolvem ou se estão só adiando a DLQ |

Nenhuma métrica de "total de webhooks cadastrados" ou similar: ninguém vai olhar, e a informação sai de um `SELECT COUNT(*)` quando for preciso.

**Label que deliberadamente não existe:** `customerId`. `webhookEndpointId` já identifica o destino e não replica identificador de cliente na stack de métricas.

### 9.2 Logs

Todos via Pino (`src/shared/logger/index.ts:32`), com `base: { service: 'order-management-api', env }` e `timestamp: isoTime` (`:20-21`). O worker usa `logger.child({ component: 'webhook-worker' })` — é o que separa as linhas dos dois processos, que compartilham o mesmo `service`.

| Evento | Nível | Processo | Campos |
| --- | --- | --- | --- |
| `webhook_event_enqueued` | `info` | API | `requestId`, `eventIds[]`, `orderId`, `customerId`, `fromStatus`, `toStatus`, `endpointCount` |
| `webhook_worker_started` | `info` | worker | `pollIntervalMs`, `batchSize` |
| `webhook_outbox_reclaimed` | `warn` | worker | `reclaimedCount` |
| `webhook_poll_cycle` | `debug` | worker | `fetched`, `pendingTotal`, `durationMs` |
| `webhook_delivery_attempt` | `info` | worker | `requestId`, `eventId`, `webhookEndpointId`, `attemptNumber`, `httpStatus`, `durationMs`, `outcome` |
| `webhook_delivery_failed` | `warn` | worker | `requestId`, `eventId`, `webhookEndpointId`, `attemptNumber`, `errorCode`, `nextAttemptAt` |
| `webhook_event_dead_lettered` | `error` | worker | `requestId`, `eventId`, `deadLetterId`, `webhookEndpointId`, `attemptCount`, `failureReason` |
| `webhook_dlq_replayed` | `info` | API | `requestId`, `userId`, `deadLetterId`, `eventId`, `webhookEndpointId` — é a auditoria pedida em `[09:36] Sofia:` |
| `webhook_secret_rotated` | `info` | API | `requestId`, `userId`, `webhookEndpointId`, `previousSecretExpiresAt` |
| `webhook_secret_expired` | `info` | worker | `expiredCount` |
| `webhook_worker_cycle_error` | `error` | worker | `err`, `durationMs` |
| `webhook_worker_shutdown` | `info` | worker | `signal`, `inFlight` |

Os nomes em snake_case seguem o padrão já usado em `src/server.ts:10,14-15` (`server_started`, `shutdown_initiated`, `http_server_closed`) e em `request-logger.middleware.ts:23` (`http_request`).

#### O que **não** pode ser logado

| Nunca aparece em log | Por quê |
| --- | --- |
| `secret` e `previousSecret` | é material secreto recuperável; Diego relatou caso real de cliente que "vazou secret em log de aplicação dele" (`[09:22] Diego:`) |
| valor de `X-Signature` | assinatura válida em log é insumo de forja/replay |
| objeto de configuração de webhook inteiro | logar o registro completo vaza o segredo por tabela; logar sempre campos selecionados |
| `Authorization` do request de entrada | já coberto por `redact` (`src/shared/logger/index.ts:5`) |
| corpo da resposta do cliente | é dado de sistema de terceiro; fica só em `webhook_deliveries.responseBody` |

**Alteração obrigatória em arquivo existente.** `redactPaths` (`src/shared/logger/index.ts:4-11`) cobre hoje `req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.passwordHash`, `*.token` e `*.accessToken` — **nenhum caminho de segredo de webhook**. Acrescentar:

```ts
'*.secret',
'*.previousSecret',
'*.signature',
```

O [ADR-006](./adrs/ADR-006-reuso-dos-padroes-do-projeto.md) já registrou este gatilho ("no momento em que a secret circular dentro de um objeto entregue ao logger, a lista `redactPaths` precisa ganhar uma entrada"), e o [ADR-004](./adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) o listou como consequência negativa. Não contraria o "nada novo" de `[09:29] Bruno:` — é configuração, não dependência. O `censor` continua `'[REDACTED]'` (`:18`).

`redact` é rede de segurança, não estratégia: o código não entrega objeto com segredo ao logger em primeiro lugar.

### 9.3 Tracing

Não há SDK de tracing no projeto, e a feature não introduz um. A correlação é feita por **campo propagado**, e a infraestrutura para isso **já existe**: `requestLogger` (`src/middlewares/request-logger.middleware.ts:5-28`) lê `x-request-id` da entrada ou gera um `uuidv4`, grava em `req.id` (`:7`), devolve em `X-Request-Id` (`:8`) e o inclui em `http_request` no evento `finish` (`:12-24`).

**Cadeia completa de correlação, de ponta a ponta:**

```
Cliente HTTP
  └─ x-request-id (ou uuidv4 gerado em request-logger.middleware.ts:6)
       └─ req.id
            └─ OrderController.changeStatus passa req.id ao service
                 └─ publishWebhookEvent grava em webhook_outbox.requestId  ← elo que atravessa os processos
                      └─ worker lê a linha e injeta requestId em todo log da entrega
                           └─ webhook_delivery_attempt / _failed / _dead_lettered
                                └─ replay copia requestId da DLQ para a nova linha da outbox
```

Com isso, `grep requestId=<valor>` devolve, numa única busca: a requisição `PATCH /orders/:id/status` original, o enfileiramento do evento, cada uma das até 5 tentativas de entrega ao longo de ~14h36min (8.2), a eventual entrada na DLQ e o replay — mesmo tendo tudo isso acontecido em **dois processos diferentes** e com horas de distância.

**Limites dos spans:**

| Span | Começa | Termina |
| --- | --- | --- |
| Span HTTP (já existe) | `requestLogger` atribui `req.id` (`:6-7`) | evento `finish` de `res` (`:12`), que emite `http_request` |
| Span do evento (novo, lógico) | inserção da linha na `webhook_outbox`, dentro da transação (fluxo 5.1, passo 8) | a linha sai da outbox: `DELIVERED` ou movida para a DLQ |
| Span de tentativa (novo, lógico) | reserva da linha em `PROCESSING` (fluxo 5.2.2, passo 3.1) | gravação de `webhook_deliveries`, com `durationMs` medido por `process.hrtime.bigint()` |

**O span do evento não é enviado ao cliente.** Nenhum header de correlação nosso viaja no request de saída (seção 6.8.2); o `requestId` é interno.

**Duas alterações de assinatura sustentam a cadeia** — são extensões deste FDD, não da reunião:

1. `OrderService.changeStatus(id, input, userId)` → `changeStatus(id, input, userId, requestId?)`. Parâmetro **opcional**, para não quebrar chamada existente nem teste.
2. `publishWebhookEvent(tx, order, fromStatus, toStatus)` — assinatura proposta em `[09:41] Bruno:` — ganha um quinto parâmetro `requestId?`. Sem ele, a correlação morre na fronteira do processo e o `webhook_outbox.requestId` fica nulo.

`OrderController.changeStatus` (`src/modules/orders/order.controller.ts:41`) passa a repassar `req.id`.

---

## 10. Integração com o sistema existente

`[09:30] Larissa:` — "reuso máximo do que já existe. AppError, Pino, error middleware, padrão de módulos, padrão de schemas Zod, padrão de códigos de erro. Webhook fica como módulo igual aos outros." Esta seção transforma essa frase em lista de arquivos.

### 10.1 Tabela mestra — existente versus a criar

| Arquivo | Situação | O que acontece com ele |
| --- | --- | --- |
| `src/modules/orders/order.service.ts` | **EXISTENTE — ALTERADO** | `changeStatus:126-179` ganha a chamada de publicação entre `:167` e `:169`; assinatura ganha `requestId?` |
| `src/modules/orders/order.controller.ts` | **EXISTENTE — ALTERADO** | `changeStatus:38-46` passa `req.id` ao service (`:41`) |
| `prisma/schema.prisma` | **EXISTENTE — ALTERADO** | 4 models e 1 enum novos; relações inversas em `Customer` e `Order` |
| `src/config/env.ts` | **EXISTENTE — ALTERADO** | novas variáveis no `envSchema:3-10` |
| `src/shared/logger/index.ts` | **EXISTENTE — ALTERADO** | `redactPaths:4-11` ganha `*.secret`, `*.previousSecret`, `*.signature` |
| `src/app.ts` | **EXISTENTE — ALTERADO** | `buildControllers:26-53` instancia o wiring do módulo |
| `src/routes/index.ts` | **EXISTENTE — ALTERADO** | tipo `Controllers:13-19` e montagem de `/webhooks` e `/admin/webhooks` em `:21-30` |
| `tests/setup.ts` | **EXISTENTE — ALTERADO** | `beforeEach:8-16` ganha 4 `deleteMany` na ordem correta de FK |
| `package.json` | **EXISTENTE — ALTERADO** | script `worker`; **nenhuma dependência nova** |
| `src/middlewares/error.middleware.ts` | **EXISTENTE — NÃO MUDA** | ver 10.3 |
| `src/middlewares/auth.middleware.ts` | **EXISTENTE — NÃO MUDA** | ver 10.3 |
| `src/middlewares/validate.middleware.ts` | **EXISTENTE — NÃO MUDA** | ver 10.3 |
| `src/middlewares/request-logger.middleware.ts` | **EXISTENTE — NÃO MUDA** | ver 10.3 |
| `src/shared/errors/app-error.ts` | **EXISTENTE — NÃO MUDA** | classe base das exceções do módulo |
| `src/shared/errors/http-errors.ts` | **EXISTENTE — NÃO MUDA** | bases reaproveitadas; nenhuma classe `WEBHOOK_*` entra aqui |
| `src/shared/errors/index.ts` | **EXISTENTE — NÃO MUDA** | barrel dos erros compartilhados |
| `src/shared/http/response.ts` | **EXISTENTE — NÃO MUDA** | `paginated():22` reaproveitado em 6.2 e 6.5 |
| `src/config/database.ts` | **EXISTENTE — NÃO MUDA** | `createPrismaClient():4` é o que o worker chama |
| `src/server.ts` | **EXISTENTE — NÃO MUDA** | é o **molde** de `src/worker.ts` |
| `src/modules/orders/order.status.ts` | **EXISTENTE — NÃO MUDA** | define quais `toStatus` existem |
| `src/modules/orders/order.repository.ts` | **EXISTENTE — NÃO MUDA** | a publicação usa o `tx`, não o repository |
| `src/modules/users/user.routes.ts` | **EXISTENTE — NÃO MUDA** | `:12-18` é o **precedente** da ordem dos middlewares |
| `docker-compose.yml` | **EXISTENTE — NÃO MUDA** | nenhum serviço novo; sem Redis, sem broker ([ADR-001](./adrs/ADR-001-outbox-no-mysql.md)) |
| `tsconfig.build.json` | **EXISTENTE — NÃO MUDA** | `include: ["src/**/*.ts"]` (`:9`) já compila `src/worker.ts` para `dist/worker.js`; ver 10.3 |
| `prisma/seed.ts` | **EXISTENTE — NÃO MUDA** | a massa já traz o customer `Logística Atlas Comercial Ltda` (`:76`); nenhum cadastro de webhook é semeado |
| `src/worker.ts` | **A CRIAR** | entry-point do processo do worker |
| `src/modules/webhooks/webhook.routes.ts` | **A CRIAR** | `buildWebhookRouter` e `buildAdminWebhookRouter` |
| `src/modules/webhooks/webhook.controller.ts` | **A CRIAR** | handlers de 6.1 a 6.7, incluindo 6.2.1 |
| `src/modules/webhooks/webhook.service.ts` | **A CRIAR** | regras de negócio e erros `WEBHOOK_*` |
| `src/modules/webhooks/webhook.repository.ts` | **A CRIAR** | acesso às 4 tabelas, com `select` que exclui `secret` |
| `src/modules/webhooks/webhook.schemas.ts` | **A CRIAR** | schemas Zod da seção 6 |
| `src/modules/webhooks/webhook.errors.ts` | **A CRIAR** | classes da seção 7.4 |
| `src/modules/webhooks/webhook.publisher.ts` | **A CRIAR** | `publishWebhookEvent(tx, ...)` — chamada pelo `OrderService` |
| `src/modules/webhooks/webhook.processor.ts` | **A CRIAR** | ciclo do worker, envio, retry, DLQ. Nome escolhido entre as duas opções deixadas em aberto em `[09:28] Bruno:` |
| `src/modules/webhooks/webhook.signature.ts` | **A CRIAR** | geração de segredo e HMAC com `node:crypto` — arquivo isolado porque é o alvo da revisão de `[09:46] Sofia:` |
| `src/modules/webhooks/webhook.backoff.ts` | **A CRIAR** | tabela de intervalos da seção 8.2 |
| `prisma/migrations/<timestamp>_add_webhooks/migration.sql` | **A CRIAR** | gerada por `npm run db:migrate` |
| `tests/webhooks.test.ts` | **A CRIAR** | supertest sobre `buildApp`, no molde de `tests/orders.test.ts` |
| `tests/webhook-worker.test.ts` | **A CRIAR** | ciclo do processador com `fetch` interceptado |

### 10.2 O que muda em cada arquivo alterado

**`src/modules/orders/order.service.ts`** — o ponto de integração crítico (`[09:40] Bruno:`).

Assinatura atual: `changeStatus(id: string, input: UpdateOrderStatusInput, userId: string): Promise<OrderWithRelations>` (`:126-130`). Passa a receber `requestId?: string`.

Dentro do `$transaction` (`:131`), a única linha nova entra **depois do `tx.orderStatusHistory.create` que termina em `:167`** e **antes do refetch que começa em `:169`**. A função recebe o `tx`, sem injetar repository — `[09:41] Diego:` "função pura recebendo o tx". `OrderService` ganha um `import` de `webhook.publisher.ts`; **não** ganha dependência de construtor, então a instanciação em `src/app.ts:43` (`new OrderService(orderRepository, prisma)`) permanece igual.

**`src/modules/orders/order.controller.ts`** — `:41` passa a chamar `this.orders.changeStatus(req.params.id!, req.body, req.user.id, req.id)`. É a única linha alterada do arquivo.

**`prisma/schema.prisma`** — os 4 models e o enum da seção 4, mais as relações inversas: `Customer` ganha `webhookEndpoints WebhookEndpoint[]` e `Order` ganha `webhookEvents WebhookOutbox[]`.

**`src/config/env.ts`** — variáveis novas no `envSchema` (`:3-10`), com o mesmo estilo Zod já usado:

```ts
WEBHOOK_POLL_INTERVAL_MS:   z.coerce.number().int().positive().default(2000),    // [09:09] Diego
WEBHOOK_HTTP_TIMEOUT_MS:    z.coerce.number().int().positive().default(10000),   // [09:42] Diego
WEBHOOK_MAX_RETRIES:        z.coerce.number().int().positive().default(5),       // 5 tentativas após a falha da entrega inicial, ADR-003
WEBHOOK_PAYLOAD_MAX_BYTES:  z.coerce.number().int().positive().default(65536),   // 64KB, [09:24] Larissa
WEBHOOK_SECRET_GRACE_HOURS: z.coerce.number().int().positive().default(24),      // [09:21] Sofia
WEBHOOK_WORKER_BATCH_SIZE:  z.coerce.number().int().positive(),                  // sem default: a reunião não deu número
WEBHOOK_SECRET_SWEEP_EVERY_CYCLES: z.coerce.number().int().positive().default(300),  // (proposta do FDD)
WEBHOOK_RESPONSE_BODY_MAX_CHARS:   z.coerce.number().int().positive().default(2048), // (proposta do FDD)
```

`WEBHOOK_WORKER_BATCH_SIZE` é **obrigatória e sem default**, no mesmo estilo de `DATABASE_URL` (`:7`): `[09:08] Diego:` disse "batch pequeno" e não deu número; inventar um default aqui seria precisão fabricada. `loadEnv` (`:14-25`) já derruba o processo com mensagem legível se ela faltar — inclusive no processo do worker.

**`src/routes/index.ts`** — `Controllers` (`:13-19`) ganha `webhooks: WebhookController`, e `buildApiRouter` (`:21-30`) ganha duas montagens, no mesmo padrão de `:24-28`:

```ts
router.use('/webhooks', buildWebhookRouter(controllers.webhooks));
router.use('/admin/webhooks', buildAdminWebhookRouter(controllers.webhooks));
```

É o primeiro prefixo `admin` do projeto, ponto que o [ADR-008](./adrs/ADR-008-controle-de-acesso-dos-endpoints.md) deixou explicitamente para o FDD resolver.

**`src/app.ts`** — `buildControllers` (`:26-53`) ganha três linhas no padrão manual repository → service → controller já usado em `:42-44`, e a chave `webhooks` no objeto de retorno (`:46-52`). Nada muda em `buildApp` (`:55-76`): `express.json({ limit: '1mb' })` (`:59`), `requestLogger` (`:60`), o `404` (`:69-71`) e o `errorMiddleware` (`:73`) atendem o módulo novo como estão.

**`tests/setup.ts`** — o `beforeEach` (`:8-16`) faz `deleteMany` em ordem segura de FK. As tabelas novas precisam sair **antes** de `order`, `customer` e `user`:

```ts
await prisma.webhookDelivery.deleteMany();   // FK → webhook_endpoints
await prisma.webhookDeadLetter.deleteMany(); // FK → webhook_endpoints
await prisma.webhookOutbox.deleteMany();     // FK → webhook_endpoints, orders
await prisma.webhookEndpoint.deleteMany();   // FK → customers
await prisma.orderStatusHistory.deleteMany();  // linhas :9-15 existentes, inalteradas
// ...
```

Errar essa ordem quebra a suíte inteira com violação de FK, não só o teste novo — `vitest.config.ts` usa `singleFork` e `fileParallelism: false`, então todos os arquivos compartilham o mesmo banco.

**`package.json`** — o script `worker` de `[09:11] Larissa:`, espelhando o par `dev`/`start` existente (`:11-13`):

```json
"worker":     "node --env-file=.env dist/worker.js",
"worker:dev": "tsx watch --env-file=.env src/worker.ts"
```

`worker:dev` é **(proposta do FDD)** — a reunião citou só `npm run worker`.

### 10.3 O que **não** muda — e por quê

Esta subseção é tão importante quanto a anterior: cada linha aqui é trabalho que **não** precisa ser feito.

| Arquivo existente | Por que não precisa mudar |
| --- | --- |
| `src/middlewares/error.middleware.ts` | O ramo `err instanceof AppError` (`:15-24`) já serializa qualquer erro do módulo em `{ error: { code, message, details? } }`. Como todas as classes de 7.4 herdam de `AppError`, os códigos `WEBHOOK_*` saem no envelope correto **sem uma linha de mudança** — `[09:29] Bruno:` "vai pegar nossos erros sem precisar mudar nada". O tratamento de `ZodError` (`:26-35`) e de `P2002`/`P2025` (`:37-54`) também serve o módulo como está |
| `src/middlewares/auth.middleware.ts` | `authenticate` (`:27-47`) e `requireRole` (`:49-61`) atendem os dois níveis de acesso do [ADR-008](./adrs/ADR-008-controle-de-acesso-dos-endpoints.md). Nenhuma camada de autorização nova; nenhum papel novo em `AuthUser` (`:6-10`) |
| `src/middlewares/validate.middleware.ts` | `validate({ body, query, params })` (`:11-37`) cobre todos os endpoints da seção 6, e a conversão de `ZodError` em `ValidationError` com `details: [{ path, message }]` (`:26-32`) é o formato de erro de validação da seção 7.2 |
| `src/middlewares/request-logger.middleware.ts` | **Já é o mecanismo de correlação** da seção 9.3: lê ou gera `x-request-id` (`:6`), grava em `req.id` (`:7`), devolve o header (`:8`). O tracing assíncrono se apoia nele sem alterá-lo |
| `src/shared/errors/app-error.ts` / `http-errors.ts` / `index.ts` | Bases suficientes. `BadRequestError`, `ConflictError` e `UnprocessableEntityError` aceitam código customizado; as classes `WEBHOOK_*` moram no módulo, e não em `shared`, porque são específicas dele |
| `src/shared/http/response.ts` | `paginated()` (`:22`) e `PaginatedResponse<T>` (`:8`) já são o formato de listagem do projeto; 6.2 e 6.5 os usam sem variação |
| `src/config/database.ts` | `createPrismaClient()` (`:4-8`) é fábrica e serve exatamente ao `PrismaClient` próprio do worker (`[09:30] Bruno:`). O singleton `prisma` (`:10`) continua sendo o da API |
| `src/server.ts` | Não muda: é o **molde**. `bootstrap()` (`:6`), `app.listen` (`:9`), shutdown em `SIGINT`/`SIGTERM` (`:20-21`) com `prisma.$disconnect()` (`:16`) e `process.exit(0)` (`:17`), mais o `catch` de bootstrap (`:24-27`) — `src/worker.ts` replica essa estrutura trocando o `listen` pelo laço de polling |
| `src/modules/orders/order.status.ts` | A máquina (`:3-10`) e os predicados `shouldDebitStock`/`shouldReplenishStock` (`:29-37`) não mudam. O webhook só **lê** o `to` já validado por `canTransition` (`:12-14`) |
| `src/modules/orders/order.repository.ts` | A publicação acontece no `tx`, dentro do service. Nenhuma consulta nova de order |
| `src/modules/users/user.routes.ts` | Nada muda; `:12-18` é o **precedente copiado** para o router admin: `authenticate` → `requireRole('ADMIN')` → `validate({ params })` → handler |
| `docker-compose.yml` | Só `mysql:8.0`. Nenhum serviço novo — é a confirmação operacional do [ADR-001](./adrs/ADR-001-outbox-no-mysql.md). O worker roda no mesmo container/host da aplicação, como segundo processo |
| `tsconfig.build.json` | **O build do worker já está resolvido, e ninguém precisa perceber isso na hora errada.** `include: ["src/**/*.ts"]` (`:9`) varre a árvore inteira de `src/`, e `exclude` (`:10`) lista apenas `node_modules`, `dist`, `tests`, `prisma` e `**/*.test.ts`. Logo, `src/worker.ts` **(a criar)** é compilado por `npm run build` para `dist/worker.js` sem uma linha de configuração nova — que é exatamente o caminho que o script `worker` do `package.json` (10.2) invoca. Um `tsconfig` com `files` explícito exigiria alteração; este não |
| `prisma/seed.ts` | Nenhum cadastro de webhook é semeado: a feature não tem estado inicial obrigatório, e a massa existente já basta para exercitar os fluxos (o customer de `:76` é a Atlas da seção 1) |
| `src/modules/auth`, `customers`, `products`, `users` | Intocados. `user.service.ts` merece uma ressalva de leitura — ver 4.1: o `BCRYPT_ROUNDS = 10` (`:16`) é o padrão de senha do projeto e **não** serve de precedente para o segredo de webhook |

### 10.4 Convenções do código que a implementação copia, não reinventa

| Convenção | Onde está o precedente |
| --- | --- |
| Estrutura de módulo (controller/service/repository/routes/schemas) | `src/modules/orders/` (`[09:27] Bruno:`) |
| Wiring manual repository → service → controller | `src/app.ts:42-44` |
| `router.use(authenticate)` no topo para CRUD sem restrição de papel | `src/modules/orders/order.routes.ts:14` |
| `authenticate` + `requireRole('ADMIN')` + `validate` por rota | `src/modules/users/user.routes.ts:12-18` |
| Schema de param `z.object({ id: z.string().uuid() })` | `src/modules/orders/order.schemas.ts:4` |
| Paginação `page`/`pageSize` com máximo 100 | `src/modules/orders/order.schemas.ts:24-25` |
| Export dos tipos inferidos do Zod | `src/modules/orders/order.schemas.ts:32-34` |
| `res.status(201).json(...)` na criação, `204` sem corpo na remoção | `src/modules/orders/order.controller.ts:32,51` |
| Enum e `@@map` snake_case, PK `uuid` `@db.Char(36)`, `DATETIME(3)`, utf8mb4 | `prisma/schema.prisma`, `prisma/migrations/20260519182739_init/migration.sql` |
| Nomes de evento de log em snake_case | `src/server.ts:10`, `request-logger.middleware.ts:23` |
| Factories e `getTestApp()` nos testes | `tests/helpers/factories.ts:10-15` |

---

## 11. Dependências e compatibilidade

### 11.1 Dependências — **nenhuma nova**

Este é um resultado, não uma ausência: dependência nova é custo de supply chain, de aprovação e de revisão. A feature inteira cabe no que já está instalado ou na biblioteca padrão do Node.

| Necessidade | Como é atendida | Já existe? |
| --- | --- | --- |
| HMAC-SHA256 ([ADR-004](./adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)) | `createHmac('sha256', …)` do módulo nativo `node:crypto` | biblioteca padrão |
| Geração do segredo | `randomBytes` do `node:crypto` | biblioteca padrão |
| Chamada HTTP de saída | `fetch` global — estável no Node >= 18 e o projeto exige >= 20 (`package.json`, `engines`) | runtime |
| Timeout de 10s da chamada | `AbortSignal.timeout()` | runtime |
| UUID dos eventos | `@default(uuid())` do Prisma; `uuid` 11.0.3 já está instalado e em uso em `request-logger.middleware.ts:2` | instalado |
| Validação de schema | `zod` 3.23.8 | instalado |
| Log estruturado | `pino` 9.5.0 (`[09:29] Bruno:`) | instalado |
| Acesso ao banco | `@prisma/client` 5.22.0 | instalado |
| Testes HTTP | `supertest` 7.0.0 + `vitest` 2.1.4 | instalado (dev) |

**Não há cliente HTTP** (`axios`, `node-fetch`) **nem biblioteca de criptografia de terceiros** no `package.json`, e não passa a haver. O único acréscimo ao arquivo é o script `worker` — `[09:11] Larissa:` — que hoje não existe entre `dev`, `build`, `start`, `db:migrate`, `db:reset`, `db:seed`, `test`, `test:watch`, `lint` e `format`.

**Infraestrutura:** nenhuma. `docker-compose.yml` sobe apenas `mysql:8.0`; não há Redis nem broker, e nenhum é acrescentado ([ADR-001](./adrs/ADR-001-outbox-no-mysql.md)).

**Runtime:** o piso de Node sobe de fato de "≥ 20 declarado" para "≥ 20 necessário", porque `fetch` e `AbortSignal.timeout` passam a ser usados em produção. `engines` já cobre isso; nenhuma mudança de arquivo.

### 11.2 Compatibilidade retroativa

**O que quebra para quem já consome a API: nada.** E vale registrar por quê, com precisão:

| Superfície | Mudança | Impacto |
| --- | --- | --- |
| `GET/POST/DELETE /api/v1/orders`, `/customers`, `/products`, `/users`, `/auth` | nenhuma | zero |
| Corpo da resposta de `PATCH /orders/:id/status` | nenhuma — continua sendo a order com `items`, `history` e `customer` (`order.service.ts:169-177`) | zero |
| **Códigos de erro de `PATCH /orders/:id/status`** | **acréscimo:** passa a poder retornar `422 WEBHOOK_PAYLOAD_TOO_LARGE` (seção 7.1) | cliente que trata erro por `error.code` e ignora código desconhecido não quebra; cliente que assume conjunto fechado de códigos precisa saber. **É a única mudança observável de contrato existente** |
| Latência de `PATCH /orders/:id/status` | acréscimo de um `findMany` em `webhook_endpoints` e de N `insert` na `webhook_outbox`, dentro da transação já existente | mensurável; ver risco R-1 |
| Schema do banco | **puramente aditivo**: 4 tabelas e 1 enum novos. Nenhuma coluna existente muda de nome, tipo ou nulidade; nenhum índice existente é removido | migration reversível por `DROP TABLE`; rollback não perde dado de pedido |
| Formato de erro | `{ error: { code, message, details? } }` inalterado | zero |
| `express.json({ limit: '1mb' })` (`src/app.ts:59`) | inalterado | **não confundir** com o teto de 64 KB do evento: `1mb` limita o corpo que **entra** na nossa API; 64 KB limita o payload do evento que **sai** para o cliente. Camadas diferentes, sem relação |

**Ordem de deploy (importa):**

1. `prisma migrate deploy` — aditivo, seguro com a versão antiga da API rodando.
2. API. A partir daqui eventos começam a ser enfileirados.
3. Worker. Antes dele, eventos **acumulam mas não se perdem** — a outbox é durável.

Inverter 2 e 3 não causa dano: o worker sobe e não encontra nada pendente. Subir a API antes da migration quebra `changeStatus` na primeira mudança de status, porque a tabela não existe — e, pela garantia de [ADR-001](./adrs/ADR-001-outbox-no-mysql.md), isso derruba a operação de pedido.

**Rollback:** reverter a API para a versão anterior para de enfileirar; a outbox pode ficar com linhas pendentes que o worker (se mantido) entrega normalmente. Derrubar o worker e manter a API é o estado degradado seguro: acumula, não perde.

---

## 12. Critérios de aceitação técnicos

Cada item é verificável por teste automatizado ou inspeção de código. Referência à seção que o origina entre parênteses.

**Atomicidade e emissão (5.1)**

1. Forçar erro após a inserção na outbox, dentro da transação de `changeStatus`, **não deixa evento órfão**: o status do pedido permanece o anterior e `webhook_outbox` fica sem linha. Teste de integração.
2. `changeStatus` bem-sucedido para um cliente com um cadastro ativo que escuta o status de destino cria **exatamente uma** linha em `webhook_outbox`, com `status = PENDING` e `attemptCount = 0`.
3. Cliente sem nenhum cadastro que escute o status de destino: mudança de status ocorre normalmente e **nenhuma linha** é criada (`[09:34] Bruno:`).
4. Cliente com dois cadastros ativos escutando o mesmo status: **duas** linhas, com `X-Event-Id` distintos e mesmo `payload`.
5. Cadastro com `active = false` não gera linha, mesmo escutando o status.
6. Criação de pedido (`POST /orders`) **não** gera evento (exclusão 3.2).
7. O `payload` gravado é snapshot: alterar a order depois **não** altera a linha da outbox. Inspeção + teste.

**Contratos HTTP (seção 6)**

8. Os 8 endpoints respondem os códigos da seção 6, incluindo `403 FORBIDDEN` para `OPERATOR` no replay — hoje **não existe nenhuma asserção de `403` no repositório** (`tests/auth.test.ts`, `tests/orders.test.ts`), lacuna registrada no [ADR-008](./adrs/ADR-008-controle-de-acesso-dos-endpoints.md).
9. `secret` aparece **apenas** nas respostas de `POST /webhooks` (201) e de rotação (200). Teste que faz `GET`, `PATCH` e listagem e assere ausência do campo em todos.
10. `POST /webhooks` com URL `http://…` responde `400` com `error.code === 'WEBHOOK_INVALID_URL'` (`[09:23] Sofia:`, `[09:28] Bruno:`).
11. `GET /webhooks/:id/deliveries` devolve o formato `{ data, pagination }` de `src/shared/http/response.ts:8`, ordenado por `createdAt` decrescente, com no máximo 100 por página.

11-b. `GET /webhooks/:id/deliveries?success=false` devolve **apenas** entregas com `success = false`, e `?success=xyz` responde `400 VALIDATION_ERROR` — `z.coerce.boolean()` não pode ser usado em query param, porque `Boolean('false') === true`.

12. Todo erro do módulo responde com `error.code` começando em `WEBHOOK_` (`[09:29] Larissa:`) — exceto os erros compartilhados listados em 7.3. Verificável por inspeção da matriz e por teste sobre cada caminho de exceção.

**Entrega e assinatura (5.2, 6.8)**

13. O `X-Signature` enviado é `HMAC-SHA256(body, secret)` em hex, calculado **sobre a mesma string** enviada como corpo. Teste: recalcular a assinatura no lado do teste e comparar.
14. Os cinco headers de 6.8.2 estão presentes em todo envio, com `X-Event-Id` igual ao `id` da linha da outbox e `X-Webhook-Id` igual ao id do cadastro.
15. Resposta `2xx` do cliente marca a linha como `DELIVERED` e cria uma linha em `webhook_deliveries` com `success = true` e `durationMs` preenchido.
16. Resposta `500` do cliente mantém o evento na fila com `status = FAILED` e `nextAttemptAt` exatamente **1 minuto** à frente na primeira falha (8.2).
17. Cliente que não responde em 10 segundos é tratado como falha (`[09:42] Diego:`); teste com servidor que atrasa a resposta.
18. `X-Event-Id` é **idêntico** em todas as tentativas do mesmo evento ([ADR-005](./adrs/ADR-005-at-least-once-com-x-event-id.md)). Teste que força falha e captura os headers das duas chamadas.

18-b. Com a primeira entrega do evento A falhando e um evento B do mesmo pedido inserido em seguida, B é entregue antes de A — comportamento esperado, documentado em 5.2.2, e o teste assere que ambos chegam com `from_status`/`to_status` corretos.

**Retry, DLQ e replay (5.3, 5.4, 5.5)**

19. Esgotada a progressão de backoff, a linha **sai** da `webhook_outbox` e entra em `webhook_dead_letter` com `payload`, `failureReason` e `failedAt` (`[09:18] Diego:`).
20. Replay recria a linha na outbox **com o mesmo `X-Event-Id`** e grava `replayedById` com o id do usuário `ADMIN` que chamou (`[09:36] Sofia:`).
21. Segundo replay da mesma linha responde `409 WEBHOOK_ALREADY_REPLAYED`.
22. Linha deixada em `PROCESSING` antes do bootstrap volta para `PENDING` e é reenviada (5.2.3) — teste que simula crash inserindo a linha nesse estado e chamando a rotina de bootstrap.

**Segurança e observabilidade (9)**

23. Nenhuma linha de log contém o valor de `secret`, `previousSecret` ou `X-Signature`. Verificável por teste que captura a saída do Pino em um fluxo completo de rotação e de entrega, e por inspeção de `redactPaths`.

23-b. Cadastro com `previousSecretExpiresAt` no passado tem `previousSecret` zerado pela varredura do worker, e uma nova rotação passa a ser aceita com `200` em vez de `409` ([ADR-004](./adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)).

24. `webhook_outbox.requestId` é igual ao `X-Request-Id` devolvido pela resposta do `PATCH /orders/:id/status` que originou o evento (9.3).
25. Os eventos de log da tabela 9.2 são emitidos com os campos listados, e `webhook_event_enqueued` **não** é emitido quando a transação sofre rollback (5.1).

**Integração e infraestrutura (10, 11)**

26. `git diff` do `package.json` mostra **apenas** acréscimo de script — zero dependências novas (11.1).
27. A suíte existente (`tests/orders.test.ts`, `tests/auth.test.ts`) passa sem alteração, com `tests/setup.ts` já contendo os quatro `deleteMany` novos na ordem correta.
28. `npm run worker` sobe o processo, conecta com `PrismaClient` próprio e responde a `SIGTERM` encerrando o laço e chamando `$disconnect()` (`[09:30] Bruno:`, molde de `src/server.ts:13-21`).
29. A migration gerada é aditiva: `git diff` do `migration.sql` contém apenas `CREATE TABLE`, `CREATE INDEX` e `ADD CONSTRAINT`, sem `ALTER` destrutivo em tabela existente (11.2).

**Latência (2)**

30. Da resposta do `PATCH /orders/:id/status` à primeira chamada HTTP ao cliente decorrem **menos de 10 segundos** em ambiente de teste (`[09:02] Marcos:`), medido pela métrica `webhook_event_lag_ms` (9.1).

---

## 13. Riscos e mitigação

| # | Risco técnico | Por que é real aqui | Mitigação |
| --- | --- | --- | --- |
| **R-1** | **A transação de `changeStatus` alarga e piora a contenção de lock em `orders` e `products`.** | A transação já faz `update` em `orders` (`:158`), `create` em `order_status_history` (`:159-167`) e um `update` de estoque **por item** (`debitStock:225-230`); ganha agora um `findMany` em `webhook_endpoints` e N `insert`. Consequência já assumida no [ADR-001](./adrs/ADR-001-outbox-no-mysql.md) | O `findMany` é servido pelo índice `@@index([customerId, active])` e o payload é renderizado uma vez para todos os destinatários. Acompanhar `durationMs` do `http_request` (`request-logger.middleware.ts:20`) para `PATCH /orders/:id/status` antes e depois do deploy |
| **R-2** | **Um defeito no módulo de webhooks derruba a mudança de status de pedido.** | Inserção dentro da transação significa que qualquer exceção na renderização ou na inserção provoca rollback (`[09:40] Bruno:`). `WEBHOOK_PAYLOAD_TOO_LARGE` é o caminho documentado; um bug não previsto vira `500` no `PATCH` | Manter `publishWebhookEvent` mínima: uma consulta, um filtro em memória, uma serialização, N inserts. Nenhuma chamada de rede, nenhuma leitura extra de `orders`. Cobrir com os critérios 1 a 7 da seção 12 |
| **R-3** | **Worker morto passa despercebido.** | Nenhuma requisição falha, nenhum erro é logado, os clientes simplesmente param de receber ([ADR-002](./adrs/ADR-002-worker-separado-com-polling.md)) | Gauge `webhook_outbox_pending_total` a cada ciclo (9.1) — a ausência da métrica **é** o sinal, tão importante quanto o valor dela. Supervisão de processo com restart automático; o bootstrap já recupera linhas `PROCESSING` |
| **R-4** | **Segredo vazando em log.** | O `redact` atual não conhece `secret` (`src/shared/logger/index.ts:4-11`), e Diego relatou caso real desse vazamento do lado de um cliente (`[09:22] Diego:`) | Acrescentar os três caminhos ao `redactPaths` (9.2); serializar cadastro sempre com `select` explícito no repository (6.2); critério de aceitação 23. Isolar HMAC e geração de segredo em `webhook.signature.ts` para a revisão de `[09:46] Sofia:` |
| **R-5** | **Grace period de rotação sem regra de assinatura definida.** | O [ADR-004](./adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) deixou **(em aberto)** qual segredo assina durante as 24h. Implementar "assina com o novo" torna a janela inútil — o cliente que ainda não migrou passa a receber assinatura que não valida, exatamente o que o grace period existe para evitar | **Bloqueia o passo 4 do fluxo 5.2.2.** Levar as três opções à revisão de `[09:46] Sofia:` antes de codar o envio. O modelo de dados (`secret`, `previousSecret`, `previousSecretExpiresAt`) suporta qualquer das três sem migration adicional |
| **R-6** | **`DELETE /webhooks/:id` apaga histórico de entregas e DLQ junto**, por `onDelete: Cascade`. | Cliente que quer só parar de receber pode remover o cadastro e perder a evidência que `GET /deliveries` (`[09:34] Marcos:`) existe para dar | Documentar no portal do desenvolvedor (`[09:40] Marcos:`) que a pausa é `PATCH { "active": false }` e a remoção é definitiva. Se virar problema recorrente, `DELETE` passa a ser remoção lógica — decisão nova, com ADR |
| **R-7** | **`webhook_outbox` e `webhook_deliveries` crescem sem limite.** | Arquivamento foi declarado fora do escopo (`[09:08] Diego:`), e o snapshot duplica dado por linha ([ADR-007](./adrs/ADR-007-snapshot-do-payload-na-insercao.md)) — agora em três tabelas | Nenhuma mitigação dentro desta feature, por decisão. Acompanhar o tamanho das tabelas; o gatilho de reabertura registrado no [ADR-001](./adrs/ADR-001-outbox-no-mysql.md) é impacto em consulta, backup ou custo |
| **R-8** | **Qualquer usuário autenticado cadastra webhook para qualquer `customerId` e recebe um segredo válido na resposta — e, por 6.6, também rotaciona o segredo de qualquer cadastro existente**, invalidando a integração em uso e ficando com o novo valor. Também pode redirecionar os eventos de outro cliente para uma URL própria | `customerId` vem do body e não do JWT (`[09:32] Larissa:`), e o CRUD não exige papel (`[09:37] Sofia:`) — vazamento entre clientes viabilizado por desenho, aceito nesta fase no [ADR-008](./adrs/ADR-008-controle-de-acesso-dos-endpoints.md) | Fora do escopo desta feature. O log de auditoria genérico (`userId` em `http_request`, `request-logger.middleware.ts:21`) permite reconstruir quem criou o quê. Endurecimento depende de vincular `customerId` ao token — reabertura prevista no ADR-008 |
| **R-9** | **Cliente sem dedup processa evento duplicado.** | At-least-once é garantia explícita ([ADR-005](./adrs/ADR-005-at-least-once-com-x-event-id.md)); crash do worker entre enviar e marcar (5.2.3) produz duplicata real | `X-Event-Id` estável entre tentativas e no replay (critério 18). Documentação em destaque no portal (`[09:26] Marcos:`). `webhook_outbox_reclaimed_total` > 0 indica quando duplicatas foram provavelmente geradas |
| **R-10** | **Cliente lento drena a vazão do worker único.** | Processamento é sequencial e cada envio pode levar até 10s; um cliente próximo do timeout atrasa a fila de todos | `webhook_delivery_duration_ms` por endpoint (9.1) identifica o responsável. Paralelismo e rate limiting foram adiados (`[09:13] Diego:`, `[09:39] Larissa:`); se o sintoma aparecer, é gatilho para reabrir o [ADR-002](./adrs/ADR-002-worker-separado-com-polling.md) |

---

## Rastreabilidade

**Transcrição** — `[09:00]`, `[09:02]`, `[09:03]`, `[09:04]`, `[09:06]`, `[09:08]`, `[09:09]`, `[09:11]`, `[09:12]`, `[09:13]`, `[09:15]`, `[09:16]`, `[09:17]`, `[09:18]`, `[09:20]`, `[09:21]`, `[09:22]`, `[09:23]`, `[09:24]`, `[09:25]`, `[09:26]`, `[09:27]`, `[09:28]`, `[09:29]`, `[09:30]`, `[09:31]`, `[09:32]`, `[09:33]`, `[09:34]`, `[09:35]`, `[09:36]`, `[09:37]`, `[09:39]`, `[09:40]`, `[09:41]`, `[09:42]`, `[09:43]`, `[09:44]`, `[09:46]`, `[09:48]`, `[09:51]`, `[09:52]` — cada uma citada em pelo menos uma afirmação deste documento e conferida contra [`TRANSCRICAO.md`](../TRANSCRICAO.md). O contexto de `[09:07]` (Redis descartado) e de `[09:19]` (abertura do bloco de segurança) está nos ADRs 001 e 004, não aqui.

**Decisões** — [ADR-001](./adrs/ADR-001-outbox-no-mysql.md), [ADR-002](./adrs/ADR-002-worker-separado-com-polling.md), [ADR-003](./adrs/ADR-003-retry-com-backoff-e-dlq.md), [ADR-004](./adrs/ADR-004-hmac-sha256-secret-por-endpoint.md), [ADR-005](./adrs/ADR-005-at-least-once-com-x-event-id.md), [ADR-006](./adrs/ADR-006-reuso-dos-padroes-do-projeto.md), [ADR-007](./adrs/ADR-007-snapshot-do-payload-na-insercao.md), [ADR-008](./adrs/ADR-008-controle-de-acesso-dos-endpoints.md).

**Código verificado** — `src/app.ts`, `src/server.ts`, `src/routes/index.ts`, `src/config/env.ts`, `src/config/database.ts`, `src/middlewares/auth.middleware.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/validate.middleware.ts`, `src/middlewares/request-logger.middleware.ts`, `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/shared/http/response.ts`, `src/shared/logger/index.ts`, `src/modules/orders/order.service.ts`, `src/modules/orders/order.controller.ts`, `src/modules/orders/order.routes.ts`, `src/modules/orders/order.schemas.ts`, `src/modules/orders/order.status.ts`, `src/modules/users/user.routes.ts`, `src/modules/users/user.service.ts`, `prisma/schema.prisma`, `prisma/migrations/20260519182739_init/migration.sql`, `prisma/seed.ts`, `tests/setup.ts`, `tests/helpers/factories.ts`, `package.json`, `tsconfig.build.json`, `docker-compose.yml`.
