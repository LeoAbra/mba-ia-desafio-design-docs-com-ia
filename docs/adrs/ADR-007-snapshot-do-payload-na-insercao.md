# ADR-007 — Payload renderizado como snapshot na inserção e filtragem na origem

- **Status:** Aceito
- **Data:** Quinta-feira, reunião técnica de definição (ver `TRANSCRICAO.md`)
- **Decisores:** Larissa (Tech Lead), Bruno (Engenheiro Pleno, Pedidos), Diego (Engenheiro Sênior, Plataforma). A parte de filtragem na inserção foi fechada com Marcos (PM) ainda na call; a parte de snapshot do payload foi fechada nos minutos finais, depois que Marcos e Sofia saíram.
- **Contexto técnico:** tabela `webhook_outbox` (a criar) e módulo `src/modules/webhooks/` (a criar); ponto de escrita dentro de `src/modules/orders/order.service.ts` → `changeStatus`.

## Contexto

O evento de webhook nasce dentro da mesma transação que muda o status do pedido ([ADR-001](./ADR-001-outbox-no-mysql.md)). Isso deixa uma pergunta em aberto sobre o que exatamente essa linha da outbox carrega, e ela não é cosmética: define o que o worker precisa saber na hora de entregar.

A janela entre o registro do evento e a entrega não é curta. O retry vai até `1m/5m/30m/2h/12h` ([ADR-003](./ADR-003-retry-com-backoff-e-dlq.md)), então a última tentativa de um evento acontece **2h36min** depois do fato que o originou — e mais ainda se a emenda pendente do ADR-003 elevar o teto de tentativas. Nesse intervalo o pedido muda: o `PATCH /orders/:id/status` (`src/modules/orders/order.routes.ts:19`) permite novas transições a qualquer momento e cada uma delas grava um novo registro em `order_status_history`. Um evento que fosse remontado no instante do envio leria a order como ela está agora, não como estava quando o status mudou.

Bruno colocou o dilema em uma frase, [09:51]: o evento guarda o payload renderizado, ou guarda só `order_id` e renderiza na hora do envio?

Existe uma segunda pergunta do mesmo tipo, feita mais cedo na reunião. Cada endpoint de webhook cadastra a lista de status que quer ouvir ([09:33] Marcos). Se o cliente só quer `SHIPPED` e `DELIVERED`, alguém precisa descartar os outros — e a escolha é entre gastar linha na outbox e filtrar no envio, ou não gastar linha nenhuma. Diego levantou exatamente isso em [09:34]. As duas perguntas são a mesma decisão vista de dois ângulos: quanto trabalho fica na origem (dentro da transação de pedido, que Bruno já descreveu como pesada em [09:04] — atualiza `orders`, insere em `order_status_history` e decrementa `stock_quantity`) e quanto fica no worker.

## Decisão

A linha da `webhook_outbox` guarda o payload JSON já renderizado no momento da inserção. É um snapshot do estado do pedido no instante em que o status mudou, não uma referência a ser resolvida depois: mesmo que o pedido mude em seguida, o evento continua refletindo o estado de então ([09:52] Larissa). O worker não consulta a order para montar o corpo do request — ele lê a linha, assina e envia.

O snapshot é produzido dentro da mesma transação da mudança de status, pela função `publishWebhookEvent(tx, order, fromStatus, toStatus)` que Bruno propôs em [09:41], recebendo o `tx` client da transação em curso. O conteúdo do JSON é o payload enxuto definido em [09:43] por Diego — sem `items`, para não inflar a linha; a especificação campo a campo do payload e dos headers mora no FDD, não aqui.

A filtragem por status de interesse acontece na origem, junto com a inserção: se nenhum webhook ativo daquele customer escuta o status de destino, a linha não é criada ([09:34] Bruno — "Na inserção. Se nenhum webhook do customer quer aquele status, nem insere. Economiza linha na tabela", com concordância de Diego no mesmo minuto). A outbox só contém eventos que têm destinatário.

## Alternativas consideradas

### Guardar só `order_id` e renderizar no envio — descartada

