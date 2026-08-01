# ADR-004 — Assinatura HMAC-SHA256 com segredo por endpoint e rotação com grace period

- **Status:** Aceito
- **Data:** Quinta-feira, reunião técnica de definição (ver `TRANSCRICAO.md`)
- **Decisores:** Larissa (Tech Lead), Sofia (Engenheira de Segurança), Diego (Engenheiro Sênior, Plataforma), Bruno (Engenheiro Pleno, Pedidos), Marcos (Product Manager)
- **Contexto técnico:** Módulo de webhooks do Order Management System (Node.js + TypeScript + Express + Prisma + MySQL) — autenticidade e integridade das requisições de saída e o ciclo de vida do segredo de cada endpoint cadastrado

## Contexto

A feature de webhooks inverte o sentido do tráfego do sistema. Até hoje o Order Management System só recebe requisições autenticadas por JWT (`src/middlewares/auth.middleware.ts`, `src/modules/auth/auth.service.ts`) e nunca precisou provar a própria identidade para um terceiro. Com o webhook, passamos a fazer chamadas HTTP de saída carregando dados de pedidos — número do pedido, status, cliente, valor — para endpoints que rodam fora da nossa infraestrutura, operados pela Atlas Comercial, MaxDistribuição e Nova Cargo.

Sofia abriu o bloco de segurança nomeando o problema em [09:19]: "a gente tá expondo eventos com dados de pedidos pra um endpoint fora da nossa infra. O cliente tem que conseguir validar que a requisição veio realmente da gente, e que ninguém adulterou o payload no meio." São dois requisitos distintos — autenticidade da origem e integridade do corpo — e nenhum mecanismo do sistema atual resolve algum deles: o JWT protege a entrada, não a saída.

A restrição que fecha o cerco é operacional, não criptográfica. Somos três clientes B2B hoje, mas cada um deles é um sistema alheio, com práticas de log e de armazenamento que não controlamos — e Diego lembrou em [09:22] um caso real ocorrido do lado do cliente: "A gente já teve cliente que vazou secret em log de aplicação dele uma vez." Some-se o prazo: Marcos reportou em [09:00] que a Atlas ameaça migrar para o concorrente se a entrega não sair no trimestre, e Larissa estimou três sprints no total em [09:46], com a revisão de segurança embutida no fim.

O dilema, então, não é "assinar ou não assinar". É decidir com que granularidade o material secreto existe e o que acontece quando ele vaza — porque ele vai vazar do lado de alguém, e a resposta a esse vazamento não pode ser "derrubar a integração de todos os clientes ao mesmo tempo".

## Decisão

Assinamos o corpo de cada requisição de webhook com **HMAC-SHA256** e enviamos a assinatura no header **`X-Signature`**. O cliente recalcula o HMAC sobre o corpo recebido e compara com o header antes de confiar no evento. A verificação acontece do lado do cliente — nós apenas assinamos.

O segredo é **único por endpoint de webhook cadastrado**, não global da plataforma. A tabela de configuração de webhook (a criar; o nome não foi definido na reunião) guarda os campos enumerados por Bruno em [09:21]: url, secret, customer_id e estado ativo. O segredo é gerado por nós e devolvido ao cliente na resposta da criação do webhook ([09:31] Marcos). Um cliente com três endpoints cadastrados tem três segredos distintos.

O segredo é **rotacionável pela API**: existe endpoint para o cliente pedir um novo segredo. Na rotação, o segredo antigo continua **válido em paralelo por 24 horas** — o **grace period** — para que o cliente tenha janela de migrar os sistemas dele sem perder eventos. Passadas as 24 horas, o segredo antigo é descartado do nosso lado: nenhuma requisição nossa volta a ser assinada com ele e ele deixa de constar como válido para o endpoint.

Enquanto durar o grace period existem dois segredos válidos para o mesmo endpoint. Qual deles é usado para assinar durante a janela — o novo, o antigo, ou ambos em headers distintos — não foi decidido na reunião e fica em aberto; é pré-requisito para que a janela de 24h cumpra o papel de permitir migração sem perda de eventos.

