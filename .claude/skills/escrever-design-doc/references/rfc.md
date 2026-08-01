# Playbook — RFC

**Pergunta:** como resolver, e o que ainda está em aberto.
**Altitude:** arquitetura.
**Extensão:** 2 a 4 páginas. A restrição de tamanho é a regra de design do documento, não uma sugestão.

Um RFC é um convite à revisão, não um relatório. Ele é escrito para ser **contestado** por pares antes de virar código — por isso as duas seções que só ele tem são *alternativas consideradas* e *questões em aberto*. Um RFC sem questões em aberto está escondendo alguma coisa ou foi escrito depois de tudo pronto.

## Estrutura

1. Metadados (autor, status, data, revisores)
2. Resumo executivo (TL;DR)
3. Contexto e problema
4. Proposta técnica
5. Alternativas consideradas
6. Questões em aberto
7. Impacto e riscos
8. Decisões relacionadas (links para ADRs)

## Seção a seção

**Metadados.** Revisores são as pessoas que efetivamente participaram da discussão, com seus papéis. Status reflete o momento real do documento (`Em revisão`, `Aprovado`), não um otimismo.

**TL;DR.** Um parágrafo. Alguém que leia só ele precisa saber o que está sendo proposto e o formato geral da solução. Escreva por último.

**Contexto e problema.** Aqui o RFC diverge do PRD, e a diferença é o que evita duplicação: o PRD narra a **dor do cliente**; o RFC narra a **restrição técnica** que a solução precisa vencer. O que já existe no sistema, o que falta, e por que a solução ingênua não serve.

> "A transação de mudança de status já atualiza o pedido, insere no histórico e movimenta estoque. Acrescentar uma chamada HTTP nesse caminho acopla a disponibilidade do cliente à nossa: cliente lento trava mudança de status de outros pedidos, e cliente fora do ar não pode causar rollback de uma mudança legítima."

**Proposta técnica.** Visão geral: componentes, fluxo entre eles, por que o formato geral resolve o problema. Um diagrama ou um fluxo numerado em texto vale mais que três parágrafos.

O corte com o FDD, que é onde quase todo RFC erra: **o RFC nomeia os componentes e o fluxo; o FDD especifica os contratos.** Diga que existe uma tabela de outbox alimentada na mesma transação; não liste as colunas. Diga que a chamada é assinada; não escreva o algoritmo de canonicalização. Se você está escrevendo um payload de exemplo, você já está no FDD.

**Alternativas consideradas.** Pelo menos duas alternativas **reais** — que foram colocadas na mesa e descartadas — cada uma com o trade-off exato que motivou o descarte. Formato:

> **Redis Streams como transporte de eventos.** Traria consumo reativo e escala horizontal desde o início. Descartada porque exigiria subir e operar infraestrutura nova para um time pequeno, sem ganho proporcional: a outbox no MySQL já existente atende o requisito de latência. — `[hh:mm] Nome:`

Alternativa inventada é pior que alternativa ausente: sugere que a decisão foi mais deliberada do que foi.

**Questões em aberto.** Pelo menos dois pontos levantados e não decididos. Cada um com: o que foi levantado, quem levantou, por que ficou em aberto, e — o que a IA sempre esquece — **o que precisa acontecer para decidir**. Questão em aberto sem gatilho de decisão fica em aberto para sempre.

> **Rate limiting de saída.** Cliente com 50 pedidos mudando de status em um minuto recebe 50 chamadas. Ficou como observar-e-decidir: sem dado de produção, qualquer limite seria arbitrário. **Gatilho:** primeira reclamação de cliente ou pico observado acima de N envios/minuto no mesmo endpoint. — `[hh:mm] Nome:`

**Impacto e riscos.** Impacto no sistema existente (o que muda em código que já roda), no operacional (o que passa a existir para monitorar), e nos consumidores. Risco em nível arquitetural — o PRD já cobriu risco de produto.

**Decisões relacionadas.** Índice dos ADRs: link + uma linha de resumo cada. Esta seção é o que autoriza o RFC a ser curto. Não reproduza a análise do ADR; aponte para ela.

## Armadilhas específicas do RFC

- **Virar FDD.** Sintoma inequívoco: payload de exemplo, assinatura de função, nome de coluna, código de erro, matriz de status HTTP. Se apareceu, mova para o FDD e deixe uma frase.
- **Reproduzir os ADRs.** Se a seção de alternativas do RFC tem o mesmo tamanho da do ADR, apague a do RFC e linke.
- **Questões em aberto de fachada.** Listar como aberto o que a reunião fechou, para parecer humilde. Confronte com a lista de decisões.
- **Estourar o tamanho.** Passou de quatro páginas: o excesso quase sempre é FDD disfarçado. Corte por realocação, não por resumo.
