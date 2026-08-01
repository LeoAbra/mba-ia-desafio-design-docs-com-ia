# ADR-005 — Entrega at-least-once com idempotência delegada via X-Event-Id

- **Status:** Aceito
- **Data:** Quinta-feira, reunião técnica de definição (ver `TRANSCRICAO.md`)
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos), Sofia (Eng. de Segurança), Marcos (Product Manager)
- **Contexto técnico:** Sistema de Webhooks de Notificação de Pedidos — contrato de entrega HTTP entre o worker de webhooks e o endpoint do cliente B2B (garantia de entrega e semântica de duplicidade)

## Contexto

O mecanismo de entrega já estava desenhado quando esta questão apareceu: evento gravado em outbox dentro da transação de mudança de status, worker separado lendo em polling, e uma política de retentativa para o caso de o cliente estar fora do ar. Diego resumiu o retry em `[09:15] Diego:` — backoff exponencial, teto de tentativas e DLQ ao fim. E o worker corta a chamada HTTP num timeout (`[09:42] Diego:` "10 segundos. Cliente lento que não responde em 10s a gente trata como falha e marca pra retry").

Esses dois fatos, juntos, criam um problema que não tem solução dentro do nosso processo: quando o worker não recebe resposta, ele não sabe se o cliente **não processou** o evento ou se processou e a resposta se perdeu no caminho (ou estourou os 10s). Se retentarmos, corremos o risco de o cliente processar o mesmo evento duas vezes. Se não retentarmos, corremos o risco de o evento nunca chegar — e a razão de existir da feature é justamente os três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) pararem de fazer polling no `GET /orders` (`[09:00] Marcos:`).

Não existe terceira opção sem coordenação com o outro lado. Diego colocou a escolha na mesa em `[09:24] Diego:`: "a gente vai garantir at-least-once. Pode acontecer de o cliente receber o mesmo evento duas vezes. Ele tem que estar preparado." Bruno imediatamente foi ao ponto prático em `[09:25] Bruno:` — "E como ele diferencia?". Sofia levantou a objeção de fundo em `[09:25] Sofia:` — "Isso joga responsabilidade pro cliente".

O dilema é esse: ou perdemos evento, ou entregamos evento repetido; e se entregamos repetido, alguém precisa deduplicar — nós, com estado e coordenação, ou o cliente, com um identificador que a gente forneça. A escolha foi fechada em menos de dois minutos de discussão (`[09:24] Diego:` a `[09:26] Larissa:`), sem que custo de infraestrutura ou prazo fossem invocados como argumento — o que pesou foi apenas a impossibilidade técnica de distinguir os dois casos sem coordenação com o cliente.

## Decisão

Garantimos **at-least-once**. O cliente pode receber o mesmo evento mais de uma vez, e a deduplicação é responsabilidade dele.

Para viabilizar essa deduplicação, cada evento carrega um identificador próprio:

- **Header:** `X-Event-Id`
- **Valor:** um **UUID** gerado no momento em que o evento entra na outbox — não no momento do envio (`[09:25] Diego:` "um UUID gerado quando o evento entra na outbox. É único por evento"). O mesmo UUID viaja em todas as retentativas do mesmo evento; é isso que torna a dedup do lado do cliente possível.
- **Unicidade:** por evento, não por pedido. Um pedido que passa por PAID, PROCESSING e SHIPPED gera até três eventos e três `X-Event-Id` distintos — só vira linha na outbox o status que algum webhook do cliente assinou (ver [ADR-007](./ADR-007-snapshot-do-payload-na-insercao.md)).
- **Formato:** UUID, fixado por Diego junto com o header (`[09:25] Diego:` "um UUID gerado quando o evento entra na outbox"). É coerente com o padrão de identificador das entidades de domínio do projeto em `prisma/schema.prisma` (`@id @default(uuid()) @db.Char(36)`). A escolha do tipo da chave primária das tabelas do módulo — UUID em vez de auto incremental, decidida por `[09:51] Larissa:` — foi tratada em separado e pertence ao [ADR-006](./ADR-006-reuso-dos-padroes-do-projeto.md).
- **Contrato com o cliente:** o cliente guarda os `X-Event-Id` já processados e descarta repetição. Não fazemos nenhuma verificação do lado de cá para saber se ele fez isso.

