# TRACKER — Rastreabilidade do Pacote de Design Docs

Este tracker é a referência cruzada do pacote (PRD, RFC, FDD e ADR-001 a ADR-008). Cada linha liga um item registrado em um dos documentos à sua origem verificável: uma fala da reunião em `TRANSCRICAO.md`, identificada pelo par exato `[hh:mm] Nome:`, ou um arquivo real deste repositório. Ele existe como instrumento contra alucinação: **linha sem localização preenchível não entra no tracker, e item sem origem identificável não entra no documento**. Quem revisa um documento pode partir de qualquer afirmação, achar a linha aqui e abrir a fonte em um passo.

A coluna **Localização** é âncora de origem, não atribuição de autoria. Quando o conteúdo é conclusão explícita da reunião, o timestamp é o ponto em que ela foi dita. Quando o conteúdo é extensão analítica do documento — algo que a reunião não afirmou —, a própria célula de conteúdo diz isso ("proposta do FDD", "analise do FDD", "extensao analitica"), e o timestamp indica apenas o assunto ao qual a análise se prende. **Nenhum achado de análise é atribuído a um participante da reunião.**

---

## Métricas de Cobertura

| Métrica | Valor |
| --- | --- |
| Total de linhas no tracker | **630** |
| docs/PRD.md | 86 |
| docs/RFC.md | 95 |
| docs/FDD.md | 259 |
| docs/adrs/ (ADR-001 a ADR-008) | 190 |
| Linhas com Fonte = TRANSCRICAO | 530 (**84,1%**) |
| Linhas com Fonte = CODIGO | 100 (**15,9%**) |
| Timestamps inválidos após validação | 0 |
| Caminhos de arquivo inexistentes e não marcados "(a criar)" | 0 |

**Alvos do pacote:** ≥80% de cobertura dos itens identificáveis, ≥70% de linhas com fonte TRANSCRICAO e ≥5 linhas com fonte CODIGO. Todos atendidos — 84,1% das linhas têm fonte TRANSCRICAO (alvo 70%) e 100 linhas têm fonte CODIGO (alvo 5).

**Validação executada:** os pares `[hh:mm] Nome:` distintos usados como localização foram conferidos um a um contra `TRANSCRICAO.md`; todos existem com aquele falante naquele minuto. Os **30 caminhos distintos** usados como Localização das 100 linhas com Fonte = CODIGO foram abertos: **os 30 existem no repositório**. Os 2 caminhos ainda inexistentes (`src/worker.ts` e `src/modules/webhooks/`) aparecem apenas em células de conteúdo, sempre marcados "(a criar)", nunca na coluna Localização.

---

## docs/PRD.md

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-RF-01 | docs/PRD.md | Requisito Funcional | Cadastrar configuracao de webhook com endereco de recebimento e status de interesse; segredo e gerado pela plataforma e devolvido na criacao; o cliente dono e informado explicitamente na requisicao, nao deduzido do login | TRANSCRICAO | [09:31] Marcos: |
| PRD-RF-02 | docs/PRD.md | Requisito Funcional | Listar as configuracoes de webhook de um cliente | TRANSCRICAO | [09:33] Bruno: |
| PRD-RF-03 | docs/PRD.md | Requisito Funcional | Editar uma configuracao de webhook ja cadastrada | TRANSCRICAO | [09:33] Bruno: |
| PRD-RF-04 | docs/PRD.md | Requisito Funcional | Remover uma configuracao de webhook | TRANSCRICAO | [09:33] Bruno: |
| PRD-RF-05 | docs/PRD.md | Requisito Funcional | Cada configuracao escolhe quais mudancas de status quer receber e nao recebe as demais; se nenhuma configuracao assina aquele status o evento nao e sequer produzido | TRANSCRICAO | [09:33] Marcos: |
| PRD-RF-06 | docs/PRD.md | Requisito Funcional | Cliente consulta o historico de entregas de uma configuracao com resultado, conteudo enviado, resposta recebida e tempo de resposta por tentativa, na ordem de grandeza dos ultimos 100 envios | TRANSCRICAO | [09:34] Marcos: |
| PRD-RF-07 | docs/PRD.md | Requisito Funcional | Cliente solicita segredo novo para uma configuracao; o segredo anterior permanece valido em paralelo por 24 horas e depois deixa de valer | TRANSCRICAO | [09:21] Sofia: |
| PRD-RF-08 | docs/PRD.md | Requisito Funcional | O evento e produzido junto com a propria mudanca de status; se o registro do evento falha a mudanca de status nao acontece, nao existindo status alterado sem evento entre os status assinados | TRANSCRICAO | [09:40] Bruno: |
| PRD-RF-09 | docs/PRD.md | Requisito Funcional | Entregar cada evento assinado para o cliente provar autoria e integridade, e identificar unicamente cada evento para o cliente reconhecer reenvios | TRANSCRICAO | [09:19] Sofia: |
| PRD-RF-10 | docs/PRD.md | Requisito Funcional | Retentar automaticamente a entrega quando o destino nao responde ou responde com erro, com intervalos crescentes ate o teto de tentativas | TRANSCRICAO | [09:15] Diego: |
| PRD-RF-11 | docs/PRD.md | Requisito Funcional | Preservar de forma consultavel o evento que esgotou as tentativas, com conteudo original, motivo da falha e momento da desistencia | TRANSCRICAO | [09:18] Diego: |
| PRD-RF-12 | docs/PRD.md | Requisito Funcional | Reprocessamento manual de evento que esgotou as tentativas, restrito ao papel ADMIN e registrando quem executou a acao para auditoria | TRANSCRICAO | [09:36] Sofia: |
| PRD-RNF-01 | docs/PRD.md | Requisito Nao Funcional | Latencia alvo de notificacao abaixo de 10 segundos entre a mudanca de status e o recebimento pelo cliente | TRANSCRICAO | [09:02] Marcos: |
| PRD-RNF-02 | docs/PRD.md | Requisito Nao Funcional | Atraso maximo de 2 segundos entre a mudanca de status e o inicio da primeira tentativa de entrega | TRANSCRICAO | [09:09] Diego: |
| PRD-RNF-03 | docs/PRD.md | Requisito Nao Funcional | Tempo maximo de espera pela resposta do cliente de 10 segundos; sem resposta a tentativa e falha e entra em retentativa | TRANSCRICAO | [09:42] Diego: |
| PRD-RNF-04 | docs/PRD.md | Requisito Nao Funcional | Numero maximo de 5 tentativas de entrega por evento | TRANSCRICAO | [09:15] Diego: |
| PRD-RNF-05 | docs/PRD.md | Requisito Nao Funcional | Espacamento entre tentativas de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas, somando na reuniao quase 15 horas | TRANSCRICAO | [09:17] Diego: |
| PRD-RNF-06 | docs/PRD.md | Requisito Nao Funcional | Janela de convivencia de 24 horas do segredo anterior apos a rotacao | TRANSCRICAO | [09:21] Sofia: |
| PRD-RNF-07 | docs/PRD.md | Requisito Nao Funcional | Tamanho maximo do conteudo do evento de 64 KB, acima disso e erro e nao truncamento | TRANSCRICAO | [09:24] Diego: |
| PRD-RNF-08 | docs/PRD.md | Requisito Nao Funcional | Endereco de recebimento precisa usar transporte cifrado; endereco em texto claro e recusado no cadastro com erro de validacao | TRANSCRICAO | [09:23] Sofia: |
| PRD-RNF-09 | docs/PRD.md | Requisito Nao Funcional | Prova de autoria e integridade por assinatura do corpo com mecanismo padrao de mercado para o qual o cliente ja tenha biblioteca pronta | TRANSCRICAO | [09:20] Sofia: |
| PRD-RNF-10 | docs/PRD.md | Requisito Nao Funcional | Um segredo por configuracao de webhook, nunca um segredo global da plataforma | TRANSCRICAO | [09:21] Sofia: |
| PRD-RNF-11 | docs/PRD.md | Requisito Nao Funcional | A entrega ao cliente nao pode bloquear a mudanca de status de outros pedidos; cliente lento ou fora do ar nao trava a operacao | TRANSCRICAO | [09:04] Bruno: |
| PRD-RNF-12 | docs/PRD.md | Requisito Nao Funcional | Conteudo do evento com dados essenciais do pedido e sem a lista de itens; quem precisar de detalhe consulta o pedido | TRANSCRICAO | [09:43] Diego: |
| PRD-DEC-01 | docs/PRD.md | Decisao | O evento e registrado junto com a mudanca de status em tudo ou nada; se o registro do evento falha, a mudanca de status falha junto e o operador ve a operacao recusada | TRANSCRICAO | [09:40] Bruno: |
| PRD-DEC-02 | docs/PRD.md | Decisao | Entrega assincrona despachada fora da transacao de mudanca de status, ao custo de ate 2 segundos de atraso fixo e nenhuma garantia de ordem | TRANSCRICAO | [09:06] Diego: |
| PRD-DEC-03 | docs/PRD.md | Decisao | Cinco tentativas espacadas e, esgotadas, o evento fica preservado para reprocessamento manual, que depende de alguem olhar porque nao ha aviso automatico | TRANSCRICAO | [09:17] Larissa: |
| PRD-DEC-04 | docs/PRD.md | Decisao | Cada configuracao tem segredo proprio rotacionavel com 24h de convivencia, ao custo de o cliente implementar a verificacao e conviver com dois segredos validos | TRANSCRICAO | [09:22] Sofia: |
| PRD-DEC-05 | docs/PRD.md | Decisao | A entrega e at-least-once, com a deduplicacao pelo identificador do evento sendo responsabilidade do cliente | TRANSCRICAO | [09:26] Larissa: |
| PRD-DEC-06 | docs/PRD.md | Decisao | A feature reusa os padroes ja vigentes na API, de modo que formato de erro, autenticacao e paginacao sao os mesmos que o cliente ja consome | TRANSCRICAO | [09:30] Larissa: |
| PRD-DEC-07 | docs/PRD.md | Decisao | O evento carrega o estado do pedido no instante da mudanca, entregando um retrato do momento e nao o estado atual | TRANSCRICAO | [09:52] Larissa: |
| PRD-DEC-08 | docs/PRD.md | Decisao | Reprocessamento exige papel ADMIN e o cadastro exige apenas autenticacao, de modo que qualquer usuario autenticado pode alterar a configuracao de qualquer cliente | TRANSCRICAO | [09:36] Larissa: |
| PRD-DEC-09 | docs/PRD.md | Decisao | Filtro de status aplicado no momento da producao do evento e nao no envio, de modo que status nao assinado por nenhuma configuracao nao gera registro | TRANSCRICAO | [09:34] Bruno: |
| PRD-DEC-10 | docs/PRD.md | Decisao | O cliente dono da configuracao e passado no corpo ou no caminho da requisicao e nao vem do token de autenticacao | TRANSCRICAO | [09:32] Larissa: |
| PRD-ALT-01 | docs/PRD.md | Alternativa | Tres tentativas de entrega em vez de cinco, descartada porque mataria o evento em 30 minutos e ja houve cliente com 2 horas de indisponibilidade planejada | TRANSCRICAO | [09:16] Diego: |
| PRD-ALT-02 | docs/PRD.md | Alternativa | Segredo unico global da plataforma, descartado porque o vazamento de um comprometeria todos os clientes | TRANSCRICAO | [09:21] Sofia: |
| PRD-ALT-03 | docs/PRD.md | Alternativa | Truncar o conteudo do evento acima do teto de tamanho, descartada em favor de retornar erro | TRANSCRICAO | [09:23] Sofia: |
| PRD-ALT-04 | docs/PRD.md | Alternativa | Cliente dono da configuracao deduzido implicitamente do token de autenticacao, descartada porque o token e do usuario operador e nao do cliente | TRANSCRICAO | [09:32] Bruno: |
| PRD-ALT-05 | docs/PRD.md | Alternativa | Cliente escolher o proprio segredo de validacao, descartada porque o segredo e gerado pela plataforma e devolvido na criacao | TRANSCRICAO | [09:31] Marcos: |
| PRD-CONTRATO-01 | docs/PRD.md | Contrato | A consulta repetida de pedidos que a feature substitui usa o endpoint de listagem de pedidos ja existente e autenticado | CODIGO | src/modules/orders/order.routes.ts |
| PRD-CONTRATO-02 | docs/PRD.md | Contrato | A mudanca de status que dispara o evento acontece no servico de pedidos existente, dentro da transacao que atualiza pedido, historico e estoque | CODIGO | src/modules/orders/order.service.ts |
| PRD-RESTR-01 | docs/PRD.md | Restricao | O fluxo e de saida apenas, os eventos saem da plataforma para o cliente e nunca o contrario | TRANSCRICAO | [09:02] Marcos: |
| PRD-RESTR-02 | docs/PRD.md | Restricao | Prazo comercial de fim de novembro colocado pela Atlas | TRANSCRICAO | [09:45] Marcos: |
| PRD-RESTR-03 | docs/PRD.md | Restricao | Capacidade de tres sprints de time, ja com a revisao de seguranca dentro da janela | TRANSCRICAO | [09:46] Larissa: |
| PRD-RESTR-04 | docs/PRD.md | Restricao | Revisao de seguranca com no minimo dois dias uteis reservados antes do deploy, com foco na assinatura e na geracao do segredo | TRANSCRICAO | [09:46] Sofia: |
| PRD-RESTR-05 | docs/PRD.md | Restricao | Documentacao de integracao no portal do desenvolvedor sob responsabilidade de Marcos, bloqueante para o deploy | TRANSCRICAO | [09:26] Marcos: |
| PRD-RESTR-06 | docs/PRD.md | Restricao | Revisao do documento de design com Bruno e Diego antes do inicio da implementacao | TRANSCRICAO | [09:50] Larissa: |
| PRD-RESTR-07 | docs/PRD.md | Restricao | Confirmacao do prazo e atualizacao dos clientes por Marcos, dependencia de comunicacao comercial sem efeito de bloqueio | TRANSCRICAO | [09:47] Marcos: |
| PRD-RESTR-08 | docs/PRD.md | Restricao | Dependencia externa: o cliente precisa manter um endereco de recebimento disponivel e com transporte cifrado | TRANSCRICAO | [09:23] Sofia: |
| PRD-RESTR-09 | docs/PRD.md | Restricao | Dependencia externa: o cliente precisa verificar a assinatura de cada evento recebido do lado dele | TRANSCRICAO | [09:20] Sofia: |
| PRD-RESTR-10 | docs/PRD.md | Restricao | Dependencia externa: o cliente precisa deduplicar eventos repetidos pelo identificador unico do evento | TRANSCRICAO | [09:25] Diego: |
| PRD-RESTR-11 | docs/PRD.md | Restricao | Os papeis de usuario existentes no sistema sao apenas ADMIN e OPERATOR, o que delimita o controle de acesso possivel nesta fase | CODIGO | prisma/schema.prisma |
| PRD-RESTR-12 | docs/PRD.md | Restricao | A verificacao de papel reaproveita o mecanismo de autorizacao ja existente na base de codigo | CODIGO | src/middlewares/auth.middleware.ts |
| PRD-TO-01 | docs/PRD.md | Trade-off | Tensao entre publicos: o cliente quer receber tudo o mais rapido possivel, mas a operacao interna nao pode ter a mudanca de status presa esperando cliente lento | TRANSCRICAO | [09:04] Bruno: |
| PRD-TO-02 | docs/PRD.md | Trade-off | Filtrar o status na producao do evento economiza registro, mas status nao assinado por ninguem nunca gera evento e fica fora da reconciliacao de perda | TRANSCRICAO | [09:34] Bruno: |
| PRD-TO-03 | docs/PRD.md | Trade-off | Escopo deliberadamente cortado e revisao de seguranca contabilizada dentro das tres sprints para caber no prazo comercial | TRANSCRICAO | [09:47] Larissa: |
| PRD-TO-04 | docs/PRD.md | Trade-off | Ausencia de garantia de ordem aceita porque os clientes nunca pediram ordering global, so querem saber se cada pedido deles mudou | TRANSCRICAO | [09:14] Marcos: |
| PRD-TO-05 | docs/PRD.md | Trade-off | At-least-once joga a responsabilidade de deduplicacao para o cliente, custo aceito por ser mais simples que garantir exactly-once | TRANSCRICAO | [09:25] Sofia: |
| PRD-RISCO-01 | docs/PRD.md | Risco | Cliente vaza o segredo de validacao, permitindo forjar notificacoes; ancorado em caso real de cliente que vazou segredo em log de aplicacao | TRANSCRICAO | [09:22] Diego: |
| PRD-RISCO-02 | docs/PRD.md | Risco | Cliente indisponivel por horas e evento perdido, ancorado no caso de indisponibilidade de duas horas em manutencao planejada | TRANSCRICAO | [09:16] Diego: |
| PRD-RISCO-03 | docs/PRD.md | Risco | Perder a Atlas Comercial por atraso, ja que o cliente sugeriu migrar para o concorrente se a entrega nao sair no prazo | TRANSCRICAO | [09:00] Marcos: |
| PRD-RISCO-04 | docs/PRD.md | Risco | Cliente nao deduplica e processa o mesmo evento duas vezes, gerando efeito duplicado no sistema dele | TRANSCRICAO | [09:25] Sofia: |
| PRD-RISCO-05 | docs/PRD.md | Risco | Evento que falhou definitivamente sem ninguem perceber, porque nao ha aviso automatico e o reprocessamento e manual | TRANSCRICAO | [09:37] Larissa: |
| PRD-RISCO-06 | docs/PRD.md | Risco | Rajada de notificacoes sobre um mesmo cliente, ancorada na hipotese de 50 pedidos mudando de status em um minuto | TRANSCRICAO | [09:38] Diego: |
| PRD-RISCO-07 | docs/PRD.md | Risco | Configuracao de um cliente alterada por usuario de outro contexto, ja que o cliente dono vem da requisicao e o endurecimento de papeis foi adiado | TRANSCRICAO | [09:32] Larissa: |
| PRD-RISCO-08 | docs/PRD.md | Risco | Analise do PRD e nao conclusao da reuniao: nao existe vinculo entre usuario e cliente no modelo de dados, entao verificar posse e uma pergunta que o sistema nao tem como responder hoje | CODIGO | prisma/schema.prisma |
| PRD-QA-01 | docs/PRD.md | Questao em Aberto | Divergencia entre o teto de 5 tentativas e a progressao de intervalos: apenas quatro esperas sao consumidas, somando 2h36min, e o intervalo de 12h fica inalcancavel | TRANSCRICAO | [09:17] Diego: |
| PRD-QA-02 | docs/PRD.md | Questao em Aberto | Percentual de cobertura do alvo de 10 segundos: a reuniao fixou o limiar mas nao fixou percentual algum, e os 95 por cento sao proposta do PRD pendente de ratificacao | TRANSCRICAO | [09:02] Marcos: |
| PRD-QA-03 | docs/PRD.md | Questao em Aberto | Janela de adocao dos tres clientes: a reuniao nao fixou prazo e os 60 dias apos a disponibilizacao sao proposta do PRD a ratificar | TRANSCRICAO | [09:47] Marcos: |
| PRD-QA-04 | docs/PRD.md | Questao em Aberto | Limiar de aprovacao de ate 1 por cento de eventos exigindo reprocessamento manual por mes e proposta do PRD, a reuniao nao fixou nenhum | TRANSCRICAO | [09:37] Larissa: |
| PRD-QA-05 | docs/PRD.md | Questao em Aberto | Qual segredo assina durante a janela de convivencia de 24 horas, o novo, o antigo ou ambos, nao foi decidido e precisa sair da revisao de seguranca | TRANSCRICAO | [09:21] Sofia: |
| PRD-QA-06 | docs/PRD.md | Questao em Aberto | Quantificar a economia da consulta repetida exige medir antes do piloto quantas consultas os tres clientes fazem por dia; nenhum dado de volume ou custo foi apresentado | TRANSCRICAO | [09:00] Marcos: |
| PRD-QA-07 | docs/PRD.md | Questao em Aberto | Limitacao da taxa de envio ficou como observar e decidir depois, sem limite nesta fase e sem dado de producao para decidir | TRANSCRICAO | [09:39] Larissa: |
| PRD-FE-01 | docs/PRD.md | Fora de Escopo | Alerta por e-mail ao cliente quando o webhook dele esta falhando, adiado para fase futura depois de medir o impacto | TRANSCRICAO | [09:37] Larissa: |
| PRD-FE-02 | docs/PRD.md | Fora de Escopo | Painel visual para o cliente acompanhar os webhooks dele, recusado por ser projeto separado do time de frontend; a lacuna vira documentacao | TRANSCRICAO | [09:40] Larissa: |
| PRD-FE-03 | docs/PRD.md | Fora de Escopo | Limitacao da taxa de envio para um mesmo cliente, adiada com a orientacao de observar e implementar se virar problema | TRANSCRICAO | [09:39] Diego: |
| PRD-FE-04 | docs/PRD.md | Fora de Escopo | Arquivamento ou expurgo do acervo de eventos ja entregues, classificado pelo proprio proponente como fora do escopo desta feature | TRANSCRICAO | [09:08] Diego: |
| PRD-FE-05 | docs/PRD.md | Fora de Escopo | Recebimento de webhooks enviados pelo cliente para a plataforma; o fluxo e apenas de saida porque eles querem receber e nao mandar | TRANSCRICAO | [09:02] Marcos: |
| PRD-FE-06 | docs/PRD.md | Fora de Escopo | Garantia de ordem global de entrega entre pedidos diferentes, descartada como problema do futuro e registrada como limitacao conhecida | TRANSCRICAO | [09:13] Larissa: |
| PRD-FE-07 | docs/PRD.md | Fora de Escopo | Restricao de papel no cadastro de webhooks, adiada com a folga autorizada para qualquer papel autenticado nesta fase | TRANSCRICAO | [09:37] Sofia: |
| PRD-METRICA-01 | docs/PRD.md | Metrica | OBJ-1 notificar em tempo real: intervalo entre a mudanca de status e a confirmacao de recebimento, com limiar de 10 segundos fixado na reuniao | TRANSCRICAO | [09:02] Marcos: |
| PRD-METRICA-02 | docs/PRD.md | Metrica | OBJ-2 eliminar a consulta repetitiva: 3 de 3 clientes que pediram formalmente a feature com configuracao ativa recebendo eventos | TRANSCRICAO | [09:00] Marcos: |
| PRD-METRICA-03 | docs/PRD.md | Metrica | OBJ-3 nao perder mudanca de status: meta zero mudancas que deveriam gerar evento e nao geraram | TRANSCRICAO | [09:40] Bruno: |
| PRD-METRICA-04 | docs/PRD.md | Metrica | OBJ-4 sobreviver a indisponibilidade do cliente: fracao dos eventos falhados que precisou de reprocessamento manual, com a janela de retentativas em disputa | TRANSCRICAO | [09:17] Diego: |
| PRD-METRICA-05 | docs/PRD.md | Metrica | OBJ-5 entregar dentro do prazo comercial: data de disponibilizacao em producao contra a data de fim de novembro acordada com a Atlas | TRANSCRICAO | [09:45] Marcos: |
| PRD-METRICA-06 | docs/PRD.md | Metrica | Numero absoluto de eventos que esgotaram tentativas por mes e o insumo condicionado para reabrir o alerta automatico depois de medir o impacto | TRANSCRICAO | [09:37] Larissa: |

