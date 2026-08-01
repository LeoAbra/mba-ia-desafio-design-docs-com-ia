# Da Reunião ao Documento — processo de produção do pacote de design docs

Este repositório é um **fork do repositório base do desafio**
([devfullcycle/mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia)).
O README original continha o enunciado; este conteúdo o substitui e documenta **como o pacote de
documentação foi produzido**. O enunciado permanece disponível na
[seção final](#o-enunciado-original) e no repositório base.

---

## 1. Sobre o desafio

O ponto de partida são duas fontes e nada mais: a transcrição literal de uma reunião técnica de
aproximadamente 55 minutos (`TRANSCRICAO.md`, cinco participantes, formato `[hh:mm] Nome: fala`) e o
código de um Order Management System real em produção — Node.js + TypeScript + Express + Prisma +
MySQL, com módulos de autenticação, usuários, clientes, produtos e pedidos, máquina de estados de
pedido e controle transacional de estoque. A reunião fecha a decisão de construir um **Sistema de
Webhooks de Notificação de Pedidos**, mas não gera nenhum registro além da própria gravação. A tarefa
é transformar essas duas fontes em um pacote de design docs acionável — PRD, RFC, FDD, ADRs e um
tracker de rastreabilidade — em nível suficiente para um desenvolvedor abrir o FDD e começar a codar.

A restrição que define o trabalho é **não inventar**. Todo requisito, decisão, número, nome de header
ou código de erro precisa ser rastreável a um timestamp da transcrição ou a um caminho de arquivo que
existe no repositório. Isso muda a natureza da tarefa: o desafio não é fazer a IA escrever bastante —
ela escreve bastante com facilidade —, é fazer a IA escrever **só o que a fonte sustenta**, e depois
provar linha a linha que foi isso que aconteceu. A entrega é puramente documental: nenhum arquivo em
`src/`, `prisma/` ou `tests/` foi alterado.

---

## 2. Ferramentas de IA usadas

O ambiente foi um só — **Claude Code, com o modelo Claude Opus**. Mas três recursos distintos dentro
dele cumpriram papéis diferentes o suficiente para merecerem nomes separados:

| Recurso | Papel no trabalho |
| --- | --- |
| **Skills customizadas** (`.claude/skills/`) | Cinco skills escritas especificamente para este desafio. Elas não são prompts avulsos: encodam a metodologia — o que ganha um ADR, qual é a altitude de cada documento, o critério de corte da citação verbatim, as seis checagens da auditoria. Ficam no repositório e são carregadas pelo modelo quando o contexto casa com a descrição. |
| **Orquestração multiagente** | Subagentes paralelos com papéis explicitamente diferentes: extratores por dimensão, escritores por documento, **auditores adversariais** cuja tarefa é derrubar o texto do escritor, e corretores que aplicam o achado. O ganho real não é velocidade: é que o auditor **não é** o escritor, e por isso não defende o próprio texto. |
| **Leitura direta do repositório** | O modelo lê `src/`, `prisma/`, `package.json` e `tests/` diretamente, com Grep e Read. É o que permitiu ancorar os documentos em caminhos de arquivo reais em vez de plausíveis — e é o que produziu os achados da [seção 6](#6-o-que-a-leitura-do-código-revelou-e-a-transcrição-não-dizia), impossíveis de obter lendo só a transcrição. |

As cinco skills:

| Skill | O que encoda |
| --- | --- |
| [`fatos-rastreaveis`](./.claude/skills/fatos-rastreaveis/SKILL.md) | Extração dimensionada em 9 dimensões + verificação adversarial + críticos de completude com lentes distintas |
| [`escrever-adr`](./.claude/skills/escrever-adr/SKILL.md) | Formato MADR, critério do que ganha ADR, obrigatoriedade da consequência negativa |
| [`escrever-design-doc`](./.claude/skills/escrever-design-doc/SKILL.md) | Método comum de PRD/RFC/FDD, com playbooks separados em `references/` |
| [`altitude-docs`](./.claude/skills/altitude-docs/SKILL.md) | Contrato de altitude entre os quatro documentos e a tabela de resolução de duplicação |
| [`auditar-rastreabilidade`](./.claude/skills/auditar-rastreabilidade/SKILL.md) | As seis checagens da auditoria e a montagem do tracker |

---

## 3. Fluxo de trabalho adotado

A ordem real de produção:

```
base de fatos verificada  →  ADRs  →  RFC  →  FDD  →  PRD  →  Tracker  →  README
```

**A base de fatos vem antes de tudo e não é entregável.** É um artefato de trabalho: uma tabela em
que cada linha carrega sua própria prova — citação verbatim mais `[hh:mm] Nome:` ou caminho de
arquivo. Documento escrito em cima dela herda rastreabilidade; documento escrito direto da
transcrição herda alucinação.

**Por que os ADRs primeiro.** As decisões são o esqueleto do resto. Um RFC escrito antes dos ADRs
acaba **decidindo coisas por acidente**, no meio de um parágrafo — o autor precisa de uma premissa
para continuar a frase, inventa uma, e ela nunca passa por escrutínio porque nunca se apresentou como
decisão. Com os ADRs fechados, RFC e FDD tratam cada decisão como premissa e linkam, em vez de
reabri-la.

**Por que o PRD por último.** Sendo o documento de altitude mais alta, com ADRs, RFC e FDD prontos ele
vira consolidação e não descoberta. Quem escreve o PRD primeiro escreve um PRD que promete o que a
engenharia ainda não sabe se consegue entregar.

**O padrão que se repetiu em cada etapa** — e é o núcleo do método:

```
escrever  →  auditar adversarialmente  →  corrigir verificando na fonte antes de aplicar
```

Três papéis, três agentes distintos. O escritor produz. O **auditor** recebe uma instrução explícita
de que o default é ceticismo e de que seu trabalho é derrubar itens, não aprová-los. O **corretor**
recebe o achado com a instrução inversa: tentar **refutá-lo** abrindo a fonte, e só aplicar o que
sobreviver. Sem esse terceiro passo, a auditoria gera retrabalho em cima de texto que já estava
correto — auditor erra para o lado de reportar falso positivo, porque encontrar problema é o
incentivo da tarefa.

---

## 4. Prompts personalizados

### 4.1 Auditor adversarial de ADR

**Problema que resolve:** um escritor de ADR revisando o próprio texto não encontra nada, porque
revisão de leitura não pega o erro dominante — fato real atribuído ao timestamp errado. O texto
*parece* certo. Este prompt separa o revisor do autor e dá a ele nove checagens mecânicas, cada uma
correspondendo a uma categoria de erro observada na prática. A instrução final — **fornecer o texto
exato de substituição** — existe porque achado sem substituto vira debate; achado com substituto vira
patch.

```
Você é auditor adversarial de UM ADR. Você NÃO é o autor dele e não tem interesse
em defendê-lo. Seu default é CETICISMO: sua tarefa é DERRUBAR afirmações, não
aprová-las. Um relatório sem achados é um relatório suspeito — releia.

Fontes de verdade, nesta ordem: (1) o código do repositório, (2) TRANSCRICAO.md,
(3) a base de fatos. Nada mais. Abra os arquivos; não confie na sua memória do
que eles dizem.

Rode as NOVE checagens abaixo como passes independentes. Passes misturados
encontram menos.

1.  CITAÇÃO VERBATIM — toda citação entre aspas existe LITERALMENTE na
    transcrição? Busque a string exata. Paráfrase entre aspas é achado.
2.  TIMESTAMP E FALANTE — todo `[hh:mm] Nome:` citado existe, com AQUELE
    falante naquele minuto, E a fala daquele ponto sustenta a afirmação?
    Timestamp plausível cuja fala não diz aquilo é o erro mais comum e o
    menos visível. Confira os três: minuto, nome, conteúdo.
3.  DECISORES — cada pessoa listada como decisora tem fala real no trecho da
    reunião em que a decisão é fechada? Decisor silencioso é fabricação.
4.  NÚMEROS E LITERAIS — todo número, intervalo, timeout, nome de header,
    nome de tabela, nome de evento e código de erro bate com a fonte?
    Número específico onde a fonte não deu número é pior que número ausente.
    Confira também a ARITMÉTICA interna e a coerência entre bullets: piso
    e teto do mesmo número em parágrafos vizinhos é autocontradição.
5.  CAMINHOS DE ARQUIVO — todo caminho citado existe no repositório? Arquivo
    que a feature vai criar está marcado inequivocamente "(a criar)"?
    E o que o ADR AFIRMA sobre o arquivo é verdade quando você o abre?
6.  ALTERNATIVA REAL — cada alternativa descartada foi mesmo levantada na
    reunião, por alguém, com o argumento que a derrubou? Alternativa
    plausível que ninguém discutiu é ruído: grepe o termo na transcrição.
7.  CONTAMINAÇÃO DE ESCOPO — algo descartado ou adiado na reunião aparece
    aqui como decisão, requisito ou consequência positiva?
8.  FRONTEIRA — este ADR está decidindo algo que pertence a outro ADR do
    conjunto, ou descendo para detalhe de implementação que é do FDD?
9.  VAGO E SUPERAFIRMAÇÃO — alguma frase sobreviveria à troca do nome da
    feature? Algum benefício é vendido sem o custo correspondente? Existe
    pelo menos uma consequência NEGATIVA concreta e nomeada?

Para CADA achado, devolva exatamente:

  - categoria: timestamp-invalido | numero-divergente | citacao-inexistente |
      caminho-inexistente | alternativa-inventada | escopo-contaminado |
      fronteira-violada | vago | decisor-fabricado
  - severidade: BLOQUEANTE | ALTA | MEDIA | BAIXA
  - trecho atual: a citação exata do ADR, copiada
  - por que está errado: a evidência da fonte que o contradiz, com
      `[hh:mm] Nome:` ou `caminho/arquivo.ts:linha`
  - TEXTO EXATO DE SUBSTITUIÇÃO: o parágrafo reescrito, pronto para colar.
      Não descreva a correção. Escreva-a.

Não proponha melhoria de estilo. Não elogie. Se não houver achado numa
checagem, diga qual verificação você fez para concluir isso.
```

### 4.2 Extração dimensionada com regra de corte da citação verbatim

**Problema que resolve:** "extraia os requisitos da transcrição" devolve o subconjunto óbvio. As
dimensões *fora de escopo* e *questões em aberto* simplesmente não aparecem — um extrator otimista não
caça negação. A solução foi uma dimensão por agente, com a lente explícita, e um critério de corte
binário que impede o item duvidoso de sobreviver como "incerto".

```
Você é extrator de UMA única dimensão da transcrição: <DIMENSÃO>.
Ignore tudo que não for dessa dimensão — outro agente cuida das demais.
Sua lente: <o que essa dimensão caça>.

REGRA DE CORTE, inegociável:
  Se você não consegue colar um recorte LITERAL da transcrição que sustente
  a afirmação, a afirmação NÃO EXISTE. Item sem citação verbatim é
  DESCARTADO — não é marcado como "incerto", não é registrado com ressalva,
  não entra com nota de rodapé. Sai.

Leia a transcrição INTEIRA antes de extrair qualquer coisa. Duas armadilhas:
  - A CAUDA: a conversa continua depois que parte dos participantes sai da
    call. As decisões dos últimos minutos, entre duas pessoas, costumam ser
    as mais concretas (modelagem, tipo de chave, formato de dado) e são as
    que mais escapam.
  - A CORREÇÃO DE ROTA: alguém propõe X no minuto 10 e o grupo converge para
    Y no minuto 25. Registrar X é documentar o oposto da decisão final.
    Nunca fixe um item antes de ler o fechamento do assunto.

Cada item extraído carrega SETE campos, sem exceção:
  id          — <FAMÍLIA>-NN, sequencial
  tipo        — a dimensão
  conteudo    — uma linha. Se não cabe em uma linha, o item está composto
                demais: quebre em dois.
  detalhe     — o que a implementação precisa saber
  fonte       — TRANSCRICAO | CODIGO
  localizacao — TRANSCRICAO: exatamente `[hh:mm] Nome:`, COPIADO do arquivo.
                Timestamp chutado é erro, não aproximação.
                CODIGO: caminho relativo real, opcionalmente :linha.
                Abra o arquivo e confirme que existe antes de citá-lo.
  citacao     — o recorte verbatim que sustenta o item

Saída: uma tabela markdown com essas sete colunas. Sem introdução, sem
conclusão, sem "espero ter ajudado".
```

### 4.3 Corretor com refutação obrigatória antes de aplicar

**Problema que resolve:** aplicar cegamente a lista do auditor introduz erro novo. Auditor erra para o
lado do falso positivo — encontrar problema é o incentivo da tarefa dele. Este prompt inverte o ônus
da prova: o corretor precisa **tentar demonstrar que o documento está certo**, e só aplica o que
sobreviver à tentativa.

```
Você recebe UM ADR e o relatório do auditor sobre ele. Você NÃO é o auditor
e não deve confiar nele.

Para CADA achado, antes de tocar no arquivo, sua primeira tarefa é REFUTAR o
achado: abrir a fonte primária e tentar demonstrar que o DOCUMENTO está certo
e o AUDITOR está errado. Não é formalidade. É o passo que impede retrabalho
em cima de texto que já estava correto.

  - achado de timestamp/citação  -> abra TRANSCRICAO.md, ache o minuto, leia
                                    a fala inteira e as duas ao redor
  - achado de número/literal     -> grepe o literal na transcrição E no código
  - achado de caminho de arquivo -> abra o caminho; confirme existência E o
                                    que ele de fato faz
  - achado de alternativa        -> grepe o termo na transcrição inteira.
                                    Zero ocorrências confirma o achado.

Depois da tentativa de refutação, classifique:

  SOBREVIVEU  -> aplique a correção.
  REFUTADO    -> NÃO altere o documento. Registre por que o auditor errou,
                 com a evidência que o refuta.
  AJUSTADO    -> o achado procede, mas o texto de substituição proposto pelo
                 auditor tem problema (redundância com o que já está no
                 parágrafo, ou truncamento de conteúdo correto que já existia).
                 Aplique uma versão corrigida e diga o que mudou e por quê.

Ao aplicar: mude o MÍNIMO necessário. Não reescreva parágrafo que não foi
objeto de achado. Não "melhore" nada de passagem. Não remova conteúdo correto
para encurtar.

Devolva o relatório de aplicação: um item por achado, com o veredito, a
evidência que o sustenta e o diff conceitual do que mudou.
```

---

## 5. Iterações e ajustes

Foram **três ciclos principais** de geração → auditoria → correção — base de fatos, ADRs e os três
design docs —, mais uma passada final de tracker, revisão de fronteira entre documentos e este README.

### Ciclo 1 — base de fatos (22 agentes)

Nove extratores por dimensão, nove auditores adversariais, três críticos de completude com lentes
distintas e um editor consolidador.

O achado do ciclo foi metodológico: **extração genérica devolve o subconjunto óbvio**. As dimensões
*fora de escopo* e *questões em aberto* só apareceram com extrator dedicado.

E a correção de rota mais cara do trabalho foi pega **apenas** pelo crítico de lente negativa: em
`[09:11] Bruno:` ele diz que o worker vai "usar o mesmo Prisma client"; em `[09:30] Bruno:`, provocado
por Diego, ele se corrige — *"Separado. PrismaClient é por processo. Mesmo banco, mesma DATABASE_URL,
mas instância nova porque é outro processo Node."* Quem lê só a primeira ocorrência documenta o
**oposto** da decisão final, com timestamp válido e citação verbatim correta.

### Ciclo 2 — ADRs (25 agentes, ~53 achados)

Oito escritores, oito auditores, oito corretores e um revisor de conjunto. Os achados por categoria:

| Categoria | Ocorrências | O que é |
| --- | --- | --- |
| `timestamp-invalido` | 16 | timestamp plausível, mas a fala naquele ponto não diz aquilo |
| `numero-divergente` | 12 | número ou literal que a reunião não disse |
| `vago` | 7 | frase que sobreviveria à troca do nome da feature |
| `fronteira-violada` | 5 | ADR invadindo o território de outro ADR |
| `escopo-contaminado` | 4 | item descartado ou adiado aparecendo como decisão |
| `citacao-inexistente` | 4 | citação que não existe literalmente na fonte |
| `caminho-inexistente` | 3 | arquivo citado que não existe no repositório |
| `alternativa-inventada` | 2 | alternativa plausível que ninguém discutiu |

> **O modo de falha dominante não foi inventar fato absurdo — foi atribuir fato real ao timestamp
> errado.** 16 de ~53. A IA sabia o que tinha sido decidido e errava *quem* decidiu e *quando*.

Isso tem uma consequência prática: **revisão de leitura não pega**. O documento lido isoladamente é
coerente, técnico e plausível. Só a verificação linha a linha contra a fonte encontra o erro — que é
exatamente o motivo de o tracker existir.

#### Exemplos concretos (todos corrigidos)

1. **Autocontradição numérica.** O ADR-002 afirmava latência "até 2 segundos" e, no bullet seguinte,
   que 2s era *piso*. A transcrição diz `[09:10] Larissa:` — *"A latência mínima vai ser 2 segundos no
   pior caso"* —, ou seja, **teto**. Duas leituras opostas do mesmo número dentro do mesmo documento,
   a poucas linhas de distância.

2. **Alternativa inventada.** O ADR-002 listava `setInterval` como alternativa descartada. A palavra
   não aparece **uma vez sequer** na transcrição (`grep -c "setInterval" TRANSCRICAO.md` → `0`).
   Também apareceram "broker", "offset" e "consumer group" — vocabulário de mensageria que ninguém
   usou na reunião, e que a IA trouxe porque *combina* com o assunto.

3. **Erro sobre o próprio código.** O ADR-002 afirmava que `createLogger()` já era usada por
   `src/server.ts`. Falso: `src/server.ts:4` importa apenas o singleton `logger`; `createLogger()` só
   é chamada dentro do próprio `src/shared/logger/index.ts:32`. Erro invisível para quem não abre os
   dois arquivos — e que induziria a implementação a um padrão inexistente.

4. **Decisor fabricado.** Sofia aparecia como decisora em blocos em que não emite nenhuma fala. Ela
   fala em 09:02–09:03 e só volta em 09:19; nos intervalos 09:08–09:14 e 09:29–09:30 está silenciosa.

5. **Ameaça comercial inflada.** O ADR-003 dizia que os três clientes podiam migrar. A fonte
   (`[09:00] Marcos:`) atribui a hipótese **só à Atlas**, e em tom condicional.

6. **Superafirmação de benefício.** O ADR-001 vendia "zero infraestrutura nova". Falso: o worker é uma
   segunda unidade de deploy (`[09:11] Diego:` — *"o worker tem que rodar como processo separado"*).
   Corrigido para "nenhum componente com estado novo", com o custo do segundo processo explicitado.

#### O harness de verificação também errou

O passo de refutação obrigatória se justificou por um motivo desconfortável: **o próprio harness de
verificação gerou falso positivo**. A varredura automatizada que confronta cada `[hh:mm] Nome:` citado
contra a transcrição acusou divergências que, ao abrir a fonte manualmente, não existiam — o defeito
estava na comparação do verificador, não nos documentos. Ele precisou ser depurado antes que sua saída
contasse como evidência. Ferramenta de verificação não é fonte de verdade; a fonte é.

Nos oito ADRs, todos os achados do auditor sobreviveram à tentativa de refutação — mas **dois textos de
substituição propostos pelo auditor foram ajustados na aplicação**, por produzirem redundância (*"O
trade-off que a derrubou: o argumento que a derrubou foi…"*) ou por truncar conteúdo correto que já
existia no parágrafo. Aplicação cega da lista teria degradado o documento.

### Ciclo 3 — RFC, FDD e PRD

Mesmo padrão — escrita, auditoria adversarial, correção verificada —, mais uma revisão específica de
**fronteira entre os quatro documentos**: altitude, duplicação, consistência numérica e de nomes, e
integridade dos links. É a passada que impede o RFC de descer para detalhe de payload e o PRD de
subir decisão técnica.

---

## 6. O que a leitura do código revelou e a transcrição não dizia

Esta seção é o diferencial concreto de ter dado ao modelo acesso ao repositório em vez de só à ata.
**São achados de análise, não falas da reunião** — e estão marcados como tal nos documentos, nunca
atribuídos a um participante.

**A ordem da transação do `changeStatus` na transcrição diverge do código.** Em `[09:40] Bruno:` a
transação é descrita como *"faz update na order, insere no history e atualiza estoque"*. A ordem real
em `src/modules/orders/order.service.ts` é outra: busca do pedido com `include: { items: true }`
(`:132-136`), validação de transição (`:138-149`), **estoque** (`:151-156`), `order.update`
(`:158`), `orderStatusHistory.create` (`:159-167`) e refetch (`:169-177`). Documentação que repete a
ata induz erro na implementação — o FDD registra a sequência real, posiciona a inserção na outbox no
ponto certo e declara explicitamente que o código prevalece sobre a descrição da reunião.

**Não existe vínculo `User` ↔ `Customer` no schema.** Em `[09:32] Marcos:` afirma-se que *"a gente tem
usuários que representam o cliente"*. O `prisma/schema.prisma` não sustenta isso: `model User` tem
apenas as relações `createdOrders` e `statusHistoryChanges`, e `model Customer` tem apenas `orders`.
Não há campo, chave estrangeira nem tabela de junção ligando um usuário a um customer. Combinado com a
decisão do ADR-008 (CRUD de webhook exige apenas autenticação), a consequência é concreta: qualquer
usuário autenticado pode cadastrar um endpoint apontando para uma URL própria informando o
`customer_id` de qualquer cliente, e o sistema **não tem como verificar posse** — a informação não
existe no modelo. Entra no FDD como risco de segurança com mitigação, e dá causa raiz nomeada à
questão em aberto de RBAC no RFC.

**Nenhuma dependência nova é necessária.** `node:crypto` cobre o HMAC-SHA256, o `fetch` global cobre a
chamada HTTP (`engines.node >= 20`) e `uuid@11` já está instalado. Descobrir isso exigiu abrir o
`package.json` — supor teria produzido uma seção de dependências com bibliotecas que o projeto não
precisa.

Outros achados que entraram nos documentos pela leitura do código:

- **Já existe mecanismo de correlação.** `src/middlewares/request-logger.middleware.ts` lê ou gera
  `X-Request-Id` e o devolve no header — é nele que o tracing do fluxo assíncrono se apoia, em vez de
  inventar um novo.
- **Já existe precedente de RBAC.** `src/modules/users/user.routes.ts:15` usa `requireRole('ADMIN')`
  na ordem `authenticate → requireRole → validate → controller`.
- **Dois limites de payload diferentes.** `express.json({ limit: '1mb' })` em `src/app.ts:59` é limite
  de *entrada* da API; o teto de 64KB é do evento de *saída*. Camadas distintas, que um documento
  desatento fundiria.
- **O `redactPaths` do Pino não cobre `secret` nem `signature`.** Em `src/shared/logger/index.ts` ele
  cobre `authorization`, `cookie`, `password`, `passwordHash`, `token` e `accessToken` — o FDD exige a
  extensão da lista na seção de observabilidade.
- **O "padrão de módulo" do projeto tem exceção.** O módulo `auth` não tem repository próprio; o
  ADR-006 registra isso em vez de afirmar uma uniformidade que não existe.

---

## 7. Como navegar a entrega

Ordem sugerida de leitura — do porquê ao como:

| # | Arquivo | O que você encontra |
| --- | --- | --- |
| 1 | [`docs/PRD.md`](./docs/PRD.md) | Por que a feature existe: os três clientes B2B que a pediram, o risco comercial, escopo e fora de escopo, requisitos funcionais e não funcionais, métricas com meta quantitativa, riscos e critérios de aceitação. Sobrevive a uma troca completa de arquitetura. |
| 2 | [`docs/RFC.md`](./docs/RFC.md) | A proposta técnica submetida a revisão: abordagem geral (outbox + worker), alternativas reais descartadas com o trade-off de cada uma, questões deixadas em aberto e links para os ADRs. Conciso — não desce ao detalhe do FDD. |
| 3 | [`docs/adrs/README.md`](./docs/adrs/README.md) | Índice dos ADRs, com resumo de cada decisão **e o preço pago por ela**, o mapa de como as oito se relacionam e o que ficou fora desta fase. É o melhor ponto de entrada nas decisões. |
| 4 | [`docs/adrs/ADR-001` … `ADR-008`](./docs/adrs/) | Uma decisão por arquivo, em MADR: outbox no MySQL, worker separado com polling, retry com backoff e DLQ, HMAC-SHA256 com segredo por endpoint, at-least-once com `X-Event-Id`, reuso dos padrões do projeto, snapshot do payload na inserção e controle de acesso dos endpoints. Cada um com alternativa real e consequência negativa nomeada. |
| 5 | [`docs/FDD.md`](./docs/FDD.md) | Como construir: fluxos detalhados, contratos HTTP com payloads de exemplo, matriz de erros `WEBHOOK_*`, resiliência, observabilidade e a seção **"Integração com o sistema existente"**, ancorada em caminhos de arquivo reais. É o documento que um dev abre para começar a codar. |
| 6 | [`docs/TRACKER.md`](./docs/TRACKER.md) | A referência cruzada: 624 linhas ligando cada item ao seu `[hh:mm] Nome:` ou caminho de arquivo. 84,0% com fonte `TRANSCRICAO`, 100 linhas com fonte `CODIGO`. É o instrumento que prova que nada foi inventado — e o lugar onde começar se você desconfiar de qualquer afirmação dos documentos acima. |
| 7 | [`TRANSCRICAO.md`](./TRANSCRICAO.md) | A fonte primária, não alterada. |

E o método, se você quiser reproduzi-lo:

| Arquivo | Conteúdo |
| --- | --- |
| [`.claude/skills/fatos-rastreaveis/SKILL.md`](./.claude/skills/fatos-rastreaveis/SKILL.md) | Como extrair uma base de fatos verificada de uma conversa não estruturada |
| [`.claude/skills/escrever-adr/SKILL.md`](./.claude/skills/escrever-adr/SKILL.md) | O que ganha ADR, formato MADR e as regras de escrita |
| [`.claude/skills/escrever-design-doc/SKILL.md`](./.claude/skills/escrever-design-doc/SKILL.md) | Método comum de PRD/RFC/FDD, com playbooks em [`references/`](./.claude/skills/escrever-design-doc/references/) |
| [`.claude/skills/altitude-docs/SKILL.md`](./.claude/skills/altitude-docs/SKILL.md) | O contrato de altitude e a resolução de duplicação entre documentos |
| [`.claude/skills/auditar-rastreabilidade/SKILL.md`](./.claude/skills/auditar-rastreabilidade/SKILL.md) | As seis checagens da auditoria e a montagem do tracker |

---

## O enunciado original

O enunciado completo do desafio — cenário, requisitos, critérios de aceite e estrutura obrigatória do
entregável — está no README do repositório base, do qual este repositório é fork:

**https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia**

O código da aplicação (`src/`, `prisma/`, `tests/` e configurações) permanece **inalterado**: ele serve
de contexto e referência, e a entrega deste desafio é puramente documental.
