---
name: auditar-rastreabilidade
description: "Audita um pacote de design docs contra a fonte primária e o código, caçando alucinação: afirmações sem origem, caminhos de arquivo inexistentes, timestamps inválidos, itens descartados vendidos como requisito e contradições entre documentos. Também monta a tabela de rastreabilidade (tracker) que liga cada item à sua origem. Use depois de gerar documentos com IA e antes de entregar. Frases-gatilho: 'auditar os documentos', 'a IA inventou alguma coisa?', 'montar o tracker', 'rastreabilidade', 'revisar antes de entregar'."
---

# Auditar rastreabilidade

IA gerando design doc alucina de um jeito específico e previsível: não inventa fatos absurdos, inventa fatos **plausíveis**. Um timeout de 30 segundos onde a reunião disse 10. Um endpoint que ninguém pediu mas que "faria sentido ter". Um arquivo `src/modules/webhooks/webhook.service.ts` citado como se já existisse.

Nenhum desses erros é detectável lendo o documento. Todos são detectáveis confrontando o documento com a fonte. É isso que esta skill faz.

## Duas entregas

1. **O tracker** — tabela de referência cruzada, ligando cada item à origem. Entregável.
2. **O relatório de auditoria** — o que não fechou. Artefato de trabalho; vira lista de correções.

O tracker não é só documentação: é o **instrumento** da auditoria. Se você não consegue preencher a coluna `Localização` de uma linha, aquela linha não tem origem identificável — foi inventada. Corrija ou remova. A dificuldade de preencher é o sinal.

## Formato do tracker

```
| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
```

- **ID** — identificador estável e legível: `PRD-RF-01`, `RFC-ALT-02`, `FDD-CONTRATO-03`, `ADR-002`. O prefixo diz o documento; o meio diz a família.
- **Documento** — caminho do arquivo onde o item aparece.
- **Tipo** — Requisito Funcional, Requisito Não Funcional, Decisão, Restrição, Trade-off, Fora de Escopo, Questão em Aberto, Contrato, Alternativa.
- **Conteúdo** — uma linha. Se não cabe em uma linha, o item está composto demais; quebre.
- **Fonte** — `TRANSCRICAO` ou `CODIGO`. Sem terceira opção. "Inferência" não é fonte.
- **Localização** — `[hh:mm] Nome:` para transcrição; caminho de arquivo real para código.

## As seis checagens

Rode cada uma como passe independente. Passes misturados encontram menos.

### 1. Caminhos de arquivo

Extraia todo caminho citado em todos os documentos e verifique existência no repositório. Distinga duas categorias, porque a segunda é legítima:

- **Arquivo existente** citado como referência → deve existir. Se não existe, é erro grave.
- **Arquivo a criar** pela feature → deve estar marcado inequivocamente como novo ("a criar", "novo"). Arquivo futuro descrito no presente é armadilha para quem for implementar.

### 2. Timestamps

Todo `[hh:mm] Nome:` citado precisa existir na transcrição, com aquele falante, e o conteúdo daquela fala precisa sustentar a afirmação. Três erros distintos:

- timestamp inexistente;
- timestamp existente com falante errado;
- timestamp e falante corretos, mas a fala não diz aquilo.

O terceiro é o mais comum e o menos visível.

### 3. Números e literais

Varra todo número e todo identificador literal dos documentos — timeouts, tamanhos, quantidades, intervalos, nomes de header, nomes de tabela, nomes de evento, códigos de erro — e confronte um a um com a fonte. Esta é a checagem que pega a alucinação plausível.

### 4. Contaminação de escopo

Pegue a lista de itens descartados e adiados da base de fatos. Para cada um, busque nos documentos: ele aparece em algum lugar como requisito, decisão ou contrato? Se aparece fora de uma seção de "fora de escopo" ou "questões em aberto", é contaminação.

O inverso também: item que a reunião **fechou** aparecendo como "em aberto" subestima o que já está decidido.

### 5. Contradição interna

Confronte os documentos entre si e cada um com o código. Um número que diverge entre RFC e FDD; um ADR que diz uma coisa e o FDD que assume outra; uma descrição do comportamento atual do sistema que não bate com o que o código faz.

Ler o código importa aqui: documento que descreve errado o sistema existente induz erro na implementação.

### 6. Altitude

Cada documento está na sua altura? Conteúdo duplicado entre dois documentos é sintoma. Veja `altitude-docs` para o critério e a resolução.

## Métricas de cobertura

Meça e reporte:

- **Cobertura**: itens identificáveis nos documentos que têm linha no tracker. Alvo ≥ 80%.
- **Proporção de fonte**: linhas com `Fonte = TRANSCRICAO` e timestamp válido. Documentação de reunião com maioria vinda do código sinaliza que os documentos estão descrevendo o sistema atual em vez da feature.
- **Ancoragem em código**: linhas com `Fonte = CODIGO` e caminho real. Zero significa que os documentos flutuam sobre o repositório sem tocá-lo.

## Severidade e ação

| Severidade | Achado | Ação |
|---|---|---|
| **Bloqueante** | Fato sem origem; caminho inexistente; número divergente da fonte | Corrigir ou remover antes de entregar |
| **Alta** | Item descartado apresentado como requisito; contradição entre documentos | Corrigir |
| **Média** | Conteúdo na altitude errada; item sem linha no tracker | Realocar ou acrescentar |
| **Baixa** | Redação vaga, adjetivo sem número | Refinar se houver tempo |

## Verificação adversarial dos achados

Auditor também erra — e erra para o lado de reportar falso positivo, porque encontrar problema é o incentivo da tarefa. Antes de agir sobre a lista, submeta cada achado bloqueante a um verificador independente cuja instrução é **refutar** o achado: abrir a fonte e demonstrar que o documento está certo. Achado que sobrevive à tentativa de refutação vira correção. Achado que não sobrevive vira ruído descartado.

Sem esse passo, a auditoria gera retrabalho em cima de documento que já estava correto.