## Alternativas consideradas

### Segredo global da plataforma — descartada

Um único segredo compartilhado entre o Order Management System e todos os clientes, no molde do que o projeto já faz com `JWT_SECRET` em `src/config/env.ts:8` — um valor único, carregado por variável de ambiente, servindo o sistema inteiro.

Sofia derrubou em [09:21] com uma frase que é o argumento inteiro: "cada endpoint de webhook do cliente tem que ter uma secret única. Não é uma secret global da nossa plataforma. Senão se vaza uma, vaza tudo." O trade-off é o raio de explosão: com segredo global, o vazamento no log de aplicação de um cliente — cenário que Diego relatou já ter ocorrido com um cliente nosso em [09:22] — obrigaria a rotacionar o segredo de todos os clientes simultaneamente, quebrando integrações de quem não teve culpa nenhuma. Com segredo por endpoint, o incidente fica contido no endpoint afetado.

### Nenhuma alternativa foi levantada para o algoritmo

Registrado por honestidade: a escolha do algoritmo foi consenso imediato, não disputa. Bruno perguntou em [09:20] "HMAC com qual algoritmo?" e Sofia respondeu na mesma linha: "SHA-256. HMAC-SHA256 é o padrão de mercado, todo cliente sério tem biblioteca pra isso." Ninguém propôs SHA-1, SHA-512, assinatura assimétrica ou qualquer outro esquema, e o critério declarado foi disponibilidade de biblioteca do lado do cliente — não força criptográfica. Nenhum outro mecanismo de autenticação de saída foi levantado na reunião; se alguém propuser um no futuro, será uma decisão nova e não uma revisão desta.

### Nenhuma alternativa foi levantada para o grace period

Igualmente registrado: Sofia propôs a rotação com validade paralela de 24h em [09:21] já com o número fechado, Diego reforçou a necessidade em [09:22] com o caso real de vazamento, e Sofia consolidou em [09:22]: "Decidido: HMAC-SHA256 sobre o corpo do request, secret por endpoint, suporte a rotação com grace period de 24h." Rotação com corte imediato (sem janela) ou com janela mais longa nunca chegaram a ser propostas. As 24h são um valor de consenso, não o resultado de uma comparação — e é isso que as torna o parâmetro mais barato de revisar neste ADR.

## Consequências

**Positivas**

- O vazamento de um segredo compromete um único endpoint de um único cliente. A resposta ao incidente é rotacionar aquele cadastro, sem tocar nas integrações dos demais.
- O cliente consegue detectar payload adulterado em trânsito e requisição forjada por terceiro, usando apenas biblioteca padrão — segundo Sofia em [09:20], "todo cliente sério tem biblioteca pra isso".
- A rotação é auto-serviço via API ([09:21] Sofia): o cliente que suspeitar de vazamento pede o novo segredo pelo próprio endpoint, sem depender de intervenção manual do nosso lado.
- O grace period de 24h torna a rotação uma operação sem downtime. Sem ele, rotacionar significaria perder eventos entre o corte e o deploy do cliente — e o cliente adiaria a rotação justamente quando ela é urgente.

**Negativas**

