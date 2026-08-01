---
name: fatos-rastreaveis
description: "Extrai uma base de fatos rastreável a partir de uma fonte primária (transcrição de reunião, ata, thread de decisão) cruzada com o código do repositório, e verifica cada item adversarialmente contra a fonte antes de liberá-lo para uso. Use ANTES de escrever qualquer design doc (PRD, RFC, FDD, ADR) quando o material de origem for uma conversa não estruturada. Frases-gatilho: 'extrair requisitos da transcrição', 'o que ficou decidido nessa reunião', 'levantar os fatos antes de documentar', 'base de fatos', 'de onde veio esse requisito'."
---

# Fatos rastreáveis

Documento de design bom não começa escrevendo documento. Começa separando **o que foi realmente dito** do **que a IA acha que faria sentido ter sido dito**.

Esta skill produz um artefato intermediário — a **base de fatos** — em que cada linha carrega sua própria prova: um timestamp da fonte ou um caminho de arquivo real. Documentos escritos em cima da base de fatos herdam a rastreabilidade de graça. Documentos escritos direto da transcrição herdam alucinação de graça.

## Quando NÃO usar

- A fonte já é estruturada (um PRD existente, um schema, uma spec). Não há o que desambiguar.
- O pedido é para escrever um documento sobre algo que ainda não foi discutido. Não há fonte primária; extrair é inventar.

## Princípio inegociável

> Se você não consegue colar um recorte literal da fonte que sustente uma afirmação, a afirmação não existe.

Não é uma diretriz de estilo. É o critério de corte. Item sem citação verbatim é descartado, não "marcado como incerto".

## Procedimento

### Passo 1 — Ler a fonte inteira, uma vez, sem extrair nada

Leia a transcrição do começo ao fim antes de qualquer extração. Duas armadilhas específicas de transcrição de reunião:

- **A cauda.** Conversas continuam depois que parte dos participantes sai da call. Decisões tomadas nos últimos minutos, entre duas pessoas, são frequentemente as mais concretas (modelagem, tipo de chave, formato de dado) e são as que mais escapam.
- **A correção de rota.** Alguém propõe X no minuto 10, o grupo converge para Y no minuto 25. Extrator desatento registra X como decisão. Sempre leia até o fechamento antes de fixar um item.

### Passo 2 — Varrer por dimensões, em paralelo

Não peça "extraia os requisitos". Peça uma dimensão por vez, com um extrator dedicado a cada. Extração genérica devolve o subconjunto óbvio; extração dimensionada devolve a cauda longa.

As nove dimensões:

| Dimensão | O que caça | Família de ID |
|---|---|---|
| Decisões fechadas | Onde o grupo convergiu e alguém fechou | `DEC-NN` |
| Requisitos funcionais | Comportamento observável que o sistema deve ter | `RF-NN` |
| Requisitos não funcionais | Números, limites, prazos, algoritmos, protocolos | `RNF-NN` |
| Fora de escopo | Recusado, adiado, "próxima fase", "de outro time" | `FE-NN` |
| Questões em aberto | Levantado e não resolvido, "a gente observa" | `QA-NN` |
| Alternativas descartadas | Bifurcação técnica + o trade-off que derrubou | `ALT-NN` |
| Contratos e modelagem | Rotas, campos, headers, tabelas, códigos — literais | `CT-NN` |
| Âncoras de código | Arquivos reais do repo e o ponto de integração | `COD-NN` |
| Contexto de negócio | Quem pediu, a dor, o risco comercial, o prazo | `NEG-NN` |

Cada item extraído carrega sete campos: `id`, `tipo`, `conteudo` (uma linha), `detalhe`, `fonte` (`TRANSCRICAO` ou `CODIGO`), `localizacao`, `citacao` (verbatim).

Formato de `localizacao`:
- `TRANSCRICAO` → exatamente `[hh:mm] Nome:`, copiado da fonte. Timestamp chutado é erro, não aproximação.
- `CODIGO` → caminho relativo real, opcionalmente `:linha`. Verifique que o arquivo existe antes de citá-lo.

### Passo 3 — Verificar adversarialmente, dimensão a dimensão

Cada dimensão extraída passa por um auditor cujo trabalho é **derrubar itens**, não aprová-los. O auditor não é o extrator, e o prompt do auditor deve dizer isso com todas as letras: o default é ceticismo.

Cinco checagens por item:

1. A citação aparece **literalmente** na fonte? Se não → `REJEITADO`.
2. Timestamp e nome do falante estão exatos? Erro de um minuto é erro → `CORRIGIDO`.
3. A citação **sustenta** a afirmação, ou o extrator extrapolou? Extrapolação → reescreva para o que a fonte suporta, ou rejeite.
4. O item é na verdade algo **descartado ou adiado** vendido como decisão? → `REJEITADO`, com nota.
5. Item de código: o caminho existe e o trecho está mesmo lá? → caminho inexistente é `REJEITADO`.

Vereditos: `CONFIRMADO`, `CORRIGIDO`, `REJEITADO`. Só os dois primeiros seguem adiante.

### Passo 4 — Críticos de completude, com lentes distintas

Verificação remove item ruim; não acha item faltante. Rode críticos com lentes **diferentes entre si** — redundância não acha o que a redundância não vê:

- **Lente linha-a-linha** — releia a fonte inteira, incluindo a cauda, e aponte o que não está na lista.
- **Lente negativa** — cace exclusivamente negações, recusas, adiamentos, ressalvas, condicionais ("por enquanto", "se virar problema") e duvidas não respondidas. É a lente que popula `FE-NN` e `QA-NN`, as duas famílias que extrator otimista subrepresenta.
- **Lente de implementação** — leia como quem vai codar: campos, nomes literais, ordem de operações, pontos de integração. Aponta também contradições entre a lista e o código real.

### Passo 5 — Consolidar

Um editor único deduplica, reclassifica, renumera com IDs estáveis por família e faz uma última validação por amostragem agressiva contra a fonte.

O arquivo final abre com uma **síntese executiva** — as decisões principais nomeadas, a lista de fora de escopo, a lista de questões em aberto, e o mapa de arquivos reais do código. É essa síntese que os autores dos documentos leem primeiro; as tabelas completas são consulta.

## Saída

Um arquivo Markdown com uma seção por família, cada uma uma tabela:

```
| ID | Tipo | Conteúdo (resumo) | Detalhe | Fonte | Localização | Citação |
```

A base de fatos é **artefato de trabalho**, não entregável. Mantenha-a fora do diretório de entrega.

## Contrato com as skills seguintes

`escrever-adr`, `escrever-design-doc` e `auditar-rastreabilidade` assumem que a base de fatos existe e que todo ID citado por elas resolve para uma linha aqui. Um documento que precisa afirmar algo sem ID correspondente está sinalizando uma de duas coisas: a extração ficou incompleta, ou o autor está inventando. Nos dois casos, volte para cá antes de escrever.