Foi a outra metade da pergunta de Bruno em [09:51]. Deixaria a linha da outbox mínima e permitiria mudar o formato do payload sem tocar nas linhas já gravadas. Larissa derrubou em [09:52] com o argumento de fidelidade temporal: "Se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou. Senão tem caso esquisito". Diego endossou no mesmo minuto — "Concordo, snapshot na inserção" — e Bruno fechou em [09:52]: "Beleza, snapshot. Decidido". Não houve terceira opção sobre a mesa; a discussão durou três falas.

### Filtrar no momento do envio — descartada

Levantada por Diego em [09:34] como pergunta aberta ("Filtra na inserção do outbox ou na hora de mandar?"). Manteria a outbox como registro completo de tudo que aconteceu, independente de quem escuta. Bruno derrubou no mesmo minuto pelo custo de armazenamento: gravar linha que ninguém vai consumir gasta espaço à toa ([09:34] Bruno — "Na inserção. Se nenhum webhook do customer quer aquele status, nem insere. Economiza linha na tabela"). O custo de leitura não foi citado nesse ponto; é consequência derivada aqui, coerente com a preocupação de acúmulo que o próprio Bruno já havia levantado em [09:07] ("E performance? Se acumular muito evento na tabela, o worker não fica lento?") — linha que ninguém consome ainda assim entra na tabela que o worker varre em polling. Diego concordou em [09:34].

## Consequências

**Positivas**

- O que o cliente recebe corresponde ao estado do pedido no instante da transição, mesmo quando a entrega só acontece na quinta tentativa, 2h36min depois (progressão de [09:17] Diego, com a ressalva aritmética do [ADR-003](./ADR-003-retry-com-backoff-e-dlq.md)).
- O worker não precisa ler `orders` a cada tentativa de envio: uma leitura da outbox basta para montar o request completo. Menos consultas ao MySQL por retry, no mesmo banco que já sustenta a API.
- O payload persistido é literalmente o que foi enviado, o que torna consistentes por construção os dois artefatos de auditoria já decididos: a `webhook_dead_letter` com a payload e o motivo da falha ([09:18] Diego) e o histórico `GET /webhooks/:id/deliveries` com payload e response ([09:34] Marcos).
- Filtrar na origem mantém a outbox proporcional ao que tem destinatário, não ao volume total de mudanças de status da plataforma.

**Negativas**

- Duplicamos dado. Cada linha da outbox replica campos que já vivem em `orders` (`orderNumber`, `status`, `customerId`, `totalCents` — ver `prisma/schema.prisma`, model `Order`), e essa cópia é replicada de novo na `webhook_dead_letter` e no histórico de deliveries. O mesmo fato ocupa espaço em até três lugares.
- A tabela cresce mais rápido em bytes do que cresceria com uma linha de referência. Isso pressiona justamente a política de arquivamento de linhas entregues que Diego declarou fora do escopo desta feature em [09:08].
- Evento já enfileirado é imutável na prática: mudar o formato do payload passa a valer só para eventos novos. Um erro de renderização não se corrige com deploy — o que está na fila sai errado do mesmo jeito.
- A transação de mudança de status fica mais longa. Ela já faz `update` em `orders` (`src/modules/orders/order.service.ts:158`), `create` em `order_status_history` (`:159-167`) e `update` de estoque por item via `debitStock`/`replenishStock` (chamados em `:151-156`, implementados em `:204-243`); agora ganha a leitura da configuração de webhooks do customer, a serialização do JSON e a inserção na outbox. É custo cobrado no caminho crítico do pedido, não no worker.
- Filtrar na origem torna a decisão de "quem escuta o quê" irreversível para o passado: evento descartado na inserção não existe. Webhook cadastrado depois, ou que passe a escutar um status novo, não recebe nada do que já aconteceu.

**Neutras / limitações conhecidas**

