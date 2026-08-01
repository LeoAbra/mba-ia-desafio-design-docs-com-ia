# Architectural Decision Records

Um ADR aqui é o registro de **uma** decisão arquitetural já tomada: o que foi decidido, qual
dilema a exigiu, quais alternativas reais foram descartadas e com que argumento, e qual o
preço que estamos pagando por ela. O público-alvo é quem herdar o sistema daqui a dois anos e
perguntar "por que raios está assim?" — por isso cada ADR carrega, obrigatoriamente, uma
consequência negativa e a rastreabilidade da fonte (timestamp da reunião ou caminho de arquivo
verificado no repositório). Parâmetro de configuração e validação trivial **não** viram ADR:
timeout HTTP, tamanho máximo de payload, obrigatoriedade de TLS na URL e o conjunto de headers
foram classificados pela própria reunião como requisito não funcional e moram no FDD/PRD.

Os oito ADRs abaixo nasceram de uma única reunião técnica de definição do **Sistema de Webhooks
de Notificação de Pedidos** (quinta-feira, 09:00, ~55 minutos — transcrição literal em
[`TRANSCRICAO.md`](../../TRANSCRICAO.md)), com Larissa (Tech Lead), Marcos (PM), Bruno
(Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma) e Sofia (Eng. de Segurança).

**Convenção de nomenclatura:** `ADR-NNN-titulo-em-kebab-case.md`, numeração sequencial a partir
de `001`, sem reuso de número mesmo quando um ADR é descontinuado. O título afirma a decisão
(`outbox-no-mysql`), não o tema (`sobre-a-outbox`). O formato é MADR: cabeçalho com Status, Data,
Decisores e Contexto técnico, seguido de **Contexto**, **Decisão**, **Alternativas consideradas**,
**Consequências** e **Rastreabilidade**. Status possíveis: `Aceito`, `Proposto`,
`Substituído por ADR-XXX`, `Descontinuado`.

## Índice

| ADR | Decisão | Status |
| --- | --- | --- |
| [ADR-001](./ADR-001-outbox-no-mysql.md) | Padrão outbox no MySQL para emissão de eventos de pedido | Aceito |
| [ADR-002](./ADR-002-worker-separado-com-polling.md) | Worker em processo separado com polling de 2 segundos | Aceito |
| [ADR-003](./ADR-003-retry-com-backoff-e-dlq.md) | Retry com backoff exponencial de 5 tentativas e DLQ em tabela separada | Aceito |
| [ADR-004](./ADR-004-hmac-sha256-secret-por-endpoint.md) | Assinatura HMAC-SHA256 com segredo por endpoint e rotação com grace period | Aceito |
| [ADR-005](./ADR-005-at-least-once-com-x-event-id.md) | Entrega at-least-once com idempotência delegada via `X-Event-Id` | Aceito |
| [ADR-006](./ADR-006-reuso-dos-padroes-do-projeto.md) | Reuso dos padrões existentes do projeto no módulo de webhooks | Aceito |
| [ADR-007](./ADR-007-snapshot-do-payload-na-insercao.md) | Payload renderizado como snapshot na inserção e filtragem na origem | Aceito |
| [ADR-008](./ADR-008-controle-de-acesso-dos-endpoints.md) | Replay de DLQ exige role `ADMIN`; CRUD de webhooks exige apenas autenticação | Aceito |

## Resumo por ADR

- **[ADR-001](./ADR-001-outbox-no-mysql.md) — Padrão outbox no MySQL.** O evento é inserido em
  `webhook_outbox` dentro da mesma transação que muda o status do pedido, garantindo que commit
  implica evento registrado e rollback apaga o evento junto; nenhuma infraestrutura nova sobe.
  Preço: o throughput de eventos passa a depender do MySQL transacional e a tabela cresce sem
  política de arquivamento definida.
- **[ADR-002](./ADR-002-worker-separado-com-polling.md) — Worker separado com polling de 2s.**
  Processo Node próprio (`src/worker.ts` + `npm run worker`, ambos a criar) lê os pendentes mais
  antigos em lote pequeno a cada 2 segundos, com `PrismaClient` próprio. Preço: até 2s de latência
  por evento, consulta permanente ao banco, um segundo processo para operar e nenhuma garantia de
  ordenação global.
