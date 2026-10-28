# Template de System Design: Aula 06 - Observabilidade & Diagnóstico em Produção

> **Contexto do Cenário: O Checkout que Trava sem Dar Erro**
>
> Sua equipe responde pelo serviço de **Checkout** do marketplace, que chama **Pagamento** e **Estoque**. Os três rodam em imagens *Distroless*.
>
> 1. **O Incidente:** Na abertura da Black Friday, com **5.000 TPS**, uma versão nova do Checkout entrou em *Canary*. Minutos depois, os painéis continuavam verdes, com latência média de 120 ms, CPU abaixo de 40% e nenhum pico de HTTP 500.
> 2. **O Impacto:** A conversão despencou. Milhares de reclamações chegaram ao suporte, todas com a mesma cena. O cliente clica em "Finalizar Pedido", a tela fica carregando e o aplicativo desiste por timeout.
> 3. **As Restrições:** A imagem não tem `sh` e não há SSH nos servidores. Cada serviço escreve logs em texto livre, do seu jeito, sem um identificador que ligue a chamada do Checkout à chamada que ele fez ao Pagamento. E dado de cliente que vai parar em log continua sujeito à LGPD.
>
> **Instruções:** Responda às decisões arquiteturais abaixo fundamentando suas escolhas estritamente com base nos requisitos e restrições deste cenário. Documente suas decisões no repositório contínuo do seu projeto através de commits atômicos e bem descritivos.

---

## 1. Logs e Correlação

- **Campos do Log:** Que campos todo log do Checkout precisa ter para você encontrar, no meio de milhares de linhas, as da requisição que travou? Liste os campos e diga para que serve cada um.
  > - `@timestamp`: Data e hora em formato ISO-8601 com precisão em milissegundos.
  > - `trace_id`: Identificador global e único da transação distribuída (padrão W3C Trace Context).
  > - `span_id`: Identificador do passo/operação específica dentro do serviço.
  > - `service.name`: Nome do microsserviço emitente (`checkout-service`).
  > - `log.level`: Nível do log (`INFO`, `WARN`, `ERROR`).
  > - `http.method` & `http.url`: Método e rota parametrizada chamada (ex: `POST /api/v1/checkout`).
  > - `customer.id_hash`: Pseudônimo criptográfico do cliente para correlação sem expor PII.
  > - `thread.name`: Nome/ID da Virtual Thread para identificar travamento em operações bloqueantes.
  > - `duration_ms`: Tempo de execução acumulado até aquele ponto do log.
- **O Caminho do Contexto:** Como o mesmo identificador de rastreamento passa do Checkout para o Pagamento e o Estoque numa chamada HTTP?
  > O rastreamento através de chamadas HTTP síncronas entre Checkout, Pagamento e Estoque adota o protocolo **W3C Trace Context**. O client HTTP do Checkout injeta o `trace_id` e o `span_id` no cabeçalho HTTP padrão `traceparent`. Os serviços receptores utilizam interceptores do OpenTelemetry / Micrometer Tracing para extrair esse cabeçalho e populá-lo automaticamente no MDC (*Mapped Diagnostic Context*) da aplicação, garantindo que todos os logs e sub-spans compartilhem o mesmo `trace_id`.
- **LGPD nos Logs:** Quais dados de cliente ficam fora de logs e traces? Para os identificadores que precisam aparecer, qual tratamento você aplica e quem pode ler esses registros?
  > - *Dados Proibidos:* Nomes de clientes, CPFs, e-mails, endereços, números de cartão de crédito e tokens de autorização/Bearer Tokens não devem ser gravados em logs sob nenhuma hipótese.
  > - *Tratamento de Identificadores:* O `customer_id` é anonimizado utilizando hash SHA-256 com salt da aplicação (`hash(customer_id + salt)`), permitindo correlacionar chamadas do mesmo usuário sem violar a LGPD.
  > - *Controle de Acesso:* Os logs armazenados na plataforma de observabilidade possuem controle de acesso baseado em função (RBAC), restritos aos engenheiros da equipe de operações/SRE com auditoria de acesso ativada.

---

## 2. Métricas