---

## docs/RFC.md

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| RFC-RF-01 | docs/RFC.md | Requisito Funcional | Produtor de evento e chamado de dentro do tx de changeStatus recebendo o cliente transacional em curso, via funcao publishWebhookEvent(tx, order, fromStatus, toStatus) | TRANSCRICAO | [09:41] Bruno: |
| RFC-RF-02 | docs/RFC.md | Requisito Funcional | Falha ao inserir o evento na outbox derruba a transacao de mudanca de status (rollback), sem caso de status mudar e evento nao sair | TRANSCRICAO | [09:40] Bruno: |
| RFC-RF-03 | docs/RFC.md | Requisito Funcional | Outbox no MySQL existente e alimentada na mesma transacao, guardando o evento ja renderizado no instante da mudanca de status | TRANSCRICAO | [09:52] Larissa: |
| RFC-RF-04 | docs/RFC.md | Requisito Funcional | Cadastro de webhooks armazena url, secret, customer_id e estado ativo por destino | TRANSCRICAO | [09:21] Bruno: |
| RFC-RF-05 | docs/RFC.md | Requisito Funcional | Filtragem por status acontece na insercao: se nenhum webhook do customer quer aquele status, a linha da outbox nem e criada | TRANSCRICAO | [09:34] Bruno: |
| RFC-RF-06 | docs/RFC.md | Requisito Funcional | Worker de entrega e processo Node separado da API com PrismaClient proprio, mesmo banco e mesma DATABASE_URL | TRANSCRICAO | [09:30] Bruno: |
| RFC-RF-07 | docs/RFC.md | Requisito Funcional | Worker faz polling em loop a cada 2 segundos buscando os eventos pendentes mais antigos, processa e marca | TRANSCRICAO | [09:09] Diego: |
| RFC-RF-08 | docs/RFC.md | Requisito Funcional | Chamada HTTP do worker usa timeout de 10 segundos; destino que nao responde no prazo e tratado como falha e marcado para retry | TRANSCRICAO | [09:42] Diego: |
| RFC-RF-09 | docs/RFC.md | Requisito Funcional | Retry com backoff 1m/5m/30m/2h/12h, total de cinco tentativas | TRANSCRICAO | [09:17] Larissa: |
| RFC-RF-10 | docs/RFC.md | Requisito Funcional | Esgotadas as tentativas o evento vai para tabela webhook_dead_letter separada, com payload, motivo da falha e timestamp | TRANSCRICAO | [09:18] Diego: |
| RFC-RF-11 | docs/RFC.md | Requisito Funcional | Toda tentativa de entrega fica registrada com resultado sucesso/falha, payload, response e tempo de resposta | TRANSCRICAO | [09:34] Marcos: |
| RFC-RF-12 | docs/RFC.md | Requisito Funcional | Superficie HTTP e um modulo em src/modules/webhooks/ (a criar) com controller, service, repository, routes e schemas | TRANSCRICAO | [09:27] Bruno: |
| RFC-RF-13 | docs/RFC.md | Requisito Funcional | Secret e rotacionavel por API; ao rotacionar a antiga permanece valida por 24 horas em paralelo e depois morre | TRANSCRICAO | [09:21] Sofia: |
| RFC-RF-14 | docs/RFC.md | Requisito Funcional | Replay da DLQ e manual, exige role ADMIN e registra quem executou para auditoria | TRANSCRICAO | [09:36] Sofia: |
| RFC-RF-15 | docs/RFC.md | Requisito Funcional | Payload e snapshot enxuto do pedido sem items; cliente que quiser detalhe bate no GET /orders/:id depois | TRANSCRICAO | [09:43] Diego: |
| RFC-RF-16 | docs/RFC.md | Requisito Funcional | Cliente deve deduplicar pelo event_id enviado no header X-Event-Id, UUID gerado quando o evento entra na outbox | TRANSCRICAO | [09:25] Diego: |
| RFC-RF-17 | docs/RFC.md | Requisito Funcional | Marcos assume documentar as obrigacoes do cliente no portal do desenvolvedor | TRANSCRICAO | [09:26] Marcos: |
| RFC-RNF-01 | docs/RFC.md | Requisito Nao Funcional | Alvo de latencia frouxo: qualquer coisa abaixo de 10 segundos ja e considerado tempo real pelos clientes | TRANSCRICAO | [09:02] Marcos: |
| RFC-RNF-02 | docs/RFC.md | Requisito Nao Funcional | TLS obrigatorio: URL do webhook tem que ser https e cadastro com http e recusado com erro de validacao no schema Zod | TRANSCRICAO | [09:23] Sofia: |
| RFC-RNF-03 | docs/RFC.md | Requisito Nao Funcional | Teto de 64KB por evento, com erro caso ultrapasse em vez de truncar | TRANSCRICAO | [09:24] Diego: |
| RFC-RNF-04 | docs/RFC.md | Requisito Nao Funcional | TLS obrigatorio e limite de 64KB foram classificados como requisito nao funcional e nao como decisao arquitetural separada | TRANSCRICAO | [09:24] Larissa: |
| RFC-RNF-05 | docs/RFC.md | Requisito Nao Funcional | Autenticidade por HMAC-SHA256 sobre o corpo do request, com secret unica por endpoint | TRANSCRICAO | [09:22] Sofia: |
| RFC-RNF-06 | docs/RFC.md | Requisito Nao Funcional | Garantia de entrega at-least-once: o cliente pode receber o mesmo evento duas vezes e tem que estar preparado | TRANSCRICAO | [09:24] Diego: |
| RFC-RNF-07 | docs/RFC.md | Requisito Nao Funcional | Solucao nao adiciona dependencia nova: usa node:crypto, fetch global do Node 20 (engines node >=20) e o uuid 11.0.3 ja instalado | CODIGO | package.json |
| RFC-RNF-08 | docs/RFC.md | Requisito Nao Funcional | Worker precisa de deploy, supervisao e desligamento gracioso no molde do shutdown por SIGINT/SIGTERM ja existente | CODIGO | src/server.ts |
| RFC-DEC-01 | docs/RFC.md | Decisao | Padrao outbox no MySQL existente, com insercao na mesma transacao SQL que atualiza orders e order_status_history | TRANSCRICAO | [09:08] Larissa: |
| RFC-DEC-02 | docs/RFC.md | Decisao | Worker em polling de 2 segundos; latencia minima de 2 segundos no pior caso e aceita | TRANSCRICAO | [09:10] Larissa: |
| RFC-DEC-03 | docs/RFC.md | Decisao | Cinco tentativas com backoff exponencial 1m/5m/30m/2h/12h antes de considerar falha permanente | TRANSCRICAO | [09:17] Larissa: |
| RFC-DEC-04 | docs/RFC.md | Decisao | DLQ persistida em tabela separada e nao como estado failed na propria outbox | TRANSCRICAO | [09:18] Diego: |
| RFC-DEC-05 | docs/RFC.md | Decisao | HMAC-SHA256 sobre o corpo, secret por endpoint e suporte a rotacao com grace period de 24h | TRANSCRICAO | [09:22] Sofia: |
| RFC-DEC-06 | docs/RFC.md | Decisao | At-least-once com X-Event-Id para dedup do lado do cliente | TRANSCRICAO | [09:26] Larissa: |
| RFC-DEC-07 | docs/RFC.md | Decisao | Reuso maximo do que ja existe: AppError, Pino, error middleware, padrao de modulos, schemas Zod e codigos de erro | TRANSCRICAO | [09:30] Larissa: |
| RFC-DEC-08 | docs/RFC.md | Decisao | Snapshot do payload renderizado na insercao, para o evento refletir o estado de quando o status mudou | TRANSCRICAO | [09:52] Larissa: |
| RFC-DEC-09 | docs/RFC.md | Decisao | Replay de DLQ exige role ADMIN e reaproveita o requireRole ja existente | TRANSCRICAO | [09:36] Larissa: |
| RFC-DEC-10 | docs/RFC.md | Decisao | CRUD de configuracao de webhook aceita qualquer role autenticada por enquanto, com endurecimento adiado | TRANSCRICAO | [09:37] Sofia: |
| RFC-DEC-11 | docs/RFC.md | Decisao | Worker roda como processo separado e nao dentro da instancia da API, para nao morrer junto com o restart da API | TRANSCRICAO | [09:11] Diego: |
| RFC-DEC-12 | docs/RFC.md | Decisao | Funcao pura recebendo o tx da transacao atual em vez de injetar repository inteiro no OrderService | TRANSCRICAO | [09:41] Diego: |
| RFC-DEC-13 | docs/RFC.md | Decisao | Id da outbox e UUID, seguindo o padrao do resto do projeto onde tudo e uuid | TRANSCRICAO | [09:51] Larissa: |
| RFC-DEC-14 | docs/RFC.md | Decisao | Prefixo WEBHOOK_ em todos os codigos de erro do modulo | TRANSCRICAO | [09:29] Larissa: |
| RFC-DEC-15 | docs/RFC.md | Decisao | Revisao de seguranca de HMAC e geracao de secret por Sofia, minimo dois dias uteis, e pre-condicao de deploy | TRANSCRICAO | [09:46] Sofia: |
| RFC-DEC-16 | docs/RFC.md | Decisao | Prazo estimado de tres sprints com a revisao de seguranca incluida no fim | TRANSCRICAO | [09:47] Larissa: |
| RFC-DEC-17 | docs/RFC.md | Decisao | Logica de processamento fica em arquivo dentro do modulo; a grafia webhook.processor.ts (a criar) foi fechada depois pelo FDD | TRANSCRICAO | [09:28] Bruno: |
| RFC-DEC-18 | docs/RFC.md | Decisao | Sessao de revisao tecnica do design doc com Bruno e Diego antes de comecar a implementacao | TRANSCRICAO | [09:50] Larissa: |
| RFC-ALT-01 | docs/RFC.md | Alternativa | Despacho sincrono dentro de changeStatus, descartado por por a disponibilidade do cliente no caminho critico e nao permitir rollback | TRANSCRICAO | [09:04] Bruno: |
| RFC-ALT-02 | docs/RFC.md | Alternativa | Redis Streams ou fila externa como transporte, descartado por exigir subir infra nova num time pequeno (overengineering) | TRANSCRICAO | [09:07] Diego: |
| RFC-ALT-03 | docs/RFC.md | Alternativa | Trigger de banco para acordar o worker, descartada porque a trigger so executa SQL e nao notifica processo externo | TRANSCRICAO | [09:09] Diego: |
| RFC-ALT-04 | docs/RFC.md | Alternativa | Politica de 3 tentativas, descartada por matar o evento em 30 minutos, janela menor que indisponibilidade de duas horas ja observada | TRANSCRICAO | [09:16] Diego: |
| RFC-ALT-05 | docs/RFC.md | Alternativa | Retry indefinido com backoff, descartado por deixar evento pendurado para sempre se o cliente sumiu | TRANSCRICAO | [09:15] Diego: |
| RFC-ALT-06 | docs/RFC.md | Alternativa | DLQ como estado failed na propria outbox, descartada por sujar a leitura da outbox principal | TRANSCRICAO | [09:18] Diego: |
| RFC-ALT-07 | docs/RFC.md | Alternativa | Entrega exactly-once, descartada por exigir coordenacao dos dois lados; at-least-once com event_id e o padrao de mercado | TRANSCRICAO | [09:25] Diego: |
| RFC-ALT-08 | docs/RFC.md | Alternativa | Secret global da plataforma, descartada porque se vaza uma vaza tudo | TRANSCRICAO | [09:21] Sofia: |
| RFC-ALT-09 | docs/RFC.md | Alternativa | Guardar so order_id na outbox e renderizar na hora do envio, descartada porque o evento refletiria o estado do momento do envio | TRANSCRICAO | [09:51] Bruno: |
| RFC-ALT-10 | docs/RFC.md | Alternativa | Id auto incremental para a outbox, descartado em favor de UUID por consistencia com o projeto | TRANSCRICAO | [09:51] Diego: |
| RFC-CONTRATO-01 | docs/RFC.md | Contrato | POST de cadastro de webhook com url, lista de status desejados; secret e gerada pela plataforma e devolvida na criacao | TRANSCRICAO | [09:31] Marcos: |
| RFC-CONTRATO-02 | docs/RFC.md | Contrato | PATCH para editar, DELETE para remover e GET para listar os webhooks de um customer | TRANSCRICAO | [09:33] Bruno: |
| RFC-CONTRATO-03 | docs/RFC.md | Contrato | GET /webhooks/:id/deliveries expoe o historico de entregas ao cliente | TRANSCRICAO | [09:34] Marcos: |
| RFC-CONTRATO-04 | docs/RFC.md | Contrato | POST /admin/webhooks/dead-letter/:id/replay recoloca o evento na outbox como pendente | TRANSCRICAO | [09:35] Diego: |
| RFC-CONTRATO-05 | docs/RFC.md | Contrato | Headers do envio: X-Event-Id com o UUID, X-Signature com o HMAC, X-Timestamp do envio e Content-Type application/json | TRANSCRICAO | [09:44] Diego: |
| RFC-CONTRATO-06 | docs/RFC.md | Contrato | Header adicional X-Webhook-Id com o id do endpoint cadastrado, para cliente com varios destinos identificar o cadastro | TRANSCRICAO | [09:44] Sofia: |
| RFC-CONTRATO-07 | docs/RFC.md | Contrato | Rotas do modulo de webhooks entram sob o prefixo /api/v1 montado pelo buildApiRouter existente | CODIGO | src/routes/index.ts |
| RFC-CONTRATO-08 | docs/RFC.md | Contrato | customer_id e passado no body ou no path do endpoint autenticado, nao vem do JWT | TRANSCRICAO | [09:32] Larissa: |
| RFC-CONTRATO-09 | docs/RFC.md | Contrato | Payload JSON com event_id, event_type order.status_changed, timestamp ISO 8601, order_id, order_number, from_status, to_status, customer_id e total_cents | TRANSCRICAO | [09:43] Diego: |
| RFC-RESTR-01 | docs/RFC.md | Restricao | OrderService.changeStatus abre um prisma.$transaction unico e ja pesado que qualquer emissao de evento atravessa | CODIGO | src/modules/orders/order.service.ts |
| RFC-RESTR-02 | docs/RFC.md | Restricao | Validacao de transicao de status por canTransition vive em arquivo proprio e ja roda dentro da transacao | CODIGO | src/modules/orders/order.status.ts |
| RFC-RESTR-03 | docs/RFC.md | Restricao | Nenhum apetite para operar mais um componente de infraestrutura porque o time e pequeno | TRANSCRICAO | [09:07] Diego: |
| RFC-RESTR-04 | docs/RFC.md | Restricao | O docker-compose do projeto sobe apenas o servico mysql:8.0, sem broker nem cache | CODIGO | docker-compose.yml |
| RFC-RESTR-05 | docs/RFC.md | Restricao | MySQL nao tem listener nativo tipo NOTIFY/LISTEN do Postgres para acordar processo externo | TRANSCRICAO | [09:09] Diego: |
| RFC-RESTR-06 | docs/RFC.md | Restricao | Restricao dupla: o registro do evento precisa ser atomico com a mudanca de status e a entrega precisa ser assincrona | TRANSCRICAO | [09:06] Diego: |
| RFC-RESTR-07 | docs/RFC.md | Restricao | Escopo estritamente outbound: os webhooks so saem da plataforma para os clientes, nao ha recebimento | TRANSCRICAO | [09:02] Marcos: |
| RFC-RESTR-08 | docs/RFC.md | Restricao | Worker entra como entry-point nova no molde de src/server.ts, com script npm run worker (a criar) | TRANSCRICAO | [09:11] Larissa: |
| RFC-RESTR-09 | docs/RFC.md | Restricao | Modulo de webhooks segue o molde existente de controller, service, repository, routes e schemas do modulo de pedidos | CODIGO | src/modules/orders/ |
| RFC-TRADE-01 | docs/RFC.md | Trade-off | Polling cabe com folga no orcamento de latencia, folga aceita explicitamente pelo produto; o valor em segundos e do FDD 8.1 | TRANSCRICAO | [09:10] Marcos: |
| RFC-TRADE-02 | docs/RFC.md | Trade-off | Nao ha garantia de ordenacao global: so por order_id e enquanto for single-worker, documentado como limitacao conhecida | TRANSCRICAO | [09:13] Larissa: |
| RFC-TRADE-03 | docs/RFC.md | Trade-off | At-least-once joga a responsabilidade de deduplicacao para o cliente | TRANSCRICAO | [09:25] Sofia: |
| RFC-TRADE-04 | docs/RFC.md | Trade-off | Sem alerta automatico ao cliente, a DLQ vira trabalho humano: alguem precisa olhar e disparar o replay manualmente | TRANSCRICAO | [09:37] Larissa: |
| RFC-TRADE-05 | docs/RFC.md | Trade-off | Extensao analitica do RFC, nao conclusao da reuniao: o compromisso de latencia so pode ser declarado em percentil sobre a primeira tentativa bem-sucedida, nao em valor absoluto | TRANSCRICAO | [09:42] Diego: |
| RFC-RISCO-01 | docs/RFC.md | Risco | Risco numero um: defeito no produtor de evento vira defeito na operacao de pedidos, porque fora da transacao perde a garantia toda | TRANSCRICAO | [09:41] Diego: |
| RFC-RISCO-02 | docs/RFC.md | Risco | Segundo processo em producao: se o worker morre nada quebra de imediato, a outbox acumula em silencio e nenhum alerta foi decidido | TRANSCRICAO | [09:11] Diego: |
| RFC-RISCO-03 | docs/RFC.md | Risco | Extensao analitica do RFC, nao conclusao da reuniao: no limite que o desenho admite uma entrega bem-sucedida ultrapassa o alvo sem que nada tenha falhado; aritmetica no FDD 8.1.1 | TRANSCRICAO | [09:42] Diego: |
| RFC-RISCO-04 | docs/RFC.md | Risco | Questao em aberto do RFC: a transcricao afirma 5 tentativas e cinco intervalos ao mesmo tempo, e nenhuma fala desempata; janela vale 2h36min ate a emenda ao ADR-003 | TRANSCRICAO | [09:17] Diego: |
| RFC-RISCO-05 | docs/RFC.md | Risco | Secret recuperavel por cadastro porque e gerada pela plataforma e devolvida na criacao do webhook | TRANSCRICAO | [09:31] Marcos: |
| RFC-RISCO-06 | docs/RFC.md | Risco | Extensao analitica do RFC: nao ha vinculo entre User e Customer no schema, entao a posse do customer_id nao pode ser verificada | CODIGO | prisma/schema.prisma |
| RFC-RISCO-07 | docs/RFC.md | Risco | Extensao analitica do RFC: fechar o ponto exige alteracao de schema antes de qualquer regra de papel, o que tira o endurecimento do escopo de middleware | CODIGO | prisma/schema.prisma |
| RFC-RISCO-08 | docs/RFC.md | Risco | Entrega pode chegar fora de ordem em retentativa; a ordenacao por created_at so se sustenta com worker unico | TRANSCRICAO | [09:12] Diego: |
| RFC-QA-01 | docs/RFC.md | Questao em Aberto | Rate limiting de saida fica como ponto em aberto: observar e implementar se virar problema | TRANSCRICAO | [09:39] Diego: |
| RFC-QA-02 | docs/RFC.md | Questao em Aberto | Endurecimento do controle de acesso do CRUD de webhooks foi adiado para mais pra frente | TRANSCRICAO | [09:37] Sofia: |
| RFC-QA-03 | docs/RFC.md | Questao em Aberto | Escala para multiplos workers e ordenacao global fica em aberto; saidas citadas foram particionar por order_id ou lock pessimista | TRANSCRICAO | [09:13] Diego: |
| RFC-QA-04 | docs/RFC.md | Questao em Aberto | Arquivamento das linhas ja entregues depois de 30 dias ou assim ficou sem definicao e fora do escopo da feature | TRANSCRICAO | [09:08] Diego: |
| RFC-QA-05 | docs/RFC.md | Questao em Aberto | Qual secret assina durante o grace period de 24h nao foi decidido: a reuniao fechou a janela sem entrar no mecanismo | TRANSCRICAO | [09:21] Sofia: |
| RFC-QA-06 | docs/RFC.md | Questao em Aberto | RESOLVIDO: replay de DLQ recoloca o evento na outbox e o FDD definiu que o identificador de evento e preservado | TRANSCRICAO | [09:18] Diego: |
| RFC-FE-01 | docs/RFC.md | Fora de Escopo | Alerta por e-mail ao cliente quando o webhook dele falha fica para proxima fase, depois de medir o impacto | TRANSCRICAO | [09:37] Larissa: |
| RFC-FE-02 | docs/RFC.md | Fora de Escopo | Painel visual para o cliente ver os webhooks e projeto separado do time de frontend; nesta fase so endpoints | TRANSCRICAO | [09:40] Larissa: |
| RFC-FE-03 | docs/RFC.md | Fora de Escopo | Webhooks de entrada estao fora: os clientes querem receber, nao mandar, entao e outbound webhook | TRANSCRICAO | [09:03] Sofia: |
| RFC-METRICA-01 | docs/RFC.md | Metrica | Medir p95 de tempo de resposta por endpoint a partir do historico de entregas, que ja registra tempo de resposta | TRANSCRICAO | [09:34] Marcos: |
| RFC-METRICA-02 | docs/RFC.md | Metrica | Extensao analitica do RFC: o compromisso de latencia so pode ser declarado em percentil sobre a primeira tentativa bem-sucedida | TRANSCRICAO | [09:02] Marcos: |

