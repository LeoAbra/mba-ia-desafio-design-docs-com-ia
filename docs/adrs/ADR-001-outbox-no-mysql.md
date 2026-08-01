# ADR-001 — Padrão outbox no MySQL para emissão de eventos de pedido

- **Status:** Aceito
- **Data:** Quinta-feira, reunião técnica de definição (ver `TRANSCRICAO.md`)
- **Decisores:** Larissa (Tech Lead, conduzindo), Diego (Engenheiro Sênior, Plataforma), Bruno (Engenheiro Pleno, time de Pedidos), Sofia (Engenheira de Segurança), Marcos (Product Manager)
- **Contexto técnico:** módulo de Pedidos (`src/modules/orders`) e persistência MySQL via Prisma; novo módulo de webhooks (a criar)

## Contexto

Três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — pediram formalmente para
serem notificados quando o status dos pedidos deles muda. Hoje eles fazem polling no
`GET /orders`, o que deixou a integração lenta e cara do lado deles, e a Atlas sinalizou que
pode migrar para o concorrente se a entrega não sair até o fim do trimestre
(`[09:00] Marcos:`). O requisito de latência é frouxo: "qualquer coisa abaixo de 10 segundos
já é tempo real" (`[09:02] Marcos:`). O escopo é apenas outbound — sai da nossa plataforma
para o cliente, não o contrário (`[09:02] Marcos:`, `[09:03] Sofia:`).

A restrição está no sistema que já roda. A mudança de status de pedido é uma transação
única e já pesada: `OrderService.changeStatus` abre um `prisma.$transaction` que valida a
transição, debita ou repõe estoque (`stockQuantity` de cada produto do pedido), atualiza a
linha em `orders` e insere em `order_status_history`
(`src/modules/orders/order.service.ts:126`). Bruno mediu o risco de acrescentar uma chamada
HTTP no meio disso: "qualquer cliente lento vai travar mudança de status pra outros pedidos"
(`[09:04] Bruno:`). E há o problema de integridade, que é pior que o de performance: se o
endpoint do cliente estiver fora do ar no instante do commit, não existe resposta boa —
"o que a gente faz, dá rollback na mudança de status? Não dá" (`[09:04] Bruno:`).

Do outro lado, o time é pequeno e não tem apetite para operar mais um componente de
infraestrutura só por causa desta feature (`[09:07] Diego:`). Prazo estimado: três sprints
(`[09:46] Larissa:`).

O dilema que abriu a reunião foi exatamente esse: "a gente dispara isso sincronamente no
service de orders quando o status muda, ou faz algum tipo de fila/outbox?"
(`[09:03] Larissa:`) — sabendo que qualquer fila real significa infra nova, e que despacho
síncrono significa acoplar a disponibilidade do pedido à disponibilidade do cliente.

## Decisão

Usamos o padrão **outbox no MySQL que já temos**. Não há despacho HTTP síncrono no caminho
da mudança de status (`[09:06] Diego:`) e não subimos nenhum componente de infraestrutura
novo (`[09:07] Diego:`). O fechamento da decisão é de Larissa: "Tá decidido então: outbox em
MySQL" (`[09:08] Larissa:`).

Parâmetros concretos da decisão:

- Quando o status de um pedido muda, o evento é inserido como uma linha na tabela
  **`webhook_outbox`** (a criar) **dentro da mesma transação SQL** que atualiza `orders` e
  insere em `order_status_history` (`[09:06] Diego:`).
- A atomicidade é a razão de ser da decisão: se a transação principal commitou, o evento
  está registrado; se deu rollback, o evento some junto — "não tem inconsistência possível"
  (`[09:06] Diego:`). Se a inserção na outbox falhar, a mudança de status sofre rollback:
  "não pode ter caso de status mudar e evento não sair" (`[09:40] Bruno:`).