- **Existe uma janela de 24h em que dois segredos assinam validamente para o mesmo endpoint.** Se a rotação foi disparada *porque* o segredo antigo vazou, o segredo comprometido continua sendo aceito pelo cliente por até um dia inteiro. Estamos trocando resposta imediata a incidente por continuidade de serviço, conscientemente.
- **Passamos a armazenar material secreto de terceiros por linha de tabela.** O sistema hoje guarda apenas hash de senha (`prisma/schema.prisma:28`, `passwordHash`, gerado com bcrypt em `src/modules/users/user.service.ts:26` e apenas comparado em `src/modules/auth/auth.service.ts:36`) — um dado que ninguém precisa recuperar. Segredo de HMAC é diferente: para assinar, precisamos do valor utilizável, então ele não pode ser tratado como um hash de senha. Cada endpoint cadastrado é mais uma linha de segredo recuperável no nosso MySQL, e um dump do banco passa a valer muito mais do que valia.
- **O redact do logger atual não cobre esse campo.** `src/shared/logger/index.ts:4-11` censura `req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.passwordHash`, `*.token` e `*.accessToken` — nenhum caminho de segredo de webhook está na lista. Enquanto isso não for tratado, qualquer log de objeto de configuração de webhook pode reproduzir do nosso lado exatamente o vazamento que Diego relatou do lado do cliente em [09:22].
- **A verificação é opcional para o cliente.** Nós assinamos, mas não temos como saber se o cliente confere a assinatura. Um cliente que ignore o `X-Signature` aceita qualquer requisição que chegue na URL dele, e a proteção que este ADR paga para construir vira enfeite. O ônus fica na documentação do portal de desenvolvedor ([09:26] Marcos, no contexto de idempotência) e fora do nosso controle.
- **Custo de cronograma travado.** Sofia condicionou o deploy à revisão dela em [09:46]: "Reservem pelo menos dois dias úteis pra eu revisar o código de segurança antes do deploy. HMAC e geração de secret eu quero olhar com calma" — e reforçou na saída, em [09:49]. São dois dias úteis de bloqueio no fim do cronograma de três sprints ([09:46] Larissa), num prazo que já é pressionado pela Atlas ([09:00] Marcos). O caminho crítico de entrega passa por este ADR.

**Neutras / limitações conhecidas**

- **Não temos nenhuma primitiva criptográfica em uso hoje.** Uma busca por `crypto`, `createHmac` e `randomBytes` em `src/` não retorna ocorrência alguma: as únicas dependências criptográficas do projeto são `bcrypt` (hash em `src/modules/users/user.service.ts:26`, comparação em `src/modules/auth/auth.service.ts:36`) e `jsonwebtoken` (assinatura do JWT em `src/modules/auth/auth.service.ts:49`, verificação em `src/middlewares/auth.middleware.ts`) — nenhuma delas serve para HMAC. Geração de segredo e cálculo de assinatura são código novo, sem precedente interno para copiar — o que explica a exigência de revisão dedicada de Sofia.
- **A reunião não definiu como o segredo é armazenado em repouso.** Cifrado no banco, em cofre externo, ou em texto puro na coluna — o assunto não foi discutido. Fica em aberto e cai na revisão de segurança de [09:46]. Gatilho de reabertura: a decisão de storage tomada nessa revisão deve virar registro próprio, não uma nota de rodapé deste ADR.
- **O HMAC cobre o corpo da requisição, não a janela temporal.** Uma requisição capturada íntegra pode ser reenviada e continuará com assinatura válida. Diego incluiu `X-Timestamp` no conjunto de headers em [09:44] justamente "pra cliente conseguir detectar replay attack se quiser" — o "se quiser" é literal: a proteção contra replay é opcional e fica no cliente. Gatilho de reabertura: se algum cliente reportar reprocessamento por replay, ou se a base sair do perfil de três parceiros B2B conhecidos, passa a valer a pena embutir o timestamp no escopo assinado e recusar janelas expiradas do nosso lado. O conjunto de headers em si é parâmetro de contrato e mora no FDD.
- **As 24h de grace period são um número de consenso, não medido.** O gatilho para revisá-lo é empírico: cliente que não consiga migrar dentro da janela e perca eventos puxa o número para cima; um incidente real de vazamento em que a janela mantenha um segredo comprometido vivo tempo demais puxa para baixo — possivelmente para um esquema com revogação imediata explícita, que hoje não existe.
- **O segredo por endpoint não substitui controle de acesso.** Quem pode cadastrar e alterar endpoints de webhook — e portanto quem pode ver segredo de qual cliente — é decisão separada, registrada em [ADR-008](./ADR-008-controle-de-acesso-dos-endpoints.md).

## Rastreabilidade

**Transcrição** (`TRANSCRICAO.md`)

