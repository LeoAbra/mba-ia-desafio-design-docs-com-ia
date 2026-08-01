# RFC — Sistema de Webhooks de Notificação de Pedidos

## 1. Metadados

| Campo | Valor |
| --- | --- |
| **Título** | Sistema de Webhooks de Notificação de Pedidos (outbound) |
| **Autor / Status** | Larissa — Tech Lead (`[09:50] Larissa:`) · Em revisão |
| **Data / Origem** | Quinta-feira, reunião técnica de definição — transcrição em [`TRANSCRICAO.md`](../TRANSCRICAO.md) |
| **Sistema afetado** | Order Management System — Node.js + TypeScript + Express + Prisma + MySQL |
| **Revisores** | Marcos (PM), Bruno (Eng. Pleno), Diego (Eng. Sênior), Sofia (Eng. Segurança); sessão técnica com Bruno e Diego antes de codar (`[09:50] Larissa:`) |
| **Pré-condição de deploy** | Revisão de segurança de HMAC e geração de segredo por Sofia, mínimo dois dias úteis (`[09:46] Sofia:`, `[09:47] Larissa:`) |

---

## 2. Resumo executivo (TL;DR)

Notificação outbound de mudança de status sem acoplar a disponibilidade dos clientes B2B à nossa
transação de negócio. Na mesma transação que muda o status, gravamos o evento já renderizado numa
tabela de outbox no MySQL que já temos; um processo Node separado, com Prisma próprio, lê os
pendentes em polling curto e entrega por HTTP assinado, com retentativas espaçadas e dead letter
queue. Garantia at-least-once com identificador estável de evento para o cliente deduplicar;
autenticidade por HMAC-SHA256 com segredo por endpoint. Não sobe infraestrutura nova e **não
adiciona nenhuma dependência** ([FDD §11.1](./FDD.md)).

---

## 3. Contexto e problema

A dor do cliente, o risco comercial e o prazo estão no [PRD §2](./PRD.md). Do produto chegam dois
parâmetros de fronteira: latência frouxa — "qualquer coisa abaixo de 10 segundos já é tempo real" —
e escopo estritamente outbound (`[09:02] Marcos:`, `[09:03] Sofia:`).

A restrição que trava a solução ingênua é o sistema que já roda: `OrderService.changeStatus`
(`src/modules/orders/order.service.ts:126`) abre um `prisma.$transaction` único e já pesado, e é
esse caminho que qualquer emissão de evento atravessa.

Uma chamada HTTP ali dentro põe a disponibilidade do cliente no caminho crítico e não tem saída se o
endpoint está fora do ar no instante do commit: "dá rollback na mudança de status? Não dá"
(`[09:04] Bruno:`). Empurrar o evento para fora da transação abre a janela em que o status mudou e o
evento não foi registrado. A restrição é dupla: **o registro do evento precisa ser atômico com a
mudança de status, e a entrega precisa ser assíncrona em relação a ela.**

Duas restrições de contorno pesaram e reaparecem na seção 5: nenhum apetite para operar mais um
componente de infraestrutura (`[09:07] Diego:`) e ausência de mecanismo nativo do MySQL para
notificar processo externo (`[09:09] Diego:`).

---

## 4. Proposta técnica

### Componentes e fluxo

1. **Produtor de evento** — chamado de dentro do `tx` de `changeStatus`, recebendo o cliente
   transacional em curso em vez de um repositório injetado (`[09:41] Diego:`); falha na gravação
   derruba a transação (`[09:40] Bruno:`).
2. **Outbox** — tabela no MySQL existente, alimentada na mesma transação
   ([ADR-001](./adrs/ADR-001-outbox-no-mysql.md)), guardando o evento **já renderizado** no instante
   da mudança ([ADR-007](./adrs/ADR-007-snapshot-do-payload-na-insercao.md)).
3. **Cadastro de webhooks** — endpoint, segredo e status por destino; a filtragem acontece **na
   inserção**: se nenhum destino quer aquele status, a linha nem é criada (`[09:34] Bruno:`).
4. **Worker de entrega** — processo Node **separado** da API, com `PrismaClient` próprio no mesmo
   banco ([ADR-002](./adrs/ADR-002-worker-separado-com-polling.md)), a criar (`[09:11] Larissa:`):
   lê os pendentes em polling curto, assina o corpo e entrega por HTTP com timeout
   (`[09:42] Diego:`).