---

## docs/FDD.md

### Requisitos, decisões e alternativas

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-RF-01 | docs/FDD.md | Requisito Funcional | Notificacao outbound substitui o polling que os tres clientes B2B fazem em GET /orders | TRANSCRICAO | [09:02] Marcos: |
| FDD-RF-02 | docs/FDD.md | Requisito Funcional | Gatilho do evento e exclusivamente a mudanca de status do pedido | TRANSCRICAO | [09:06] Diego: |
| FDD-RF-03 | docs/FDD.md | Requisito Funcional | Evento gravado em webhook_outbox dentro da mesma transacao que atualiza orders e order_status_history | TRANSCRICAO | [09:06] Diego: |
| FDD-RF-04 | docs/FDD.md | Requisito Funcional | Worker separado le a outbox e dispara as chamadas HTTP | TRANSCRICAO | [09:06] Diego: |
| FDD-RF-05 | docs/FDD.md | Requisito Funcional | Worker como entry-point src/worker.ts (a criar) e script npm run worker (a criar) | TRANSCRICAO | [09:11] Larissa: |
| FDD-RF-06 | docs/FDD.md | Requisito Funcional | Polling a cada 2 segundos buscando os eventos pendentes mais antigos | TRANSCRICAO | [09:09] Diego: |
| FDD-RF-07 | docs/FDD.md | Requisito Funcional | Retry com backoff e, esgotado o teto de tentativas, falha permanente movida para DLQ | TRANSCRICAO | [09:15] Diego: |
| FDD-RF-08 | docs/FDD.md | Requisito Funcional | DLQ em tabela webhook_dead_letter com payload, motivo da falha e timestamp | TRANSCRICAO | [09:18] Diego: |
| FDD-RF-09 | docs/FDD.md | Requisito Funcional | Endpoint admin de replay manual recoloca o evento na outbox como pendente | TRANSCRICAO | [09:18] Diego: |
| FDD-RF-10 | docs/FDD.md | Requisito Funcional | HMAC-SHA256 sobre o corpo enviado, assinatura no header X-Signature | TRANSCRICAO | [09:20] Sofia: |
| FDD-RF-11 | docs/FDD.md | Requisito Funcional | Segredo unico por endpoint de webhook, nao global da plataforma | TRANSCRICAO | [09:21] Sofia: |
| FDD-RF-12 | docs/FDD.md | Requisito Funcional | Rotacao de segredo via API com o segredo antigo valido por 24h em paralelo | TRANSCRICAO | [09:21] Sofia: |
| FDD-RF-13 | docs/FDD.md | Requisito Funcional | URL do webhook obrigatoriamente https; http e recusado | TRANSCRICAO | [09:23] Sofia: |
| FDD-RF-14 | docs/FDD.md | Requisito Funcional | Entrega at-least-once com X-Event-Id para o cliente deduplicar do lado dele | TRANSCRICAO | [09:25] Diego: |
| FDD-RF-15 | docs/FDD.md | Requisito Funcional | Cadastro de webhook por POST; a plataforma gera a secret e devolve na criacao | TRANSCRICAO | [09:31] Marcos: |
| FDD-RF-16 | docs/FDD.md | Requisito Funcional | customerId vai no body ou no path do request, nunca e extraido do JWT | TRANSCRICAO | [09:32] Larissa: |
| FDD-RF-17 | docs/FDD.md | Requisito Funcional | CRUD completo: PATCH para editar, DELETE para remover, GET para listar por customer | TRANSCRICAO | [09:33] Bruno: |
| FDD-RF-18 | docs/FDD.md | Requisito Funcional | Cada cadastro escolhe a lista de status que quer ouvir (subscribedStatuses) | TRANSCRICAO | [09:33] Marcos: |
| FDD-RF-19 | docs/FDD.md | Requisito Funcional | Filtragem por status acontece na insercao; sem destinatario interessado nao insere linha | TRANSCRICAO | [09:34] Bruno: |
| FDD-RF-20 | docs/FDD.md | Requisito Funcional | Historico de entregas com sucesso/falha, payload, response e tempo de resposta | TRANSCRICAO | [09:34] Marcos: |
| FDD-RF-21 | docs/FDD.md | Requisito Funcional | Replay exige role ADMIN reaproveitando o requireRole existente | TRANSCRICAO | [09:36] Larissa: |
| FDD-RF-22 | docs/FDD.md | Requisito Funcional | Endpoint de replay registra quem executou, para auditoria (coluna replayedById) | TRANSCRICAO | [09:36] Sofia: |
| FDD-RF-23 | docs/FDD.md | Requisito Funcional | publishWebhookEvent(tx, order, fromStatus, toStatus) recebe o tx da transacao corrente | TRANSCRICAO | [09:41] Bruno: |
| FDD-RF-24 | docs/FDD.md | Requisito Funcional | Timeout de 10s no fetch: cliente que nao responde e tratado como falha e marcado para retry | TRANSCRICAO | [09:42] Diego: |
| FDD-RF-25 | docs/FDD.md | Requisito Funcional | Payload renderizado como snapshot na insercao, refletindo o estado de quando o status mudou | TRANSCRICAO | [09:52] Larissa: |
| FDD-RF-26 | docs/FDD.md | Requisito Funcional | Worker instancia PrismaClient proprio, mesmo banco e mesma DATABASE_URL, outro processo Node | TRANSCRICAO | [09:30] Bruno: |
| FDD-RF-27 | docs/FDD.md | Requisito Funcional | Bootstrap devolve linhas PROCESSING orfas para PENDING - proposta do FDD apoiada no worker unico | TRANSCRICAO | [09:12] Diego: |
| FDD-RF-28 | docs/FDD.md | Requisito Funcional | Varredura periodica do worker zera previousSecret vencido apos as 24h | TRANSCRICAO | [09:21] Sofia: |
| FDD-RF-29 | docs/FDD.md | Requisito Funcional | Cadastro inativo faz o evento ir direto a DLQ sem envio - proposta do FDD sobre o estado ativo | TRANSCRICAO | [09:21] Bruno: |
| FDD-RF-30 | docs/FDD.md | Requisito Funcional | Worker registra SIGINT/SIGTERM e faz shutdown gracioso no molde do bootstrap existente | CODIGO | src/server.ts |
| FDD-RF-31 | docs/FDD.md | Requisito Funcional | Gatilho HTTP do fluxo A e a rota de mudanca de status ja publicada | CODIGO | src/modules/orders/order.routes.ts |
| FDD-RF-52 | docs/FDD.md | Requisito Funcional | Log webhook_dlq_replayed com userId, deadLetterId e eventId atende a auditoria de quem fez o replay | TRANSCRICAO | [09:36] Sofia: |
| FDD-RF-53 | docs/FDD.md | Requisito Funcional | Mitigacao de R-6 e R-10: documentar no portal do desenvolvedor a pausa por active false e a possibilidade de duplicatas | TRANSCRICAO | [09:40] Marcos: |
| FDD-RF-54 | docs/FDD.md | Requisito Funcional | Mitigacao de R-4: isolar HMAC e geracao de segredo em arquivo proprio para a revisao de seguranca antes do deploy | TRANSCRICAO | [09:46] Sofia: |
| FDD-RNF-01 | docs/FDD.md | Requisito Nao Funcional | Commit implica evento na outbox e rollback nao deixa evento orfao; falha na insercao derruba o status | TRANSCRICAO | [09:40] Bruno: |
| FDD-RNF-02 | docs/FDD.md | Requisito Nao Funcional | Nenhuma chamada HTTP de saida dentro da transacao de changeStatus | TRANSCRICAO | [09:04] Bruno: |
| FDD-RNF-03 | docs/FDD.md | Requisito Nao Funcional | Disponibilidade do pedido nao pode ficar acoplada a disponibilidade do cliente | TRANSCRICAO | [09:04] Bruno: |
| FDD-RNF-04 | docs/FDD.md | Requisito Nao Funcional | Nenhuma dependencia nova em package.json; Pino, Zod e error middleware ja existem e o unico acrescimo ao arquivo e o script worker | TRANSCRICAO | [09:29] Bruno: |
| FDD-RNF-05 | docs/FDD.md | Requisito Nao Funcional | Cada entrega auditavel pelo cliente sem acesso ao banco | TRANSCRICAO | [09:34] Marcos: |
| FDD-RNF-06 | docs/FDD.md | Requisito Nao Funcional | Cliente precisa provar que a requisicao veio de nos e que o payload nao foi adulterado | TRANSCRICAO | [09:19] Sofia: |
| FDD-RNF-07 | docs/FDD.md | Requisito Nao Funcional | Processamento sequencial do lote, uma chamada HTTP por vez, sem paralelismo | TRANSCRICAO | [09:12] Diego: |
| FDD-RNF-08 | docs/FDD.md | Requisito Nao Funcional | Serializar o corpo uma unica vez e assinar exatamente a string enviada - detalhe fechado pelo FDD | TRANSCRICAO | [09:20] Sofia: |
| FDD-RNF-09 | docs/FDD.md | Requisito Nao Funcional | secret, previousSecret e previousSecretExpiresAt nunca serializados nas leituras - regra do FDD | TRANSCRICAO | [09:21] Sofia: |
| FDD-RNF-10 | docs/FDD.md | Requisito Nao Funcional | Prefixo WEBHOOK_ obrigatorio em todos os codigos de erro do modulo | TRANSCRICAO | [09:29] Larissa: |
| FDD-RNF-12 | docs/FDD.md | Requisito Nao Funcional | Contrato dos endpoints ja publicados sob o prefixo existente permanece inalterado, inclusive corpo de resposta e envelope de erro | CODIGO | src/app.ts |
| FDD-RNF-52 | docs/FDD.md | Requisito Nao Funcional | secret, previousSecret e o valor de X-Signature nunca podem aparecer em log de aplicacao | TRANSCRICAO | [09:22] Diego: |
| FDD-RNF-53 | docs/FDD.md | Requisito Nao Funcional | Corpo da resposta do cliente nao vai para log, fica apenas em webhook_deliveries.responseBody | TRANSCRICAO | [09:34] Marcos: |
| FDD-RNF-55 | docs/FDD.md | Requisito Nao Funcional | Piso de Node passa de declarado a necessario (>= 20) porque fetch e AbortSignal.timeout viram uso de producao | CODIGO | package.json |
| FDD-DEC-01 | docs/FDD.md | Decisao | Padrao outbox no MySQL existente em vez de fila ou broker | TRANSCRICAO | [09:08] Larissa: |
| FDD-DEC-02 | docs/FDD.md | Decisao | Worker em polling de 2s; latencia minima de 2 segundos aceita como custo | TRANSCRICAO | [09:10] Larissa: |
| FDD-DEC-03 | docs/FDD.md | Decisao | Worker roda em processo separado da instancia da API | TRANSCRICAO | [09:11] Diego: |
| FDD-DEC-04 | docs/FDD.md | Decisao | Instancia unica de worker, com ordenacao implicita por order_id | TRANSCRICAO | [09:12] Diego: |
| FDD-DEC-05 | docs/FDD.md | Decisao | Teto de 5 tentativas de entrega por evento | TRANSCRICAO | [09:16] Larissa: |
| FDD-DEC-06 | docs/FDD.md | Decisao | Progressao de backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa: |
| FDD-DEC-07 | docs/FDD.md | Decisao | DLQ em tabela separada, nao flag failed na propria outbox | TRANSCRICAO | [09:18] Diego: |
| FDD-DEC-08 | docs/FDD.md | Decisao | HMAC-SHA256 sobre o corpo, secret por endpoint, rotacao com grace period de 24h | TRANSCRICAO | [09:22] Sofia: |
| FDD-DEC-09 | docs/FDD.md | Decisao | Teto de 64KB no payload, com erro caso ultrapasse | TRANSCRICAO | [09:24] Larissa: |
| FDD-DEC-10 | docs/FDD.md | Decisao | At-least-once com X-Event-Id para dedup do lado do cliente | TRANSCRICAO | [09:26] Larissa: |
| FDD-DEC-11 | docs/FDD.md | Decisao | Modulo src/modules/webhooks (a criar) com controller, service, repository, routes e schemas, igual aos outros dominios | TRANSCRICAO | [09:27] Bruno: |
| FDD-DEC-12 | docs/FDD.md | Decisao | Logica de processamento em arquivo do modulo, entry separada em src/worker.ts (a criar) | TRANSCRICAO | [09:28] Bruno: |
| FDD-DEC-13 | docs/FDD.md | Decisao | Reuso maximo do existente: AppError, Pino, error middleware, padrao de modulos, schemas Zod e codigos de erro | TRANSCRICAO | [09:30] Larissa: |
| FDD-DEC-14 | docs/FDD.md | Decisao | Snapshot na insercao confirmado como decisao de fechamento da reuniao | TRANSCRICAO | [09:52] Bruno: |
| FDD-DEC-15 | docs/FDD.md | Decisao | Chave primaria UUID CHAR(36), seguindo o padrao do resto do projeto | TRANSCRICAO | [09:51] Larissa: |
| FDD-DEC-16 | docs/FDD.md | Decisao | Enum WebhookOutboxStatus com PENDING, PROCESSING, FAILED e DELIVERED | TRANSCRICAO | [09:08] Diego: |
| FDD-DEC-17 | docs/FDD.md | Decisao | Tabela de configuracao guarda url, secret, customerId e estado ativo | TRANSCRICAO | [09:21] Bruno: |
| FDD-DEC-18 | docs/FDD.md | Decisao | Indices da outbox sobre status e created_at, materializados em compostos pelo FDD | TRANSCRICAO | [09:08] Diego: |
| FDD-DEC-19 | docs/FDD.md | Decisao | Ordem de consumo do lote e createdAt ASC | TRANSCRICAO | [09:12] Diego: |
| FDD-DEC-20 | docs/FDD.md | Decisao | O id da linha da outbox e o proprio X-Event-Id; nao ha segunda coluna de identificador | TRANSCRICAO | [09:25] Diego: |
| FDD-DEC-21 | docs/FDD.md | Decisao | Fan-out de uma linha de outbox por cadastro que escuta o status - lacuna resolvida por analise do FDD | TRANSCRICAO | [09:44] Sofia: |
| FDD-DEC-22 | docs/FDD.md | Decisao | eventId de webhook_deliveries e referencia solta, nao FK, para sobreviver a ida do evento a DLQ | TRANSCRICAO | [09:18] Diego: |
| FDD-DEC-23 | docs/FDD.md | Decisao | Unicidade em eventId na DLQ torna a movimentacao idempotente - proposta do FDD | TRANSCRICAO | [09:18] Diego: |
| FDD-DEC-24 | docs/FDD.md | Decisao | responseBody truncado em 2048 caracteres com sufixo de aviso - proposta do FDD | TRANSCRICAO | [09:34] Marcos: |
| FDD-DEC-25 | docs/FDD.md | Decisao | eventType literal order.status_changed | TRANSCRICAO | [09:43] Diego: |
| FDD-DEC-26 | docs/FDD.md | Decisao | Header X-Webhook-Id acrescentado para o cliente saber qual cadastro caiu no envio | TRANSCRICAO | [09:44] Sofia: |
| FDD-DEC-27 | docs/FDD.md | Decisao | Conjunto de campos basicos do corpo fechado em total_cents pelo FDD; ampliar exige decisao nova | TRANSCRICAO | [09:43] Diego: |
| FDD-DEC-28 | docs/FDD.md | Decisao | Assinatura em hex minusculo de 64 caracteres - proposta do FDD a confirmar na revisao de seguranca | TRANSCRICAO | [09:46] Sofia: |
| FDD-DEC-29 | docs/FDD.md | Decisao | Conjunto de headers de saida e fechado; nenhum header de correlacao nosso vai ao cliente | TRANSCRICAO | [09:44] Diego: |
| FDD-DEC-30 | docs/FDD.md | Decisao | X-Event-Id original preservado no replay - fechado pelo FDD como unica leitura compativel com a dedup | TRANSCRICAO | [09:25] Diego: |
| FDD-DEC-31 | docs/FDD.md | Decisao | Replay responde 202 Accepted porque reenfileira, nao entrega - proposta do FDD | TRANSCRICAO | [09:18] Diego: |
| FDD-DEC-32 | docs/FDD.md | Decisao | Movimentacao para DLQ cria a linha e apaga a da outbox na mesma transacao curta | TRANSCRICAO | [09:18] Diego: |
| FDD-DEC-33 | docs/FDD.md | Decisao | Nada e notificado automaticamente quando um evento cai na DLQ; so restam metrica e log | TRANSCRICAO | [09:37] Larissa: |
| FDD-DEC-34 | docs/FDD.md | Decisao | Mitigacao da ordem e de contrato: from_status/to_status e aviso no portal do desenvolvedor | TRANSCRICAO | [09:26] Marcos: |
| FDD-DEC-35 | docs/FDD.md | Decisao | WEBHOOK_INVALID_URL lancado no service, nao no schema Zod, para nao virar VALIDATION_ERROR generico | TRANSCRICAO | [09:28] Bruno: |
| FDD-DEC-36 | docs/FDD.md | Decisao | DELETE e remocao fisica com cascade; quem quer so parar de receber usa PATCH active false | TRANSCRICAO | [09:33] Bruno: |
| FDD-DEC-37 | docs/FDD.md | Decisao | customerId e secret nao sao editaveis pelo PATCH - proposta do FDD | TRANSCRICAO | [09:33] Bruno: |
| FDD-DEC-38 | docs/FDD.md | Decisao | Publicacao entra apos orderStatusHistory.create e antes do refetch, dentro do mesmo $transaction | CODIGO | src/modules/orders/order.service.ts |
| FDD-DEC-39 | docs/FDD.md | Decisao | Paginacao dos endpoints novos reusa page, pageSize e limite maximo ja praticados no projeto | CODIGO | src/modules/orders/order.schemas.ts |
| FDD-DEC-40 | docs/FDD.md | Decisao | Duracao da entrega medida com process.hrtime.bigint, no mesmo padrao do middleware existente | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-DEC-51 | docs/FDD.md | Decisao | Reparticao de codigos: erro de forma vira VALIDATION_ERROR no validate, regra do modulo vira WEBHOOK_* no service | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-DEC-52 | docs/FDD.md | Decisao | Erros compartilhados VALIDATION_ERROR, UNAUTHORIZED, FORBIDDEN, CONFLICT, NOT_FOUND e INTERNAL_SERVER_ERROR reaproveitados sem alteracao | CODIGO | src/shared/errors/http-errors.ts |
| FDD-DEC-54 | docs/FDD.md | Decisao | BACKOFF_INTERVALS_MS como tabela de constantes e nao formula, porque os intervalos nao tem razao constante | TRANSCRICAO | [09:17] Diego: |
| FDD-DEC-55 | docs/FDD.md | Decisao | Falha retentavel abrange fora de 2xx, timeout e erro de rede; falha permanente e exclusivamente o esgotamento da progressao | TRANSCRICAO | [09:15] Diego: |
| FDD-DEC-59 | docs/FDD.md | Decisao | Metricas emitidas como eventos estruturados do Pino, com o nome no campo metric, para troca futura por exporter | TRANSCRICAO | [09:29] Bruno: |
| FDD-DEC-60 | docs/FDD.md | Decisao | Worker usa logger.child com component webhook-worker para separar as linhas dos dois processos que compartilham o mesmo service | CODIGO | src/shared/logger/index.ts |
| FDD-DEC-61 | docs/FDD.md | Decisao | redactPaths ganha *.secret, *.previousSecret e *.signature; e configuracao do Pino, nao dependencia nova | CODIGO | src/shared/logger/index.ts |
| FDD-DEC-62 | docs/FDD.md | Decisao | Correlacao por requestId propagado, sem SDK de tracing; o middleware ja le ou gera x-request-id e devolve X-Request-Id | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-DEC-64 | docs/FDD.md | Decisao | schema.prisma ganha 4 models e 1 enum novos, mais relacoes inversas em Customer e Order | CODIGO | prisma/schema.prisma |
| FDD-DEC-65 | docs/FDD.md | Decisao | WEBHOOK_WORKER_BATCH_SIZE e obrigatoria e sem default porque a reuniao disse apenas batch pequeno, sem numero | TRANSCRICAO | [09:08] Diego: |
| FDD-DEC-68 | docs/FDD.md | Decisao | Nome webhook.processor.ts (a criar) escolhido entre as duas opcoes que a reuniao deixou em aberto para a logica do worker | TRANSCRICAO | [09:28] Bruno: |
| FDD-DEC-71 | docs/FDD.md | Decisao | HMAC-SHA256 e geracao de segredo com node:crypto (createHmac e randomBytes), sem biblioteca de terceiros | TRANSCRICAO | [09:20] Sofia: |
| FDD-DEC-72 | docs/FDD.md | Decisao | Chamada HTTP de saida com fetch global e timeout com AbortSignal.timeout, sem axios nem node-fetch | CODIGO | package.json |
| FDD-DEC-73 | docs/FDD.md | Decisao | Ordem de deploy: migration, depois API, depois worker; antes do worker os eventos acumulam mas nao se perdem | TRANSCRICAO | [09:06] Diego: |
| FDD-ALT-01 | docs/FDD.md | Alternativa | Envio sincrono dentro do service de orders - descartado por travar mudanca de status | TRANSCRICAO | [09:04] Bruno: |
| FDD-ALT-02 | docs/FDD.md | Alternativa | Redis Streams ou infra equivalente - descartado como overengineering para time pequeno | TRANSCRICAO | [09:07] Diego: |
| FDD-ALT-03 | docs/FDD.md | Alternativa | Trigger de banco para reatividade - descartado porque MySQL nao tem NOTIFY/LISTEN | TRANSCRICAO | [09:09] Diego: |
| FDD-ALT-04 | docs/FDD.md | Alternativa | 3 tentativas - descartado por matar o evento em 30 minutos | TRANSCRICAO | [09:16] Diego: |
| FDD-ALT-05 | docs/FDD.md | Alternativa | Retry indefinido com backoff - descartado por deixar evento pendurado para sempre se o cliente sumir | TRANSCRICAO | [09:15] Diego: |
| FDD-ALT-06 | docs/FDD.md | Alternativa | Marcar failed na propria outbox em vez de tabela separada - descartado | TRANSCRICAO | [09:18] Diego: |
| FDD-ALT-07 | docs/FDD.md | Alternativa | Exactly-once - descartado por exigir coordenacao dos dois lados | TRANSCRICAO | [09:25] Diego: |
| FDD-ALT-08 | docs/FDD.md | Alternativa | customer_id implicito do JWT - revertido na propria reuniao porque o JWT e do usuario operador | TRANSCRICAO | [09:32] Bruno: |
| FDD-ALT-09 | docs/FDD.md | Alternativa | Filtrar status na hora do envio - descartado em favor da filtragem na insercao | TRANSCRICAO | [09:34] Bruno: |
| FDD-ALT-10 | docs/FDD.md | Alternativa | Injetar repository de webhook no OrderService - descartado em favor de funcao pura recebendo o tx | TRANSCRICAO | [09:41] Diego: |
| FDD-ALT-11 | docs/FDD.md | Alternativa | Truncar payload acima do limite - descartado em favor de erro | TRANSCRICAO | [09:23] Sofia: |
| FDD-ALT-12 | docs/FDD.md | Alternativa | Particionar por order_id ou lock pessimista para escalar - adiado como problema do futuro | TRANSCRICAO | [09:13] Diego: |
| FDD-ALT-13 | docs/FDD.md | Alternativa | Guardar o segredo com bcrypt como o projeto guarda senha e inaplicavel, e o BCRYPT_ROUNDS nao serve de precedente - analise do FDD sobre o codigo | CODIGO | src/modules/users/user.service.ts |
| FDD-ALT-50 | docs/FDD.md | Alternativa | Jitter no backoff descartado: nao foi mencionado na reuniao e o volume de tres clientes B2B nao gera thundering herd | TRANSCRICAO | [09:00] Marcos: |
| FDD-ALT-52 | docs/FDD.md | Alternativa | Paralelismo de workers adiado; particionar por order_id ou usar lock pessimista fica como opcao futura | TRANSCRICAO | [09:13] Diego: |
| FDD-ALT-53 | docs/FDD.md | Alternativa | Proposta do FDD para o risco R-6: transformar DELETE em remocao logica seria decisao nova, com ADR proprio | TRANSCRICAO | [09:34] Marcos: |