- A tabela tem estados de processamento **pendente, processando, falhou, entregue**, com
  índice no campo de status e em `created_at` (`[09:08] Diego:`).
- O ponto de integração é o método `changeStatus` do `OrderService`
  (`src/modules/orders/order.service.ts:126`). A inserção entra por uma função que recebe o
  client de transação corrente — `publishWebhookEvent(tx, order, fromStatus, toStatus)` —
  em vez de injetar o repository de webhooks inteiro dentro do serviço de pedidos
  (`[09:41] Bruno:`, `[09:41] Diego:`).
- Um worker separado lê a outbox e dispara as chamadas HTTP (`[09:06] Diego:`). Se esse
  worker é processo próprio, com que frequência lê e em que lote é decisão de [ADR-002](./ADR-002-worker-separado-com-polling.md), não
  desta ADR.
- O arquivamento das linhas já entregues (algo como 30 dias) foi explicitamente declarado
  **fora do escopo desta feature** (`[09:08] Diego:`).

## Alternativas consideradas

### Despacho HTTP síncrono dentro do `OrderService` — descartada

Disparar a chamada ao webhook do cliente dentro da própria transação de mudança de status.
Foi uma das duas opções que Larissa colocou ao abrir o dilema, não uma defesa dela
(`[09:03] Larissa:`) — ela se alinhou contra logo em seguida: "Concordo. Pensei a mesma
coisa." (`[09:04] Larissa:`). Foi derrubada por Bruno em dois argumentos: (1) a transação já faz update em `orders`, insert em `order_status_history`
e decremento de estoque — "se a gente acrescentar um HTTP call no meio disso, qualquer
cliente lento vai travar mudança de status pra outros pedidos" (`[09:04] Bruno:`); e (2) não
existe tratamento aceitável para o cliente indisponível — "se o cliente tiver fora do ar, o
que a gente faz, dá rollback na mudança de status? Não dá" (`[09:04] Bruno:`). Diego
confirmou ao entrar na call: "síncrono está fora de questão" (`[09:06] Diego:`).

O trade-off que a derrubou: acoplaria a disponibilidade de uma operação interna e crítica
do negócio à disponibilidade de um endpoint de terceiro.

### Redis Streams / fila externa — descartada

Levantada por Larissa como o caminho convencional de mensageria: "a alternativa seria botar
Redis Streams ou alguma coisa parecida, mas a gente acabaria precisando subir mais infra"
(`[09:07] Larissa:`). Derrubada por Diego pelo custo operacional: "a gente é um time pequeno.
Subir Redis Cluster pra isso é overengineering. Outbox no MySQL existente resolve"
(`[09:07] Diego:`).

O argumento que a derrubou foi só o custo operacional: um broker dedicado é um segundo
sistema com estado para operar, monitorar e recuperar, e o time é pequeno
(`[09:07] Diego:`). Throughput e desacoplamento, que seriam o ganho, ainda não são
problema no volume atual. (Que um broker também exigiria uma outbox para garantir
atomicidade com o commit do pedido é análise desta ADR, não foi dito na reunião.)

## Consequências

**Positivas**

- Atomicidade real entre a mudança de status e o registro do evento: um commit implica um
  evento registrado; um rollback apaga o evento junto. Não existe estado em que o pedido
  mudou e a notificação se perdeu (`[09:06] Diego:`).
- A chamada HTTP sai do caminho crítico da transação. Cliente lento ou offline não trava a
  mudança de status de outros pedidos nem provoca rollback do pedido (`[09:04] Bruno:`).
- Nenhum componente de infraestrutura com estado novo: mesmo MySQL, mesmo Prisma, mesma
  `DATABASE_URL` (`[09:30] Bruno:`) — foi exatamente o argumento de evitar overengineering
  num time pequeno (`[09:07] Diego:`), e cabe na janela de três sprints (`[09:46] Larissa:`).
  Não é "zero infraestrutura": a leitura da outbox exige um segundo processo Node para
  operar e monitorar (`[09:11] Diego:`, `[09:11] Larissa:`), custo que é detalhado em
  [ADR-002](./ADR-002-worker-separado-com-polling.md).