5. **Retry e dead letter queue** — backoff em intervalos crescentes com teto de tentativas;
   esgotadas, o evento vai para tabela separada
   ([ADR-003](./adrs/ADR-003-retry-com-backoff-e-dlq.md)). Toda tentativa fica registrada com
   resultado, resposta e tempo (`[09:34] Marcos:`).
6. **Superfície HTTP** — módulo em `src/modules/webhooks/` **(a criar)**, no molde de
   `src/modules/orders/`: CRUD, rotação de segredo, histórico e replay da DLQ.

Parâmetro, formato, campo e comportamento detalhado são do [FDD](./FDD.md) §5–§7 e não se repetem
aqui.

### Por que o formato resolve

A atomicidade vem de graça porque a outbox mora no mesmo banco transacional do pedido: commit
implica evento registrado, rollback apaga o evento junto (`[09:06] Diego:`). O desacoplamento vem do
processo separado: cliente lento ou fora do ar afeta a fila, nunca a transação de negócio. E o
polling cabe com folga no orçamento de latência (`[09:10] Marcos:`) — folga que vale para a primeira
tentativa bem-sucedida, com a ressalva da seção 7.

**Contrato e segurança do envio.** Snapshot enxuto do pedido, sem itens, para não inflar
(`[09:43] Diego:`); HMAC-SHA256 sobre o corpo, com segredo por endpoint e janela de convivência na
rotação ([ADR-004](./adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)). TLS obrigatório e teto de
tamanho por evento a reunião classificou como requisito não funcional — valores em
[PRD §7](./PRD.md). Nada disso custa dependência nova.

---

## 5. Alternativas consideradas

Todas descartadas na reunião. A análise completa mora no ADR indicado; aqui fica o trade-off.

| Alternativa | O que ganharia | Por que foi descartada | Origem · ADR |
| --- | --- | --- | --- |
| Despacho síncrono dentro de `changeStatus` | Nenhum componente novo | Põe a disponibilidade do cliente no caminho crítico, e rollback de mudança legítima não é opção | `[09:04] Bruno:` · [001](./adrs/ADR-001-outbox-no-mysql.md) |
| Redis Streams ou fila externa | Consumo reativo, escala horizontal | Infraestrutura nova para operar num time pequeno; a outbox no MySQL já atende a latência | `[09:07] Diego:` · [001](./adrs/ADR-001-outbox-no-mysql.md) |
| Trigger de banco acordando o worker | Elimina o polling | MySQL não tem listener nativo para processo externo — trigger só executa SQL | `[09:09] Diego:` · [002](./adrs/ADR-002-worker-separado-com-polling.md) |
| 3 tentativas de entrega | Desiste mais cedo | Janela menor que a indisponibilidade já observada em cliente | `[09:16] Diego:` · [003](./adrs/ADR-003-retry-com-backoff-e-dlq.md) |
| Retry indefinido | Nunca desiste | Evento pendurado para sempre quando o cliente sumiu | `[09:15] Diego:` · [003](./adrs/ADR-003-retry-com-backoff-e-dlq.md) |
| DLQ como estado na própria outbox | Uma tabela a menos | Suja a leitura da outbox, que é fila de trabalho do worker | `[09:18] Diego:` · [003](./adrs/ADR-003-retry-com-backoff-e-dlq.md) |
| Entrega exactly-once | Dispensa dedup no cliente | Exige coordenação dos dois lados, complexidade desproporcional | `[09:25] Diego:` · [005](./adrs/ADR-005-at-least-once-com-x-event-id.md) |
| Segredo global da plataforma | Cadastro e rotação mais simples | Um vazamento comprometeria todos os clientes de uma vez | `[09:21] Sofia:` · [004](./adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) |
| Guardar só a referência e renderizar no envio | Tabela menor, sem dado duplicado | O evento refletiria o estado no envio, não no instante da mudança | `[09:52] Larissa:` · [007](./adrs/ADR-007-snapshot-do-payload-na-insercao.md) |

---

## 6. Questões em aberto