### Contratos, matriz de erros e restrições

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /api/v1/webhooks devolve 201 com o segredo exibido uma unica vez | TRANSCRICAO | [09:31] Marcos: |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET /api/v1/webhooks com customerId obrigatorio na query, resposta paginada e sem secret | TRANSCRICAO | [09:33] Bruno: |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | GET /api/v1/webhooks/:id para leitura unitaria - endpoint nao citado na reuniao, proposta do FDD | TRANSCRICAO | [09:33] Bruno: |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | PATCH /api/v1/webhooks/:id edita url, subscribedStatuses e active e devolve 200 | TRANSCRICAO | [09:33] Bruno: |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | DELETE /api/v1/webhooks/:id devolve 204 sem corpo | TRANSCRICAO | [09:33] Bruno: |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | GET /api/v1/webhooks/:id/deliveries devolve o historico ordenado por createdAt DESC | TRANSCRICAO | [09:34] Marcos: |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | POST /api/v1/webhooks/:id/secret/rotate sem corpo - o caminho exato e proposta do FDD | TRANSCRICAO | [09:21] Sofia: |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | POST /api/v1/admin/webhooks/dead-letter/:id/replay devolve 202 Accepted | TRANSCRICAO | [09:35] Diego: |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato | Request de saida POST com X-Event-Id, X-Signature, X-Timestamp e Content-Type application/json | TRANSCRICAO | [09:44] Diego: |
| FDD-CONTRATO-10 | docs/FDD.md | Contrato | Corpo do evento com nove campos snake_case: event_id, event_type, timestamp, order_id, order_number, from_status, to_status, customer_id, total_cents | TRANSCRICAO | [09:43] Diego: |
| FDD-CONTRATO-11 | docs/FDD.md | Contrato | 2xx e sucesso; qualquer outra resposta ou ausencia de resposta em 10s e falha retentavel | TRANSCRICAO | [09:42] Diego: |
| FDD-CONTRATO-16 | docs/FDD.md | Contrato | Prefixo real da API e /api/v1; os caminhos citados na reuniao sao relativos a ele | CODIGO | src/app.ts |
| FDD-CONTRATO-17 | docs/FDD.md | Contrato | Envelope de erro com error.code e error.message, e details apenas quando o erro o carrega | CODIGO | src/middlewares/error.middleware.ts |
| FDD-CONTRATO-18 | docs/FDD.md | Contrato | Todos os endpoints exigem Bearer JWT com sub, email e role | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-CONTRATO-19 | docs/FDD.md | Contrato | 403 FORBIDDEN para role OPERATOR no replay, serializado sem codigo novo | CODIGO | src/shared/errors/http-errors.ts |
| FDD-CONTRATO-20 | docs/FDD.md | Contrato | Toda resposta traz X-Request-Id gerado ou refletido pelo middleware existente | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-CONTRATO-50 | docs/FDD.md | Contrato | Eventos de log do modulo em snake_case (webhook_event_enqueued, webhook_delivery_attempt, webhook_event_dead_lettered e demais), no molde de server_started | CODIGO | src/server.ts |
| FDD-CONTRATO-51 | docs/FDD.md | Contrato | Extensao do FDD: changeStatus e publishWebhookEvent ganham parametro opcional requestId para atravessar os dois processos | TRANSCRICAO | [09:41] Bruno: |
| FDD-CONTRATO-52 | docs/FDD.md | Contrato | order.service.ts alterado: publicacao entra dentro da transacao de changeStatus e a assinatura ganha requestId opcional | CODIGO | src/modules/orders/order.service.ts |
| FDD-CONTRATO-53 | docs/FDD.md | Contrato | order.controller.ts alterado em uma unica linha para repassar req.id ao service | CODIGO | src/modules/orders/order.controller.ts |
| FDD-CONTRATO-54 | docs/FDD.md | Contrato | env.ts ganha WEBHOOK_POLL_INTERVAL_MS, WEBHOOK_HTTP_TIMEOUT_MS, WEBHOOK_MAX_RETRIES, WEBHOOK_PAYLOAD_MAX_BYTES e WEBHOOK_SECRET_GRACE_HOURS | CODIGO | src/config/env.ts |
| FDD-CONTRATO-55 | docs/FDD.md | Contrato | routes/index.ts monta /webhooks e /admin/webhooks, primeiro prefixo admin do projeto | CODIGO | src/routes/index.ts |
| FDD-ERRO-50 | docs/FDD.md | Contrato | WEBHOOK_NOT_FOUND (404) quando o id nao existe em webhook_endpoints | TRANSCRICAO | [09:28] Bruno: |
| FDD-ERRO-51 | docs/FDD.md | Contrato | WEBHOOK_INVALID_URL (400) quando a URL cadastrada nao usa esquema https | TRANSCRICAO | [09:23] Sofia: |
| FDD-ERRO-52 | docs/FDD.md | Contrato | WEBHOOK_PAYLOAD_TOO_LARGE (422) quando o snapshot serializado passa de 64 KB, com rollback da mudanca de status | TRANSCRICAO | [09:24] Larissa: |
| FDD-ERRO-53 | docs/FDD.md | Contrato | WEBHOOK_ENDPOINT_INACTIVE (409 na API e motivo de DLQ no worker) para cadastro com active = false | TRANSCRICAO | [09:21] Bruno: |
| FDD-ERRO-54 | docs/FDD.md | Contrato | WEBHOOK_SECRET_ROTATION_IN_PROGRESS (409) ao rotacionar enquanto o grace period de 24h ainda vigora | TRANSCRICAO | [09:21] Sofia: |
| FDD-ERRO-55 | docs/FDD.md | Contrato | WEBHOOK_DEAD_LETTER_NOT_FOUND (404) no replay com id inexistente em webhook_dead_letter | TRANSCRICAO | [09:18] Diego: |
| FDD-ERRO-56 | docs/FDD.md | Contrato | WEBHOOK_ALREADY_REPLAYED (409) ao replayar linha de DLQ com replayedAt ja preenchido | TRANSCRICAO | [09:18] Diego: |
| FDD-ERRO-57 | docs/FDD.md | Contrato | WEBHOOK_DELIVERY_TIMEOUT emitido pelo worker quando a chamada HTTP nao responde em 10 segundos | TRANSCRICAO | [09:42] Diego: |
| FDD-ERRO-58 | docs/FDD.md | Contrato | WEBHOOK_DELIVERY_FAILED emitido pelo worker para resposta fora de 2xx ou erro de rede, DNS ou TLS | TRANSCRICAO | [09:15] Diego: |
| FDD-RESTR-01 | docs/FDD.md | Restricao | changeStatus ja e uma transacao pesada que atualiza orders, history e estoque | TRANSCRICAO | [09:04] Bruno: |
| FDD-RESTR-02 | docs/FDD.md | Restricao | Worker usa mesmo banco e mesma stack; so nao pode ser o mesmo processo | TRANSCRICAO | [09:11] Diego: |
| FDD-RESTR-03 | docs/FDD.md | Restricao | Alterar subscribedStatuses nao afeta eventos ja enfileirados, porque a filtragem e na insercao | TRANSCRICAO | [09:34] Bruno: |
| FDD-RESTR-04 | docs/FDD.md | Restricao | Ordem dos middlewares segue o precedente authenticate, requireRole, validate e handler | CODIGO | src/modules/users/user.routes.ts |
| FDD-RESTR-05 | docs/FDD.md | Restricao | Tabelas em snake_case via @@map com colunas em camelCase, inclusive no SQL gerado | CODIGO | prisma/migrations/20260519182739_init/migration.sql |
| FDD-RESTR-06 | docs/FDD.md | Restricao | Timestamps DATETIME(3) e charset utf8mb4 herdados do schema existente | CODIGO | prisma/migrations/20260519182739_init/migration.sql |
| FDD-RESTR-07 | docs/FDD.md | Restricao | Coluna Json tem precedente real em Customer.address; enums em SCREAMING_SNAKE_CASE | CODIGO | prisma/schema.prisma |
| FDD-RESTR-08 | docs/FDD.md | Restricao | to_status nunca assume PENDING porque nenhuma transicao chega nesse estado | CODIGO | src/modules/orders/order.status.ts |
| FDD-RESTR-09 | docs/FDD.md | Restricao | order_number segue o formato ORD- mais seis digitos gerado pelo service de pedidos | CODIGO | src/modules/orders/order.service.ts |
| FDD-RESTR-10 | docs/FDD.md | Restricao | Node >= 20 declarado em engines viabiliza fetch global sem dependencia nova | CODIGO | package.json |
| FDD-RESTR-11 | docs/FDD.md | Restricao | MySQL 8.0 com character-set-server utf8mb4 definido no ambiente de desenvolvimento | CODIGO | docker-compose.yml |
| FDD-RESTR-12 | docs/FDD.md | Restricao | O cliente Atlas Comercial ja existe na massa de dados, servindo de fixture real para os testes | CODIGO | prisma/seed.ts |
| FDD-RESTR-13 | docs/FDD.md | Restricao | O prefixo admin do replay e o primeiro do projeto no roteador que hoje so monta routers de dominio | CODIGO | src/routes/index.ts |
| FDD-RESTR-14 | docs/FDD.md | Restricao | Worker instancia o PrismaClient pela factory existente, com a mesma DATABASE_URL | CODIGO | src/config/database.ts |
| FDD-RESTR-15 | docs/FDD.md | Restricao | Logger do worker e filho do singleton Pino ja configurado no projeto | CODIGO | src/shared/logger/index.ts |
| FDD-RESTR-50 | docs/FDD.md | Restricao | Os statusCode 504 e 502 dos erros de worker nao trafegam: existem so porque o construtor de AppError exige statusCode | CODIGO | src/shared/errors/app-error.ts |
| FDD-RESTR-51 | docs/FDD.md | Restricao | NotFoundError nao pode ser reaproveitada para WEBHOOK_NOT_FOUND porque fixa o codigo NOT_FOUND sem override | CODIGO | src/shared/errors/http-errors.ts |
| FDD-RESTR-52 | docs/FDD.md | Restricao | BadRequestError, ConflictError e UnprocessableEntityError aceitam codigo customizado e sao as bases corretas das classes WEBHOOK_* | CODIGO | src/shared/errors/http-errors.ts |
| FDD-RESTR-53 | docs/FDD.md | Restricao | Transacoes do Prisma seguem o default do client; nenhum transactionOptions novo e introduzido | CODIGO | src/config/database.ts |
| FDD-RESTR-54 | docs/FDD.md | Restricao | Nada esvazia a DLQ sozinho: se ninguem olhar, o evento nunca chega ao cliente | TRANSCRICAO | [09:18] Diego: |
| FDD-RESTR-55 | docs/FDD.md | Restricao | Nenhum cliente de metricas nem SDK de tracing entra no projeto; observabilidade fica sobre o Pino ja instalado | TRANSCRICAO | [09:29] Bruno: |
| FDD-RESTR-56 | docs/FDD.md | Restricao | tests/setup.ts precisa dos 4 deleteMany novos antes de order, customer e user, sob pena de violacao de FK na suite inteira | CODIGO | tests/setup.ts |
| FDD-RESTR-57 | docs/FDD.md | Restricao | error, auth, validate e request-logger middlewares nao mudam: ja tratam AppError, Zod, Prisma e correlacao | TRANSCRICAO | [09:29] Bruno: |
| FDD-RESTR-58 | docs/FDD.md | Restricao | docker-compose.yml nao ganha servico novo: sobe apenas mysql:8.0, sem Redis e sem broker | CODIGO | docker-compose.yml |
| FDD-RESTR-59 | docs/FDD.md | Restricao | tsconfig.build.json ja compila src/worker.ts (a criar) para dist/worker.js sem nenhuma linha de configuracao nova | CODIGO | tsconfig.build.json |
| FDD-RESTR-61 | docs/FDD.md | Restricao | Unica mudanca observavel de contrato existente: PATCH /orders/:id/status pode passar a retornar 422 WEBHOOK_PAYLOAD_TOO_LARGE | TRANSCRICAO | [09:24] Larissa: |
| FDD-RESTR-62 | docs/FDD.md | Restricao | Migration puramente aditiva: 4 tabelas e 1 enum novos, sem ALTER destrutivo, reversivel por DROP TABLE e sem mudanca em coluna existente | CODIGO | prisma/migrations/20260519182739_init/migration.sql |
| FDD-RESTR-63 | docs/FDD.md | Restricao | O limite de 1mb do express.json rege o corpo que entra e nao se confunde com o teto de 64 KB do evento que sai | CODIGO | src/app.ts |
| FDD-TRADE-01 | docs/FDD.md | Trade-off | Polling troca reatividade por simplicidade; a latencia minima de 2s foi aceita explicitamente | TRANSCRICAO | [09:10] Larissa: |
| FDD-TRADE-02 | docs/FDD.md | Trade-off | At-least-once joga a responsabilidade de deduplicar para o cliente | TRANSCRICAO | [09:25] Sofia: |
| FDD-TRADE-03 | docs/FDD.md | Trade-off | Worker unico da vazao previsivel e ordem por created_at, ao custo de nao escalar | TRANSCRICAO | [09:12] Diego: |
| FDD-TRADE-04 | docs/FDD.md | Trade-off | Cinco tentativas cobrem indisponibilidade longa, mas seguram o evento por horas | TRANSCRICAO | [09:16] Diego: |
| FDD-TRADE-05 | docs/FDD.md | Trade-off | Snapshot na insercao mantem o evento fiel ao momento da mudanca, ao custo de duplicar dado | TRANSCRICAO | [09:52] Larissa: |
| FDD-TRADE-50 | docs/FDD.md | Trade-off | Tensao apontada pelo FDD entre tratar https como mera validacao Zod e ter codigo proprio WEBHOOK_INVALID_URL do modulo | TRANSCRICAO | [09:23] Sofia: |
| FDD-RISCO-01 | docs/FDD.md | Risco | O retry quebra a ordem por order_id mesmo com worker unico - extensao de analise do FDD, nao levantada na reuniao | TRANSCRICAO | [09:13] Larissa: |
| FDD-RISCO-02 | docs/FDD.md | Risco | Crash entre o POST e a marcacao deixa a linha presa em PROCESSING e provoca reentrega - analise do FDD, nao levantada na reuniao | TRANSCRICAO | [09:24] Diego: |
| FDD-RISCO-03 | docs/FDD.md | Risco | X-Timestamp fica fora do escopo assinado e nao barra atacante ativo - analise do FDD | TRANSCRICAO | [09:44] Diego: |
| FDD-RISCO-04 | docs/FDD.md | Risco | Segredo mantido em texto claro na coluna, com decisao de seguranca ainda pendente | TRANSCRICAO | [09:46] Sofia: |
| FDD-RISCO-05 | docs/FDD.md | Risco | Desativar cadastro com fila acumulada move tudo para a DLQ de uma vez, e o replay e unitario - analise do FDD | TRANSCRICAO | [09:18] Diego: |
| FDD-RISCO-07 | docs/FDD.md | Risco | Cinquenta mudancas de status geram cinquenta chamadas, sem teto por cliente por janela | TRANSCRICAO | [09:38] Diego: |
| FDD-RISCO-08 | docs/FDD.md | Risco | O estoque e movimentado antes do update da order, contrariando a descricao da ata - analise do FDD sobre o codigo | CODIGO | src/modules/orders/order.service.ts |
| FDD-RISCO-51 | docs/FDD.md | Risco | R-1: a transacao de changeStatus alarga e piora a contencao de lock em orders e products - analise do FDD sobre o codigo | CODIGO | src/modules/orders/order.service.ts |
| FDD-RISCO-52 | docs/FDD.md | Risco | R-2: um defeito no modulo de webhooks derruba a mudanca de status, porque a insercao esta dentro da transacao | TRANSCRICAO | [09:40] Bruno: |
| FDD-RISCO-53 | docs/FDD.md | Risco | R-3: worker morto passa despercebido; nenhuma requisicao falha, a fila acumula em silencio e os clientes param de receber | TRANSCRICAO | [09:11] Diego: |
| FDD-RISCO-54 | docs/FDD.md | Risco | R-4: segredo vazando em log, com caso real relatado de cliente que expos secret em log de aplicacao | TRANSCRICAO | [09:22] Diego: |
| FDD-RISCO-55 | docs/FDD.md | Risco | R-5: grace period de rotacao sem regra de assinatura definida bloqueia o passo de assinatura do worker | TRANSCRICAO | [09:21] Sofia: |
| FDD-RISCO-56 | docs/FDD.md | Risco | R-6: DELETE /webhooks/:id apaga historico de entregas e DLQ junto, por onDelete Cascade - analise do FDD, nao discutida na reuniao | TRANSCRICAO | [09:34] Marcos: |
| FDD-RISCO-57 | docs/FDD.md | Risco | R-7: divergencia aritmetica apontada pelo FDD entre 5 tentativas e cinco intervalos; janela real de 2h36min | TRANSCRICAO | [09:17] Diego: |
| FDD-RISCO-58 | docs/FDD.md | Risco | R-8: webhook_outbox e webhook_deliveries crescem sem limite porque arquivamento ficou fora do escopo | TRANSCRICAO | [09:08] Diego: |
| FDD-RISCO-59 | docs/FDD.md | Risco | R-9: qualquer usuario autenticado cadastra e rotaciona webhook de qualquer customerId, viabilizando vazamento entre clientes | TRANSCRICAO | [09:37] Sofia: |
| FDD-RISCO-60 | docs/FDD.md | Risco | R-10: cliente sem dedup processa evento duplicado, consequencia assumida da garantia at-least-once | TRANSCRICAO | [09:24] Diego: |
| FDD-RISCO-61 | docs/FDD.md | Risco | R-11: cliente lento drena a vazao do worker unico, porque o processamento e sequencial e cada envio pode levar 10s | TRANSCRICAO | [09:12] Diego: |
| FDD-RISCO-62 | docs/FDD.md | Risco | 8.1.1: analise do FDD, nao conclusao da reuniao - o timeout de 10s sozinho iguala o alvo de 10s, entao um destino que responde em 9s entrega em ~11s sem que nada tenha falhado; reduzir o timeout e trade-off, nao ajuste | TRANSCRICAO | [09:42] Diego: |
| FDD-QA-01 | docs/FDD.md | Questao em Aberto | Qual segredo assina durante o grace period de 24h: o novo, o antigo ou ambos - bloqueia o passo de assinatura e vai para a revisao de seguranca | TRANSCRICAO | [09:21] Sofia: |
| FDD-QA-02 | docs/FDD.md | Questao em Aberto | Tamanho e codificacao do segredo gerado ficam para a revisao de seguranca | TRANSCRICAO | [09:46] Sofia: |
| FDD-QA-03 | docs/FDD.md | Questao em Aberto | Divergencia aritmetica: 5 chamadas consomem 4 esperas e nao fecham as quase 15 horas prometidas | TRANSCRICAO | [09:17] Diego: |
| FDD-QA-04 | docs/FDD.md | Questao em Aberto | Rate limiting de saida fica como ponto a observar e decidir depois, e e mitigacao pendente do risco do worker unico | TRANSCRICAO | [09:39] Diego: |
| FDD-QA-05 | docs/FDD.md | Questao em Aberto | Tamanho do lote do polling ficou como batch pequeno, sem numero; vira env obrigatoria sem default | TRANSCRICAO | [09:08] Diego: |
| FDD-QA-06 | docs/FDD.md | Questao em Aberto | Autorizacao do endpoint de rotacao nao foi discutida; candidato a exigir ADMIN | TRANSCRICAO | [09:46] Sofia: |
| FDD-QA-07 | docs/FDD.md | Questao em Aberto | Derrubar a mudanca de status por payload acima de 64KB e inferencia do FDD e precisa de ratificacao | TRANSCRICAO | [09:24] Larissa: |
| FDD-QA-08 | docs/FDD.md | Questao em Aberto | Incluir o timestamp no escopo assinado - opcao a levar a revisao de seguranca | TRANSCRICAO | [09:46] Sofia: |
| FDD-QA-50 | docs/FDD.md | Questao em Aberto | WEBHOOK_SECRET_REQUIRED so voltaria se a rotacao passar a aceitar segredo informado pelo cliente, hipotese para a revisao de seguranca | TRANSCRICAO | [09:46] Sofia: |
| FDD-QA-52 | docs/FDD.md | Questao em Aberto | Emenda ao ADR-003 antes do go-live para reconciliar o teto de tentativas com os cinco intervalos de backoff | TRANSCRICAO | [09:17] Diego: |
| FDD-FE-01 | docs/FDD.md | Fora de Escopo | Rate limiting de saida por cliente | TRANSCRICAO | [09:39] Larissa: |
| FDD-FE-02 | docs/FDD.md | Fora de Escopo | Arquivamento ou expurgo de linhas ja entregues da outbox | TRANSCRICAO | [09:08] Diego: |
| FDD-FE-03 | docs/FDD.md | Fora de Escopo | Recepcao de webhooks inbound; os clientes querem receber, nao mandar | TRANSCRICAO | [09:02] Marcos: |
| FDD-FE-04 | docs/FDD.md | Fora de Escopo | Notificacao por e-mail ao cliente quando o webhook dele falha | TRANSCRICAO | [09:37] Larissa: |
| FDD-FE-05 | docs/FDD.md | Fora de Escopo | Painel ou dashboard visual; apenas endpoints HTTP | TRANSCRICAO | [09:40] Larissa: |
| FDD-FE-06 | docs/FDD.md | Fora de Escopo | Deduplicacao do nosso lado; nenhuma tabela de ids processados e nenhum lock | TRANSCRICAO | [09:25] Diego: |
| FDD-FE-07 | docs/FDD.md | Fora de Escopo | Ordenacao global entre pedidos e execucao de mais de um worker | TRANSCRICAO | [09:13] Larissa: |
| FDD-FE-08 | docs/FDD.md | Fora de Escopo | Restricao de papel no CRUD; qualquer usuario autenticado opera qualquer customerId | TRANSCRICAO | [09:37] Sofia: |
| FDD-FE-09 | docs/FDD.md | Fora de Escopo | Classificar 4xx como falha permanente; o unico criterio definitivo e o teto de tentativas, entao 410 Gone e 404 do cliente sao tratados como um 503 | TRANSCRICAO | [09:15] Diego: |
| FDD-FE-10 | docs/FDD.md | Fora de Escopo | Replay automatico ou em lote; um evento por chamada, disparado por humano | TRANSCRICAO | [09:18] Diego: |
| FDD-FE-11 | docs/FDD.md | Fora de Escopo | Cifra do segredo em repouso; a coluna guarda o valor utilizavel, tema da revisao de seguranca | TRANSCRICAO | [09:46] Sofia: |
| FDD-FE-12 | docs/FDD.md | Fora de Escopo | Versionamento do payload; evento enfileirado sai no formato com que foi gravado | TRANSCRICAO | [09:52] Larissa: |
| FDD-FE-13 | docs/FDD.md | Fora de Escopo | Emissao de evento na criacao do pedido; o gatilho e exclusivamente a mudanca de status | TRANSCRICAO | [09:06] Diego: |
| FDD-FE-14 | docs/FDD.md | Fora de Escopo | Envio de items no payload; o cliente busca detalhes em GET /orders/:id | TRANSCRICAO | [09:43] Diego: |
| FDD-FE-50 | docs/FDD.md | Fora de Escopo | WEBHOOK_SECRET_REQUIRED descartado da matriz: sem caminho de ocorrencia porque a plataforma gera o segredo | TRANSCRICAO | [09:31] Marcos: |
| FDD-METRICA-01 | docs/FDD.md | Metrica | Orcamento de latencia ponta a ponta de 10 segundos, definido pelos clientes como tempo real | TRANSCRICAO | [09:02] Marcos: |
| FDD-METRICA-02 | docs/FDD.md | Metrica | Janela de quase 15 horas entre a primeira falha e a ultima tentativa, conforme a progressao proposta | TRANSCRICAO | [09:17] Diego: |
| FDD-METRICA-03 | docs/FDD.md | Metrica | Os ultimos ~100 deliveries correspondem ao pageSize maximo de 100 ja praticado no projeto | CODIGO | src/modules/orders/order.schemas.ts |
| FDD-METRICA-50 | docs/FDD.md | Metrica | Metrica proposta pelo FDD: webhook_event_lag_ms (histograma) mede createdAt ate a primeira chamada HTTP e valida o orcamento de 10 segundos | TRANSCRICAO | [09:02] Marcos: |
| FDD-METRICA-51 | docs/FDD.md | Metrica | Metrica proposta pelo FDD: webhook_outbox_pending_total (gauge) emitida uma vez por ciclo de polling para mostrar se a fila esta drenando | TRANSCRICAO | [09:09] Diego: |
| FDD-METRICA-52 | docs/FDD.md | Metrica | Metrica proposta pelo FDD: webhook_delivery_attempts_total (contador) com labels outcome, webhookEndpointId e httpStatus | TRANSCRICAO | [09:34] Marcos: |
| FDD-METRICA-53 | docs/FDD.md | Metrica | Metrica proposta pelo FDD: webhook_delivery_duration_ms (histograma) por endpoint identifica quem esta perto de estourar o timeout de 10s | TRANSCRICAO | [09:42] Diego: |
| FDD-METRICA-54 | docs/FDD.md | Metrica | Metrica proposta pelo FDD: webhook_dead_letter_total (contador) com labels webhookEndpointId e reason, substituto operacional do alerta por e-mail adiado | TRANSCRICAO | [09:37] Larissa: |
| FDD-METRICA-55 | docs/FDD.md | Metrica | Metrica proposta pelo FDD: webhook_outbox_reclaimed_total (contador) diferente de zero no bootstrap indica crash com entrega em voo e duplicata provavel | TRANSCRICAO | [09:24] Diego: |
| FDD-METRICA-56 | docs/FDD.md | Metrica | Metrica proposta pelo FDD: webhook_retry_scheduled_total (contador) por attemptNumber mostra se a progressao de backoff resolve ou so adia a DLQ | TRANSCRICAO | [09:17] Diego: |
| FDD-AC-50 | docs/FDD.md | Requisito Funcional | Criterio: erro apos a insercao na outbox nao deixa evento orfao; status permanece o anterior e a outbox fica sem linha | TRANSCRICAO | [09:40] Bruno: |
| FDD-AC-51 | docs/FDD.md | Requisito Funcional | Criterio: cliente sem cadastro que escute o status nao gera linha e a mudanca de status ocorre normalmente | TRANSCRICAO | [09:34] Bruno: |
| FDD-AC-52 | docs/FDD.md | Requisito Funcional | Criterio: cliente com dois cadastros escutando o mesmo status gera duas linhas com X-Event-Id distintos e mesmo payload | TRANSCRICAO | [09:44] Sofia: |
| FDD-AC-53 | docs/FDD.md | Requisito Funcional | Criterio: o payload gravado e snapshot; alterar a order depois nao altera a linha da outbox | TRANSCRICAO | [09:52] Larissa: |
| FDD-AC-54 | docs/FDD.md | Requisito Funcional | Criterio: criacao de pedido nao gera evento, pois o gatilho e exclusivamente a mudanca de status | TRANSCRICAO | [09:06] Diego: |
| FDD-AC-55 | docs/FDD.md | Requisito Funcional | Criterio: replay por OPERATOR responde 403 FORBIDDEN; hoje nao existe asseracao de 403 no repositorio | CODIGO | tests/auth.test.ts |
| FDD-AC-56 | docs/FDD.md | Requisito Funcional | Criterio: secret aparece apenas nas respostas de criacao (201) e de rotacao (200), nunca em GET, PATCH ou listagem | TRANSCRICAO | [09:31] Marcos: |
| FDD-AC-57 | docs/FDD.md | Requisito Funcional | Criterio: POST /webhooks com URL http responde 400 com error.code igual a WEBHOOK_INVALID_URL | TRANSCRICAO | [09:23] Sofia: |
| FDD-AC-58 | docs/FDD.md | Requisito Funcional | Criterio: X-Signature e HMAC-SHA256(body, secret) em hex calculado sobre a mesma string enviada como corpo | TRANSCRICAO | [09:20] Sofia: |
| FDD-AC-59 | docs/FDD.md | Requisito Funcional | Criterio: os cinco headers de saida presentes em todo envio, com X-Event-Id e X-Webhook-Id corretos | TRANSCRICAO | [09:44] Diego: |
| FDD-AC-60 | docs/FDD.md | Requisito Funcional | Criterio: cliente que nao responde em 10 segundos e tratado como falha, testado com servidor que atrasa a resposta | TRANSCRICAO | [09:42] Diego: |
| FDD-AC-61 | docs/FDD.md | Requisito Funcional | Criterio: resposta 500 mantem o evento como FAILED com nextAttemptAt exatamente 1 minuto a frente na primeira falha | TRANSCRICAO | [09:17] Diego: |
| FDD-AC-62 | docs/FDD.md | Requisito Funcional | Criterio: X-Event-Id identico em todas as tentativas do mesmo evento, inclusive no replay | TRANSCRICAO | [09:25] Diego: |
| FDD-AC-63 | docs/FDD.md | Requisito Funcional | Criterio: esgotada a progressao, a linha sai da outbox e entra na DLQ com payload, failureReason e failedAt | TRANSCRICAO | [09:18] Diego: |
| FDD-AC-64 | docs/FDD.md | Requisito Funcional | Criterio: replay recria a linha na outbox com o mesmo X-Event-Id e grava replayedById do usuario ADMIN | TRANSCRICAO | [09:36] Sofia: |
| FDD-AC-65 | docs/FDD.md | Requisito Funcional | Criterio: nenhuma linha de log contem secret, previousSecret ou X-Signature em fluxo completo de rotacao e entrega | TRANSCRICAO | [09:22] Diego: |
| FDD-AC-66 | docs/FDD.md | Requisito Funcional | Criterio: git diff do package.json mostra apenas acrescimo de script, com zero dependencias novas | TRANSCRICAO | [09:29] Bruno: |
| FDD-AC-67 | docs/FDD.md | Requisito Funcional | Criterio: a suite existente passa sem alteracao com os quatro deleteMany novos na ordem correta de FK | CODIGO | tests/setup.ts |
| FDD-AC-68 | docs/FDD.md | Requisito Funcional | Criterio: npm run worker sobe o processo com PrismaClient proprio e responde a SIGTERM encerrando o laco | TRANSCRICAO | [09:30] Bruno: |
| FDD-AC-69 | docs/FDD.md | Requisito Funcional | Criterio: menos de 10 segundos entre a resposta do PATCH de status e a primeira chamada HTTP ao cliente | TRANSCRICAO | [09:02] Marcos: |
| FDD-AC-70 | docs/FDD.md | Requisito Funcional | Criterio: GET /webhooks/:id/deliveries devolve data mais pagination, ordenado por createdAt desc, no maximo 100 por pagina | TRANSCRICAO | [09:34] Marcos: |
| FDD-AC-71 | docs/FDD.md | Requisito Funcional | Criterio: todo erro do modulo responde com error.code comecando em WEBHOOK_, exceto os erros compartilhados | TRANSCRICAO | [09:29] Larissa: |

