---
name: escrever-design-doc
description: "Escreve PRD, RFC ou FDD a partir de uma base de fatos rastreável e de ADRs já fechados, respeitando a altitude de cada documento. Use ao produzir documentação de design de uma feature — requisitos de produto, proposta técnica para revisão, ou especificação de implementação. Frases-gatilho: 'escrever o PRD', 'montar o RFC', 'especificar a feature', 'documento de design', 'FDD'."
---

# Escrever design doc

Três documentos, três alturas, um método. Esta skill cobre o método comum; o playbook específico de cada tipo está em `references/`.

Carregue **apenas** a referência do documento que você está escrevendo:

- `references/prd.md` — Product Requirements Document
- `references/rfc.md` — Request for Comments
- `references/fdd.md` — Feature Design Document

## Pré-requisitos

1. **Base de fatos verificada** (`fatos-rastreaveis`). Sem ela, você vai escrever ficção coerente.
2. **ADRs fechados** (`escrever-adr`), se o documento for RFC ou FDD. Ambos tratam decisões como premissa e linkam; nenhum dos dois decide.
3. **Contrato de altitude lido** (`altitude-docs`). É o que impede os três documentos de virarem o mesmo documento em três arquivos.

## Método comum

### Passo 1 — Selecionar da base de fatos, não da fonte

Filtre a base de fatos pelas famílias que pertencem à altitude do documento. Escrever direto da transcrição é o erro que anula todo o trabalho de extração: você reintroduz o que a verificação removeu.

| Documento | Famílias que consome |
|---|---|
| PRD | `NEG`, `RF`, `RNF`, `FE`, `QA` (como risco) |
| RFC | `DEC`, `ALT`, `QA`, `RNF`, `NEG` (só o problema) |
| FDD | `CT`, `COD`, `RF`, `RNF`, `DEC` (como premissa) |

### Passo 2 — Escrever com o ID ao lado

Durante a redação, mantenha o ID da base de fatos junto de cada afirmação. Você remove os IDs do texto final se atrapalharem a leitura, mas eles alimentam o tracker e, mais importante, **denunciam o parágrafo órfão**: o trecho que você escreveu sem conseguir apontar de onde veio é exatamente o trecho inventado.

### Passo 3 — Cortar o genérico

Primeira geração de IA vem cheia de frase que sobreviveria em qualquer documento de qualquer empresa. Elas são detectáveis por um teste: **se a frase continua verdadeira depois de trocar o nome da feature, ela não diz nada.**

- "A solução deve ser escalável e de fácil manutenção" → corte.
- "É importante garantir a segurança dos dados" → corte, ou substitua pelo mecanismo concreto.
- "O sistema deve ter boa performance" → substitua pelo número que a fonte deu.

Toda afirmação de qualidade vira número ou vira nada.

### Passo 4 — Verificar a fronteira

Releia procurando conteúdo que pertence a outro documento do pacote. Aplique a tabela de resolução de `altitude-docs`. Duplicação não se resolve cortando: se resolve movendo para a altitude certa e deixando um link.

## Erros recorrentes de IA nestes documentos

Reconheça e corrija na revisão — todos aparecem na primeira geração:

**Simetria falsa.** A IA equilibra seções: se a feature tem três riscos reais e cinco requisitos, ela inventa o quarto risco para fechar em cinco. Seção enxuta e honesta vale mais que seção simétrica e inflada.

**Requisito ressuscitado.** Ideia descartada na reunião reaparece como requisito, porque isoladamente ela faz sentido. Confronte com a lista `FE-NN` antes de fechar.

**Precisão fabricada.** Número específico onde a fonte não deu número. "Retry a cada 30 segundos" quando ninguém falou em 30 segundos soa mais técnico e é pior — é o tipo de erro que chega na implementação.

**Presente do futuro.** Descrever arquivo, tabela ou endpoint que ainda não existe usando o presente, misturado com descrição do sistema atual. Quem for implementar não vai distinguir o que precisa criar do que já está lá. Marque o que é novo.

**Vazamento de altitude para baixo.** É sempre para baixo: o RFC desce para detalhe de implementação, o PRD desce para decisão técnica. Ninguém sobe acidentalmente. Na revisão, procure especificamente o parágrafo técnico demais para o documento em que está.

## Antes de fechar qualquer um dos três

- [ ] Toda afirmação rastreável a um ID da base de fatos
- [ ] Nenhum item da lista de fora de escopo aparecendo como requisito
- [ ] Todo número conferido contra a fonte
- [ ] Todo caminho de arquivo verificado; arquivos novos marcados como novos
- [ ] Nenhuma frase que sobreviveria à troca do nome da feature
- [ ] Nenhuma seção que pertence a outro documento do pacote
- [ ] O documento tem pelo menos uma seção que só ele tem
