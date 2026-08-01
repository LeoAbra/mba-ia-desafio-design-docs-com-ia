# PRD — Sistema de Webhooks de Notificação de Pedidos

- **Altitude:** produto / negócio. Este documento responde *por que a feature existe e o que ela precisa entregar*, não *como construir*.
- **Leitor-alvo:** Product Manager, liderança e quem precisa decidir se o investimento se justifica.
- **Fonte factual:** [`TRANSCRICAO.md`](../TRANSCRICAO.md) — reunião técnica de definição, quinta-feira 09:00, ~55 minutos, com Larissa (Tech Lead), Marcos (PM), Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma) e Sofia (Eng. de Segurança).
- **Documentos irmãos:** [RFC](./RFC.md) — como resolver · [ADRs 001–008](./adrs/README.md) — por que cada decisão · [FDD](./FDD.md) — como construir.
- **Convenção:** toda afirmação sustentada por fala da reunião carrega a marca `[hh:mm] Nome:`. Números sem essa marca são derivações declaradas explicitamente no texto.

---

## 1. Resumo e contexto da feature

Três clientes B2B — **Atlas Comercial, MaxDistribuição e Nova Cargo** — pediram formalmente para ser
notificados quando o status dos pedidos deles muda na nossa plataforma (`[09:00] Marcos:`). A feature
entrega essa notificação: o cliente cadastra um endereço de recebimento, escolhe quais mudanças de
status quer ouvir, e passa a receber cada mudança sem precisar perguntar.

O fluxo é **de saída apenas** — os eventos saem da nossa plataforma para o cliente, nunca o
contrário. Sofia abriu a reunião com exatamente essa pergunta de escopo (`[09:02] Sofia:`) e Marcos
fechou: "Só saindo da gente pra eles. Eles querem receber, não mandar" (`[09:02] Marcos:`).

O compromisso de negócio é duplo: **latência abaixo de 10 segundos**, que é o que os clientes chamam
de tempo real (`[09:02] Marcos:`), e **nenhuma mudança de status sem evento correspondente** — "Não
pode ter caso de status mudar e evento não sair" (`[09:40] Bruno:`). Prazo comercial: fim de novembro
(`[09:45] Marcos:`), estimado em três sprints incluindo a revisão de segurança (`[09:46] Larissa:`).

---

## 2. Problema e motivação

### A dor do cliente hoje

Os três clientes descobrem mudanças de status **perguntando repetidamente**: "Hoje eles ficam batendo
no `GET /orders` de tempos em tempos pra ver se mudou alguma coisa, e isso tá deixando a integração
lenta e cara pra eles" (`[09:00] Marcos:`). Duas palavras do próprio cliente definem o problema:

- **Lenta** — o cliente só descobre a mudança no próximo ciclo de consulta dele, que ele controla e
  que não tem relação nenhuma com o instante em que o status realmente mudou. Marcos traduziu a
  expectativa: "O importante é que não fique pendurado e eles tenham que ficar atualizando
  manualmente" (`[09:02] Marcos:`).
- **Cara** — cada consulta é custo de infraestrutura dos dois lados, pago mesmo quando nada mudou.
  Nenhuma medição de volume ou de custo de consulta repetida foi apresentada na reunião; o custo é o
  que o cliente relatou qualitativamente — "lenta e cara pra eles" (`[09:00] Marcos:`). Quantificar a
  economia exige medir, antes do piloto, quantas consultas ao recurso de pedidos os três clientes
  fazem por dia e qual fração delas retorna estado inalterado.

### O custo de não fazer

O risco é comercial e foi declarado na abertura da reunião: **"A Atlas chegou a sugerir que se a
gente não entregar isso até fim do trimestre, eles podem migrar pro nosso concorrente"**
(`[09:00] Marcos:`). A data alvo que a Atlas colocou é fim de novembro (`[09:45] Marcos:`).

Não é um pedido de um cliente isolado que dá para negociar: **três clientes B2B pediram formalmente a
mesma coisa, na mesma semana** (`[09:00] Marcos:`). O que está em jogo, portanto, não é só a receita
da Atlas — é a resposta a um requisito que o segmento inteiro está pedindo, com um concorrente já
nomeado como alternativa disponível.

Enquanto isso não existe, cada novo cliente B2B integra do jeito que hoje já é reconhecido como lento
e caro, e a nossa plataforma continua absorvendo o volume de consultas repetidas que a feature elimina.

---

## 3. Público-alvo e cenários de uso

A feature tem **dois públicos com necessidades opostas** e que, em parte, usam a mesma superfície.

### 3.1 Cliente B2B integrador (Atlas Comercial, MaxDistribuição, Nova Cargo)

Sistema do cliente, operado por um time técnico do lado dele. Quer **receber** e não quer operar nada
além disso.

| Cenário | O que o cliente faz | O que espera |
| --- | --- | --- |
| Integração inicial | Cadastra o endereço de recebimento e escolhe os status de interesse; recebe o segredo de validação na criação (`[09:31] Marcos:`) | Passar a receber eventos sem escrever rotina de consulta |
| Assinatura seletiva | "Só quero saber quando vira `SHIPPED` e `DELIVERED`" (`[09:33] Marcos:`) | Não receber o que não interessa a ele |
| Recebimento de evento | Recebe a notificação da mudança e valida que ela veio mesmo de nós e não foi adulterada (`[09:19] Sofia:`, `[09:20] Sofia:`) | Reagir em menos de 10 segundos (`[09:02] Marcos:`) |
| Diagnóstico do próprio lado | Consulta o histórico de entregas: "esses são os últimos 100 webhooks que vocês mandaram pra mim, sucesso/falha, payload, response, tempo de resposta" (`[09:34] Marcos:`) | Descobrir sozinho se o problema é dele, sem abrir chamado |
| Troca de credencial | Pede um segredo novo pela API e migra os sistemas dele dentro de uma janela (`[09:21] Sofia:`) | Trocar sem parar de receber evento |

