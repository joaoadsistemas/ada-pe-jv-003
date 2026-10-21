# Template de System Design: Aula 04 - Microsserviços & Padrão Saga

> **Contexto do Cenário: O Pedido Pago Sem Produto na Black Friday**
>
> Sua equipe está arquitetando a transação distribuída de compra na Black Friday para o marketplace:
>
> 1. **O Incidente:** Um cliente comprou uma Smart TV de R$ 3.000 via Pix. O gateway de pagamentos debitou o valor imediatamente e confirmou a transação. No milissegundo seguinte, o serviço de Estoque esgotou as unidades e rejeitou a reserva. O dinheiro sumiu da conta do cliente, o produto não existe para entrega e o pedido travou em um estado inconsistente.
> 2. **O Desafio Distribuído:** Não existe transação única (`@Transactional`) entre bancos de dados autônomos de Checkout, Pagamento e Estoque.
> 3. **A Carga:** 5.000 TPS de checkout simultâneo e milhões de clientes consultando o status da compra em "Meus Pedidos".
>
> **Instruções:** Responda às decisões arquiteturais abaixo fundamentando suas escolhas estritamente com base nos requisitos e restrições deste cenário. Documente suas decisões no repositório contínuo do seu projeto através de commits atômicos e bem descritivos.

---

## 1. Transações Distribuídas: Coordenação e Trade-offs

### 1.1 Volumetria e Pragmatismo: 2PC vs Saga

- **Análise Crítica:** Em quais cenários arquiteturais uma transação ACID local ou Two-Phase Commit (2PC) resolveria o problema com menor custo operacional? Por que, para o cenário de 5.000 TPS na Black Friday com serviços e bancos autônomos, o 2PC é inviável e a consistência eventual se torna necessária?
  > Uma transação ACID local (ex: via `@Transactional` do Spring) resolve o problema com custo operacional mínimo em arquiteturas monolíticas que compartilham a mesma base de dados. Já o protocolo Two-Phase Commit (2PC) pode ser aplicado em sistemas distribuídos de baixíssima volumetria, onde a sobrecarga de bloqueio central é tolerável.
  >Para o cenário de 5.000 TPS na Black Friday com bancos autônomos, o 2PC torna-se totalmente inviável devido aos seguintes fatores:
  >1. Contenção de Locks Distribuídos: O 2PC exige que todos os bancos participantes mantenham bloqueios (locks) de gravação abertos durante a fase de *Prepare* até a confirmação do *Commit*. A 5.000 TPS, isso causa exaustão do pool de conexões, latência catastrófica e deadlocks massivos.
  >2. Degradação da Disponibilidade: No 2PC, se um único participante oscilar ou demorar para responder, toda a transação distribuída trava. 
  >A Consistência Eventual via Padrão Saga isola cada operação em uma transação local rápida e independente em cada microsserviço, utilizando o Transactional Outbox Pattern para garantir a comunicação assíncrona confiável através de mensageria sem travar recursos de banco de dados.

### 1.2 O Modelo de Coordenação da Saga

- **A Escolha:** ( ) Coreografia (Orientada a Eventos)  (x) Orquestração (Process Manager / Máquina de Estados)
- **Justificativa e Mitigação das Desvantagens do seu Modelo:**
  - *Se você escolheu Coreografia:* Como seu design resolve a falta de visibilidade centralizada (saber em tempo real onde o pedido está) e previne dependências cíclicas à medida que novos serviços entrarem no fluxo de compra?
  - *Se você escolheu Orquestração:* Qual componente atua como Orquestrador, como ele se comunica com os participantes (síncrono ou assíncrono) e como você evita que ele se transforme em um "serviço Deus", tirando a inteligência e a autonomia dos outros microsserviços?
  > Escolhi o modelo de Orquestração devido à alta complexidade do fluxo transacional do e-commerce (Checkout, Pagamento, Reserva de Estoque, Entrega e NFe) e à necessidade de rastreabilidade explícita do estado do pedido em tempo real.
  >Detalhamento do Design:
  >1. Componente Orquestrador: O próprio `Checkout-Service` atuará como o Orquestrador (Process Manager), gerenciando a máquina de estados local do pedido.
  >2. Comunicação Assíncrona: O orquestrador se comunica com os participantes de forma estritamente assíncrona orientada a eventos através de mensageria (Kafka/RabbitMQ), garantindo baixo acoplamento e resiliência a picos de tráfego.
  >3. Prevenção do "Serviço Deus" (Princípios de DDD): O orquestrador não contém regras de negócio dos domínios participantes. Ele apenas orquestra o fluxo de comandos (`ReservarEstoqueCommand`, `ProcessarPagamentoCommand`) e escuta os eventos de resposta (`EstoqueReservadoEvent`, `EstoqueEsgotadoEvent`). As regras de validação de saldo, cálculo de taxas de pagamento ou alocação física de estoque permanecem 100% isoladas na camada de domínio (`Domain Core`) de cada microsserviço especialista.