- O ponto de integração no domínio de pedidos é mínimo: uma função que recebe o `tx`, sem
  injetar o repository de webhooks no `OrderService` (`[09:41] Diego:`).

**Negativas**

- **O throughput de eventos passa a depender do MySQL transacional.** A mesma instância que
  atende a API de pedidos passa a absorver a escrita de cada evento e a leitura contínua do
  worker. Não há isolamento: pico de eventos e pico de pedidos disputam o mesmo banco e o
  mesmo pool de conexões.
- **A tabela cresce sem política de limpeza definida.** O arquivamento das linhas entregues
  foi conscientemente empurrado para fora desta feature (`[09:08] Diego:`), então entramos
  em produção sabendo que `webhook_outbox` cresce monotonicamente até alguém decidir o
  contrário. O custo aparece em tamanho de backup, tempo de restore e degradação de
  varredura por índice.
- **A transação de mudança de status, que já era pesada, fica mais pesada.** Adicionamos mais
  um insert dentro do `$transaction` de `changeStatus`, alargando a janela de lock de uma
  operação que já mexe em `orders`, `order_status_history` e no estoque dos produtos.
- **Um defeito no módulo de webhooks pode derrubar a operação principal de pedidos.** Como a
  inserção na outbox está dentro da transação e falha nela significa rollback
  (`[09:40] Bruno:`), um bug na montagem do evento passa a ser capaz de impedir a mudança de
  status de um pedido. É o preço direto da garantia de "não pode ter caso de status mudar e
  evento não sair".

**Neutras / limitações conhecidas**

- Usar o banco transacional como broker é adequado ao volume atual — mudanças de status dos
  pedidos de três clientes B2B (`[09:00] Marcos:`) —, não a um volume arbitrário de eventos.
  **Gatilho de reabertura:** se a latência ponta a ponta de entrega passar a estourar o teto
  de 10 segundos acordado com os clientes (`[09:02] Marcos:`) por contenção no MySQL, ou se
  a escrita/leitura da outbox aparecer nas métricas de lock e de pool de conexões da API de
  pedidos, a alternativa Redis Streams / fila externa volta à mesa — junto com o custo
  operacional que a derrubou hoje (`[09:07] Diego:`).
- A ausência de política de arquivamento é uma dívida aceita, não um esquecimento.
  **Gatilho:** quando o volume de linhas entregues impactar consulta, backup ou custo de
  armazenamento, o expurgo/arquivamento (a ordem de grandeza citada foi 30 dias) precisa de
  decisão própria (`[09:08] Diego:`).
- O `webhook_outbox` não é um contrato público. É estrutura interna: clientes veem apenas o
  webhook entregue e o histórico de entregas exposto por API (`[09:34] Marcos:`), o que
  mantém liberdade para alterar o modelo da tabela sem quebrar integração externa.

## Rastreabilidade

**Transcrição** (`TRANSCRICAO.md`)

- `[09:00] Marcos:` — pedido formal de Atlas Comercial, MaxDistribuição e Nova Cargo; hoje
  fazem polling no `GET /orders`; risco de churn da Atlas até o fim do trimestre.
- `[09:02] Marcos:` — abaixo de 10 segundos já é "tempo real" para os clientes.
- `[09:02] Marcos:` / `[09:03] Sofia:` — escopo apenas outbound.
- `[09:03] Larissa:` — o dilema: despacho síncrono no service de orders ou fila/outbox.
- `[09:04] Bruno:` — a transação já atualiza `orders`, insere em `order_status_history` e
  decrementa estoque; HTTP call no meio trava mudança de status de outros pedidos.