Este cliente **assume responsabilidade operacional nova**: precisa manter um endereço de recebimento
no ar, validar cada evento e tolerar receber o mesmo evento duas vezes (`[09:25] Sofia:` —
"Isso joga responsabilidade pro cliente"). Marcos assumiu documentar isso de forma destacada
(`[09:26] Marcos:`).

### 3.2 Operador interno da plataforma

Usuário do nosso sistema, autenticado, com dois papéis distintos:

| Cenário | Quem | O que faz |
| --- | --- | --- |
| Cadastro em nome do cliente | Qualquer usuário autenticado nesta fase (`[09:37] Sofia:` — "Por enquanto sim") | Cadastra, edita, lista e remove configurações de webhook. O cliente ao qual a configuração pertence é informado explicitamente, **não é deduzido do login** (`[09:32] Bruno:`, `[09:32] Larissa:`) |
| Recuperação de evento perdido | Somente papel `ADMIN` (`[09:36] Sofia:` — "Mexer em fila de entrega de notificação não é coisa de operador") | Reprocessa manualmente um evento que a plataforma desistiu de entregar; a ação fica registrada com autoria (`[09:36] Sofia:`) |
| Operação de pedidos (indireto) | Operador que muda o status de um pedido | Não faz nada de webhook, mas é quem dispara o evento — e é ele quem sente se a emissão do evento falhar (ver risco R-5) |

A tensão entre os dois públicos é requisito simultâneo aqui: o cliente quer receber **tudo**, sempre,
o mais rápido possível (`[09:02] Marcos:`); o operador interno não pode ter a mudança de status presa
esperando o cliente — "qualquer cliente lento vai travar mudança de status pra outros pedidos"
(`[09:04] Bruno:`).

---

## 4. Objetivos e métricas de sucesso

Cada objetivo carrega métrica, meta e forma de medição. Metas derivadas de número da reunião estão
marcadas como **derivada**, com a derivação explicitada.

### OBJ-1 — Notificar dentro do que o cliente chama de tempo real

- **Métrica:** intervalo entre o instante em que o status do pedido muda e o instante em que o
  cliente confirma o recebimento do evento.
- **Meta:** **95% dos eventos entregues em menos de 10 segundos**, medidos sobre a primeira tentativa
  bem-sucedida. **Derivada e pendente de ratificação por Marcos:** a reunião fixou o limiar de 10
  segundos — "Pra eles, qualquer coisa abaixo de 10 segundos já é 'tempo real'" (`[09:02] Marcos:`) —
  mas **não fixou percentual algum**; os 95% são proposta deste PRD. Cobertura de 100% é incompatível
  com os próprios requisitos não funcionais: PRD-RNF-02 admite até 2 segundos de espera antes da
  primeira tentativa e PRD-RNF-03 admite até 10 segundos de resposta do cliente antes de a tentativa
  ser considerada falha, de modo que uma entrega lenta porém bem-sucedida pode ultrapassar os 10
  segundos sem descumprir nada.
- **Como se mede:** o histórico de entregas registra, por tentativa, o resultado e o tempo de
  resposta (`[09:34] Marcos:`); comparado com o instante da mudança de status, dá o intervalo ponta a
  ponta. Eventos que entram em nova tentativa saem desta métrica e caem no OBJ-4 — a reunião fixou o
  alvo de 10 segundos para o caminho sem falha, não para o caminho de retentativa.

### OBJ-2 — Eliminar a consulta repetitiva como forma de integração

- **Métrica:** número de clientes B2B, entre os que pediram a feature, com pelo menos uma
  configuração de webhook ativa recebendo eventos.
- **Meta:** **3 de 3** — Atlas Comercial, MaxDistribuição e Nova Cargo. **Derivada:** o número 3 é a
  quantidade de clientes que fizeram o pedido formal (`[09:00] Marcos:`); a reunião não estabeleceu
  meta de adoção própria.
- **Como se mede:** configurações ativas por cliente, cruzadas com entregas bem-sucedidas no
  histórico, **em até 60 dias após a disponibilização em produção** (prazo proposto por este PRD, a
  ratificar com Marcos — a reunião não fixou janela de adoção). Marcos é o canal com os clientes:
  confirma o prazo (`[09:47] Marcos:`) e os atualiza (`[09:49] Marcos:`).

### OBJ-3 — Não perder mudança de status

- **Métrica:** mudanças de status que deveriam gerar evento e não geraram.
- **Meta:** **zero**. Origem literal: "Não pode ter caso de status mudar e evento não sair"
  (`[09:40] Bruno:`).
- **Como se mede:** reconciliação entre o histórico de mudanças de status dos pedidos e os eventos
  registrados para envio, no mesmo período, descontadas as mudanças cujo status ninguém assinou —
  essas, por decisão, não geram evento (`[09:34] Bruno:`).

### OBJ-4 — Sobreviver à indisponibilidade do cliente sem intervenção humana

- **Métrica:** entre os eventos que falharam ao menos uma vez na entrega, a fração que precisou de
  reprocessamento manual. (A formulação anterior — "destino fora do ar temporariamente" — foi
  descartada: nada no sistema classifica a causa da falha do lado do cliente, então não havia
  instrumento que produzisse o número.)
- **Meta:** entregar sem intervenção humana todo evento cujo destino volte ao ar dentro da janela de
  retentativas coberta pelo desenho, que é de **~14h36min** — as "quase 15 horas entre primeira falha
  e última tentativa" registradas na reunião (`[09:17] Diego:`, aceito em `[09:17] Marcos:`). Com o
  teto de **5 tentativas** (`[09:15] Diego:`, fechado em `[09:17] Larissa:`), cada um dos cinco
  intervalos precede uma retentativa, contados a partir da falha da entrega inicial: 1m, 5m, 30m, 2h
  e 12h. A janela cobre com folga o caso real de **2 horas de indisponibilidade em manutenção
  planejada** (`[09:16] Diego:`) e fica dentro da faixa de "até 12 ou 24 horas" que a reunião
  buscava (`[09:15] Diego:`).