---

## 2. Máquina de Estados, Compensação e Confiabilidade

### 2.1 Máquina de Estados e Fluxos Ponta a Ponta

- **Estados do Pedido:** Defina a máquina de estados explícita do ciclo de vida da transação no seu modelo (incluindo os estados iniciais, intermediários, de compensação e finais).
  > - Iniciais: `PEDIDO_CRIADO`
  > - Intermediários: `AGUARDANDO_PAGAMENTO`, `PAGAMENTO_APROVADO`, `RESERVANDO_ESTOQUE`
  > - Compensação: `ESTOQUE_INDISPONIVEL`, `SOLICITANDO_ESTORNO`
  > - Finais: `PEDIDO_CONCLUIDO` (Sucesso), `PEDIDO_CANCELADO_COM_ESTORNO` (Falha compensada)
- **Detalhamento dos Fluxos:**
  - **Caminho Feliz:** Descreva o fluxo de ponta a ponta da transação concluída com sucesso entre Checkout, Pagamento e Estoque.
  - **Caminho de Falha & Compensação:** Descreva o passo a passo a partir da falha na reserva de estoque após o pagamento aprovado. Como a ação compensatória (estorno) é disparada e executada sem intervenção humana?
  > - Caminho Feliz:
   > 1. Checkout cria pedido (`PEDIDO_CRIADO`) e publica comando de pagamento.
    >2. Pagamento processa e publica `PAGAMENTO_APROVADO`.
    >3. Orquestrador avança para `RESERVANDO_ESTOQUE` e envia comando de reserva ao Estoque.
    >4. Estoque confirma reserva de itens e publica `ESTOQUE_RESERVADO`.
    >5. Orquestrador altera o status final para `PEDIDO_CONCLUIDO`.

  >- Caminho de Falha & Compensação (Incidente da Smart TV):
   > 1. Pagamento aprovado (`PAGAMENTO_APROVADO`).
    >2. Orquestrador solicita reserva de estoque, mas o serviço de Estoque responde com `EstoqueEsgotadoEvent`.
    >3. O Orquestrador intercepta a falha e altera o estado do pedido para `ESTOQUE_INDISPONIVEL`.
    >4. O Orquestrador dispara a Ação Compensatória: publica `SolicitarEstornoCommand` para o serviço de Pagamento contendo o ID da transação e a Idempotency-Key original.
    >5. O serviço de Pagamento realiza o estorno do Pix via API do gateway/banco e confirma com `PagamentoEstornadoEvent`.
    >6. O Orquestrador finaliza o ciclo transicionando o pedido para `PEDIDO_CANCELADO_COM_ESTORNO`, sem necessidade de intervenção humana.

### 2.2 Resiliência da Compensação e Consistência

- **E se a compensação falhar ou oscilar?** A chamada ou evento de estorno pode sofrer timeout ou instabilidade transitória. Como seu design garante que o cliente não fique no prejuízo se a tentativa de compensação falhar ou for reprocessada?
  > As mensagens de compensação serão reprocessadas automaticamente adotando políticas de Retry com Backoff Exponencial. Para evitar duplicidade de estorno no parceiro, todas as chamadas de compensação utilizarão obrigatoriamente a Idempotency-Key original da transação. Caso ocorra uma falha persistente e ininterrupta (ex: queda total da API do banco/gateway), a transação é enviada para uma Dead Letter Queue (DLQ) de Compensação com alertas em tempo real para auditoria e conciliação financeira, garantindo que o cliente não fique no prejuízo.