---

## docs/adrs/ADR-001-outbox-no-mysql.md

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisao | Adotar o padrao outbox no MySQL que ja existe para emissao dos eventos de webhook, sem infraestrutura nova | TRANSCRICAO | [09:08] Larissa: |
| ADR-001-DEC-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisao | Evento inserido na tabela webhook_outbox dentro da mesma transacao SQL que atualiza orders e insere em order_status_history | TRANSCRICAO | [09:06] Diego: |
| ADR-001-DEC-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisao | Atomicidade e a razao de ser: se a transacao commitou o evento esta registrado, se deu rollback o evento some junto | TRANSCRICAO | [09:06] Diego: |
| ADR-001-DEC-03 | docs/adrs/ADR-001-outbox-no-mysql.md | Restricao | Falha ao inserir na outbox provoca rollback da mudanca de status - nao pode haver caso de status mudar e evento nao sair | TRANSCRICAO | [09:40] Bruno: |
| ADR-001-DEC-04 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisao | Outbox tem estados pendente, processando, falhou e entregue, com indice no campo de status e em created_at | TRANSCRICAO | [09:08] Diego: |
| ADR-001-DEC-05 | docs/adrs/ADR-001-outbox-no-mysql.md | Contrato | Ponto de integracao e a funcao publishWebhookEvent(tx, order, fromStatus, toStatus) recebendo o tx da transacao corrente | TRANSCRICAO | [09:41] Bruno: |
| ADR-001-DEC-06 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisao | Funcao pura recebendo o tx em vez de injetar o repository de webhooks inteiro dentro do OrderService | TRANSCRICAO | [09:41] Diego: |
| ADR-001-ALT-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa | Despacho HTTP sincrono dentro do OrderService - descartada: cliente lento trava mudanca de status de outros pedidos | TRANSCRICAO | [09:04] Bruno: |
| ADR-001-ALT-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa | Redis Streams ou fila externa - descartada por custo operacional; time pequeno e subir Redis Cluster seria overengineering | TRANSCRICAO | [09:07] Diego: |
| ADR-001-CONTRATO-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Contrato | webhook_outbox e estrutura interna e nao contrato publico; o cliente so ve o webhook entregue e o historico de entregas por API | TRANSCRICAO | [09:34] Marcos: |
| ADR-001-RNF-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Requisito Nao Funcional | Latencia de notificacao abaixo de 10 segundos ja e considerada tempo real pelos clientes B2B | TRANSCRICAO | [09:02] Marcos: |
| ADR-001-REST-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Restricao | Escopo apenas outbound: os eventos saem da nossa plataforma para o cliente, nunca o contrario | TRANSCRICAO | [09:02] Marcos: |
| ADR-001-REST-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Restricao | Time pequeno veta subir componente de infraestrutura novo so por causa desta feature | TRANSCRICAO | [09:07] Diego: |
| ADR-001-REST-03 | docs/adrs/ADR-001-outbox-no-mysql.md | Restricao | Prazo estimado de tres sprints para a feature inteira, com a revisao de seguranca incluida | TRANSCRICAO | [09:46] Larissa: |
| ADR-001-TO-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Sincrono acoplaria a disponibilidade de uma operacao interna critica a disponibilidade de um endpoint de terceiro | TRANSCRICAO | [09:04] Bruno: |
| ADR-001-CONS-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Consequencia negativa apontada na analise do ADR: o throughput de eventos passa a depender do mesmo MySQL transacional que atende a API de pedidos | CODIGO | prisma/schema.prisma |
| ADR-001-CONS-04 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | A transacao de changeStatus, que ja atualiza orders, grava history e ajusta estoque, ganha mais um insert e alarga a janela de lock | CODIGO | src/modules/orders/order.service.ts |
| ADR-001-CONS-05 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisao | Nenhum componente com estado novo: mesmo MySQL, mesmo Prisma e mesma DATABASE_URL para API e worker | TRANSCRICAO | [09:30] Bruno: |
| ADR-001-CONS-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Risco | webhook_outbox cresce monotonicamente porque nenhuma politica de limpeza foi definida nesta fase | TRANSCRICAO | [09:08] Diego: |
| ADR-001-CONS-03 | docs/adrs/ADR-001-outbox-no-mysql.md | Risco | Um bug na montagem do evento passa a poder impedir a mudanca de status de um pedido, por estar dentro da transacao | TRANSCRICAO | [09:40] Bruno: |
| ADR-001-RISCO-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Risco | Gatilho de reabertura: latencia ponta a ponta estourar os 10 segundos por contencao no MySQL traz a fila externa de volta a mesa | TRANSCRICAO | [09:02] Marcos: |
| ADR-001-FE-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Fora de Escopo | Arquivamento das linhas ja entregues, na ordem de 30 dias, declarado fora do escopo desta feature | TRANSCRICAO | [09:08] Diego: |
| ADR-001-COD-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Contrato | OrderService.changeStatus abre prisma.$transaction que valida a transicao, ajusta estoque, atualiza order e grava order_status_history | CODIGO | src/modules/orders/order.service.ts |
| ADR-001-COD-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Contrato | createPrismaClient() e o singleton prisma sao o ponto de criacao do PrismaClient usado pela aplicacao | CODIGO | src/config/database.ts |