- `[09:04] Larissa:` — "Concordo. Pensei a mesma coisa." — alinhamento contra o síncrono.
- `[09:04] Bruno:` — cliente fora do ar não pode causar rollback da mudança de status.
- `[09:06] Diego:` — "síncrono está fora de questão"; definição do padrão outbox e da
  garantia de atomicidade (commit registra o evento, rollback apaga junto).
- `[09:07] Larissa:` — Redis Streams como alternativa, ao custo de subir mais infra.
- `[09:07] Diego:` — time pequeno; Redis Cluster para isso é overengineering.
- `[09:08] Diego:` — estados pendente/processando/falhou/entregue, índice em status e
  `created_at`; arquivamento após ~30 dias fora do escopo.
- `[09:08] Larissa:` — "Tá decidido então: outbox em MySQL".
- `[09:11] Diego:` / `[09:11] Larissa:` — worker como processo separado da API, com entry
  própria (`src/worker.ts`) — decisão detalhada em [ADR-002](./ADR-002-worker-separado-com-polling.md), citada aqui só como custo.
- `[09:30] Bruno:` — worker usa `PrismaClient` próprio, mesmo banco e mesma `DATABASE_URL`.
- `[09:34] Marcos:` — histórico de entregas exposto ao cliente por API.
- `[09:40] Bruno:` — inserção na `webhook_outbox` dentro da mesma transação de `changeStatus`;
  falha na outbox provoca rollback.
- `[09:41] Diego:` — fora da transação, perde-se a garantia inteira.
- `[09:41] Bruno:` / `[09:41] Diego:` — `publishWebhookEvent(tx, order, fromStatus, toStatus)`
  recebendo o `tx`, sem injetar o repository no `OrderService`.
- `[09:48] Larissa:` — resumo final confirma outbox no MySQL com transação atômica.

**Código existente** (verificado)

- `src/modules/orders/order.service.ts:126` — `OrderService.changeStatus`, o
  `prisma.$transaction` onde a inserção na outbox passa a acontecer: valida a transição,
  chama `debitStock`/`replenishStock`, faz `tx.order.update` e
  `tx.orderStatusHistory.create`.
- `src/modules/orders/order.service.ts:204` — `debitStock`, que decrementa `stockQuantity`
  dos produtos dentro da mesma transação (o peso citado por Bruno).
- `prisma/schema.prisma` — models `Order` (`@@map("orders")`), `OrderStatusHistory`
  (`@@map("order_status_history")`) e `Product`; datasource `mysql`.
- `src/config/database.ts` — `createPrismaClient()` e o singleton `prisma` exportado
  (`src/config/database.ts:10`): é o ponto de criação do `PrismaClient` usado pela
  aplicação. O script `prisma/seed.ts:4` instancia o seu próprio, fora desse caminho.

**Código a criar**

- `prisma/schema.prisma` — model da tabela `webhook_outbox` (a criar).
- `src/modules/webhooks/` — módulo de webhooks, incluindo a função
  `publishWebhookEvent(tx, ...)` chamada pelo `OrderService` (a criar).

**Relacionado**

- [ADR-002](./ADR-002-worker-separado-com-polling.md) — worker em processo separado com polling de 2 segundos (como a outbox é lida).
- [ADR-003](./ADR-003-retry-com-backoff-e-dlq.md) — retry com backoff exponencial e DLQ em tabela separada.
- [ADR-005](./ADR-005-at-least-once-com-x-event-id.md) — entrega at-least-once com `X-Event-Id` (o identificador gerado na inserção).
- [ADR-006](./ADR-006-reuso-dos-padroes-do-projeto.md) — reuso dos padrões do projeto (convenções de schema e estrutura do módulo
  `src/modules/webhooks/` que hospeda a função de publicação).
- [ADR-007](./ADR-007-snapshot-do-payload-na-insercao.md) — payload renderizado como snapshot na inserção e filtragem na origem (o que a
  linha da outbox guarda e quando ela deixa de ser criada).
