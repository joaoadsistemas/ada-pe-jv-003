# Template de System Design: Aula 02 - APIs Resilientes

> **Instruções:** Faça uma cópia deste arquivo e preencha as seções abaixo com as conclusões da sua pesquisa e os debates. O foco é a integração da API de Checkout com o Serviço de Pagamento Externo para suportar 10.000 usuários simultâneos.

## 1. Escolha de Paradigma da API

- **A Escolha:** (x) Spring MVC com Virtual Threads ou ( ) Spring WebFlux (Reativo)?
- **Justificativa:** Considerando a capacidade de suportar 10k acessos concorrentes, por que você fez essa escolha técnica?
  > Para uma API de Checkout, a maior parte do tempo de resposta é gasta aguardando I/O na chamada ao gateway de pagamento. Com o Java 21 e Virtual Threads ativadas no Spring MVC, o modelo Thread-per-Request se torna extremamente leve, permitindo escalar para 10.000 conexões simultâneas sem esgotar a memória do sistema operacional ou exigir gerenciamento manual de pools de threads complexos. Escolhi o Spring MVC com Virtual Threads em relação ao WebFlux porque ele entrega o mesmo nível de taxa de transferência (throughput) e resiliência em cenários I/O-bound, mas mantém a simplicidade do modelo imperativo tradicional, facilita o manuseio de depuração (stack traces simples), possui menor curva de aprendizado para o time e excelente integração com bibliotecas do ecossistema Spring.

## 2. Desenho de Resiliência

Dependemos de uma API terceira instável. Quais recursos do padrão de resiliência você vai utilizar?

- **Proteção contra Falhas e Lentidão:**
  Quais padrões (Circuit Breaker, TimeLimiter, RateLimiter, Bulkhead, Retry) você implementaria na chamada para o Parceiro de Pagamentos? Por que?
 
   1. TimeLimiter: Define um timeout rígido (ex: 2.5s) por chamada ao parceiro, evitando que requisições presas comprometam a experiência do usuário.
   2. Circuit Breaker: Monitora falhas e timeouts. Ao atingir o limiar (ex: 50% de falha nas últimas 100 chamadas), abre o circuito para interromper o tráfego ao parceiro instável e falhar rapidamente (fail-fast) com um fallback amigável. Muda para Half-Open periodicamente para testar a recuperação.
   3. Bulkhead: Limita o número máximo de chamadas concorrentes para o gateway de pagamento (ex: max 500 conexões), isolando os recursos da API de Checkout para que lentidões no pagamento não travem outros serviços da aplicação.
   4. RateLimiter: Garante que não ultrapassaremos o limite máximo de requisições por segundo (RPS) permitido no contrato com a API terceira.
   5. Retry: Configurado apenas para erros específicos de falha de conexão/transientes com backoff exponencial, associado obrigatoriamente à Idempotency-Key.

- **Proteção contra Duplicidade:**
  Como a `Idempotency-Key` será utilizada na requisição do Checkout para evitar cobranças duplas? Qual o ciclo de vida dessa chave?
  A Idempotency-Key será gerada pelo Frontend (UUID v4) para cada tentativa única de finalização de pedido e enviada no cabeçalho HTTP (`X-Idempotency-Key`).
  Ciclo de vida da chave:
  1. Criação e Trava: Ao receber a requisição, o Checkout realiza uma operação atômica no Redis (`SETNX`) com o estado `IN_PROGRESS` e TTL inicial de 60 segundos para evitar concorrência.
  2. Execução: A API de Checkout aciona a API do Parceiro de Pagamentos enviando essa mesma chave (se o parceiro suportar idempotência).
  3. Conclusão e Cache de Resposta: Após a resposta do parceiro, o valor no Redis é atualizado com o status final (ex: HTTP 200/400 e o payload resumido da transação) e o TTL é renovado para 24 horas.
  4. Deduplicação: Caso o usuário reenvie a mesma requisição (devido a lentidão ou múltiplos cliques), o sistema identifica a chave existente no Redis e retorna a resposta armazenada imediatamente, sem reprocessar a cobrança no parceiro.

## 3. Os Edge Cases (Engenharia do Caos)

Cole aqui as 3 perguntas de risco que você formulou e defina como a arquitetura responde a cada uma delas.

1. E se... a conexão com o gateway de pagamento sofrer timeout de rede sem confirmação de recebimento?
  - Resposta do Sistema: O sistema acionará a política de Retry associada obrigatoriamente à Idempotency-Key. Caso o parceiro já tenha recebido a chamada anterior, a chave impede a dupla cobrança. Se as tentativas de Retry também estourarem o TimeLimiter, o estado do pedido é marcado como PENDENTE ou FALHOU para conciliação assíncrona, evitando bloquear o cliente.

2. E se... o banco de dados/cache Redis de idempotência cair no pico de acessos?
  - Resposta do Sistema: A infraestrutura utilizará o Redis em modo de Alta Disponibilidade (Cluster/Sentinel com réplicas primário-secundário e failover automático). Caso ocorra indisponibilidade total do cluster Redis, o sistema adotará o comportamento Fail-Safe, utilizando um cache local em memória na aplicação para mitigação temporária.

3. E se... o Circuit Breaker abrir completamente devido à queda total do provedor de pagamentos externo?
  - Resposta do Sistema: O sistema acionará o Fallback do Resilience4j imediatamente (Fail-Fast), respondendo com HTTP 503 (Service Unavailable) e uma mensagem amigável sem tentar se comunicar com o parceiro. Isso protege os recursos de CPU e memória da API de Checkout e preserva o restante do sistema.

## 4. Testando a Resiliência

- **Estratégia:** Como você vai garantir que o Circuit Breaker abre na hora certa e o Timeout não deixa a Thread presa?
  A validação da arquitetura de resiliência será realizada através de testes integrados e testes de carga automatizados:
  1. Testes de Latência e Timeout (WireMock):
     Configuraremos stubs no WireMock com atraso artificial na resposta (ex: `withFixedDelay(5000)`). Testaremos se o TimeLimiter do Resilience4j cancela a requisição exatamente aos 2.5s e libera a Virtual Thread sem vazamento de memória ou threads presas.

  2. Testes de Circuit Breaker (Fault Injection com WireMock):
     Simularemos instabilidade no parceiro configurando o WireMock para retornar erros HTTP 500 sucessivos. Validaremos se o Circuit Breaker altera seu estado para OPEN após atingir a taxa de falhas configurada e se a API passa a responder via Fallback em milissegundos (Fail-Fast).

  3. Testes de Carga e Concorrência (K6 / JMeter):
     Executaremos scripts de teste de estresse simulando os 10.000 usuários simultâneos no fluxo de checkout. Monitoraremos a estabilidade das Virtual Threads do Java 21, a eficácia do Bulkhead na contenção de conexões paralelas e a validação atômica da Idempotency-Key no Redis sob alta concorrência.