- **Visão do Cliente (RED):** Quais métricas do Checkout entram no painel? Explique como você mede a duração para que o travamento de uma parte dos clientes não desapareça na média.
  > Monitoremos a **Taxa (Rate)** com `http_requests_total`, **Erros** com `http_requests_errors_total` e **Duração (Duration)** através do histograma de latência   `http_server_requests_seconds_bucket`.
 > - *Por que a média camuflou o travamento:* A latência média (120 ms) é distorcida por requisições rápidas/com falha simples. Para capturar o travamento, o painel deve exibir obrigatoriamente os **percentis p95, p99 e p99.9**. O gráfico mostrará o p99 encostando no limite do timeout do aplicativo (30s+), revelando o travamento do cliente enquanto a média aparentava estabilidade.
- **Visão dos Recursos (USE):** Que recursos você monitora? Que sinal acusaria saturação antes do primeiro HTTP 500?
  > Monitoraremos utilização e saturação do **Pool HikariCP**, conexões HTTP de saída e threads da JVM.
  > - *Sinal de Saturação que antecede o HTTP 500:* A métrica `hikaricp_pending_threads` (threads aguardando conexão com o banco) e `hikaricp_connections_timeout_total`. Quando chamadas a serviços terceiros/estoque demoram, as transações do banco ficam abertas e esgotam o pool do HikariCP. As conexões pendentes disparam a saturação e travam o sistema antes que a aplicação chegue a emitir um erro HTTP 500.
- **Labels:** Quais labels você permite nessas métricas e quais proíbe? Diga o efeito de cada escolha no servidor de métricas.
  > - *Permitidas (Baixa Cardinalidade):* `http_status` (200, 400, 500), `http_method` (POST, GET), `uri` (com o caminho genérico `/api/v1/checkout`), `environment` (`prod`).
  > - *Proibidas (Alta Cardinalidade):* `customer_id`, `order_id`, `user_ip` e parâmetros de query string únicos.
  > - *Efeito:* Labels permitidas mantêm o consumo de memória do Prometheus/VictoriaMetrics estável. Labels proibidas causariam uma **explosão de cardinalidade**, multiplicando as séries temporais até esgotar a memória RAM e derrubar o servidor de métricas durante a Black Friday.

---

## 3. Traces e Amostragem

- **A Escolha:** ( ) Head Sampling  ( ) Tail Sampling  (x) Combinação das duas
- **Justificativa:** Guardar 100% dos traces a 5.000 TPS custa caro. Qual é a sua regra de retenção, e como ela evita que justamente as requisições que travaram sejam descartadas?
  > A 5.000 TPS, reter 100% dos traces geraria um custo insustentável de rede e armazenamento. Adotaremos **Head Sampling** no SDK da aplicação configurado para capturar 1% das requisições normais (caminho feliz).
  > Em paralelo, configuraremos **Tail Sampling** no OpenTelemetry Collector (na camada de infraestrutura): o collector retém temporariamente os spans em memória e avalia a transação completa antes de descartar. Se a latência total do trace for superior a 2 segundos ou se contiver erros, o collector força a retenção de **100% desses traces anômalos**. Isso garante a captura detalhada de exatamente todas as requisições que sofreram com o travamento.

---

## 4. Estratégia de Testes

- **Qual a nossa estratégia para testar isso e garantir que funciona?** Antes da próxima Black Friday, como você prova que os painéis e os alertas mostrariam este incidente?
  > Validaremos a observabilidade através de **Engenharia do Caos e Teste de Carga**:
  > 1. *Simulação de Latência (Chaos Testing):* Em ambiente de homologação, usaremos o WireMock para responder com atraso artificial de 30 segundos em 15% das chamadas do serviço de Estoque.
  > 2. *Injeção de Carga (K6 / JMeter):* Executaremos um script de estresse simulando os 5.000 TPS de checkout.
  > 3. *Validação dos Painéis e Alertas:*
  >   - Confirmaremos se o gráfico de latência **p99/p99.9** no Grafana dispara o alerta de timeout imediatamente.
  >   - Verificaremos se o painel indica a saturação de conexões no HikariCP (`hikaricp_pending_threads`).
  >   - Testaremos se o OpenTelemetry Collector reteve os traces das chamadas lentas via **Tail Sampling** e se o `trace_id` permite navegar perfeitamente do log do Checkout até o log do Estoque no Grafana Loki.