O ônus de integração é assumido explicitamente e compensado por documentação: Marcos se comprometeu a destacar o comportamento no portal do desenvolvedor (`[09:26] Marcos:` "Eu posso documentar isso bem destacado no portal de desenvolvedor pros clientes, sem problema"). Larissa fechou em `[09:26] Larissa:` — "At-least-once com X-Event-Id pra dedup do lado do cliente. Decisão." — e reafirmou no resumo final (`[09:48] Larissa:` "Idempotência por X-Event-Id, garantia at-least-once").

## Alternativas consideradas

### Exactly-once com coordenação bilateral — descartada

Entregar cada evento exatamente uma vez, garantido pela plataforma. Foi Diego quem levantou e derrubou, respondendo à objeção da Sofia em `[09:25] Diego:`: "Joga, mas é o padrão de mercado. Stripe faz assim, GitHub faz assim. Garantir exactly-once exigiria coordenação dos dois lados e fica muito mais complexo. At-least-once com event_id resolve 99% dos casos."

O trade-off que a matou, nas palavras de Diego, é que exactly-once "exigiria coordenação dos dois lados e fica muito mais complexo" (`[09:25] Diego:`); a reunião não chegou a detalhar que forma essa coordenação teria. Ou seja, exigiria que os três clientes B2B implementassem algo específico nosso — o oposto de reduzir o custo de integração deles, que é o objetivo declarado da feature (`[09:00] Marcos:`). Ninguém sustentou a alternativa depois da resposta de Diego, e ela não voltou nem no resumo final de `[09:48] Larissa:`.

### Deduplicação do nosso lado — descartada

A alternativa nunca chegou a ser formulada como proposta técnica; ela existe como a objeção de Sofia em `[09:25] Sofia:` — "Isso joga responsabilidade pro cliente" — e é registrada aqui porque é a única forma de tirar o ônus do consumidor. Diego respondeu no mesmo minuto (`[09:25] Diego:`) e ninguém sustentou a objeção depois; Marcos aceitou compensar com documentação (`[09:26] Marcos:`) e Larissa fechou (`[09:26] Larissa:`).

Registro honesto: a alternativa não teve defensor após a resposta de Diego. Deduplicar do nosso lado só resolveria reenvio originado em bug interno — não resolve o caso real, que é resposta perdida depois de o cliente já ter processado. Esse caso é indistinguível de fora, e por isso nenhum estado do nosso lado o cobriria.

## Consequências

**Positivas**

- Podemos retentar sem medo. A política de retry (5 tentativas, backoff 1m/5m/30m/2h/12h, janela de ~14h36min entre a primeira falha e a última tentativa — ver [ADR-003](./ADR-003-retry-com-backoff-e-dlq.md)) só é segura porque duplicidade é comportamento previsto no contrato, não incidente.
- Zero estado de deduplicação do nosso lado: nenhuma tabela de IDs processados, nenhuma janela de retenção, nenhum lock entre worker e cliente.
- O contrato é o mesmo de Stripe e GitHub (`[09:25] Diego:`), o que dá um precedente conhecido para explicar o comportamento no portal do desenvolvedor em vez de justificar um formato próprio.
- O esforço do nosso lado se resume a gerar e propagar um UUID que já existe na linha da outbox: nenhuma tabela de dedup, nenhum job de expiração e nenhum protocolo novo entram nas três sprints estimadas (`[09:46] Larissa:`, `[09:47] Larissa:`).

**Negativas**

- **Transferimos custo de integração para o cliente.** Foi apontado na hora por Sofia (`[09:25] Sofia:`). O cliente que não implementar dedup vai processar o mesmo evento duas vezes, e o efeito colateral disso acontece dentro do sistema dele — fora do nosso alcance para detectar ou corrigir.
- Não temos como saber se o cliente deduplica. Não há verificação, métrica ou alerta possível do nosso lado; a corretude da ponta final é presumida.
- Suporte vai receber o chamado "recebi o mesmo webhook duas vezes" e a resposta correta é "é comportamento esperado". Isso só não vira atrito se a documentação de Marcos no portal (`[09:26] Marcos:`) existir antes do primeiro cliente entrar em produção — a decisão cria uma dependência de entrega fora do time de engenharia.
- A decisão amarra o contrato público. Passar a prometer exactly-once depois significaria mudar o contrato dos três clientes que já integraram.

**Neutras / limitações conhecidas**