- **[ADR-003](./ADR-003-retry-com-backoff-e-dlq.md) — Retry com backoff e DLQ.** Cinco tentativas
  espaçadas em 1m/5m/30m/2h/12h e, esgotadas, o evento vai para `webhook_dead_letter` com payload,
  motivo e timestamp; replay é manual via endpoint admin. Preço: um evento leva **2h36min** para ser
  declarado perdido e a DLQ exige operação humana. **Divergência aritmética registrada no próprio
  ADR, com emenda pendente:** cinco chamadas consomem quatro esperas, então o intervalo de 12h é
  inalcançável e as "quase 15 horas" da reunião não se realizam. A pendência é rastreada em
  [RFC §6](../RFC.md), com responsáveis e gatilho.
- **[ADR-004](./ADR-004-hmac-sha256-secret-por-endpoint.md) — HMAC-SHA256 com segredo por
  endpoint.** Assinamos o corpo e enviamos em `X-Signature`; cada endpoint cadastrado tem segredo
  próprio, rotacionável pela API, com o segredo antigo válido em paralelo por 24 horas. Preço:
  janela de 24h com dois segredos válidos e material secreto recuperável armazenado por linha.
- **[ADR-005](./ADR-005-at-least-once-com-x-event-id.md) — At-least-once com `X-Event-Id`.**
  Garantimos at-least-once e delegamos a deduplicação ao cliente, que recebe em `X-Event-Id` o
  UUID gerado na inserção da outbox e estável entre retentativas. Preço: ônus de integração
  transferido ao consumidor, sem qualquer forma de verificar se ele deduplica.
- **[ADR-006](./ADR-006-reuso-dos-padroes-do-projeto.md) — Reuso dos padrões do projeto.**
  O módulo nasce em `src/modules/webhooks/` no molde de `src/modules/orders/`, com `AppError` e
  prefixo `WEBHOOK_`, `errorMiddleware`, `validate`, `authenticate`/`requireRole`, Pino e UUID
  `@db.Char(36)` reaproveitados sem alteração. **É o ADR ancorado no código**, com caminhos reais
  verificados. Preço: herda os limites dos padrões atuais fora do ciclo HTTP.
- **[ADR-007](./ADR-007-snapshot-do-payload-na-insercao.md) — Snapshot do payload na inserção.**
  A linha da outbox guarda o JSON já renderizado no instante da mudança de status, e a filtragem
  por status de interesse acontece na inserção — sem destinatário, nem cria linha. Preço: dado
  duplicado, tabela maior e evento enfileirado imutável na prática.
- **[ADR-008](./ADR-008-controle-de-acesso-dos-endpoints.md) — Controle de acesso dos endpoints.**
  `POST /admin/webhooks/dead-letter/:id/replay` exige role `ADMIN` via o `requireRole` existente e
  registra quem executou; o CRUD de configuração exige apenas autenticação. Preço: qualquer usuário
  autenticado pode cadastrar ou alterar o webhook de qualquer cliente — risco aceito nesta fase.

## Como os ADRs se relacionam

[ADR-001](./ADR-001-outbox-no-mysql.md) é a decisão-raiz: cria a outbox que
[ADR-002](./ADR-002-worker-separado-com-polling.md) consome,
[ADR-003](./ADR-003-retry-com-backoff-e-dlq.md) retenta,
[ADR-005](./ADR-005-at-least-once-com-x-event-id.md) identifica e
[ADR-007](./ADR-007-snapshot-do-payload-na-insercao.md) preenche.
[ADR-004](./ADR-004-hmac-sha256-secret-por-endpoint.md) e
[ADR-008](./ADR-008-controle-de-acesso-dos-endpoints.md) cobrem os dois lados da segurança —
autenticidade do que sai e autorização de quem opera —, e
[ADR-006](./ADR-006-reuso-dos-padroes-do-projeto.md) é a decisão transversal que fixa em que
convenções todos os outros são implementados.

## Fora de escopo desta fase

Registrado aqui para que não seja confundido com decisão: alerta por e-mail ao cliente com
webhook falhando (`[09:37] Larissa:` — "Email tá fora de escopo dessa fase"), rate limiting de
saída (`[09:39] Larissa:` — "observar e decidir depois"), dashboard visual para o cliente
(`[09:40] Larissa:`) e arquivamento das linhas já entregues da outbox (`[09:08] Diego:`).