- Não há versionamento de payload na outbox. Enquanto o formato não mudar, isso é irrelevante; o gatilho para reabrir é a primeira alteração incompatível de formato com a fila não vazia — nesse momento é preciso decidir entre drenar a fila antes do deploy ou marcar a versão na linha.
- O tamanho da linha é conhecido no momento da inserção, e não no envio. O payload é enxuto por decisão ([09:43] Diego, sem `items`), então fica confortavelmente abaixo do teto de tamanho de payload — que a reunião tratou como requisito não funcional e não como decisão arquitetural ([09:24] Larissa), e cujo valor mora no FDD. O gatilho para revisitar é qualquer pedido de enriquecimento do payload por parte dos clientes, que reaproximaria o snapshot desse limite e multiplicaria o custo por linha.
- A duplicação só passa a doer junto com o volume. O gatilho é a `webhook_outbox` virar problema de espaço ou de latência de varredura do worker — aí a política de arquivamento adiada em [09:08] deixa de ser opcional.

## Rastreabilidade

- Transcrição `[09:04] Bruno:` — a transação de mudança de status já é pesada: atualiza `orders`, insere em `order_status_history`, decrementa `stock_quantity`.
- Transcrição `[09:07] Bruno:` — preocupação com acúmulo de eventos na tabela e lentidão do worker.
- Transcrição `[09:08] Diego:` — arquivamento de linhas entregues após ~30 dias declarado fora do escopo desta feature.
- Transcrição `[09:17] Diego:` — backoff `1m/5m/30m/2h/12h`, quase 15h entre a primeira falha e a última tentativa.
- Transcrição `[09:18] Diego:` — `webhook_dead_letter` guarda a payload, o motivo da falha e o timestamp.
- Transcrição `[09:33] Marcos:` — filtro de eventos é a lista de status que o webhook quer ouvir; filtragem na hora de inserir na outbox.
- Transcrição `[09:34] Diego:` — "Filtra na inserção do outbox ou na hora de mandar?".
- Transcrição `[09:34] Bruno:` — "Na inserção. Se nenhum webhook do customer quer aquele status, nem insere. Economiza linha na tabela".
- Transcrição `[09:34] Diego:` — "Concordo".
- Transcrição `[09:34] Marcos:` — histórico de entregas com payload, response e tempo de resposta em `GET /webhooks/:id/deliveries`.
- Transcrição `[09:41] Bruno:` — proposta da função `publishWebhookEvent(tx, order, fromStatus, toStatus)` recebendo o `tx` da transação atual.
- Transcrição `[09:43] Diego:` — payload enxuto, sem `items`; cliente busca detalhe em `GET /orders/:id` se quiser.
- Transcrição `[09:51] Bruno:` — "o evento da outbox guarda o payload renderizado já, ou guarda só order_id e renderiza na hora do envio?".
- Transcrição `[09:52] Larissa:` — "Eu prefiro renderizado já, na hora da inserção. Se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou".
- Transcrição `[09:52] Diego:` — "Concordo, snapshot na inserção".
- Transcrição `[09:52] Bruno:` — "Beleza, snapshot. Decidido".
- Código: `src/modules/orders/order.service.ts:126-179` — `changeStatus`, a transação onde o snapshot passa a ser inserido (hoje: `tx.order.update`, `tx.orderStatusHistory.create`, débito/reposição de estoque).
- Código: `src/modules/orders/order.routes.ts:19-23` — `PATCH /:id/status`, a rota que permite o pedido continuar mudando depois do evento gerado.
- Código: `prisma/schema.prisma`, models `Order` e `OrderItem` — campos replicados no snapshot (`orderNumber`, `customerId`, `status`, `totalCents`) e os `items` deliberadamente deixados de fora.
- Código a criar: tabela `webhook_outbox` em `prisma/schema.prisma` e módulo `src/modules/webhooks/` com a função de publicação do evento.
- Relacionado: [ADR-001](./ADR-001-outbox-no-mysql.md) (outbox no MySQL, transação atômica), [ADR-003](./ADR-003-retry-com-backoff-e-dlq.md) (retry e DLQ — consumidores do payload persistido), [ADR-005](./ADR-005-at-least-once-com-x-event-id.md) (`X-Event-Id` gerado na inserção), [ADR-006](./ADR-006-reuso-dos-padroes-do-projeto.md) (padrões do projeto reaproveitados no módulo).
