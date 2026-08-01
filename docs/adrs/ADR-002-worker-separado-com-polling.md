# ADR-002 — Worker em processo separado com polling de 2 segundos

- **Status:** Aceito
- **Data:** Quinta-feira, reunião técnica de definição (ver `TRANSCRICAO.md`)
- **Decisores:** Larissa (Tech Lead), Diego (Engenheiro Sênior, Plataforma), Bruno (Engenheiro Pleno, time de Pedidos), Marcos (Product Manager, de acordo com a latência em `[09:10]`)
- **Contexto técnico:** Sistema de Webhooks de Notificação de Pedidos — consumo da tabela `webhook_outbox` e topologia de processos da aplicação (Node.js + TypeScript + Express + Prisma + MySQL)

## Contexto

A emissão do evento já estava resolvida: o evento é gravado em `webhook_outbox` dentro da
mesma transação que muda o status do pedido (ver [ADR-001](./ADR-001-outbox-no-mysql.md)). Fechada essa parte, Larissa
abriu o ponto seguinte de forma explícita — `[09:08] Larissa:` *"Tá decidido então: outbox
em MySQL. Próximo ponto: como o worker lê isso?"*.

O problema é que a tabela de outbox é passiva: linha inserida não avisa ninguém. Alguém
precisa descobrir que existe evento pendente e disparar o HTTP. E as restrições que
cercavam esse "alguém" eram todas concretas:

- **Banco é MySQL, não Postgres.** Não existe primitiva de notificação de processo externo.
- **Time pequeno e prazo curto.** A Atlas quer a feature para fim de novembro
  (`[09:45] Marcos:`) e a estimativa saiu em três sprints (`[09:46] Larissa:` — *"Eu chuto
  três sprints incluindo revisão da Sofia"*), fechada em `[09:47] Larissa:`. Não há
  espaço para subir componente de infraestrutura novo — a mesma restrição que já tinha
  derrubado Redis no ponto anterior (`[09:07] Diego:` *"a gente é um time pequeno"*).
- **Orçamento de latência folgado.** Os clientes B2B consideram "tempo real" qualquer coisa
  abaixo de 10 segundos (`[09:02] Marcos:`). Não é requisito de sub-segundo.
- **A API existente é um processo Express que reinicia.** `src/server.ts` sobe o servidor,
  registra `SIGINT`/`SIGTERM` e chama `process.exit(0)` no shutdown. Qualquer coisa
  hospedada dentro desse processo morre junto a cada deploy ou restart.

O dilema: colocar o consumo de eventos onde? Dentro da API que já existe, e aceitar que ele
suma junto com ela? Ou pagar um segundo processo para operar? E como esse consumidor
descobre que há trabalho — reagindo a um sinal do banco, ou perguntando de tempos em tempos?

## Decisão

Usamos um **worker em processo Node separado da API**, que consome a outbox por **polling
em loop a cada 2 segundos**.

Parâmetros concretos da decisão:

- **Intervalo de polling: 2 segundos.** A cada ciclo o worker busca os eventos pendentes
  mais antigos, processa e marca — `[09:09] Diego:` *"Polling em loop. A cada 2 segundos,
  busca os eventos pendentes mais antigos, processa, marca."*
- **Lote pequeno por ciclo**, lendo apenas as linhas em estado pendente. A leitura se apoia
  nos índices de status e `created_at` definidos no [ADR-001](./ADR-001-outbox-no-mysql.md) — é dependência deste worker, não
  decisão deste ADR.
- **Ordem de consumo: `created_at` da outbox**, crescente.
- **Worker único (single-worker).** Uma instância do processo, não N em paralelo
  (`[09:12] Diego:`).
- **Garantia de ordenação: por `order_id`**, implícita e válida enquanto houver um único
  worker. Não há garantia de ordenação global (`[09:12] Diego:`, `[09:13] Larissa:`).
- **Processo separado, não thread/timer dentro da API** — `[09:11] Diego:` *"o worker tem
  que rodar como processo separado, não dentro da mesma instância da API. Senão se a API
  reinicia, perde o worker."*
- **Entry-point próprio no mesmo repositório**, no molde do que já existe: `src/worker.ts`
  (a criar) espelhando `src/server.ts`, mais um script `npm run worker` (a criar em
  `package.json`) — `[09:11] Larissa:`.
- **Mesma stack, mesmo banco, mesma `DATABASE_URL`, `PrismaClient` próprio.** O worker
  instancia seu próprio client porque é outro processo Node — `[09:30] Bruno:` *"Separado.
  PrismaClient é por processo. Mesmo banco, mesma DATABASE_URL, mas instância nova porque é
  outro processo Node."* A fábrica `createPrismaClient()` de `src/config/database.ts` já
  existe e serve a esse uso.

A latência adicional de entrega passa a ter o intervalo de polling como teto, e isso foi
aceito explicitamente — `[09:10] Larissa:` *"Worker em polling, 2s. A latência mínima vai ser 2
segundos no pior caso. Aceitamos."*, com o de-acordo do produto em `[09:10] Marcos:` *"2
segundos serve, perfeito."*

## Alternativas consideradas

### Trigger de banco / mecanismo reativo — descartada

Levantada por Bruno em `[09:09] Bruno:` — *"Não dá pra usar trigger do banco pra ser mais
reativo?"*.

Derrubada por Diego no mesmo minuto, `[09:09] Diego:`: *"MySQL não tem listener nativo tipo
o NOTIFY/LISTEN do Postgres. Trigger no banco a gente até tem, mas ela não notifica processo
externo, ela só executa SQL. Pra avisar o worker, a gente teria que improvisar algo tipo
escrever em arquivo ou bater num endpoint, fica esquisito."* O argumento é de capacidade do
motor, não de gosto: a trigger do MySQL roda SQL dentro do banco e não tem como acordar um
processo Node. Qualquer ponte seria improviso. E o ganho que esse improviso compraria —
alguns segundos de latência — não é necessário: *"Polling de 2 segundos atende o requisito
de 'abaixo de 10 segundos' tranquilo."*

### Worker dentro do processo da API — descartada

É o caminho de menor esforço: hospedar o loop de polling dentro da mesma instância Express
que já está no ar. A reunião não chegou a discutir o mecanismo concreto dessa hospedagem —
ela foi eliminada antes disso. Ninguém defendeu a opção; Diego a eliminou preventivamente
em `[09:11] Diego:` com o argumento operacional — *"Senão se a API reinicia, perde o
worker."* O ciclo de vida do consumo de eventos ficaria amarrado ao ciclo de vida do servidor
HTTP, que é exatamente o que `src/server.ts` faz hoje ao derrubar tudo em `SIGTERM`.

Ninguém chegou a checar a viabilidade prática desta opção — a conversa seguiu direto para o
processo separado. As falas de `[09:11] Bruno:` (*"vai precisar conectar no mesmo banco e
usar o mesmo Prisma client"*) e de `[09:11] Diego:` (*"Sim, mesmo banco, mesma stack. Só não
pode ser o mesmo processo."*) são sobre o **worker separado** proposto por Larissa, não sobre
esta alternativa; a segunda apenas delimita o que o worker separado compartilha com a API. A
ressalva de Bruno sobre "o mesmo Prisma client" foi revista por ele próprio em
`[09:30] Bruno:` — instância de `PrismaClient` separada, por ser outro processo Node.

### Múltiplos workers em paralelo — adiada, não descartada

Bruno perguntou pelo caminho de escala em `[09:13] Bruno:` — *"E se algum dia a gente quiser
escalar?"*. Diego respondeu com as saídas técnicas e com o veredito de escopo,
`[09:13] Diego:`: *"Aí dá pra particionar por order_id, ou usar lock pessimista. Mas isso é
problema do futuro, não agora."* Não é uma alternativa rejeitada por mérito técnico: é uma
decisão adiada por falta de necessidade atual, reforçada por `[09:14] Marcos:` — *"Os
clientes nunca pediram garantia de ordering global, eles só querem saber se cada pedido
deles mudou."*

## Consequências

**Positivas**

- Zero infraestrutura nova. O worker roda na mesma stack, contra o mesmo MySQL, com o mesmo
  Prisma. Cabe no orçamento de três sprints e no tamanho do time.
- Restart, deploy ou crash da API não interrompe a entrega de webhooks, e vice-versa: são
  ciclos de vida independentes.
- Enquanto for worker único, o consumo em ordem de `created_at` entrega os eventos de um
  mesmo pedido na sequência em que os status mudaram. É bônus, não requisito: `[09:14] Marcos:`
  — *"Os clientes nunca pediram garantia de ordering global, eles só querem saber se cada
  pedido deles mudou."*
- O caminho de um evento é inspecionável com um `SELECT`: o estado de cada entrega está na
  própria linha da `webhook_outbox`, com o `created_at` que define a ordem de consumo. Não há
  componente intermediário a consultar nem estado de entrega guardado fora do MySQL.

**Negativas**

- **Todo evento espera até 2 segundos antes da chamada HTTP.** É o teto do atraso introduzido
  pelo polling, não a média nem o piso: o evento inserido logo depois de um ciclo espera o
  ciclo inteiro; o inserido pouco antes sai quase de imediato. Foi assim que a reunião aceitou
  o número — `[09:10] Larissa:` fala em 2 segundos "no pior caso". A folga do requisito (10s)
  é o que torna isso tolerável — e é folga que gastamos aqui.
- **Consultamos a `webhook_outbox` a cada 2 segundos para sempre**, inclusive na madrugada,
  inclusive quando não há nada pendente. É carga constante e permanente no mesmo MySQL que
  serve a API de pedidos.
- **Não há garantia de ordenação global** entre pedidos diferentes. Só a ordenação por
  `order_id`, e apenas por consequência de existir um único worker — não por desenho.
- **O worker único é ponto único de falha silencioso.** Se ele cai, a API continua aceitando
  mudanças de status e continua inserindo na outbox; nenhum cliente recebe nada e nenhuma
  requisição falha. O acúmulo é o único sintoma. Isso exige supervisão de processo e
  alarme de fila pendente — trabalho de operação que hoje não existe.
- **Passamos de um processo para dois em produção.** Deploy, supervisão, rotação de log,
  métricas e on-call agora cobrem `npm start` e `npm run worker`. O custo é operacional e é
  permanente.
- **Um pool de conexões MySQL a mais.** O `PrismaClient` do worker é instância própria
  (`[09:30] Bruno:`), então o banco passa a atender dois conjuntos de conexões.
- **O teto de throughput é o de um processo.** Um pico de mudanças de status é absorvido pela
  outbox, mas drenado no ritmo de um único consumidor.

**Neutras / limitações conhecidas**

- **Ordenação garantida apenas por `order_id`, enquanto for single-worker.** Registrada como
  limitação explícita em `[09:13] Larissa:` — *"Documentamos como limitação conhecida. Não é
  garantia de ordering global, só por order_id e enquanto for single-worker."*
  **Gatilho de reabertura:** no momento em que subirmos um segundo worker em paralelo, a
  garantia cai e precisa ser reconstruída — particionamento por `order_id` ou lock pessimista
  (`[09:13] Diego:`).
- **Intervalo de 2 segundos é calibrado contra o requisito de 10 segundos** (`[09:02]
  Marcos:`). **Gatilho de reabertura:** cliente que exija latência sub-segundo, ou revisão do
  SLA de notificação para baixo — aí polling deixa de ser a ferramenta certa e o problema do
  mecanismo reativo volta à mesa.
- **Escalar horizontalmente é problema declarado do futuro** (`[09:13] Diego:`). **Gatilho de
  reabertura:** o volume de eventos pendentes crescer mais rápido do que um worker consegue
  drenar em regime.
- **Rate limiting de saída ficou em aberto.** Diego levantou o risco de bombardear um cliente
  com dezenas de chamadas em sequência (`[09:38] Diego:`); a definição foi observar antes de
  implementar — `[09:39] Larissa:` *"Fica como 'observar e decidir depois'."* Se virar
  problema, o ponto exige decisão própria — a reunião não o atribuiu a nenhum ADR.

## Rastreabilidade

**Transcrição**

- `[09:02] Marcos:` — clientes consideram "tempo real" qualquer coisa abaixo de 10 segundos.
- `[09:07] Diego:` — time pequeno; restrição que veta infraestrutura adicional.
- `[09:08] Larissa:` — abre o ponto: "Próximo ponto: como o worker lê isso?".
- `[09:08] Diego:` — índices de status e `created_at` (decisão registrada no [ADR-001](./ADR-001-outbox-no-mysql.md)); worker
  lê pendentes em batch pequeno.
- `[09:09] Diego:` — polling em loop, a cada 2 segundos, pendentes mais antigos.
- `[09:09] Bruno:` — propõe trigger de banco para ser mais reativo.
- `[09:09] Diego:` — MySQL não tem NOTIFY/LISTEN; trigger não notifica processo externo.
- `[09:10] Marcos:` — "2 segundos serve, perfeito".
- `[09:10] Larissa:` — registra a decisão e aceita a latência mínima de 2s.
- `[09:11] Diego:` — worker em processo separado; se a API reinicia, perde o worker.
- `[09:11] Larissa:` — entry-point nova no molde de `src/server.ts`: `src/worker.ts` e script
  `npm run worker`.
- `[09:11] Bruno:` — mesmo banco e mesmo Prisma client (revisto por ele em `[09:30]`).
- `[09:11] Diego:` — mesma stack, mesmo banco, só não pode ser o mesmo processo.
- `[09:12] Larissa:` — levanta a questão de ordenação em mudanças rápidas de status.
- `[09:12] Diego:` — ordem por `created_at` com worker único; paralelismo quebraria a
  garantia; single-worker e ordering implícita por `order_id`.
- `[09:13] Bruno:` — e se quisermos escalar?
- `[09:13] Diego:` — particionar por `order_id` ou lock pessimista; problema do futuro.
- `[09:13] Larissa:` — documenta como limitação conhecida.
- `[09:14] Marcos:` — clientes nunca pediram ordering global.
- `[09:29] Diego:` — pergunta se o worker abre o mesmo `PrismaClient` ou um separado.
- `[09:30] Bruno:` — separado; `PrismaClient` é por processo, mesma `DATABASE_URL`.
- `[09:45] Marcos:` — prazo de fim de novembro.
- `[09:46] Larissa:` — estimativa em voz alta ("chuto três sprints"); `[09:47] Larissa:` fecha
  em três sprints com a revisão da Sofia incluída.
- `[09:48] Larissa:` — resumo final confirma "Worker separado em polling de 2 segundos".

**Código**

- `src/server.ts` — entry-point atual da API: `bootstrap()`, `app.listen`, shutdown em
  `SIGINT`/`SIGTERM` com `prisma.$disconnect()` e `process.exit(0)`. É o molde citado por
  Larissa e a prova de que um worker hospedado nesse processo morreria a cada restart.
- `src/config/database.ts` — expõe `createPrismaClient()` e a instância singleton `prisma`.
  A fábrica é o ponto de reuso para o `PrismaClient` próprio do worker.
- `src/shared/logger/index.ts` — `createLogger()` (Pino, com `redact`) e o `logger` singleton;
  `src/server.ts` importa o singleton. O worker usa a mesma infraestrutura de log.
- `package.json` — scripts atuais (`dev`, `build`, `start`, …). Não existe script de worker
  hoje: `npm run worker` é **a criar**.
- `src/worker.ts` — **a criar**. Entry-point do processo do worker.

**Relacionado**

- [ADR-001](./ADR-001-outbox-no-mysql.md) — Padrão outbox no MySQL: define a tabela `webhook_outbox` que este worker consome.
- [ADR-003](./ADR-003-retry-com-backoff-e-dlq.md) — Retry com backoff exponencial e DLQ: define o que o worker faz quando a entrega
  falha. Política de retry, tentativas e destino de falha permanente **não** são cobertos
  aqui.
- [ADR-006](./ADR-006-reuso-dos-padroes-do-projeto.md) — Reuso dos padrões existentes do projeto: define onde a lógica de processamento
  mora dentro de `src/modules/webhooks/`.
- [ADR-007](./ADR-007-snapshot-do-payload-na-insercao.md) — Snapshot do payload na inserção: define o conteúdo que o worker encontra pronto
  na linha da outbox.