---

## docs/adrs/ADR-002-worker-separado-com-polling.md

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-002 | docs/adrs/ADR-002-worker-separado-com-polling.md | Decisao | Worker em processo Node separado da API consumindo a outbox por polling em loop a cada 2 segundos | TRANSCRICAO | [09:10] Larissa: |
| ADR-002-DEC-01 | docs/adrs/ADR-002-worker-separado-com-polling.md | Decisao | Intervalo de polling de 2 segundos, buscando a cada ciclo os eventos pendentes mais antigos, processando e marcando | TRANSCRICAO | [09:09] Diego: |
| ADR-002-DEC-02 | docs/adrs/ADR-002-worker-separado-com-polling.md | Decisao | Leitura em lote pequeno por ciclo, apenas das linhas em estado pendente, apoiada nos indices de status e created_at | TRANSCRICAO | [09:08] Diego: |
| ADR-002-DEC-03 | docs/adrs/ADR-002-worker-separado-com-polling.md | Decisao | Ordem de consumo pelo created_at da outbox, crescente | TRANSCRICAO | [09:12] Diego: |
| ADR-002-DEC-04 | docs/adrs/ADR-002-worker-separado-com-polling.md | Decisao | Worker unico: uma instancia do processo, nao N em paralelo, com ordering implicita por order_id | TRANSCRICAO | [09:12] Diego: |
| ADR-002-DEC-05 | docs/adrs/ADR-002-worker-separado-com-polling.md | Restricao | O worker tem que rodar como processo separado; se ficar dentro da API, um restart da API perde o worker | TRANSCRICAO | [09:11] Diego: |
| ADR-002-DEC-06 | docs/adrs/ADR-002-worker-separado-com-polling.md | Decisao | Entry-point propria src/worker.ts (a criar) espelhando src/server.ts, mais um script npm run worker (a criar) | TRANSCRICAO | [09:11] Larissa: |
| ADR-002-DEC-07 | docs/adrs/ADR-002-worker-separado-com-polling.md | Decisao | Mesma stack, mesmo banco e mesma DATABASE_URL, com PrismaClient proprio por ser outro processo Node | TRANSCRICAO | [09:30] Bruno: |
| ADR-002-ALT-01 | docs/adrs/ADR-002-worker-separado-com-polling.md | Alternativa | Trigger de banco como mecanismo reativo - descartada: MySQL nao tem NOTIFY/LISTEN e trigger nao notifica processo externo | TRANSCRICAO | [09:09] Diego: |
| ADR-002-ALT-02 | docs/adrs/ADR-002-worker-separado-com-polling.md | Alternativa | Worker hospedado dentro do processo da API - descartada preventivamente pelo ciclo de vida amarrado ao servidor HTTP | TRANSCRICAO | [09:11] Diego: |
| ADR-002-ALT-03 | docs/adrs/ADR-002-worker-separado-com-polling.md | Alternativa | Multiplos workers em paralelo - adiada e nao descartada; particionar por order_id ou lock pessimista fica para o futuro | TRANSCRICAO | [09:13] Diego: |
| ADR-002-RNF-01 | docs/adrs/ADR-002-worker-separado-com-polling.md | Requisito Nao Funcional | Latencia adicional de ate 2 segundos por evento aceita explicitamente como pior caso do polling | TRANSCRICAO | [09:10] Larissa: |
| ADR-002-RNF-02 | docs/adrs/ADR-002-worker-separado-com-polling.md | Requisito Nao Funcional | Produto valida os 2 segundos de polling como suficientes para o requisito de tempo real | TRANSCRICAO | [09:10] Marcos: |
| ADR-002-REST-01 | docs/adrs/ADR-002-worker-separado-com-polling.md | Restricao | Prazo alvo de fim de novembro pedido pela Atlas para a feature | TRANSCRICAO | [09:45] Marcos: |
| ADR-002-CONS-01 | docs/adrs/ADR-002-worker-separado-com-polling.md | Trade-off | Ordenacao garantida apenas por order_id e enquanto for single-worker; nao ha garantia de ordering global | TRANSCRICAO | [09:13] Larissa: |
| ADR-002-CONS-02 | docs/adrs/ADR-002-worker-separado-com-polling.md | Trade-off | Ordering global nunca foi pedido: os clientes so querem saber se cada pedido deles mudou | TRANSCRICAO | [09:14] Marcos: |
| ADR-002-CONS-05 | docs/adrs/ADR-002-worker-separado-com-polling.md | Trade-off | Analise do ADR: producao passa de um para dois processos, com custo permanente de deploy, supervisao, log, metricas e on-call | TRANSCRICAO | [09:11] Diego: |
| ADR-002-CONS-06 | docs/adrs/ADR-002-worker-separado-com-polling.md | Trade-off | Analise do ADR: um pool de conexoes MySQL a mais, porque o worker instancia PrismaClient proprio | TRANSCRICAO | [09:30] Bruno: |
| ADR-002-CONS-03 | docs/adrs/ADR-002-worker-separado-com-polling.md | Risco | Analise do ADR: consultar webhook_outbox a cada 2 segundos para sempre e carga constante e permanente no mesmo MySQL da API | TRANSCRICAO | [09:09] Diego: |
| ADR-002-CONS-04 | docs/adrs/ADR-002-worker-separado-com-polling.md | Risco | Worker unico e ponto unico de falha silencioso: nada falha visivelmente, so acumula, exigindo supervisao e alarme de fila | TRANSCRICAO | [09:11] Diego: |
| ADR-002-CONS-07 | docs/adrs/ADR-002-worker-separado-com-polling.md | Risco | Analise do ADR: o teto de throughput e o de um unico processo; pico e absorvido pela outbox, mas drenado no ritmo de um consumidor | TRANSCRICAO | [09:12] Diego: |
| ADR-002-RISCO-01 | docs/adrs/ADR-002-worker-separado-com-polling.md | Risco | Gatilho de reabertura: subir um segundo worker em paralelo derruba a garantia de ordenacao por order_id | TRANSCRICAO | [09:13] Diego: |
| ADR-002-QA-01 | docs/adrs/ADR-002-worker-separado-com-polling.md | Questao em Aberto | Rate limiting de saida ficou como observar e decidir depois, sem ADR atribuido pela reuniao | TRANSCRICAO | [09:39] Larissa: |
| ADR-002-QA-02 | docs/adrs/ADR-002-worker-separado-com-polling.md | Questao em Aberto | Risco levantado de bombardear o cliente com dezenas de chamadas quando muitos pedidos mudam de status no mesmo minuto | TRANSCRICAO | [09:38] Diego: |
| ADR-002-COD-01 | docs/adrs/ADR-002-worker-separado-com-polling.md | Contrato | src/server.ts e o molde de entry-point: bootstrap, app.listen e shutdown em SIGINT/SIGTERM com process.exit(0) | CODIGO | src/server.ts |
| ADR-002-COD-02 | docs/adrs/ADR-002-worker-separado-com-polling.md | Contrato | Nao existe script de worker hoje nos scripts do projeto; npm run worker e a criar | CODIGO | package.json |
| ADR-002-COD-03 | docs/adrs/ADR-002-worker-separado-com-polling.md | Contrato | createLogger() Pino com redact e o logger singleton sao a infraestrutura de log reusada pelo worker | CODIGO | src/shared/logger/index.ts |