- **Como se mede:** duas contagens no mesmo período, extraídas do histórico de entregas e do acervo
  de eventos que esgotaram tentativas: (a) eventos entregues em alguma retentativa, sem ação humana;
  (b) eventos que esgotaram as tentativas e exigiram reprocessamento manual. **Limiar de aprovação:**
  (b) ≤ 1% de (a)+(b) por mês, medido a partir do primeiro mês completo em produção — limiar
  **proposto por este PRD**, a reunião não fixou nenhum. O número absoluto de (b) por mês é também o insumo que Larissa
  condicionou para reabrir o alerta automático (`[09:37] Larissa:` — "depois que a gente medir o
  impacto", R-5, FE-1).

### OBJ-5 — Entregar dentro do prazo comercial

- **Métrica:** data de disponibilização em produção.
- **Meta:** **fim de novembro** (`[09:45] Marcos:`), correspondente a **três sprints** incluindo a
  revisão de segurança (`[09:46] Larissa:`, `[09:47] Larissa:`).
- **Como se mede:** data de deploy contra a data acordada com a Atlas. É o objetivo que mitiga o
  risco de churn declarado em `[09:00] Marcos:`.

---

## 5. Escopo

### 5.1 Incluído

- Cadastro, consulta, edição e remoção de configurações de webhook por cliente (`[09:31] Marcos:`,
  `[09:33] Bruno:`).
- Segredo de validação **gerado pela plataforma** e devolvido ao cliente na criação
  (`[09:31] Marcos:`), com rotação sob demanda e janela de convivência (`[09:21] Sofia:`).
- Assinatura por configuração de webhook para o cliente provar autoria e integridade do que recebeu
  (`[09:19] Sofia:`, `[09:20] Sofia:`).
- Assinatura seletiva: cada configuração escolhe quais mudanças de status quer receber
  (`[09:33] Marcos:`).
- Emissão do evento na mudança de status do pedido, atrelada ao sucesso da própria mudança
  (`[09:40] Bruno:`).
- Entrega assíncrona, com retentativas automáticas e destino explícito para o que não pôde ser
  entregue (`[09:15] Diego:`, `[09:17] Larissa:`, `[09:18] Diego:`).
- Histórico de entregas consultável pelo cliente (`[09:34] Marcos:`).
- Reprocessamento manual, restrito e auditado, de evento que a plataforma desistiu de entregar
  (`[09:35] Diego:`, `[09:36] Sofia:`).

### 5.2 Fora de escopo

Esta é a seção que impede que discussão levantada e recusada volte como requisito. Cada item foi
**levantado na reunião** e **recusado ou adiado**, com autoria e timestamp.

| # | Item | Quem levantou | Quem recusou / adiou | Classificação | Justificativa registrada |
| --- | --- | --- | --- | --- | --- |
| FE-1 | **Alerta por e-mail ao cliente quando o webhook dele está falhando** ("se ele falhou 3 vezes seguidas, mandar email pra ele") | Marcos, `[09:37] Marcos:` | Larissa, `[09:37] Larissa:` | **ADIADO para fase futura** | "Email tá fora de escopo dessa fase. Talvez próxima fase, depois que a gente medir o impacto." O único aviso previsto é o cliente consultar o histórico de entregas (PRD-RF-06) |
| FE-2 | **Painel / dashboard visual para o cliente acompanhar os webhooks dele** | Marcos, `[09:39] Marcos:` | Larissa, `[09:40] Larissa:` | **FORA DE ESCOPO — outro time** | "Não, agora não. Só endpoints. Painel é projeto separado do time de frontend." Marcos assumiu cobrir a lacuna com documentação de integração (`[09:40] Marcos:`) |
| FE-3 | **Limitação da taxa de envio para um mesmo cliente** ("se o cliente tem 50 pedidos mudando de status em um minuto, a gente bombardeia ele com 50 chamadas?") | Diego, `[09:38] Diego:` | Diego e Larissa, `[09:39] Diego:` / `[09:39] Larissa:` | **ADIADO — não decidido** | "A gente observa e implementa se virar problema"; Larissa registrou como "observar e decidir depois". Não há limite nesta fase, e isso é consciente — ver risco R-6 |
| FE-4 | **Arquivamento / expurgo do acervo de eventos já entregues** (menção a "depois de 30 dias ou assim") | Diego, `[09:08] Diego:` | Diego, no mesmo minuto, `[09:08] Diego:` | **FORA DE ESCOPO desta feature** | Ele próprio classificou: "fora do escopo dessa feature". Sem política de retenção, o acervo cresce indefinidamente |
| FE-5 | **Recebimento de webhooks enviados pelo cliente para nós (entrada)** | Sofia, como pergunta de escopo, `[09:02] Sofia:` | Marcos, `[09:02] Marcos:` | **FORA DE ESCOPO permanente desta feature** | "Só saindo da gente pra eles. Eles querem receber, não mandar." Sofia confirmou o recorte em `[09:03] Sofia:` |
| FE-6 | **Garantia de ordem global de entrega entre pedidos diferentes**, e o processamento paralelo que a quebraria e exigiria mecanismo novo para ser reconstruída | Larissa, `[09:12] Larissa:`; retomado por Bruno, `[09:13] Bruno:` | Diego, `[09:13] Diego:`; Larissa, `[09:13] Larissa:` | **DESCARTADO por decisão técnica** | "Isso é problema do futuro, não agora." Larissa fechou como limitação conhecida e documentada. A premissa que sustenta o descarte é de produto: "Os clientes nunca pediram garantia de ordering global, eles só querem saber se cada pedido deles mudou" (`[09:14] Marcos:`) |
| FE-7 | **Restrição de papel no cadastro de webhooks** (endurecer quem pode operar a configuração de qual cliente) | Ninguém propôs; o tema surge por negação, quando Marcos pergunta se o CRUD pode ficar com qualquer papel autenticado (`[09:36] Marcos:`) | Sofia, `[09:37] Sofia:`, que autorizou a folga e ela mesma anunciou o endurecimento como futuro | **ADIADO para fase futura** | "Por enquanto sim. Mais pra frente a gente pode endurecer." Nesta fase, qualquer usuário autenticado opera a configuração de qualquer cliente — ver R-7 e [ADR-008](./adrs/ADR-008-controle-de-acesso-dos-endpoints.md) |

Nada fora desta lista foi descartado na reunião. Ausência de um tema aqui significa que ele não foi
discutido, não que tenha sido recusado.

---

## 6. Requisitos funcionais

Capacidades verificáveis, com identificador estável. Todas rastreadas a uma fala da reunião.

| ID | Requisito | Origem |
| --- | --- | --- |
| **PRD-RF-01** | O sistema deve permitir **cadastrar uma configuração de webhook** para um cliente, informando o endereço de recebimento e os status de interesse. O **segredo de validação é gerado pela plataforma** e devolvido ao cliente na resposta da criação — o cliente não escolhe o segredo. O cliente ao qual a configuração pertence é informado explicitamente na requisição, **não deduzido do login** | `[09:31] Marcos:`; correção do `customer_id` em `[09:32] Bruno:` → `[09:32] Marcos:` → `[09:32] Larissa:` |
| **PRD-RF-02** | O sistema deve permitir **listar as configurações de webhook de um cliente** | `[09:33] Bruno:` |
| **PRD-RF-03** | O sistema deve permitir **editar uma configuração de webhook** já cadastrada | `[09:33] Bruno:` |
| **PRD-RF-04** | O sistema deve permitir **remover uma configuração de webhook** | `[09:33] Bruno:` |
| **PRD-RF-05** | O sistema deve permitir que **cada configuração escolha quais mudanças de status quer receber**, e não deve enviar mudanças de status fora dessa seleção. Se nenhuma configuração do cliente assina aquele status, o evento não é sequer produzido | `[09:33] Marcos:`; `[09:34] Bruno:` / `[09:34] Diego:` |
| **PRD-RF-06** | O sistema deve permitir que o cliente **consulte o histórico de entregas** de uma configuração, com **resultado (sucesso ou falha), conteúdo enviado, resposta recebida e tempo de resposta** por tentativa. Marcos descreveu a ordem de grandeza como "os últimos 100 webhooks" | `[09:34] Marcos:` |
| **PRD-RF-07** | O sistema deve permitir que o cliente **solicite um segredo novo** para uma configuração. O segredo anterior **permanece válido em paralelo por 24 horas** para o cliente migrar os sistemas dele; depois disso, deixa de valer | `[09:21] Sofia:`, reafirmado em `[09:22] Sofia:` e `[09:48] Larissa:` |
| **PRD-RF-08** | O sistema deve **produzir o evento de mudança de status junto com a própria mudança**: se a mudança de status é efetivada e existe ao menos uma configuração que assinou aquele status, o evento existe; se o registro do evento falha, a mudança de status não acontece. **Não existe status alterado sem evento correspondente, entre as mudanças que alguma configuração assinou** (as demais, por decisão de PRD-RF-05, não geram evento) | `[09:40] Bruno:`, confirmado em `[09:41] Diego:` |
| **PRD-RF-09** | O sistema deve **entregar cada evento assinado**, de forma que o cliente possa verificar que a notificação veio da nossa plataforma e não foi adulterada em trânsito, e deve **identificar unicamente cada evento** para o cliente reconhecer reenvios | `[09:19] Sofia:`, `[09:20] Sofia:`, `[09:25] Diego:` |
| **PRD-RF-10** | O sistema deve **retentar automaticamente** a entrega quando o destino não responde ou responde com erro, com intervalos crescentes, até o teto de tentativas definido nos requisitos não funcionais | `[09:14] Larissa:`, `[09:15] Diego:`, `[09:17] Larissa:` |
| **PRD-RF-11** | O sistema deve **preservar o evento que esgotou as tentativas**, com o conteúdo original, o motivo da falha e o momento da desistência, de forma consultável — evento que falhou definitivamente não desaparece | `[09:18] Diego:` |
| **PRD-RF-12** | O sistema deve permitir o **reprocessamento manual de um evento que esgotou as tentativas**, restrito a usuários com papel `ADMIN`, **registrando quem executou a ação** para auditoria | `[09:35] Diego:`, `[09:36] Sofia:`, `[09:36] Larissa:` |

**Doze requisitos.** Todos foram enunciados na reunião; nenhum foi acrescentado por inferência.

---

## 7. Requisitos não funcionais

Parâmetros fixados na reunião que não mereceram decisão arquitetural própria — a própria reunião os
classificou assim para a obrigatoriedade de transporte cifrado (`[09:23] Sofia:`) e para o teto de
tamanho do evento (`[09:24] Larissa:`).

| ID | Requisito | Valor | Origem |
| --- | --- | --- | --- |
| **PRD-RNF-01** | Latência alvo de notificação | **abaixo de 10 segundos** da mudança de status até o recebimento pelo cliente | `[09:02] Marcos:` |
| **PRD-RNF-02** | Atraso máximo aceitável entre a mudança de status e o início da primeira tentativa de entrega | **2 segundos** — é a maior parcela fixa do atraso e cabe dentro do alvo de 10s. O mecanismo que produz esse atraso é decisão de engenharia ([ADR-002](./adrs/ADR-002-worker-separado-com-polling.md)) | `[09:09] Diego:`, aceito em `[09:10] Marcos:` e `[09:10] Larissa:` |
| **PRD-RNF-03** | Tempo máximo de espera pela resposta do cliente | **10 segundos**; sem resposta nesse prazo, a tentativa é considerada falha e entra em retentativa | `[09:42] Diego:` |
| **PRD-RNF-04** | Número máximo de tentativas de entrega por evento | **5** | `[09:15] Diego:`, fechado em `[09:16] Larissa:` / `[09:17] Larissa:` |
| **PRD-RNF-05** | Espaçamento entre tentativas | **1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas**, contados a partir da falha da entrega inicial: cada um dos cinco intervalos precede uma das 5 tentativas (PRD-RNF-04). Da primeira falha até a última tentativa decorrem **~14h36min** — as "quase 15 horas" registradas na reunião (`[09:17] Diego:`) | `[09:17] Diego:`, aceito em `[09:17] Marcos:` |
| **PRD-RNF-06** | Janela de convivência do segredo anterior após rotação | **24 horas** | `[09:21] Sofia:` |
| **PRD-RNF-07** | Tamanho máximo do conteúdo de um evento | **64 KB**; acima disso é **erro, não truncamento** — "Se chegou nesse tamanho, tem algo errado" | `[09:23] Sofia:`, `[09:24] Diego:`, `[09:24] Larissa:` |
| **PRD-RNF-08** | Transporte obrigatoriamente cifrado | O endereço de recebimento **precisa usar transporte cifrado**; endereço em texto claro é recusado no cadastro com erro de validação | `[09:23] Sofia:` |
| **PRD-RNF-09** | Mecanismo de prova de autoria e integridade | Assinatura do corpo enviado por **mecanismo padrão de mercado, para o qual o cliente já tenha biblioteca pronta** — "é o padrão de mercado, todo cliente sério tem biblioteca pra isso" (`[09:20] Sofia:`). O algoritmo específico é decisão de engenharia ([ADR-004](./adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)) | `[09:20] Sofia:`, `[09:22] Sofia:` |
| **PRD-RNF-10** | Unicidade do segredo | **Um segredo por configuração de webhook**, nunca um segredo global da plataforma — "Senão se vaza uma, vaza tudo" | `[09:21] Sofia:` |
| **PRD-RNF-11** | Isolamento da operação de pedidos | A entrega ao cliente **não pode bloquear a mudança de status** de outros pedidos — cliente lento ou fora do ar não trava a operação | `[09:04] Bruno:`, `[09:06] Diego:` |
| **PRD-RNF-12** | Conteúdo do evento | Dados essenciais do pedido, **sem a lista de itens**, para não inflar; quem precisar de detalhe consulta o pedido | `[09:43] Diego:`, `[09:44] Bruno:` |

---

## 8. Decisões e trade-offs principais

Uma linha por decisão, com o que ela **custa para o cliente ou para a operação**. O porquê de cada
uma mora no ADR correspondente e não é reproduzido aqui.

| Decisão | O que custa | ADR |
| --- | --- | --- |
| O evento é registrado junto com a mudança de status, tudo ou nada | Se o registro do evento falhar, a **mudança de status falha junto** e o operador interno vê a operação de pedido ser recusada | [ADR-001](./adrs/ADR-001-outbox-no-mysql.md) |
| A entrega é assíncrona, despachada fora da transação de mudança de status | Até **2 segundos** de atraso fixo antes da primeira tentativa; **nenhuma garantia de ordem** entre pedidos diferentes e, **sempre que uma entrega falha e entra em retentativa, também não há garantia de ordem entre os eventos de um mesmo pedido** — o cliente pode receber um status posterior antes de um anterior e precisa se orientar pelos status de origem e destino que o próprio evento carrega. Isso precisa estar na documentação de integração (9.1) | [ADR-002](./adrs/ADR-002-worker-separado-com-polling.md) |
| Cinco tentativas espaçadas e, esgotadas, o evento fica preservado para reprocessamento manual | Um evento pode levar **~15 horas** para ser declarado perdido, e a recuperação **depende de alguém olhar**: não há aviso automático | [ADR-003](./adrs/ADR-003-retry-com-backoff-e-dlq.md) |
| Cada configuração tem segredo próprio, rotacionável, com 24h de convivência | O cliente **precisa implementar a verificação da assinatura** e, durante 24 horas após rotacionar, conviver com **dois segredos válidos** | [ADR-004](./adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) |
| A entrega é *at-least-once* | O cliente **pode receber o mesmo evento mais de uma vez e precisa deduplicar** pelo identificador do evento; a responsabilidade é dele (`[09:25] Sofia:`) | [ADR-005](./adrs/ADR-005-at-least-once-com-x-event-id.md) |
| A feature reusa os padrões já vigentes na nossa API | Nada novo para o cliente aprender: formato de erro, autenticação e paginação são os mesmos do resto da API que ele já consome | [ADR-006](./adrs/ADR-006-reuso-dos-padroes-do-projeto.md) |
| O evento carrega o estado do pedido no instante da mudança | O cliente recebe um **retrato do momento, não o estado atual**; um evento reprocessado dias depois entrega dado antigo | [ADR-007](./adrs/ADR-007-snapshot-do-payload-na-insercao.md) |
| Reprocessamento exige o papel `ADMIN`; cadastro exige apenas autenticação | Nesta fase, **qualquer usuário autenticado pode alterar a configuração de qualquer cliente** — risco aceito e instanciado em R-7, endurecimento adiado (FE-7) | [ADR-008](./adrs/ADR-008-controle-de-acesso-dos-endpoints.md) |

---

## 9. Dependências

### 9.1 Dependências de processo

Duas bloqueiam o deploy (revisão de segurança e documentação de integração publicada); uma bloqueia o
início da implementação (revisão do documento de design); uma é comunicação comercial, sem efeito de
bloqueio. A coluna **Condição** diz qual é qual.

| Dependência | Responsável | Condição | Origem |
| --- | --- | --- | --- |
| **Revisão de segurança antes do deploy**, com foco na assinatura e na geração do segredo | Sofia | **Mínimo dois dias úteis** reservados antes de subir: "Reservem pelo menos dois dias úteis pra eu revisar o código de segurança antes do deploy. HMAC e geração de secret eu quero olhar com calma". Larissa confirmou e incluiu na estimativa de três sprints | `[09:46] Sofia:`, `[09:47] Larissa:`, reforçado em `[09:49] Sofia:` |
| **Documentação de integração no portal do desenvolvedor** | Marcos | Marcos assumiu documentar a garantia de entrega e a necessidade de o cliente deduplicar ("Eu posso documentar isso bem destacado no portal de desenvolvedor pros clientes") e, depois da recusa do painel visual, como integrar via API ("Eu documento no portal pros clientes saberem como integrar via API"). Precisa cobrir também a **ausência de garantia de ordem quando há retentativa**, inclusive entre eventos do mesmo pedido (seção 8, [FDD §5.2.2](./FDD.md)) | `[09:26] Marcos:`, `[09:40] Marcos:` |
| **Revisão do documento de design com o time** antes do início da implementação | Larissa, com Bruno e Diego | "Eu vou abrir o doc de design da feature e marcar uma sessão pro Bruno e o Diego revisarem comigo antes da gente começar a codar" | `[09:50] Larissa:` |
| **Confirmação do prazo com os clientes** | Marcos | "Atlas vai gostar. Eu confirmo prazo com eles"; "Eu atualizo os clientes hoje à tarde" | `[09:47] Marcos:`, `[09:49] Marcos:` |

### 9.2 Dependências de capacidade

- **Três sprints de time**, com a revisão de segurança já dentro da janela (`[09:46] Larissa:`,
  `[09:47] Larissa:`). O prazo comercial é fim de novembro (`[09:45] Marcos:`).

### 9.3 Dependência externa — do lado do cliente

A feature só entrega valor se o cliente fizer a parte dele, e essa parte é nova para ele:

- Manter um endereço de recebimento disponível e **com transporte cifrado** (PRD-RNF-08).
- Verificar a assinatura de cada evento recebido (`[09:19] Sofia:`, `[09:20] Sofia:`).
- **Deduplicar** eventos repetidos pelo identificador do evento (`[09:25] Diego:`) — é o que a
  documentação do item 9.1 precisa deixar destacado.

---

## 10. Riscos e mitigação

Cada risco ancorado em fato declarado na reunião. Risco sem âncora não entra.

### R-1 — Cliente vaza o segredo de validação

- **Âncora:** "A gente já teve cliente que vazou secret em log de aplicação dele uma vez"
  (`[09:22] Diego:`).
- **Probabilidade: média** — já aconteceu pelo menos uma vez com cliente nosso.
- **Impacto: alto** — quem tiver o segredo consegue forjar notificações que o cliente aceitará como
  legítimas.
- **Mitigação:** segredo **único por configuração**, e não global — o vazamento de um não compromete
  os demais clientes (`[09:21] Sofia:`, PRD-RNF-10) — somado à **rotação sob demanda com 24 horas de
  convivência** (PRD-RF-07), que permite ao cliente trocar a credencial sem parar de receber evento.
  Reforço: a geração do segredo é item explícito da revisão de segurança (`[09:46] Sofia:`).

### R-2 — Cliente indisponível por horas e evento perdido

- **Âncora:** "Já tinha cliente nosso com indisponibilidade de duas horas em manutenção planejada"
  (`[09:16] Diego:`).
- **Probabilidade: alta** — depender da disponibilidade de terceiro é a condição normal da feature, e
  o caso de 2 horas já ocorreu.
- **Impacto: médio** — o cliente não é notificado enquanto está fora; se a notificação for descartada
  cedo demais, ele nunca fica sabendo da mudança.
- **Mitigação:** **5 tentativas** espaçadas (PRD-RNF-04, PRD-RNF-05), cobrindo uma janela de
  **~14h36min** a partir da primeira falha — folga larga sobre o caso ancorado de 2 horas de
  manutenção planejada. Foi exatamente esse cenário que derrubou a proposta de 3 tentativas
  (`[09:16] Bruno:` → `[09:16] Diego:`). O tamanho da janela foi aceito pelo produto pelo que
  significa na prática: "Se um cliente meu cair por 15 horas, ele já tá com problema sério dele"
  (`[09:17] Marcos:`). Esgotadas as tentativas, o evento é preservado
  (PRD-RF-11) e pode ser reprocessado manualmente (PRD-RF-12).

### R-3 — Perder a Atlas Comercial por atraso na entrega

- **Âncora:** "A Atlas chegou a sugerir que se a gente não entregar isso até fim do trimestre, eles
  podem migrar pro nosso concorrente" (`[09:00] Marcos:`); data alvo fim de novembro
  (`[09:45] Marcos:`).
- **Probabilidade: média** — o prazo é apertado (três sprints, `[09:46] Larissa:`) e a ameaça foi
  declarada pelo cliente, não inferida.
- **Impacto: alto** — perda de cliente B2B e sinal negativo para os outros dois que pediram a mesma
  coisa.
- **Mitigação:** **escopo deliberadamente cortado** — alerta por e-mail (FE-1), painel visual (FE-2),
  limitação de taxa de envio (FE-3) e endurecimento de papéis (FE-7) ficaram fora justamente para
  caber no prazo; a revisão de segurança foi **contabilizada dentro** das três sprints, ao fim delas
  e antes do deploy, em vez de ficar fora da estimativa (`[09:47] Larissa:` — "Três sprints com a
  revisão da Sofia incluída no fim"); e Marcos confirma a data com o cliente logo após a reunião
  (`[09:47] Marcos:`, `[09:49] Marcos:`).

### R-4 — Cliente não deduplica e processa o mesmo evento duas vezes

- **Âncora:** a garantia é *at-least-once* e a deduplicação é responsabilidade do cliente —
  "Isso joga responsabilidade pro cliente" (`[09:25] Sofia:`), aceito por Diego no mesmo minuto
  (`[09:25] Diego:`).
- **Probabilidade: média** — depende inteiramente da qualidade da integração do cliente, sobre a qual
  não temos controle nem visibilidade.
- **Impacto: alto para o cliente** — evento de mudança de status processado em duplicidade pode gerar
  efeito duplicado no sistema dele (baixa, expedição, cobrança do lado dele).
- **Mitigação:** **identificador único e estável por evento**, que não muda entre reenvios
  (`[09:25] Diego:`, PRD-RF-09), somado à **documentação destacada no portal do desenvolvedor** que
  Marcos assumiu (`[09:26] Marcos:` — "Eu posso documentar isso bem destacado"). A dedução é
  auditável pelo cliente via histórico de entregas (PRD-RF-06).

### R-5 — Evento que falhou definitivamente sem ninguém perceber

- **Âncora:** o alerta por e-mail foi pedido por Marcos e recusado nesta fase (`[09:37] Marcos:` →
  `[09:37] Larissa:`); o reprocessamento é manual (`[09:18] Diego:`).
- **Probabilidade: média** — nada avisa automaticamente que um evento foi para o fim da linha.
- **Impacto: alto** — na prática, um evento que ninguém reprocessa é um evento perdido, e o cliente
  fica sem a notificação sem saber disso.
- **Mitigação:** nesta fase, o **histórico de entregas consultável pelo próprio cliente**
  (PRD-RF-06) é o canal de descoberta, e o **reprocessamento manual auditado** (PRD-RF-12) é o
  caminho de recuperação. O gatilho de reabertura foi declarado pela própria Larissa: "Talvez próxima
  fase, depois que a gente medir o impacto" (`[09:37] Larissa:`) — ou seja, medir quantos eventos
  chegam a esse estado é pré-requisito para justificar o alerta (ver OBJ-4).

### R-6 — Rajada de notificações sobre um mesmo cliente

- **Âncora:** "Se o cliente tem 50 pedidos mudando de status em um minuto, a gente bombardeia ele com
  50 chamadas?" (`[09:38] Diego:`).
- **Probabilidade: baixa a média** — depende do volume de mudanças concentradas de um mesmo cliente;
  nenhum dado de produção existe hoje.
- **Impacto: médio** — o cliente pode ficar sobrecarregado, começar a recusar as chamadas e, por
  consequência, alimentar o R-2.
- **Mitigação:** **nenhum limite nesta fase, por decisão consciente** — "A gente observa e implementa
  se virar problema" (`[09:39] Diego:`), registrado por Larissa como "observar e decidir depois"
  (`[09:39] Larissa:`). A mitigação é de processo: acompanhar o histórico de entregas (PRD-RF-06) e a
  taxa de falhas por cliente, e reabrir o tema (FE-3) na primeira ocorrência concreta.

### R-7 — Configuração de um cliente alterada por usuário de outro contexto

- **Âncora:** o cliente ao qual a configuração pertence é informado na requisição e **não é deduzido
  do login** (`[09:32] Larissa:`), e o endurecimento de papéis foi adiado (`[09:37] Sofia:` — "Por
  enquanto sim. Mais pra frente a gente pode endurecer", FE-7).
- **Causa raiz — análise deste PRD, não conclusão da reunião:** a premissa levantada em
  `[09:32] Marcos:` ("A gente tem usuários que representam o cliente") **não se sustenta no modelo de
  dados atual**. Em `prisma/schema.prisma`, o usuário tem apenas as relações de pedidos que criou e
  de mudanças de status que registrou, e o cliente tem apenas os pedidos dele; não existe campo,
  chave estrangeira nem tabela de junção ligando um usuário a um cliente. Ou seja: verificar se quem
  cadastra tem alguma relação com o cliente informado **não é uma regra que ficou para depois — é uma
  pergunta que o sistema hoje não tem como responder**.
- **Probabilidade: baixa** — exige usuário interno autenticado agindo por erro ou má-fé; não há
  registro de ocorrência. **Ressalva desta análise:** não há barreira técnica nem detecção — nada
  distingue um cadastro legítimo de um indevido no momento em que ele acontece, e a exposição não
  depende de o cliente ter usuários próprios (o gatilho abaixo), ela já existe com o quadro atual de
  usuários internos.
- **Impacto: alto** — quem cria uma configuração recebe o segredo de validação na resposta
  (PRD-RF-01); apontar a configuração de um cliente para um endereço de terceiro expõe os eventos de
  pedido daquele cliente e corrói a garantia que a assinatura pretende dar. Concretamente: qualquer
  usuário autenticado, **inclusive um `OPERATOR`**, pode cadastrar um endereço de recebimento próprio
  informando o identificador de qualquer cliente e passar a receber os eventos de pedidos desse
  cliente.
- **Mitigação:** nesta fase, **nenhuma técnica** — é risco aceito e documentado em
  [ADR-008](./adrs/ADR-008-controle-de-acesso-dos-endpoints.md). A contenção é de processo: acesso ao
  sistema restrito a usuários internos e o histórico de entregas (PRD-RF-06) como evidência de para
  onde os eventos foram. **Mitigação estrutural, quando o tema for reaberto:** criar o vínculo entre
  usuário e cliente no modelo de dados é **pré-requisito** de qualquer verificação de posse — sem
  ele, restrição de papel não resolve, porque um `ADMIN` continuaria podendo apontar a configuração
  de qualquer cliente. Isso muda o custo do endurecimento (alteração de schema e migração, não apenas
  regra de autorização) e é insumo para o planejamento de FE-7. **Gatilho de reabertura (FE-7):**
  primeiro cliente com usuários próprios, ou primeiro incidente de alteração indevida.

---

## 11. Critérios de aceitação

Nível de produto: o que precisa ser demonstrável para a feature ser considerada entregue.

| ID | Critério | Requisito coberto |
| --- | --- | --- |
| **AC-01** | Um cliente cadastrado com um endereço de transporte cifrado e uma seleção de status **recebe a notificação da mudança de status em menos de 10 segundos**, demonstrável ponta a ponta | PRD-RF-08, PRD-RF-09, PRD-RNF-01 |
| **AC-02** | O cadastro devolve **o segredo gerado pela plataforma** na criação, e o cliente consegue **validar a assinatura** de um evento recebido usando esse segredo | PRD-RF-01, PRD-RF-09, PRD-RNF-09 |
| **AC-03** | Cadastro com endereço **sem transporte cifrado** é **recusado com erro de validação** | PRD-RNF-08 |
| **AC-04** | Uma configuração que assina apenas parte dos status **não recebe** notificação dos status que não assinou | PRD-RF-05 |
| **AC-05** | Um usuário autenticado consegue **listar, editar e remover** configurações de um cliente informando explicitamente qual é o cliente na requisição. **Nesta fase não há verificação de posse** — a ausência dessa verificação é consciente (FE-7, R-7, [ADR-008](./adrs/ADR-008-controle-de-acesso-dos-endpoints.md)) e não deve ser confundida com defeito no teste | PRD-RF-02, PRD-RF-03, PRD-RF-04 |
| **AC-06** | Após rotacionar o segredo, o cliente **consegue validar todo evento recebido durante as 24 horas seguintes usando o segredo anterior ou o novo, sem perder nenhum evento**, e a validação pelo segredo anterior deixa de funcionar após a janela. **Depende de decisão em aberto:** qual segredo assina durante a janela (novo, antigo ou ambos em cabeçalhos distintos) não foi decidido na reunião — ver [RFC §6](./RFC.md) e [FDD §3.3](./FDD.md); a decisão precisa sair da revisão de segurança de `[09:46] Sofia:` antes deste critério ser verificável | PRD-RF-07, PRD-RNF-06 |
| **AC-07** | Destino fora do ar por período compatível com a janela de tentativas **recebe o evento quando volta**, sem nenhuma intervenção humana | PRD-RF-10, PRD-RNF-04, PRD-RNF-05 |
| **AC-08** | Destino que nunca responde faz o evento **ser preservado com o conteúdo original e o motivo da falha**, consultável | PRD-RF-11 |
| **AC-09** | Um usuário `ADMIN` consegue **reprocessar** um evento preservado e o cliente o recebe; a ação fica **registrada com autoria**. Usuário sem `ADMIN` é **recusado** | PRD-RF-12 |
| **AC-10** | O histórico de entregas mostra, por tentativa, **resultado, conteúdo enviado, resposta recebida e tempo de resposta** | PRD-RF-06 |
| **AC-11** | Mudança de status cuja emissão de evento falhe **não é efetivada** — não existe pedido com status novo e evento ausente **entre os status assinados por alguma configuração** | PRD-RF-08, OBJ-3 |
| **AC-12** | Um destino que demora além do limite de espera **não bloqueia** a mudança de status de outros pedidos | PRD-RNF-03, PRD-RNF-11 |
| **AC-13** | A **revisão de segurança de Sofia está concluída** antes do deploy, com no mínimo dois dias úteis dedicados | Dependência 9.1 |
| **AC-14** | A **documentação de integração está publicada no portal do desenvolvedor**, com a necessidade de deduplicação em destaque | Dependência 9.1, R-4 |

---

## 12. Estratégia de testes e validação

### 12.1 Validação funcional

- **Verificação ponta a ponta da mudança de status até o recebimento.** Larissa dimensionou esse
  teste dentro da estimativa: meia sprint para a integração com o fluxo de mudança de status e os
  testes ponta a ponta (`[09:46] Larissa:`). É o teste que sustenta AC-01 e AC-11 simultaneamente — mudança
  efetivada implica evento entregue; emissão falha implica mudança não efetivada.
- **Verificação da assinatura pelo lado receptor**: um receptor de teste precisa conseguir validar a
  assinatura com o segredo recebido no cadastro (AC-02). É o único jeito de provar o requisito do
  ponto de vista de quem consome — `[09:20] Sofia:` descreve exatamente esse fluxo de verificação do
  lado do cliente.
- **Verificação da seleção de status** (AC-04): a configuração que assina apenas parte dos status não
  pode receber os demais (`[09:33] Marcos:`).

### 12.2 Validação dos caminhos de falha

Cenários de falha que a reunião ancorou em caso concreto e que, por isso, precisam ser exercitados
explicitamente:

| Cenário a exercitar | Fato que o motiva | Critério |
| --- | --- | --- |
| Destino indisponível e retorno dentro da janela de tentativas | Cliente com 2h de manutenção planejada (`[09:16] Diego:`) | AC-07 |
| Destino que nunca responde, até esgotar as tentativas | Teto de 5 tentativas (`[09:15] Diego:`) | AC-08 |
| Destino que responde além do limite de espera | Timeout de 10 segundos (`[09:42] Diego:`) | AC-12 |
| Reprocessamento manual por `ADMIN` e recusa a quem não é `ADMIN` | `[09:36] Sofia:` | AC-09 |
| Rotação de segredo, com validação pelo segredo antigo dentro e fora da janela de 24h | `[09:21] Sofia:` | AC-06 |
| Endereço sem transporte cifrado recusado no cadastro | `[09:23] Sofia:` | AC-03 |
| Evento acima do tamanho máximo | Teto de 64 KB, com erro e não truncamento (`[09:23] Sofia:`, `[09:24] Diego:`) | PRD-RNF-07 |

### 12.3 Revisão de segurança — condição de deploy

Etapa obrigatória antes do deploy: **mínimo dois dias úteis** para Sofia revisar a assinatura e a
geração do segredo (`[09:46] Sofia:`), já contabilizados dentro das três sprints
(`[09:47] Larissa:`). Sofia reforçou o ponto no fechamento: "só não esqueçam de me agendar pra
revisão de segurança antes de subir" (`[09:49] Sofia:`).

### 12.4 Validação com o cliente

- **Piloto com os clientes que pediram a feature.** Marcos é o canal: confirma o prazo
  (`[09:47] Marcos:`) e atualiza os clientes (`[09:49] Marcos:`). O sucesso do OBJ-2 é medido por
  eles deixarem de consultar em loop e passarem a receber.
- **Documentação de integração publicada antes do piloto** (`[09:26] Marcos:`, `[09:40] Marcos:`) —
  sem ela, o cliente não sabe que precisa deduplicar (R-4) nem, na ausência do painel (FE-2), como
  acompanhar as entregas dele.

---

**Nota de fronteira.** Este documento não descreve arquitetura, modelo de dados, contratos de API nem
tecnologia. Se a engenharia trocar completamente a solução técnica, tudo acima continua válido — o
*como* está no [RFC](./RFC.md), nos [ADRs](./adrs/README.md) e no [FDD](./FDD.md).