- `[09:19] Sofia:` — expõe o problema: eventos com dados de pedidos saindo para fora da infra; cliente precisa validar origem e integridade do payload
- `[09:20] Sofia:` — "Padrão é HMAC"; assinatura do payload com secret compartilhada, enviada em header `X-Signature`, verificada pelo cliente
- `[09:20] Bruno:` — "HMAC com qual algoritmo?"
- `[09:20] Sofia:` — SHA-256; HMAC-SHA256 como padrão de mercado com biblioteca disponível em qualquer stack
- `[09:21] Sofia:` — secret única por endpoint, não global: "Senão se vaza uma, vaza tudo"
- `[09:21] Bruno:` — campos da tabela de configuração: url + secret + customer_id + estado ativo
- `[09:21] Sofia:` — secret rotacionável via API; antiga válida por 24h em paralelo; depois morre
- `[09:22] Diego:` — caso real: cliente já vazou secret em log de aplicação
- `[09:22] Sofia:` — fecha a decisão: HMAC-SHA256 sobre o corpo do request, secret por endpoint, rotação com grace period de 24h
- `[09:31] Marcos:` — secret gerada por nós e devolvida na criação do webhook
- `[09:44] Diego:` — `X-Signature` no conjunto de headers; `X-Timestamp` para o cliente detectar replay attack "se quiser"
- `[09:46] Sofia:` — dois dias úteis reservados para revisão de segurança de HMAC e geração de secret antes do deploy
- `[09:46] Larissa:` — estimativa de três sprints incluindo a revisão
- `[09:48] Larissa:` — resumo final confirma "HMAC-SHA256 sobre payload, secret por endpoint, rotação com grace period de 24h"
- `[09:49] Sofia:` — reforça o agendamento da revisão de segurança antes de subir
- `[09:00] Marcos:` — pressão de prazo: Atlas pode migrar para o concorrente se não entregarmos no trimestre

**Código** (caminhos verificados no repositório)

- `src/config/env.ts:8` — `JWT_SECRET` validado por Zod com mínimo de 16 caracteres: precedente de segredo único e global, exatamente o modelo descartado aqui
- `src/modules/auth/auth.service.ts:36,49` — `bcrypt.compare` e `jwt.sign`; `src/modules/users/user.service.ts:26` — `bcrypt.hash`; `src/middlewares/auth.middleware.ts:2` — `jsonwebtoken` na verificação. Nenhum uso de HMAC em todo o `src/`
- `src/shared/logger/index.ts:4-11` — lista de `redact` do Pino; não inclui caminho de segredo de webhook
- `prisma/schema.prisma:28` — `User.passwordHash`; único material sensível persistido hoje, e num formato não recuperável, diferente do que este ADR exige
- `src/middlewares/auth.middleware.ts` — autenticação de entrada existente; não cobre requisições de saída
- Tabela de configuração de webhook e código de assinatura — **a criar**; a reunião não fixou nome de tabela nem de arquivo para este ponto

**Relacionado**

- [ADR-001](./ADR-001-outbox-no-mysql.md) — padrão outbox no MySQL (origem do evento assinado)
- [ADR-005](./ADR-005-at-least-once-com-x-event-id.md) — entrega at-least-once com `X-Event-Id` (o mesmo cliente que valida a assinatura precisa deduplicar por evento)
- [ADR-006](./ADR-006-reuso-dos-padroes-do-projeto.md) — reuso dos padrões do projeto (logger Pino, `AppError`, prefixo `WEBHOOK_`)
- [ADR-008](./ADR-008-controle-de-acesso-dos-endpoints.md) — controle de acesso dos endpoints (quem pode cadastrar webhook e, portanto, acessar segredo)
- FDD/PRD — TLS obrigatório na URL (`[09:23] Sofia:` — "nem é decisão arquitetural, é só uma validação no schema Zod"), limite de 64KB de payload (`[09:24] Larissa:`) e o conjunto completo de headers (`[09:43]`–`[09:45]`)