---

## docs/adrs/ADR-003-retry-com-backoff-e-dlq.md

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisao | Retry com backoff exponencial de no maximo 5 tentativas por evento e dead letter queue em tabela separada | TRANSCRICAO | [09:17] Larissa: |
| ADR-003-DEC-01 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisao | Progressao de backoff fixada em 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas | TRANSCRICAO | [09:17] Diego: |
| ADR-003-DEC-02 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisao | Esgotadas as tentativas o evento vira falha permanente e vai para webhook_dead_letter com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego: |
| ADR-003-DEC-03 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Contrato | Reprocessamento manual via POST /admin/webhooks/dead-letter/:id/replay, recolocando o evento na outbox como pendente | TRANSCRICAO | [09:18] Diego: |
| ADR-003-DEC-04 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisao | Enquanto ha tentativas restantes o evento permanece na webhook_outbox, cujos estados e indices vem do ADR-001 | TRANSCRICAO | [09:08] Diego: |
| ADR-003-ALT-01 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Alternativa | 3 tentativas - descartada: mataria o evento em 30 minutos e ja houve cliente com 2h de manutencao planejada | TRANSCRICAO | [09:16] Diego: |
| ADR-003-ALT-02 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Alternativa | Retry indefinido com backoff - descartada por deixar evento pendurado para sempre se o cliente sumiu | TRANSCRICAO | [09:15] Diego: |
| ADR-003-ALT-03 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Alternativa | Marcar failed na propria outbox sem tabela separada - descartada por poluir a fila de trabalho do worker | TRANSCRICAO | [09:18] Diego: |
| ADR-003-CONS-01 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Trade-off | A DLQ exige processo humano de operacao: nada esvazia webhook_dead_letter sozinho | TRANSCRICAO | [09:18] Diego: |
| ADR-003-CONS-03 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Trade-off | Segunda tabela a manter: webhook_dead_letter entra em migration e backup e passa a precisar de retencao propria | TRANSCRICAO | [09:18] Diego: |
| ADR-003-CONS-04 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Trade-off | Cada evento de cliente instavel volta a fila de trabalho do worker unico cinco vezes antes de sair de cena | TRANSCRICAO | [09:15] Diego: |
| ADR-003-CONS-05 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Trade-off | O replay reenvia o snapshot original: evento reprocessado dias depois entrega o estado do pedido de quando o status mudou | TRANSCRICAO | [09:52] Larissa: |
| ADR-003-CONS-02 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Risco | Ninguem e avisado automaticamente quando um webhook comeca a falhar; o unico aviso e o cliente consultar o historico | TRANSCRICAO | [09:37] Larissa: |
| ADR-003-RISCO-01 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Risco | Produto aceitou o custo acreditando numa janela de quase 15 horas, numero que a emenda pendente do ADR precisa ratificar | TRANSCRICAO | [09:17] Marcos: |
| ADR-003-QA-01 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Questao em Aberto | Divergencia aritmetica apontada na analise do ADR: os cinco intervalos rendem 2h36min efetivos e o de 12h fica inalcancavel | TRANSCRICAO | [09:17] Diego: |
| ADR-003-QA-02 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Questao em Aberto | Intervalos sao fixos e iguais para todos os clientes; diferenciacao de politica por cliente nao chegou a ser discutida | TRANSCRICAO | [09:17] Diego: |
| ADR-003-QA-03 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Questao em Aberto | Nao ha reprocessamento automatico nem em lote a partir da DLQ; o replay e um evento por chamada | TRANSCRICAO | [09:18] Diego: |
| ADR-003-FE-01 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Fora de Escopo | Alerta por e-mail ao cliente apos falhas consecutivas pedido pelo PM e adiado para a proxima fase | TRANSCRICAO | [09:37] Marcos: |
| ADR-003-FE-02 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Fora de Escopo | Timeout do HTTP call fica fora deste ADR, tratado como parametro nao funcional que vive no FDD | TRANSCRICAO | [09:42] Diego: |
| ADR-003-FE-03 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Fora de Escopo | A exigencia de role ADMIN no endpoint de replay e decisao de controle de acesso e pertence ao ADR-008 | TRANSCRICAO | [09:36] Sofia: |
| ADR-003-COD-01 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Restricao | Nao ha broker nem biblioteca de fila ou retry nas dependencias de producao; toda a politica de reentrega e codigo proprio | CODIGO | package.json |
| ADR-003-COD-02 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Contrato | Precedente de tabela dedicada a historico: model OrderStatusHistory mapeado para order_status_history | CODIGO | prisma/schema.prisma |

---

## docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisao | Assinar o corpo de cada webhook com HMAC-SHA256, com secret por endpoint e rotacao com grace period de 24h | TRANSCRICAO | [09:22] Sofia: |
| ADR-004-DEC-02 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisao | Algoritmo SHA-256 escolhido por ser padrao de mercado com biblioteca disponivel em qualquer stack de cliente | TRANSCRICAO | [09:20] Sofia: |
| ADR-004-DEC-03 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisao | Secret unica por endpoint de webhook cadastrado, nunca global da plataforma | TRANSCRICAO | [09:21] Sofia: |
| ADR-004-DEC-04 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisao | Na rotacao a secret antiga continua valida em paralelo por 24 horas e depois e descartada | TRANSCRICAO | [09:21] Sofia: |
| ADR-004-RF-01 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Requisito Funcional | A secret e gerada por nos e devolvida ao cliente na resposta de criacao do webhook | TRANSCRICAO | [09:31] Marcos: |
| ADR-004-RF-02 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Requisito Funcional | Existe endpoint para o cliente pedir nova secret pela API, em auto-servico | TRANSCRICAO | [09:21] Sofia: |
| ADR-004-ALT-01 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Alternativa | Secret global da plataforma - descartada porque se vaza uma, vaza tudo, e o raio de explosao seria todos os clientes | TRANSCRICAO | [09:21] Sofia: |
| ADR-004-ALT-02 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Alternativa | Nenhuma alternativa de algoritmo foi levantada; SHA-256 foi consenso imediato apos a pergunta sobre qual algoritmo usar | TRANSCRICAO | [09:20] Bruno: |
| ADR-004-ALT-03 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Alternativa | Nenhuma alternativa ao grace period foi proposta; corte imediato ou janela mais longa nunca entraram em pauta | TRANSCRICAO | [09:22] Sofia: |
| ADR-004-CONTRATO-01 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Contrato | Tabela de configuracao de webhook guarda url, secret, customer_id e estado ativo | TRANSCRICAO | [09:21] Bruno: |
| ADR-004-CONTRATO-02 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Contrato | X-Timestamp incluido no conjunto de headers para o cliente detectar replay attack se quiser | TRANSCRICAO | [09:44] Diego: |
| ADR-004-DEC-01 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Contrato | Assinatura enviada no header X-Signature, recalculada e comparada pelo cliente antes de confiar no evento | TRANSCRICAO | [09:20] Sofia: |
| ADR-004-RNF-01 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Requisito Nao Funcional | TLS obrigatorio: a URL do webhook tem que ser https e http e recusado por validacao no schema Zod | TRANSCRICAO | [09:23] Sofia: |
| ADR-004-RNF-02 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Requisito Nao Funcional | Limite de 64KB de payload com erro caso ultrapasse, tratado como requisito nao funcional e nao como decisao arquitetural | TRANSCRICAO | [09:24] Larissa: |
| ADR-004-REST-01 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Restricao | Deploy condicionado a pelo menos dois dias uteis de revisao de seguranca de HMAC e geracao de secret | TRANSCRICAO | [09:46] Sofia: |
| ADR-004-CONS-01 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Trade-off | Durante as 24h duas secrets sao validas; se a rotacao foi por vazamento, a comprometida segue aceita por ate um dia | TRANSCRICAO | [09:21] Sofia: |
| ADR-004-CONS-05 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Trade-off | O HMAC cobre o corpo e nao a janela temporal; a protecao contra replay e opcional e fica do lado do cliente | TRANSCRICAO | [09:44] Diego: |
| ADR-004-RISCO-01 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Risco | Caso real citado na reuniao: cliente ja vazou secret em log de aplicacao do lado dele | TRANSCRICAO | [09:22] Diego: |
| ADR-004-CONS-04 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Risco | A verificacao da assinatura e opcional para o cliente e o onus fica na documentacao do portal do desenvolvedor | TRANSCRICAO | [09:26] Marcos: |
| ADR-004-CONS-02 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Risco | Analise do ADR sobre o codigo: passamos a armazenar material secreto recuperavel por linha de tabela, diferente do passwordHash que ninguem precisa recuperar | CODIGO | prisma/schema.prisma |
| ADR-004-CONS-03 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Risco | Analise do ADR sobre o codigo: a lista de redact do Pino nao cobre nenhum caminho de secret de webhook, permitindo vazamento em log do nosso lado | CODIGO | src/shared/logger/index.ts |
| ADR-004-QA-01 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Questao em Aberto | Qual secret assina durante o grace period - a nova, a antiga ou ambas em headers distintos - nao foi decidido | TRANSCRICAO | [09:21] Sofia: |
| ADR-004-QA-02 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Questao em Aberto | Como a secret e armazenada em repouso - cifrada, em cofre externo ou texto puro - nao foi discutido e cai na revisao de seguranca | TRANSCRICAO | [09:46] Sofia: |
| ADR-004-FE-01 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Fora de Escopo | Quem pode cadastrar e alterar endpoints, e portanto ver secret de qual cliente, e decisao separada registrada no ADR-008 | TRANSCRICAO | [09:36] Sofia: |
| ADR-004-COD-01 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Restricao | JWT_SECRET validado por Zod e o precedente de segredo unico e global, exatamente o modelo descartado por este ADR | CODIGO | src/config/env.ts |
| ADR-004-COD-02 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Restricao | Nao ha nenhum uso de HMAC em src/; as unicas dependencias criptograficas em uso sao bcrypt e jsonwebtoken | CODIGO | src/modules/auth/auth.service.ts |
| ADR-004-COD-03 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Restricao | A autenticacao existente cobre apenas requisicoes de entrada por JWT e nao serve para provar a nossa identidade na saida | CODIGO | src/middlewares/auth.middleware.ts |

---

## docs/adrs/ADR-005-at-least-once-com-x-event-id.md

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-005 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisao | Garantia de entrega at-least-once com a deduplicacao delegada ao cliente | TRANSCRICAO | [09:26] Larissa: |
| ADR-005-DEC-01 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisao | O mesmo UUID viaja em todas as retentativas do mesmo evento, o que torna a dedup do lado do cliente possivel | TRANSCRICAO | [09:25] Diego: |
| ADR-005-DEC-03 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisao | Precedente de mercado invocado como justificativa: Stripe e GitHub fazem assim e at-least-once resolve 99% dos casos | TRANSCRICAO | [09:25] Diego: |
| ADR-005-CONS-05 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisao | O evento e notificacao de estado e nao comando; o cliente pode sempre buscar a verdade em GET /orders/:id | TRANSCRICAO | [09:43] Diego: |
| ADR-005-ALT-01 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Alternativa | Exactly-once com coordenacao bilateral - descartada por exigir coordenacao dos dois lados e ficar muito mais complexo | TRANSCRICAO | [09:25] Diego: |
| ADR-005-ALT-02 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Alternativa | Deduplicacao do nosso lado - nunca formulada como proposta tecnica; existe apenas como objecao registrada | TRANSCRICAO | [09:25] Sofia: |
| ADR-005-CONTRATO-01 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Contrato | Header X-Event-Id com UUID gerado quando o evento entra na outbox, unico por evento e nao por pedido | TRANSCRICAO | [09:25] Diego: |
| ADR-005-DEC-02 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Contrato | O cliente guarda os X-Event-Id ja processados e descarta repeticao; nao ha verificacao do nosso lado | TRANSCRICAO | [09:25] Diego: |
| ADR-005-RF-01 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Requisito Funcional | Comportamento de duplicidade documentado em destaque no portal do desenvolvedor, compromisso assumido pelo PM | TRANSCRICAO | [09:26] Marcos: |
| ADR-005-RNF-01 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Requisito Nao Funcional | Timeout de 10 segundos no envio; cliente que nao responde nesse prazo e tratado como falha e marcado para retry | TRANSCRICAO | [09:42] Diego: |
| ADR-005-TO-01 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Trade-off | Objecao registrada na hora: a garantia at-least-once joga responsabilidade de dedup para o cliente | TRANSCRICAO | [09:25] Sofia: |
| ADR-005-CONS-01 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Trade-off | Zero estado de deduplicacao do nosso lado: nenhuma tabela de ids processados, janela de retencao ou lock | TRANSCRICAO | [09:25] Diego: |
| ADR-005-CONS-04 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Trade-off | A decisao amarra o contrato publico: prometer exactly-once depois mudaria o contrato dos tres clientes ja integrados | TRANSCRICAO | [09:00] Marcos: |
| ADR-005-CONS-02 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Risco | Nao temos como saber se o cliente deduplica; a corretude da ponta final e presumida, sem metrica nem alerta possivel | TRANSCRICAO | [09:25] Sofia: |
| ADR-005-CONS-03 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Risco | Suporte recebera chamados de webhook duplicado; depende da documentacao do portal existir antes do primeiro cliente em producao | TRANSCRICAO | [09:26] Marcos: |
| ADR-005-QA-01 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Questao em Aberto | A reuniao definiu que o replay recoloca o evento na outbox, mas nao discutiu se o X-Event-Id original e preservado | TRANSCRICAO | [09:18] Diego: |
| ADR-005-CONS-06 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Metrica | Duplicidade nao e falha operacional: nenhum alerta, dashboard ou metrica deve tratar reenvio como incidente | TRANSCRICAO | [09:24] Diego: |
| ADR-005-COD-01 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Contrato | Padrao de identificador das entidades de dominio no schema e @id @default(uuid()) @db.Char(36), formato que o X-Event-Id segue | CODIGO | prisma/schema.prisma |
| ADR-005-COD-02 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Contrato | changeStatus e a transacao onde o evento e portanto o UUID do X-Event-Id passam a ser gerados | CODIGO | src/modules/orders/order.service.ts |

---

## docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisao | Reuso maximo dos padroes existentes, sem introduzir stack, biblioteca ou convencao nova no modulo de webhooks | TRANSCRICAO | [09:30] Larissa: |
| ADR-006-DEC-01 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisao | Modulo em src/modules/webhooks (a criar) com controller, service, repository, routes e schemas espelhando os modulos existentes | TRANSCRICAO | [09:27] Bruno: |
| ADR-006-DEC-02 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisao | Logica de processamento em arquivo dentro do proprio modulo e entry-point separada em src/worker.ts (a criar) | TRANSCRICAO | [09:28] Bruno: |
| ADR-006-DEC-03 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisao | Erros do modulo herdam de AppError seguindo o molde de InsufficientStockError e InvalidStatusTransitionError | TRANSCRICAO | [09:28] Bruno: |
| ADR-006-DEC-04 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisao | Prefixo literal WEBHOOK_ em todos os codigos de erro do modulo | TRANSCRICAO | [09:29] Larissa: |
| ADR-006-DEC-05 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisao | errorMiddleware reaproveitado sem alteracao porque ja trata AppError, Zod e erros conhecidos do Prisma | TRANSCRICAO | [09:29] Bruno: |
| ADR-006-DEC-06 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisao | Log continua sendo Pino, ja presente no projeto inteiro, sem nenhuma biblioteca de log nova | TRANSCRICAO | [09:29] Bruno: |
| ADR-006-DEC-07 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisao | Tabelas novas do modulo usam UUID como chave primaria, seguindo o padrao do resto do projeto | TRANSCRICAO | [09:51] Larissa: |
| ADR-006-DEC-08 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Decisao | O worker instancia PrismaClient proprio, contra o mesmo banco e a mesma DATABASE_URL | TRANSCRICAO | [09:30] Bruno: |
| ADR-006-ALT-01 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Alternativa | id auto incremental nas tabelas do modulo - descartada pela consistencia do schema, onde tudo e uuid | TRANSCRICAO | [09:51] Diego: |
| ADR-006-ALT-02 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Alternativa | Modulo autonomo com stack propria ou biblioteca de terceiros - nunca proposta; o reuso foi consenso imediato | TRANSCRICAO | [09:28] Diego: |
| ADR-006-CONTRATO-01 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Contrato | Codigos de erro previstos: WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL e WEBHOOK_SECRET_REQUIRED, entre outros do modulo | TRANSCRICAO | [09:28] Bruno: |
| ADR-006-CONS-04 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Trade-off | Analise do ADR: chave Char(36) nao sequencial em tabela de escrita quente rende indice mais largo e insercao menos localizada | TRANSCRICAO | [09:51] Larissa: |
| ADR-006-CONS-01 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Trade-off | AppError exige statusCode obrigatorio; falha de entrega no worker nao tem status HTTP que alguem va ler | CODIGO | src/shared/errors/app-error.ts |
| ADR-006-CONS-02 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Risco | errorMiddleware so existe no processo HTTP; o worker precisa de tratamento de erro proprio sobre o Pino | CODIGO | src/middlewares/error.middleware.ts |
| ADR-006-CONS-03 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Risco | redactPaths do Pino nao inclui secret; reaproveitar sem nada novo deixa a secret por endpoint fora da lista de censura | CODIGO | src/shared/logger/index.ts |
| ADR-006-CONS-05 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Risco | Wiring manual: registrar webhooks obriga editar app.ts e routes/index.ts, com conflito de merge durante tres sprints | CODIGO | src/app.ts |
| ADR-006-QA-01 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Questao em Aberto | Nome do arquivo de processamento ficou aberto entre webhook.worker.ts e webhook.processor.ts, ambos a criar | TRANSCRICAO | [09:28] Bruno: |
| ADR-006-QA-02 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Questao em Aberto | Gatilho de reabertura: quando a secret circular dentro de objeto entregue ao logger, redactPaths precisa ganhar entrada | TRANSCRICAO | [09:22] Diego: |
| ADR-006-QA-03 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Questao em Aberto | Duas instancias de PrismaClient contra o mesmo MySQL; saturacao de conexoes obriga configurar limite de pool explicito | CODIGO | src/config/database.ts |
| ADR-006-COD-01 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Contrato | authenticate e requireRole(...roles) com papeis ADMIN e OPERATOR ja existem e sao usados como estao | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-006-COD-02 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Contrato | validate({ body, query, params }) e a porta de entrada dos schemas Zod e converte ZodError em ValidationError | CODIGO | src/middlewares/validate.middleware.ts |
| ADR-006-COD-03 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Restricao | Nenhuma dependencia nova: pino, zod, uuid e @prisma/client ja estao declarados no projeto | CODIGO | package.json |
| ADR-006-COD-04 | docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md | Contrato | src/modules/orders e o molde do modulo: controller, service, repository, routes e schemas com os mesmos nomes | CODIGO | src/modules/orders/order.routes.ts |

---

## docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-007 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Decisao | A linha da webhook_outbox guarda o payload JSON ja renderizado - snapshot do pedido no instante da mudanca de status | TRANSCRICAO | [09:52] Larissa: |
| ADR-007-DEC-01 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Decisao | Snapshot na insercao endossado por Diego e fechado por Bruno nos minutos finais da call | TRANSCRICAO | [09:52] Diego: |
| ADR-007-DEC-02 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Decisao | O worker le a linha, assina e envia, sem consultar a order para montar o corpo do request | TRANSCRICAO | [09:52] Larissa: |
| ADR-007-DEC-03 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Decisao | Snapshot produzido dentro da mesma transacao pela funcao publishWebhookEvent que recebe o tx em curso | TRANSCRICAO | [09:41] Bruno: |
| ADR-007-DEC-04 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Decisao | Payload enxuto sem items para nao inflar a linha; o cliente busca detalhe em GET /orders/:id se quiser | TRANSCRICAO | [09:43] Diego: |
| ADR-007-DEC-05 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Decisao | Filtragem por status de interesse acontece na insercao: se nenhum webhook do customer quer aquele status, nem insere | TRANSCRICAO | [09:34] Bruno: |
| ADR-007-CONTRATO-01 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Contrato | Payload JSON com event_id, event_type order.status_changed, timestamp ISO 8601, order_id, order_number, from_status, to_status, customer_id e total_cents | TRANSCRICAO | [09:43] Diego: |
| ADR-007-RF-01 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Requisito Funcional | Cada endpoint cadastra a lista de status que quer ouvir, filtrada na hora de inserir na outbox | TRANSCRICAO | [09:33] Marcos: |
| ADR-007-RF-02 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Requisito Funcional | Historico de entregas com sucesso/falha, payload, response e tempo de resposta em GET /webhooks/:id/deliveries | TRANSCRICAO | [09:34] Marcos: |
| ADR-007-ALT-01 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Alternativa | Guardar so order_id e renderizar no envio - descartada pela fidelidade temporal do evento ao momento da transicao | TRANSCRICAO | [09:52] Larissa: |
| ADR-007-ALT-02 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Alternativa | Filtrar no momento do envio - descartada pelo custo de gravar linha que ninguem vai consumir | TRANSCRICAO | [09:34] Bruno: |
| ADR-007-CONS-05 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Trade-off | Payload enxuto fica confortavelmente abaixo do teto de tamanho, que a reuniao tratou como requisito nao funcional | TRANSCRICAO | [09:24] Larissa: |
| ADR-007-CONS-01 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Trade-off | Duplicamos dado: campos de orders replicados na outbox, na dead letter e no historico de deliveries | CODIGO | prisma/schema.prisma |
| ADR-007-CONS-03 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Trade-off | A transacao de mudanca de status ganha leitura da configuracao de webhooks, serializacao do JSON e insercao na outbox | CODIGO | src/modules/orders/order.service.ts |
| ADR-007-CONS-02 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Risco | Evento ja enfileirado e imutavel na pratica: erro de renderizacao nao se corrige com deploy | TRANSCRICAO | [09:52] Larissa: |
| ADR-007-CONS-04 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Risco | Filtrar na origem e irreversivel para o passado: webhook cadastrado depois nao recebe nada do que ja aconteceu | TRANSCRICAO | [09:34] Bruno: |
| ADR-007-CONS-06 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Risco | O snapshot pressiona a politica de arquivamento de linhas entregues, declarada fora do escopo da feature | TRANSCRICAO | [09:08] Diego: |
| ADR-007-RISCO-01 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Risco | Preocupacao original registrada: acumulo de eventos na tabela pode deixar o worker lento | TRANSCRICAO | [09:07] Bruno: |
| ADR-007-QA-01 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Questao em Aberto | Pergunta que abriu o ponto: a outbox guarda o payload renderizado ou so order_id renderizado na hora do envio | TRANSCRICAO | [09:51] Bruno: |
| ADR-007-QA-02 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Questao em Aberto | Pergunta que abriu a filtragem: filtra na insercao do outbox ou na hora de mandar | TRANSCRICAO | [09:34] Diego: |
| ADR-007-QA-03 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Questao em Aberto | Nao ha versionamento de payload na outbox; o gatilho e a primeira mudanca incompativel de formato com a fila nao vazia | TRANSCRICAO | [09:43] Diego: |
| ADR-007-COD-01 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Contrato | PATCH /:id/status permite o pedido continuar mudando depois do evento gerado, o que motiva o snapshot | CODIGO | src/modules/orders/order.routes.ts |

---

## docs/adrs/ADR-008-controle-de-acesso-dos-endpoints.md

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-008 | docs/adrs/ADR-008-controle-de-acesso-dos-endpoints.md | Decisao | Replay de DLQ exige role ADMIN e o CRUD de webhooks exige apenas autenticacao, reaproveitando o requireRole existente | TRANSCRICAO | [09:36] Larissa: |
| ADR-008-DEC-01 | docs/adrs/ADR-008-controle-de-acesso-dos-endpoints.md | Decisao | A verificacao usa requireRole encadeado depois do authenticate, sem middleware de autorizacao novo | TRANSCRICAO | [09:36] Larissa: |
| ADR-008-DEC-03 | docs/adrs/ADR-008-controle-de-acesso-dos-endpoints.md | Decisao | CRUD de configuracao e consulta de deliveries exigem apenas autenticacao, sem restricao de papel, por enquanto | TRANSCRICAO | [09:37] Sofia: |
| ADR-008-DEC-02 | docs/adrs/ADR-008-controle-de-acesso-dos-endpoints.md | Requisito Funcional | O endpoint de replay tem que registrar quem executou a operacao, para auditoria | TRANSCRICAO | [09:36] Sofia: |
| ADR-008-RF-01 | docs/adrs/ADR-008-controle-de-acesso-dos-endpoints.md | Requisito Funcional | Cadastro de webhook por POST com url, lista de status de interesse e secret gerada por nos e devolvida na criacao | TRANSCRICAO | [09:31] Marcos: |
| ADR-008-RF-02 | docs/adrs/ADR-008-controle-de-acesso-dos-endpoints.md | Requisito Funcional | PATCH para editar, DELETE para remover e GET para listar os webhooks de um customer | TRANSCRICAO | [09:33] Bruno: |
| ADR-008-ALT-01 | docs/adrs/ADR-008-controle-de-acesso-dos-endpoints.md | Alternativa | Exigir ADMIN em todo o modulo - descartada por ora; aparece apenas por negacao na pergunta sobre o resto do CRUD | TRANSCRICAO | [09:36] Marcos: |
| ADR-008-ALT-02 | docs/adrs/ADR-008-controle-de-acesso-dos-endpoints.md | Alternativa | Replay acessivel a qualquer papel autenticado - descartada: mexer em fila de entrega nao e coisa de operador | TRANSCRICAO | [09:36] Sofia: |
| ADR-008-ALT-03 | docs/adrs/ADR-008-controle-de-acesso-dos-endpoints.md | Alternativa | Nenhuma alternativa de mecanismo - escopos por token, ACL por customer_id ou API key - chegou a ser cogitada | TRANSCRICAO | [09:36] Larissa: |
| ADR-008-CONTRATO-01 | docs/adrs/ADR-008-controle-de-acesso-dos-endpoints.md | Contrato | POST /admin/webhooks/dead-letter/:id/replay confirmado como a rota do replay administrativo | TRANSCRICAO | [09:35] Diego: |
| ADR-008-DEC-04 | docs/adrs/ADR-008-controle-de-acesso-dos-endpoints.md | Restricao | O customer_id vai no body ou no path e nao vem do JWT | TRANSCRICAO | [09:32] Larissa: |
| ADR-008-REST-01 | docs/adrs/ADR-008-controle-de-acesso-dos-endpoints.md | Restricao | O JWT atual e do usuario operador e nao do cliente, o que impede derivar o customer_id do token | TRANSCRICAO | [09:32] Bruno: |
| ADR-008-REST-02 | docs/adrs/ADR-008-controle-de-acesso-dos-endpoints.md | Restricao | O cliente cadastra pela nossa API direto, com JWT do nosso sistema, via usuarios que representam o cliente | TRANSCRICAO | [09:32] Marcos: |
| ADR-008-CONS-04 | docs/adrs/ADR-008-controle-de-acesso-dos-endpoints.md | Restricao | Modelo de papeis binario ADMIN ou OPERATOR limita o endurecimento futuro: nao existe papel intermediario | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-008-CONS-01 | docs/adrs/ADR-008-controle-de-acesso-dos-endpoints.md | Risco | Qualquer usuario autenticado pode criar, alterar ou apagar o webhook de qualquer cliente, viabilizando vazamento entre clientes | TRANSCRICAO | [09:32] Larissa: |
| ADR-008-CONS-02 | docs/adrs/ADR-008-controle-de-acesso-dos-endpoints.md | Risco | Quem cria webhook recebe na resposta uma secret valida, inclusive apontando para customer_id de terceiro | TRANSCRICAO | [09:31] Marcos: |
| ADR-008-CONS-03 | docs/adrs/ADR-008-controle-de-acesso-dos-endpoints.md | Risco | A auditoria do replay nasce generica: log Pino de requisicao com userId, sem retencao garantida nem imutabilidade | CODIGO | src/middlewares/request-logger.middleware.ts |
| ADR-008-RISCO-01 | docs/adrs/ADR-008-controle-de-acesso-dos-endpoints.md | Risco | A suite atual nao exercita requireRole e nao ha nenhuma asercao de 403 no repositorio; o caminho de negativa entra sem cobertura | CODIGO | tests/orders.test.ts |
| ADR-008-QA-01 | docs/adrs/ADR-008-controle-de-acesso-dos-endpoints.md | Questao em Aberto | Endurecimento do RBAC no CRUD fica declarado como mais pra frente a gente pode endurecer, sem gatilho definido na reuniao | TRANSCRICAO | [09:37] Sofia: |
| ADR-008-QA-02 | docs/adrs/ADR-008-controle-de-acesso-dos-endpoints.md | Questao em Aberto | A reuniao nao definiu se o PATCH de webhook devolve secret na resposta | TRANSCRICAO | [09:31] Marcos: |
| ADR-008-QA-03 | docs/adrs/ADR-008-controle-de-acesso-dos-endpoints.md | Questao em Aberto | O escopo declarado da revisao de seguranca foi HMAC e geracao de secret; controle de acesso nao foi mencionado | TRANSCRICAO | [09:46] Sofia: |
| ADR-008-QA-04 | docs/adrs/ADR-008-controle-de-acesso-dos-endpoints.md | Questao em Aberto | O prefixo admin seria o primeiro do projeto; a convencao de montagem fica como detalhe de implementacao para o FDD | CODIGO | src/routes/index.ts |
| ADR-008-COD-01 | docs/adrs/ADR-008-controle-de-acesso-dos-endpoints.md | Contrato | Precedente real de requireRole('ADMIN') encadeado apos authenticate no modulo de usuarios | CODIGO | src/modules/users/user.routes.ts |
| ADR-008-COD-02 | docs/adrs/ADR-008-controle-de-acesso-dos-endpoints.md | Contrato | Precedente de router.use(authenticate) sem restricao de papel, molde adotado pelo CRUD de webhooks | CODIGO | src/modules/orders/order.routes.ts |
| ADR-008-COD-03 | docs/adrs/ADR-008-controle-de-acesso-dos-endpoints.md | Contrato | ForbiddenError mapeia para 403 com codigo FORBIDDEN, ja formatado pelo middleware de erro central | CODIGO | src/shared/errors/http-errors.ts |

---

## Deliberadamente fora do tracker

Itens que existem no pacote ou nas varreduras e que **não** viraram linha aqui, com o motivo:

- **Duplicatas entre varreduras (30 linhas removidas).** Cinco varreduras paralelas produziram o mesmo item com IDs diferentes. Quando duas linhas afirmavam o mesmo fato, no mesmo documento, com a mesma âncora, ficou a de conteúdo mais completo. Exemplos: o prefixo `WEBHOOK_` nos códigos de erro apareceu como decisão e como requisito não funcional (ficou `FDD-RNF-10`); a progressão de backoff apareceu isolada e recombinada com o teto de cinco tentativas (ficaram `FDD-DEC-05` e `FDD-DEC-06`); o worker morto apareceu duas vezes como risco (ficou `FDD-RISCO-53`); quatro códigos de erro apareceram na família `CONTRATO` e na `ERRO` (ficou a família `ERRO`, que cobre a matriz inteira).
- **Famílias de ID normalizadas.** `FDD-RESTRICAO-*` passou a `FDD-RESTR-*` e `FDD-TRADEOFF-*` passou a `FDD-TRADE-*`, para que cada tipo tenha um único prefixo de família no documento. Não houve colisão de número na renumeração; os IDs de destino estavam livres.
- **Repetições legítimas entre documentos.** O mesmo fato registrado no PRD, no RFC, no FDD e num ADR **não** é duplicata: cada documento o registra sob um recorte diferente (produto, arquitetura, implementação, decisão isolada). Essas linhas permanecem, uma por documento — é justamente isso que o tracker precisa mostrar quando um deles mudar.
- **Falas de coordenação da reunião.** Cumprimentos, entrada e saída de participantes, o resumo final de `[09:48] Larissa:` e os "ok" de `[09:49]` não produzem item rastreável. O resumo, em particular, apenas recapitula decisões que já têm linha própria com a âncora do momento em que foram tomadas.
- **Nomes de tabela ainda inexistentes.** `webhook_outbox`, `webhook_endpoints`, `webhook_deliveries` e `webhook_dead_letter` aparecem em células de conteúdo, mas nunca na coluna Localização: não são caminhos de arquivo e nenhuma delas existe hoje em `prisma/schema.prisma`. Só entram como localização depois que a migration for criada.
- **Estatísticas de negócio não apresentadas.** Volume de chamadas dos três clientes, custo da integração atual e taxa de falha histórica de endpoint de cliente não aparecem no tracker porque não foram apresentados na reunião nem existem no código — ficaram registrados como questões em aberto (`PRD-QA-06`) em vez de virarem número inventado.
