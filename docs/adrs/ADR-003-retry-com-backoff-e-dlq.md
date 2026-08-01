# ADR-003 — Retry com backoff exponencial de 5 tentativas e dead letter queue em tabela separada

- **Status:** Aceito
- **Data:** Quinta-feira, reunião técnica de definição (ver `TRANSCRICAO.md`)
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos), Marcos (Product Manager)
- **Contexto técnico:** Sistema de Webhooks de Notificação de Pedidos — worker de entrega e modelo de dados (`webhook_outbox`, `webhook_dead_letter`), Node.js + TypeScript + Prisma + MySQL

## Contexto

A feature nasce de um pedido formal de três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — que hoje fazem polling em `GET /orders` para descobrir mudança de status. A Atlas, especificamente, sinalizou que pode migrar para o concorrente se a entrega não sair até o fim do trimestre (`[09:00] Marcos:`). O evento passa a ser emitido via padrão outbox no MySQL e entregue por um worker próprio ([ADR-001](./ADR-001-outbox-no-mysql.md) e [ADR-002](./ADR-002-worker-separado-com-polling.md)). Isso resolve *como* o evento sai daqui, mas não resolve o que acontece quando o outro lado não atende.

E o outro lado vai não atender. Estamos chamando endpoints HTTP que rodam fora da nossa infraestrutura, em clientes cuja disponibilidade não controlamos — Diego lembrou que já tivemos cliente com indisponibilidade de duas horas em manutenção planejada (`[09:16] Diego:`). Como a escrita na outbox acontece dentro da mesma transação que muda o status do pedido, não existe a saída de "dar rollback e tentar mais tarde": o evento já está commitado e é responsabilidade nossa (`[09:04] Bruno:`).

Não há broker, fila gerenciada nem biblioteca de retry no projeto — as dependências de produção em `package.json` são Express, Prisma, Pino (com `pino-http`), Zod, JWT, bcrypt e `uuid`. Toda política de reentrega é código nosso, rodando sobre a mesma tabela que o worker único varre em polling. Isso cria três perguntas que precisavam de resposta antes de escrever qualquer linha:

1. Quantas vezes retentamos antes de declarar a entrega perdida?
2. Com que espaçamento, sabendo que retry apertado demais mata o cliente que está em janela de manutenção e retry frouxo demais atrasa a notificação de quem só teve um soluço de rede?
3. E o evento que esgotou as tentativas — some, fica marcado como lixo na fila de trabalho do worker, ou vira outra coisa?

Sem resposta à terceira, a primeira não faz sentido: retry sem destino de falha permanente é retry infinito, e evento pendurado para sempre na outbox é exatamente o que Diego apontou como problema do retry indefinido (`[09:15] Diego:`).

## Decisão

Usamos retry com backoff exponencial de no máximo **5 tentativas de entrega por evento**, com a progressão de intervalos **1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas** (`[09:17] Diego:`, fechado em `[09:17] Larissa:` e reafirmado no resumo em `[09:48] Larissa:`).

Cada intervalo é a espera que antecede uma retentativa: a entrega inicial falha em t=0, espera-se 1m para a tentativa 1, 5m para a tentativa 2, 30m para a tentativa 3, 2h para a tentativa 4 e 12h para a tentativa 5. São cinco intervalos para cinco tentativas, e a janela entre a primeira falha e a última tentativa é de ~14h36min — as "quase 15 horas" anunciadas por Diego em `[09:17] Diego:` e aceitas por Marcos em `[09:17] Marcos:`, dentro da faixa de "12 ou 24 horas" que Diego citara em `[09:15] Diego:`.

Enquanto há tentativas restantes, o evento permanece na `webhook_outbox`. Os estados `pendente`, `processando`, `falhou` e `entregue` e os índices de status e `created_at` (`[09:08] Diego:`) são decisão do [ADR-001](./ADR-001-outbox-no-mysql.md) e aqui só são consumidos: este ADR não redefine o modelo da outbox.

