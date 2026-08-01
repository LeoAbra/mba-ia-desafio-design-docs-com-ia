# Playbook — FDD

**Pergunta:** como construir, em detalhe.
**Altitude:** implementação.
**Teste de aceitação do documento:** um desenvolvedor que não estava na reunião consegue começar a codar amanhã de manhã sem perguntar nada a ninguém. Toda seção que não contribui para isso está no documento errado.

O FDD é o único dos três que pode e deve ser longo. Profundidade aqui é qualidade; nos outros é vazamento de altitude.

## Estrutura

1. Contexto e motivação técnica
2. Objetivos técnicos
3. Escopo e exclusões
4. Modelo de dados
5. Fluxos detalhados
6. Contratos públicos
7. Matriz de erros
8. Estratégias de resiliência
9. Observabilidade
10. **Integração com o sistema existente**
11. Dependências e compatibilidade
12. Critérios de aceitação técnicos
13. Riscos e mitigação

## Seção a seção

**Contexto e motivação técnica.** Curto. O FDD assume que o leitor já leu o RFC. Recapitule em um parágrafo e linke; não renarre o problema.

**Escopo e exclusões.** Exclusão aqui é técnica, não de produto: o que o módulo deliberadamente **não** faz, para que ninguém implemente por conta própria. "O worker não faz rate limiting de saída"; "o módulo não arquiva linhas processadas".

**Modelo de dados.** Cada tabela nova com colunas, tipos, índices e justificativa de cada índice. Índice sem justificativa é cargo cult. Siga as convenções do schema existente — tipo de chave primária, convenção de nomes, estratégia de timestamps — e diga explicitamente que está seguindo.

**Fluxos detalhados.** Passo a passo numerado, do gatilho ao efeito final, incluindo os caminhos de exceção. Cubra cada fluxo do ciclo de vida, não só o caminho feliz. Para cada fluxo transacional, deixe claro **o que está dentro e o que está fora da transação** — é a informação que mais impacta correção e a que mais some na redação.

**Contratos públicos.** Para cada endpoint: método, caminho, autenticação/autorização exigida, schema de request, payload de exemplo de request **e** de response, e todos os códigos de status com sua semântica. Exemplos com dados realistas — UUID com cara de UUID, valores coerentes entre si.

Para webhooks e integrações de saída, o contrato inclui o que o **consumidor** recebe: headers com nome exato e finalidade, corpo, e o que se espera de volta (qual faixa de status é sucesso, qual é retry, qual é falha definitiva).

**Matriz de erros.** Tabela com código, status HTTP, quando ocorre, e mensagem. Códigos seguem a convenção do projeto — se o projeto usa `SCREAMING_SNAKE_CASE` com prefixo de módulo, siga sem inventar variação. Todo erro que os fluxos e contratos mencionam precisa estar na matriz; toda linha da matriz precisa ser alcançável por algum fluxo. Divergência nos dois sentidos é bug de documento.

**Estratégias de resiliência.** Timeouts com valor, política de retry com a progressão exata, o que conta como falha retentável versus permanente, comportamento na falha definitiva, e o que acontece se o próprio processador cair no meio. Esta última é a que some: crash entre "enviei" e "marquei como enviado" é o caso que justifica a garantia de entrega escolhida.

**Observabilidade.** As três coisas, nomeadas concretamente:
- **Métricas** — nome, tipo (contador/histograma/gauge), labels, e a pergunta operacional que cada uma responde. Métrica que ninguém vai olhar não entra.
- **Logs** — eventos estruturados com os campos de cada um, nível, e o que **não** pode ser logado (segredo, assinatura, dado sensível). Respeite a política de mascaramento já existente no projeto.
- **Tracing** — onde o span começa e termina, o que propaga correlação entre a requisição original e o efeito assíncrono. Em fluxo assíncrono é o único jeito de ligar causa e consequência.

**Integração com o sistema existente.** Seção de maior valor do documento e a que separa FDD real de template preenchido.

Nomeie **caminhos de arquivo reais**, verificados, e para cada um descreva concretamente o que muda ou o que é reutilizado:
- o método que ganha uma chamada nova, com a assinatura atual e onde exatamente a chamada entra na sequência;
- as classes e utilitários que são reutilizados em vez de reescritos, pelo nome real;
- os middlewares que já tratam o caso e por isso não precisam mudar — dizer que algo **não** muda é informação de alto valor;
- o entry point existente que serve de molde para um novo;
- as convenções do schema que a modelagem nova segue.

Distinga sem ambiguidade **arquivo existente** de **arquivo a criar**. Uma tabela com as duas categorias explicitamente marcadas resolve.

Se você não consegue citar arquivo real, você não leu o código — e o FDD não é acionável.

**Dependências e compatibilidade.** Bibliotecas necessárias, e quais já estão no projeto. Descobrir que a necessidade é atendida por algo já instalado ou pela biblioteca padrão é resultado de valor: dependência nova é custo. Compatibilidade retroativa: o que quebra para quem já consome a API, e por que nada quebra se for o caso.

**Critérios de aceitação técnicos.** Verificáveis por teste ou inspeção. "Evento inserido na mesma transação: rollback da mudança de status não deixa evento órfão — verificável por teste de integração forçando falha após a inserção."

## Armadilhas específicas do FDD

- **Virar ADR.** O FDD assume a decisão como premissa e linka. Se um parágrafo argumenta *por que* a decisão foi tomada, ele é do ADR.
- **Contrato incompleto.** Endpoint sem exemplo de response, ou sem os códigos de erro, não é contrato — é ementa.
- **Arquivo inventado.** Caminho que não existe e não está marcado como novo. Verifique todos, um a um.
- **Observabilidade decorativa.** "Adicionar métricas e logs adequados" não é seção de observabilidade. Nomeie cada métrica e cada evento de log.
- **Fluxo só feliz.** Se não há caminho de exceção nos fluxos, eles não foram pensados.