- **Atomicidade entre Estado e Mensageria (Dual Write):** Se a sua arquitetura envolve publicação de eventos ou mensagens entre passos, como você garante que a persistência no banco local e o despacho da mensagem ocorram sem risco de perda silenciosa se a rede ou o broker falharem entre as duas operações?
  > Para eliminar o risco de Dual Write (onde o estado é alterado no banco local, mas a mensagem falha ao enviar para o broker), utilizaremos o **Transactional Outbox Pattern**.
  A atualização da máquina de estados do pedido e a inserção do evento correspondente na tabela `outbox` ocorrem na **mesma transação ACID local do banco de dados relacional** (`@Transactional`). Um conector de CDC (Change Data Capture via Debezium) lê o log de transações do banco (WAL) e publica o evento no Kafka/RabbitMQ com garantia de entrega *At-Least-Once*, eliminando perdas silenciosas de eventos mesmo em cenários de queda brutal de rede ou aplicação.

---

## 3. Desacoplamento com CQRS e Estratégia de Testes

### 3.1 Consultas em Alta Vazão com CQRS

- **O Problema de Leitura:** Durante o pico de vendas, milhões de usuários atualizam a página de "Meus Pedidos" a cada poucos segundos.
- **Desenho da Solução:** Como você separa a responsabilidade de escrita (Command) da consulta de status (Query)? Como o modelo de leitura é alimentado e como sua arquitetura lida com a consistência eventual (o delay entre o pagamento e a visualização do status)?
  > - O Problema de Leitura:
  >Durante o pico de vendas, o volume massivo de consultas na página "Meus Pedidos" degrada o banco de dados relacional principal de checkout se as operações de escrita (Command) e leitura (Query) compartilharem as mesmas tabelas e recursos.
>- Desenho da Solução:
 > 1. Separação de Responsabilidades (CQRS): O modelo de escrita (Command) utiliza o PostgreSQL para processamento de checkout e orquestração da Saga. O modelo de consulta (Query) utiliza uma base otimizada para alta vazão de leitura e busca (ex: Elasticsearch / Redis).
  >2. Alimentação Desacoplada (Outbox + Debezium): Ao processar um comando de escrita, a aplicação grava a alteração e publica o evento de domínio na tabela `outbox` no banco transacional. O Debezium captura esses eventos e os projeta na base de leitura. Essa abordagem isola os esquemas de dados (Schema Isolation): alterações estruturais de tabelas no banco de escrita não quebram o modelo de leitura nem a API de consulta.
  >3. Gestão da Consistência Eventual na UI: Para contornar o pequeno delay de sincronização entre a escrita e a leitura, o Frontend exibe status intermediários explícitos (ex: `PROCESSANDO_PAGAMENTO`) e utiliza técnicas de *polling* ou *Server-Sent Events (SSE)* para atualizar a interface assim que a projeção de leitura for sincronizada.

### 3.2 Estratégia de Testes da Transação Distribuída

- **Validação de Resiliência:** Como comprovar em ambiente de testes e pipeline que a Saga funciona de verdade?
- **Plano de Testes:** Descreva como você estruturaria o teste automatizado para simular a rejeição na reserva de estoque após a cobrança e validar que a compensação automática transiciona o pedido para cancelado com sucesso.
  > - Validação de Resiliência:
 > A confiabilidade da Saga Orquestrada e dos fluxos de compensação será garantida através de testes de integração automatizados no pipeline de CI/CD utilizando **Testcontainers** (subindo instâncias reais de PostgreSQL e broker de mensageria em containers Docker) combinados com o **WireMock**.

>- Plano de Testes (Simulação de Falha e Compensação):
 > 1. Configuração dos Stubs (WireMock): O WireMock será configurado para simular o serviço de Estoque retornando o erro HTTP 409 / `ESTOQUE_ESGOTADO` na chamada de reserva.
  >2. Execução da Transação: O teste dispara uma requisição de checkout com pagamento aprovado no gateway simulado.
  >3. Asserção do Fluxo de Compensação: O teste valida se o Orquestrador interceptou a rejeição de estoque, se a mensagem de compensação foi inserida na tabela `outbox`, se a chamada de estorno foi enviada ao gateway via WireMock e se o estado final do pedido no PostgreSQL transicionou com sucesso para `PEDIDO_CANCELADO_COM_ESTORNO`.