Esgotadas as 5 tentativas, o evento é considerado **falha permanente** e movido para a tabela separada **`webhook_dead_letter`**, que guarda a **payload**, o **motivo da falha** e o **timestamp** (`[09:18] Diego:`). A DLQ é persistida — não é log, não é métrica: é a evidência usada para debug e para reprocessamento.

O reprocessamento é **manual**, via endpoint administrativo **`POST /admin/webhooks/dead-letter/:id/replay`**, que recoloca o evento na outbox como pendente (`[09:18] Diego:`, `[09:19] Larissa:`). Não existe replay automático a partir da DLQ: uma vez que o evento chegou lá, alguém precisa decidir reenviá-lo.

Ambas as tabelas seguem as convenções de schema já vigentes no projeto (identificador UUID e `@@map` em snake_case) — decisão de reuso registrada no [ADR-006](./ADR-006-reuso-dos-padroes-do-projeto.md), não reaberta aqui.

Fora do escopo deste ADR: o que caracteriza uma tentativa como falhada no nível do transporte (timeout do HTTP call) é parâmetro não funcional e vive no FDD; a exigência de role `ADMIN` no endpoint de replay é decisão de controle de acesso e está no [ADR-008](./ADR-008-controle-de-acesso-dos-endpoints.md).

## Alternativas consideradas

### 3 tentativas — descartada

Proposta por Bruno como política mais agressiva: `[09:16] Bruno:` "3 não é melhor? Mais agressivo." Derrubada por Diego no mesmo minuto com um argumento operacional concreto: `[09:16] Diego:` "3 é pouco. Se o cliente teve indisponibilidade de manhã, a gente retentaria três vezes em 30 minutos e mataria. Já tinha cliente nosso com indisponibilidade de duas horas em manutenção planejada." Com 3 tentativas, a janela coberta cabe dentro de meia hora — bem abaixo das duas horas de manutenção planejada que Diego relatou como caso já observado em cliente nosso. A política morreria justamente nesse cenário.

### Retry indefinido com backoff — descartada

Levantada e descartada pelo próprio Diego ao propor o teto: `[09:15] Diego:` "Algumas pessoas defendem retry indefinido com backoff, mas isso traz o problema de evento ficar pendurado pra sempre se o cliente sumiu." O trade-off que a derrubou: sem teto, um cliente que simplesmente desligou o endpoint deixa linhas vivas na outbox indefinidamente, e o worker único continua gastando ciclo de polling com elas. O teto de 5 transforma "cliente sumiu" em um estado terminal e observável, em vez de um vazamento silencioso na fila de trabalho.

### Marcar `failed` na própria outbox, sem tabela separada — descartada

Foi a pergunta explícita de Larissa ao abrir o ponto: `[09:17] Larissa:` "Próximo: DLQ. Faz numa tabela separada ou marca como 'failed' na própria outbox?" Diego optou pela tabela separada em `[09:18] Diego:` — "Mais limpa a leitura da outbox principal, e fica como evidence pra debug e reprocessamento". O argumento é de separação de responsabilidades da tabela: a outbox é fila de trabalho do worker, e misturar nela o acervo permanente de falhas polui as consultas do caminho quente (a varredura de pendentes) com linhas que nunca mais serão processadas automaticamente. A tabela de DLQ guarda campos que a outbox não precisa ter — motivo da falha e timestamp da desistência.

## Consequências

**Positivas**

- Cliente com indisponibilidade real — inclusive manutenção planejada de duas horas, caso já observado — continua recebendo o evento sem intervenção humana. A janela de ~14h36min entre a primeira falha e a última tentativa cobre com folga larga as duas horas de indisponibilidade que Diego relatou em `[09:16] Diego:`.
- Evento que falhou definitivamente não desaparece: fica em `webhook_dead_letter` com payload e motivo, o que dá base para responder ao cliente o que aconteceu e para reenviar depois via replay.
- A outbox permanece como fila de trabalho enxuta: a varredura de pendentes do worker não compete com o histórico de falhas permanentes.
- Existe um caminho de recuperação explícito via API (`POST /admin/webhooks/dead-letter/:id/replay`), em vez de reprocessamento por `UPDATE` manual no banco; o controle de acesso e o registro de quem executou o replay ficam no [ADR-008](./ADR-008-controle-de-acesso-dos-endpoints.md).

