# Playbook — PRD

**Pergunta:** por quê e o quê.
**Altitude:** produto / negócio.
**Teste de sobrevivência:** o PRD continua válido se a engenharia trocar completamente a arquitetura. Se um parágrafo morre quando a solução técnica muda, ele não é PRD.

## Estrutura

1. Resumo e contexto da feature
2. Problema e motivação
3. Público-alvo e cenários de uso
4. Objetivos e métricas de sucesso
5. Escopo (incluído e fora de escopo)
6. Requisitos funcionais
7. Requisitos não funcionais
8. Decisões e trade-offs principais
9. Dependências
10. Riscos e mitigação
11. Critérios de aceitação
12. Estratégia de testes e validação

## Seção a seção

**Problema e motivação.** Descreva a dor do cliente e o custo de não fazer, com os fatos `NEG-NN`. Nome de cliente real, comportamento atual de integração, risco comercial declarado. O que não entra: solução, tecnologia, arquitetura.

**Público-alvo e cenários.** Quem consome a feature e em que situação. Distinga o cliente externo do operador interno — costumam ter necessidades opostas e o mesmo endpoint.

**Objetivos e métricas.** Cada objetivo carrega **métrica e meta quantitativa**. Objetivo sem número é intenção.

> Ruim: "Reduzir a latência de notificação."
> Bom: "Entregar 100% dos eventos em menos de 10 segundos da mudança de status, medido do commit da transação ao recebimento do 2xx pelo cliente."

Puxe os números da fonte. Se a fonte não deu número para um objetivo, ou você deriva de um número que ela deu — declarando a derivação — ou o objetivo não entra.

**Requisitos funcionais.** Capacidade verificável, uma por linha, com ID estável. Redija como comportamento observável: "O sistema deve permitir que…", "O sistema deve enviar…". Evite requisito que descreve mecanismo interno — mecanismo é FDD.

**Requisitos não funcionais.** Números: latência, limites, tamanhos, algoritmos, janelas de tempo, obrigatoriedades de protocolo. É aqui que moram os parâmetros que a reunião decidiu mas que não mereceram ADR próprio.

**Decisões e trade-offs principais.** Uma linha por decisão, em linguagem de produto, com link para o ADR. O PRD registra que a decisão existe e o que ela custa para o cliente — não por que ela foi tomada.

> "A entrega é *at-least-once*: o cliente pode receber o mesmo evento duas vezes e precisa deduplicar pelo identificador do evento. Ver ADR-004."

**Fora de escopo.** A seção mais importante contra alucinação, e a que a IA mais deixa vazia. Liste explicitamente o que foi **descartado ou adiado**, com quem levantou e por que não entrou. Cada item vem de um `FE-NN`.

Distinga três categorias — elas têm futuros diferentes:
- adiado para fase futura (volta);
- fora de escopo permanente / outro time (não volta para este time);
- descartado por decisão técnica (só volta se a premissa mudar).

**Riscos.** Cada risco com **probabilidade, impacto e mitigação**. Risco sem mitigação é lamento. Risco genérico ("o projeto pode atrasar") não é risco: é a condição humana. Ancore nos fatos — cliente que já vazou segredo em log, indisponibilidade de duas horas em manutenção planejada, prazo comercial com ameaça de churn.

**Critérios de aceitação.** Verificáveis, no nível de produto: o que precisa ser demonstrável para a feature ser considerada entregue. Não confunda com critérios técnicos — esses são do FDD.

## Armadilhas específicas do PRD

- **Virar RFC.** Se aparecer nome de tabela, protocolo ou biblioteca, você desceu de altitude.
- **Fora de escopo vazio ou genérico.** Se a seção diz "outras funcionalidades não mencionadas", a IA não fez a filtragem — ela não sabe o que foi descartado, porque nunca procurou.
- **Métrica sem instrumentação.** Meta que ninguém sabe medir não é meta. Diga como se mede.
- **Requisito inflado.** Reunião com oito requisitos não vira PRD com vinte. Os doze extras foram inventados.
