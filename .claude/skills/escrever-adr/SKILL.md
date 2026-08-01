---
name: escrever-adr
description: "Escreve Architecture Decision Records no formato MADR a partir de uma base de fatos rastreável, uma decisão por arquivo, com alternativas reais e consequências negativas explícitas. Use ao registrar decisões arquiteturais já tomadas, ao converter decisões de uma reunião em ADRs, ou ao revisar ADRs genéricos. Frases-gatilho: 'escrever ADR', 'registrar essa decisão', 'ADRs da reunião', 'documentar decisões arquiteturais', 'MADR'."
---

# Escrever ADR

Um ADR ruim descreve o que foi escolhido. Um ADR bom explica por que as outras opções foram descartadas — porque é isso que o leitor futuro precisa saber quando quiser reabrir a decisão.

Público-alvo: a pessoa que vai herdar o sistema em dois anos e perguntar "por que raios está assim?". Escreva para ela.

## Pré-requisito

Uma base de fatos verificada (veja `fatos-rastreaveis`). Cada ADR precisa apontar para itens `DEC-NN` e `ALT-NN`. ADR escrito direto da transcrição inventa alternativa plausível em vez de registrar a alternativa real.

## O que ganha um ADR

Ganha ADR a decisão que satisfaz as três condições:

1. **Foi contestada ou tinha alternativa real.** Se ninguém cogitou outra coisa, não é decisão — é consequência.
2. **É cara de reverter.** Muda schema, contrato público, topologia de processo, ou modelo de segurança.
3. **Alguém vai questionar depois.** Se o "por quê" é auto-evidente pelo resultado, não precisa de ADR.

O que **não** ganha ADR próprio: parâmetros de configuração (timeout, tamanho de payload, nome de header) e validações triviais. Isso mora no FDD. Uma reunião fecha dezenas de microdecisões; virar ADR de cada uma dilui o conjunto e esconde as que importam.

Sinal de alerta: se a própria reunião disse *"não vejo como decisão arquitetural separada, é só requisito não funcional"*, respeite. O registro da conversa já fez a triagem.

## Estrutura MADR

```markdown
# ADR-NNN — <título afirmativo, o que foi decidido>

- **Status:** Aceito | Proposto | Substituído por ADR-XXX | Descontinuado
- **Data:** <data da decisão>
- **Decisores:** <quem estava na sala>
- **Contexto técnico:** <sistema/módulo afetado>

## Contexto

<A força que exigiu decisão. Restrições reais: time, infra, prazo, sistema
existente. Termina no dilema, não na resposta.>

## Decisão

<Afirmativa, presente do indicativo: "Usamos X". Não "vamos usar" nem
"decidiu-se por". Inclua os parâmetros concretos que fazem parte da decisão.>

## Alternativas consideradas

### <Alternativa A> — descartada
<O que era, quem levantou, e o trade-off exato que a derrubou.>

### <Alternativa B> — descartada
<Idem.>

## Consequências

**Positivas**
- <ganho concreto, não adjetivo>

**Negativas**
- <o preço que estamos pagando conscientemente>

**Neutras / limitações conhecidas**
- <o que fica aceito como limitação, com a condição que a reabriria>

## Rastreabilidade
- Transcrição: `[hh:mm] Nome:` — <o que foi dito>
- Código: `caminho/real.ts` — <o que existe lá>
- Relacionado: ADR-XXX
```

## Regras de escrita

**Nomenclatura.** `ADR-NNN-titulo-em-kebab-case.md`, numeração sequencial a partir de 001, sem reuso de número. O título afirma a decisão (`outbox-no-mysql`), não o tema (`sobre-a-outbox`).

**Uma decisão por arquivo.** Se o título precisa de "e", provavelmente são dois ADRs. Exceção legítima: decisões que só fazem sentido juntas (política de retry *e* seu destino de falha permanente são um par indivisível — retry sem DLQ é retry infinito).

**Alternativa precisa ser real.** A que foi discutida e descartada, com o argumento de quem a descartou. "Poderíamos ter usado Kafka" sem ninguém ter cogitado Kafka é ruído. Se a base de fatos não tem `ALT-NN` para aquela decisão, é sinal de que a decisão foi consenso imediato — e aí diga isso no ADR, é informação honesta.

**Consequência negativa é obrigatória.** ADR sem custo é propaganda. Toda decisão arquitetural paga alguma coisa: latência, acoplamento, responsabilidade jogada para o cliente, limitação de escala. Nomeie o preço. Se você não consegue enxergar o preço, você não entendeu a decisão.

**Limitação conhecida vem com gatilho.** Não basta dizer "não garantimos ordering global". Diga o que a reabriria: "enquanto for worker único; escalar para múltiplos workers exige particionar por chave ou lock pessimista".

**Presente do indicativo.** "Usamos outbox no MySQL." O ADR descreve um estado do mundo, não um plano.

## Ancoragem no código

Pelo menos um ADR do conjunto deve referenciar arquivos, módulos ou classes reais do repositório — tipicamente o ADR de reuso de padrões existentes ou o de integração com o sistema atual. Verifique cada caminho antes de citá-lo: caminho inexistente em ADR contamina a credibilidade do pacote inteiro.

## Antes de fechar

- [ ] Título afirma a decisão, não o tema
- [ ] Contexto termina em dilema, não em resposta
- [ ] Decisão inclui parâmetros concretos
- [ ] Pelo menos uma alternativa real, com o trade-off que a derrubou
- [ ] Pelo menos uma consequência negativa nomeada
- [ ] Limitações conhecidas vêm com o gatilho de reabertura
- [ ] Rastreabilidade com timestamp ou caminho de arquivo verificado
- [ ] Nada aqui é detalhe de implementação que pertence ao FDD