**Negativas**

- **Um evento pode levar ~15 horas até ser declarado perdido.** Entre a primeira falha e a entrada na DLQ passam-se as cinco esperas (1m + 5m + 30m + 2h + 12h), ou seja, ~14h36min. Até lá a falha só existe no log e no histórico de entregas, que ninguém é obrigado a olhar, e o cliente B2B que depende do evento fica quase um dia sem ele. É um custo que Marcos aceitou conscientemente — `[09:17] Marcos:` "Se um cliente meu cair por 15 horas, ele já tá com problema sério dele. Acho aceitável." —, mas continua sendo um custo, e o intervalo final de 12h é o que domina a janela.
- **DLQ exige processo humano de operação.** Nada esvazia a `webhook_dead_letter` sozinho. Alguém precisa olhar a tabela, decidir o replay e executá-lo. Se ninguém olhar, a DLQ vira um cemitério silencioso e o cliente nunca recebe o evento — falhar permanentemente sem que ninguém perceba é o mesmo que perder o evento.
- **Ninguém é avisado automaticamente quando um webhook começa a falhar.** Marcos pediu alerta por e-mail após falhas consecutivas em `[09:37] Marcos:` e Larissa cortou: `[09:37] Larissa:` "Não. Email tá fora de escopo dessa fase." O único aviso previsto ao cliente é ele mesmo consultar o histórico de entregas.
- **Segunda tabela a manter.** `webhook_dead_letter` acumula payloads completos, entra em migration, entra em backup e passa a precisar de política de retenção própria — sendo que a política de arquivamento da própria outbox já foi declarada fora do escopo desta feature em `[09:08] Diego:`.
- Retenção de 5 tentativas por evento significa que o custo de um cliente instável é pago em ciclos do worker único: cada evento falhado volta à fila de trabalho cinco vezes antes de sair de cena.

**Neutras / limitações conhecidas**

- **Os intervalos são fixos, iguais para todos os clientes.** Os cinco intervalos valem igualmente para todos os endpoints cadastrados; a reunião não chegou a discutir diferenciação de política por cliente. Gatilho de reabertura: um cliente cronicamente instável cujo volume de reentregas passe a atrapalhar a entrega dos demais — conversa vizinha da que ficou registrada como "observar e decidir depois" sobre rate limiting de saída em `[09:39] Diego:` e `[09:39] Larissa:`.
- **Não há reprocessamento automático nem em lote a partir da DLQ.** O replay é um evento por vez, por chamada. Gatilho: incidente que jogue muitos eventos na DLQ de uma vez (queda longa de um cliente de alto volume) — aí o replay unitário deixa de ser operável e a decisão precisa ser revista.
- **Ausência de alerta é limitação assumida, não esquecimento.** Gatilho declarado na própria reunião: `[09:37] Larissa:` — "Talvez próxima fase, depois que a gente medir o impacto." Ou seja, o gatilho declarado é medir o impacto em produção — a reunião não definiu qual métrica nem qual limiar.
- O replay recoloca o evento na outbox como pendente, e o payload é o snapshot gravado na inserção original ([ADR-007](./ADR-007-snapshot-do-payload-na-insercao.md)). Um evento reenviado horas ou dias depois entrega o estado do pedido no momento em que o status mudou, não o estado atual — comportamento desejado, mas que precisa estar claro para quem operar o replay.

## Rastreabilidade

**Transcrição** (`TRANSCRICAO.md`)