- **A dedup só funciona se o `X-Event-Id` for estável entre retentativas.** O UUID é gerado na inserção da outbox (`[09:25] Diego:`), logo as retentativas do mesmo evento reusam o mesmo valor. Gatilho de reabertura: qualquer mudança que gere um novo identificador para um evento já enviado quebra a garantia na origem.
- **O replay manual de DLQ era um ponto em aberto, resolvido na modelagem.** A reunião definiu que o replay recoloca o evento na outbox como pendente (`[09:18] Diego:`), mas não discutiu se o `X-Event-Id` original é preservado — e gerar UUID novo faria o cliente que deduplica corretamente reprocessar assim mesmo. O FDD, seção 5.5, fechou em **preservar o identificador original**, por ser a única leitura compatível com a garantia central deste ADR. Gatilho de reabertura: qualquer proposta de gerar id novo no replay volta como ADR, não como alteração de FDD.
- **A garantia at-least-once é aceitável porque o evento é uma notificação de estado, não um comando.** O payload é um snapshot do pedido no instante da mudança (ver [ADR-007](./ADR-007-snapshot-do-payload-na-insercao.md)) e o cliente pode sempre buscar a verdade em `GET /orders/:id` (`[09:43] Diego:`). Gatilho de reabertura: se algum evento futuro carregar semântica de comando com efeito colateral não idempotente, ou se um cliente exigir exactly-once em contrato, a decisão volta para a mesa.
- Duplicidade não é falha operacional. Nenhum alerta, dashboard ou métrica deve tratar reenvio como incidente.

## Rastreabilidade

**Transcrição** (`TRANSCRICAO.md`)

- `[09:15] Diego:` — backoff exponencial com teto de tentativas e DLQ; é o retry que torna a duplicidade inevitável.
- `[09:24] Diego:` — "a gente vai garantir at-least-once. Pode acontecer de o cliente receber o mesmo evento duas vezes. Ele tem que estar preparado."
- `[09:25] Bruno:` — "E como ele diferencia?" — a pergunta que exigiu o identificador.
- `[09:25] Diego:` — `X-Event-Id` com UUID gerado quando o evento entra na outbox; único por evento; dedup pelo event_id do lado do cliente.
- `[09:25] Sofia:` — "Isso joga responsabilidade pro cliente." — objeção registrada.
- `[09:25] Diego:` — padrão de mercado (Stripe, GitHub); exactly-once exigiria coordenação dos dois lados; at-least-once resolve 99% dos casos.
- `[09:26] Marcos:` — compromisso de documentar o comportamento em destaque no portal do desenvolvedor.
- `[09:26] Larissa:` — "At-least-once com X-Event-Id pra dedup do lado do cliente. Decisão."
- `[09:42] Diego:` — timeout de 10s no envio (RNF; documentado no FDD) — fonte prática de reenvio após processamento bem-sucedido do cliente.
- `[09:48] Larissa:` — resumo final confirma "Idempotência por X-Event-Id, garantia at-least-once".

**Código**

- `prisma/schema.prisma` — padrão de identificador das entidades de domínio: `@id @default(uuid()) @db.Char(36)` em `User` (linha 26), `Customer` (41), `Product` (57), `Order` (75), `OrderItem` (100) e `OrderStatusHistory` (117). Única exceção no schema: `OrderNumberSequence` (linha 134), tabela de controle interno, que usa `Int @id @default(1)`. É o formato das entidades de domínio que o `X-Event-Id` segue.
- `src/modules/orders/order.service.ts` — `OrderService.changeStatus` (a partir da linha 126): transação atual que atualiza `orders`, insere em `order_status_history` e ajusta estoque. É onde o evento (e portanto o UUID do `X-Event-Id`) passa a ser gerado.
- `src/modules/webhooks/` — módulo **a criar**; o envio HTTP com os headers do contrato vive no processador do worker.
- Tabela `webhook_outbox` — **a criar**; guarda o UUID do evento, que é o valor enviado em `X-Event-Id`.

**Relacionado**

- [ADR-001](./ADR-001-outbox-no-mysql.md) — outbox no MySQL (origem do evento e do identificador).
- [ADR-003](./ADR-003-retry-com-backoff-e-dlq.md) — retry com backoff e DLQ (a política que produz as duplicatas; e o replay, ponto em aberto acima).
- [ADR-004](./ADR-004-hmac-sha256-secret-por-endpoint.md) — HMAC-SHA256 com segredo por endpoint (o mesmo request carrega `X-Signature` e `X-Event-Id`).
- [ADR-006](./ADR-006-reuso-dos-padroes-do-projeto.md) — reuso dos padrões do projeto (convenção de UUID que o identificador de evento segue).
- [ADR-007](./ADR-007-snapshot-do-payload-na-insercao.md) — payload como snapshot na inserção (por que o evento é notificação de estado, não comando).