- **Quantas tentativas de entrega, afinal — e que janela isso compra.** Única questão que **bloqueia
  o go-live**, e única nascida de contradição *dentro* da reunião, não de omissão dela:
  `[09:17] Larissa:` e `[09:48] Larissa:` fecham "5 tentativas" **e** cinco intervalos, mas cinco
  chamadas consomem quatro esperas — e as "quase 15 horas" que Diego anunciou e Marcos aceitou
  (`[09:17] Diego:`) só se realizam se os cinco forem percorridos. **Nenhuma fala desempata, e o
  pacote não escolhe por conta própria.** As duas saídas estão dimensionadas: cinco tentativas →
  janela de **2h36min**, contra as 2h de manutenção planejada de `[09:16] Diego:`; cinco intervalos
  percorridos → **~14h36min**, que é o que a reunião acreditou aprovar. Até a escolha sair vale
  2h36min em todo o pacote — o que a Decisão do ADR-003 literalmente implementa. **Responsáveis:**
  Diego e Larissa. **Gatilho:** a revisão técnica de `[09:50] Larissa:`; o resultado vira emenda ao
  ADR-003 e se propaga a [PRD-RNF-04/05](./PRD.md), [PRD OBJ-4](./PRD.md),
  [FDD §13 R-7](./FDD.md) e ao default de `WEBHOOK_MAX_RETRIES`. — aritmética em
  [FDD §8.2](./FDD.md) · [ADR-003](./adrs/ADR-003-retry-com-backoff-e-dlq.md)

- **Rate limiting de saída.** Cliente com muitos pedidos mudando de status recebe uma rajada de
  chamadas. Aberto porque sem dado de produção qualquer limite seria arbitrário, e limitar saída
  conflita com o alvo de latência. **Gatilho:** primeira reclamação formal sobre volume, ou rajada
  observada em produção. — `[09:39] Diego:`, `[09:39] Larissa:`

- **Endurecimento do controle de acesso do CRUD.** Nesta fase qualquer usuário autenticado altera
  webhook de qualquer cliente; só o replay da DLQ exige `ADMIN`. Aberto porque a solução **não é uma
  regra de papel**: sem vínculo usuário↔cliente em `prisma/schema.prisma`, a verificação de posse é
  uma consulta que o modelo não permite fazer, e fechar o ponto exige alteração de schema antes de
  qualquer middleware. Causa raiz e exposição concreta em [PRD R-7](./PRD.md) e
  [FDD §13 R-9](./FDD.md). **Gatilho:** primeiro cliente com usuários próprios, ou incidente — mas a
  exposição já existe com o quadro de usuários internos atual. — `[09:37] Sofia:` ·
  [ADR-008](./adrs/ADR-008-controle-de-acesso-dos-endpoints.md)

- **Escala para múltiplos workers e ordenação global.** Com um worker só a ordem por pedido se
  sustenta; paralelizar quebra essa garantia implícita. Aberto porque ninguém pediu ordenação global
  (`[09:14] Marcos:`). **Gatilho:** o worker único deixar de vazar a fila dentro do alvo de latência
  em pico. — `[09:13] Diego:` · [ADR-002](./adrs/ADR-002-worker-separado-com-polling.md)

- **Arquivamento das linhas já entregues.** A outbox cresce indefinidamente; Diego mencionou
  arquivar "depois de 30 dias ou assim" e ele mesmo pôs o tema fora do escopo. Aberto por falta de
  dado de crescimento. **Gatilho:** primeira medição, ou degradação da consulta dos pendentes. —
  `[09:08] Diego:`

- **Qual segredo assina durante o grace period.** Na janela há dois segredos válidos, e não foi
  decidido se assinamos com o novo, o antigo, ou ambos em headers distintos — a reunião fechou a
  janela sem entrar no mecanismo. O modelo de dados suporta as três opções, então o FDD **não**
  fechou o ponto e o passo de assinatura fica bloqueado ([FDD §3.3](./FDD.md)). **Gatilho:** a
  revisão de segurança de `[09:46] Sofia:`, pré-condição de deploy. —
  [ADR-004](./adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)

**Fechados durante a redação do pacote, registrados para não parecerem esquecimento:** o replay
**preserva** o identificador de evento original ([FDD §5.5](./FDD.md)) — única leitura compatível
com [ADR-005](./adrs/ADR-005-at-least-once-com-x-event-id.md), e reverter exige ADR novo; e o
processador chama-se `webhook.processor.ts` ([FDD §10.1](./FDD.md)) — convenção, não arquitetura
(`[09:28] Bruno:`).

---

## 7. Impacto e riscos