- `[09:04] Bruno:` — "se o cliente tiver fora do ar, o que a gente faz, dá rollback na mudança de status? Não dá." — origem da necessidade de reentrega assíncrona.
- `[09:08] Diego:` — outbox com status `pendente`, `processando`, `falhou`, `entregue`, índices em status e `created_at`; arquivamento de entregues declarado fora do escopo.
- `[09:14] Larissa:` — abre o bloco de retry: "Se o cliente tá offline, o que a gente faz?"
- `[09:15] Diego:` — backoff exponencial, teto de tentativas, falha permanente move para DLQ.
- `[09:15] Bruno:` — "Quantas tentativas?"
- `[09:15] Diego:` — sugere 5; descarta retry indefinido (evento pendurado para sempre).
- `[09:16] Bruno:` — propõe 3 tentativas.
- `[09:16] Diego:` — derruba 3: mataria em 30 minutos; cliente já teve 2h de manutenção planejada.
- `[09:16] Larissa:` — "Cinco fica bom. Qual a progressão do backoff?"
- `[09:17] Diego:` — progressão 1m / 5m / 30m / 2h / 12h, quase 15 horas no total.
- `[09:17] Marcos:` — aceita a janela de 15 horas.
- `[09:17] Larissa:` — fecha 5 tentativas e a progressão; abre o dilema tabela separada vs. `failed` na outbox.
- `[09:18] Diego:` — `webhook_dead_letter` separada com payload, motivo e timestamp; evidência para debug e reprocessamento.
- `[09:18] Bruno:` — "E quem reprocessa? Tem endpoint?"
- `[09:18] Diego:` — replay manual via `POST /admin/webhooks/dead-letter/:id/replay`, recoloca na outbox como pendente.
- `[09:19] Larissa:` — registra o endpoint admin de replay manual.
- `[09:37] Marcos:` / `[09:37] Larissa:` — alerta por e-mail em falhas consecutivas pedido e adiado para a próxima fase.
- `[09:39] Diego:` / `[09:39] Larissa:` — rate limiting de saída fica como ponto em aberto ("observar e decidir depois").
- `[09:48] Larissa:` — resumo final confirma "Retry com backoff exponencial 1m/5m/30m/2h/12h, total 5 tentativas, depois DLQ persistida em tabela separada".

**Código**

- `package.json` — dependências de produção (`@prisma/client`, `express`, `pino`, `pino-http`, `zod`, `jsonwebtoken`, `bcrypt`, `uuid`): não há broker nem biblioteca de fila/retry no projeto; a política de reentrega é código próprio.
- `prisma/schema.prisma` — precedente de tabela dedicada a histórico (`model OrderStatusHistory` → `@@map("order_status_history")`); as convenções de identificador e nomenclatura que as novas tabelas seguem estão no [ADR-006](./ADR-006-reuso-dos-padroes-do-projeto.md).
- `src/modules/orders/order.service.ts:126` — `changeStatus`, a transação que passa a originar o evento reentregável (integração detalhada no [ADR-001](./ADR-001-outbox-no-mysql.md)).
- `prisma/schema.prisma` — **a alterar**: novos models `WebhookOutbox` (`@@map("webhook_outbox")`) e `WebhookDeadLetter` (`@@map("webhook_dead_letter")`).
- `src/modules/webhooks/webhook.processor.ts` — **a criar**: lógica de tentativa, cálculo do próximo intervalo de backoff e movimentação para a DLQ.

**Relacionado**

- [ADR-001](./ADR-001-outbox-no-mysql.md) — padrão outbox no MySQL (de onde vem o evento que é retentado).
- [ADR-002](./ADR-002-worker-separado-com-polling.md) — worker separado com polling de 2s (quem executa as tentativas).
- [ADR-006](./ADR-006-reuso-dos-padroes-do-projeto.md) — reuso dos padrões do projeto (convenções de schema que as duas novas tabelas seguem).
- [ADR-007](./ADR-007-snapshot-do-payload-na-insercao.md) — payload como snapshot na inserção (o que o replay reenvia).
- [ADR-008](./ADR-008-controle-de-acesso-dos-endpoints.md) — role `ADMIN` obrigatório no endpoint de replay.
