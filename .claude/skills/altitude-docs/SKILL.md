---
name: altitude-docs
description: "Contrato de altitude entre PRD, RFC, ADR e FDD: define qual pergunta cada documento responde, o que pertence a cada um e como resolver conteúdo duplicado entre eles. Use ao planejar um pacote de design docs, ao decidir em qual documento uma informação entra, ou ao revisar documentos que estão se repetindo. Frases-gatilho: 'isso vai no RFC ou no FDD?', 'os documentos estão repetitivos', 'qual a diferença entre PRD e RFC', 'pacote de documentação técnica'."
---

# Altitude entre documentos

Um pacote de design docs falha de duas maneiras. Ou os documentos **se repetem** — e aí o leitor não sabe qual é a fonte da verdade, e a manutenção duplica. Ou têm **buracos** — e aí a informação não está em lugar nenhum.

As duas falhas têm a mesma causa: ninguém definiu a altitude de cada documento antes de escrever.

## O contrato

| Documento | Pergunta que responde | Altitude | Leitor-alvo |
|---|---|---|---|
| **PRD** | Por quê e o quê? | Produto / negócio | PM, stakeholder, liderança |
| **RFC** | Como resolver, e o que ainda está aberto? | Arquitetura | Revisores técnicos, pares |
| **ADR** | Por que decidimos exatamente assim? | Decisão pontual | Quem herdar o sistema |
| **FDD** | Como construir, em detalhe? | Implementação | Quem vai codar amanhã |

Em uma frase: **o RFC propõe e abre para revisão, os ADRs registram cada decisão fechada, o FDD detalha como construir, o PRD justifica por que vale a pena.**

## Teste de altitude

Antes de escrever um parágrafo, pergunte para quem ele serve.

- Se o parágrafo deixa de fazer sentido caso a solução técnica mude → **PRD não é o lugar.** O PRD sobrevive a uma troca completa de arquitetura.
- Se o parágrafo contém um payload de exemplo, uma assinatura de função, um nome de coluna ou um código de erro → **FDD.** Nunca RFC.
- Se o parágrafo tem a forma "escolhemos X em vez de Y porque Z" → **ADR.** O RFC pode *referenciar* a decisão em duas linhas e linkar; não pode reproduzi-la inteira.
- Se o parágrafo descreve algo que ainda não foi decidido → **RFC, seção de questões em aberto.** FDD não especula; FDD especifica.

## Resolvendo duplicação

Conteúdo repetido entre dois documentos quase nunca significa "corte de um deles". Significa que o conteúdo está na altitude errada em pelo menos um.

Padrões recorrentes e a resolução correta:

| Sintoma | Diagnóstico | Resolução |
|---|---|---|
| RFC descrevendo o payload campo a campo | RFC desceu para altitude de FDD | RFC diz "o evento carrega um snapshot enxuto do pedido"; FDD lista os campos |
| PRD justificando escolha de tecnologia | PRD subiu decisão técnica | PRD diz o requisito de negócio (latência < 10s); ADR diz por que polling de 2s atende |
| ADR e RFC com a mesma seção de alternativas | RFC reproduziu o ADR | ADR carrega a análise completa; RFC resume em uma linha por alternativa e linka |
| FDD explicando por que a decisão foi tomada | FDD virou ADR | FDD afirma a decisão como premissa e linka o ADR; o "porquê" mora no ADR |
| PRD e RFC com o mesmo "contexto e problema" | Ambos narrando a origem | PRD narra a dor do cliente e o risco comercial; RFC narra a restrição técnica que a solução precisa vencer |

## Direção das referências

Referência cruzada é o que substitui duplicação. A direção importa:

```
PRD  ──(o que precisa existir)──►  RFC  ──(decisões que sustentam)──►  ADRs
                                    │                                    ▲
                                    └────(como construir)──► FDD ────────┘
```

- **RFC → ADR**: link direto, uma linha de resumo. O RFC é o índice das decisões.
- **FDD → ADR**: o FDD trata a decisão como premissa fechada e linka. Não reabre.
- **FDD → código**: caminhos de arquivo reais. É o que torna o FDD acionável.
- **PRD → nada técnico**: o PRD não deve precisar linkar ADR para ser compreendido.

## Ordem de produção

Contraintuitiva, e é de propósito:

1. **ADRs primeiro.** As decisões são o esqueleto. Escrever RFC antes dos ADRs produz um RFC que decide coisas por acidente, no meio de um parágrafo.
2. **RFC.** Consolida as decisões já fechadas em uma proposta coerente, e é onde alternativas descartadas e questões em aberto encontram lugar natural.
3. **FDD.** Com decisões formalizadas, o detalhamento se constrói em cima delas em vez de inventá-las.
4. **PRD por último.** Sendo o mais alto nível, com o resto na mão ele vira consolidação, não descoberta.
5. **Tracker** varrendo tudo, no fim ou em paralelo.

Quem escreve PRD primeiro acaba escrevendo um PRD que promete o que a engenharia ainda não sabe se consegue.

## Extensão obrigatória em cada documento

Peculiaridade útil: cada documento precisa de **pelo menos uma seção que só ele tem**. Se o RFC não tem nada que não esteja no FDD, o RFC não deveria existir.

- PRD exclusivo: métricas de sucesso, público-alvo, critérios de aceitação de produto.
- RFC exclusivo: alternativas com trade-off, questões em aberto, revisores.
- ADR exclusivo: consequências negativas assumidas conscientemente.
- FDD exclusivo: contratos, matriz de erros, integração com arquivos reais, observabilidade.