### O que muda em código que já roda

A alteração crítica é uma só: a transação de `changeStatus`
(`src/modules/orders/order.service.ts`) ganha mais um ponto de escrita, tornando o caminho mais
quente do domínio de pedidos dependente de uma tabela nova. É o risco número um do RFC — defeito no
produtor de evento vira defeito na operação de pedidos —, e é intencional (`[09:41] Diego:`). O
resto é aditivo e reuso puro ([ADR-006](./adrs/ADR-006-reuso-dos-padroes-do-projeto.md)); lista
arquivo a arquivo em [FDD §10](./FDD.md).

### O que passa a existir para operar

Um **segundo processo em produção**, com deploy, supervisão e desligamento gracioso. Se ele morre,
nada quebra de imediato: a outbox acumula em silêncio e a falha só aparece como notificação que não
chegou — nenhum alerta foi decidido na reunião ([FDD §8.5](./FDD.md)). A DLQ é trabalho humano por
desenho: sem alerta automático (`[09:37] Larissa:`), alguém precisa olhar, e o replay é manual.
**Quanto tempo um evento leva até ser declarado perdido depende de uma divergência ainda aberta**
(seção 6). Some-se o polling perpétuo e o segredo recuperável por cadastro — daí a revisão de
segurança ser pré-condição de deploy (seção 1).

### Risco arquitetural: o orçamento de latência não fecha no limite

**Análise deste RFC, não conclusão da reunião.** Alvo de latência, polling e timeout HTTP foram
fixados separadamente e não somam: no limite que o próprio desenho admite, uma entrega bem-sucedida
ultrapassa o alvo sem que nada tenha falhado. Consequência arquitetural única — o compromisso de
latência **não pode ser declarado em valor absoluto, apenas em percentil sobre a primeira tentativa
bem-sucedida**, como faz o [PRD OBJ-1](./PRD.md). Aritmética e mitigação em [FDD §8.1.1](./FDD.md).

### O que muda para os consumidores

Inversão de responsabilidade operacional, e a consequência arquitetural é uma: **o cliente precisa
deduplicar por identificador de evento**, porque a garantia é at-least-once
([ADR-005](./adrs/ADR-005-at-least-once-com-x-event-id.md)), a entrega pode chegar fora de ordem em
retentativa e o snapshot é do momento da mudança, não do estado atual. Lista completa em
[PRD §9.3](./PRD.md).

---

## 8. Decisões relacionadas

Todas **aceitas e verificadas**; este RFC não as reabre e o que elas deixam em aberto está na
seção 6. Índice completo em [`docs/adrs/README.md`](./adrs/README.md).

| ADR | Decisão |
| --- | --- |
| [ADR-001](./adrs/ADR-001-outbox-no-mysql.md) | Padrão outbox no MySQL |
| [ADR-002](./adrs/ADR-002-worker-separado-com-polling.md) | Worker separado com polling |
| [ADR-003](./adrs/ADR-003-retry-com-backoff-e-dlq.md) | Retry com backoff e DLQ em tabela separada |
| [ADR-004](./adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) | HMAC-SHA256 com segredo por endpoint e grace period |
| [ADR-005](./adrs/ADR-005-at-least-once-com-x-event-id.md) | At-least-once com identificador de evento, dedup no cliente |
| [ADR-006](./adrs/ADR-006-reuso-dos-padroes-do-projeto.md) | Reuso dos padrões do projeto |
| [ADR-007](./adrs/ADR-007-snapshot-do-payload-na-insercao.md) | Snapshot do payload na inserção, filtragem na origem |
| [ADR-008](./adrs/ADR-008-controle-de-acesso-dos-endpoints.md) | Replay de DLQ exige `ADMIN`; CRUD exige apenas autenticação |

**Documentos irmãos:** [`FDD.md`](./FDD.md) — contratos, modelo de dados e fluxos.
[`PRD.md`](./PRD.md) — por que a feature existe e para quem. [`TRACKER.md`](./TRACKER.md) —
rastreabilidade item a item: 630 linhas ligando cada afirmação a um `[hh:mm] Nome:` ou a um caminho
de arquivo.

**Fora de escopo desta fase**, para não ser confundido com requisito: alerta por e-mail, painel
visual e webhooks de entrada — quem pediu, quem recusou e por quê em [PRD §5.2](./PRD.md).
