# Template de System Design: Aula 03 - Banco de Dados em Larga Escala

> **Contexto do Cenário: Checkout na Black Friday**
>
> Sua equipe está preparando a persistência da API de Checkout para o pico de tráfego da Black Friday:
>
> 1. **Armazenamento:** Transações financeiras de pedidos exigem garantias ACID rigorosas e consistência imediata, enquanto dados de catálogo e sessões de usuários demandam alta vazão de leitura e tolerância a partição.
> 2. **Concorrência:** A atualização de status do pedido tem baixa disputa por registro (apenas o comprador interage), mas a **reserva de estoque** sofre altíssima contenção com centenas de clientes disputando as últimas unidades do mesmo SKU promocional.
> 3. **Incidente em Produção:** Em testes de carga, o tempo de resposta p99 subiu para 4.8s e o pool HikariCP saturou devido a consultas lentas de pedidos (`WHERE customer_id = ?` e `WHERE order_status = 'processing'`).
>
> **Instruções:** Responda às decisões arquiteturais abaixo fundamentando suas escolhas estritamente com base nos requisitos e restrições deste cenário. Adicione este bloco ao repositório contínuo do seu projeto.

---

## 1. Decisão de Armazenamento e Chave Primária (PK)

### 1.1 Motor de Banco de Dados e Paradigma de Persistência

- **A Escolha para o Checkout:** ( ) Relacional Puro (PostgreSQL) (x) Persistência Poliglota / Híbrida
- **Justificativa Arquitetural:** Avalie o Teorema CAP e o modelo de consistência (ACID vs BASE). Por que a sua escolha atende aos requisitos do Checkout sem comprometer a integridade financeira?
  > Foi adotado a Persistência Poliglota sob o padrão CQRS (Command Query Responsibility Segregation). 
  Para o modelo de escrita (Command), foi utilizado o PostgreSQL devido ao suporte rigoroso a transações ACID e consistência imediata (priorizando Consistência no Teorema CAP), garantindo a integridade dos dados financeiros. 
  Para o modelo de leitura e busca (Query), foi utilizado Change Data Capture (CDC via Debezium) capturando as alterações do Postgres e projetando os dados em um motor NoSQL/Search (como Elasticsearch). O modelo de consulta opera sob o paradigma BASE (Consistência Eventual / modelo AP), garantindo alta disponibilidade e altíssima vazão de leitura na Black Friday sem consumir recursos transacionais da base principal.
- **Além do Modelo Documental (Ecossistema NoSQL):** A persistência poliglota vai muito além da dicotomia "Relacional vs MongoDB". Considerando as necessidades complementares de uma plataforma de e-commerce (como carrinhos efêmeros, busca textual de produtos, telemetria/logs, auditoria temporal ou análise de conexões/fraude), cite e comente pelo menos outros dois tipos/famílias de bancos NoSQL que poderiam coexistir na arquitetura, indicando para quais casos de uso específicos cada um seria a ferramenta ideal e por quê.
  > 1. Banco Chave-Valor (Redis): Ideal para o gerenciamento de sessões de usuário, armazenamento efêmero de carrinhos de compra e controle de chaves de idempotência. Sua execução em memória entrega latência sub-milissegundo com suporte nativo a expiração (TTL), aliviando o banco relacional de cargas efêmeras.
  > 2. Banco de Grafos (Neo4j): Ideal para o motor de detecção de fraudes no momento do checkout. Permite mapear e analisar em tempo real relações complexas e suspeitas entre cartões de crédito, dispositivos, endereços de entrega e históricos de contas compartilhadas.

### 1.2 Estratégia da Chave Primária

- **A Escolha:** ( ) Sequence / Identity ( ) UUIDv4 (x) UUIDv7 / TSID
- **Impacto no Armazenamento:** Por que essa estratégia foi escolhida considerando inserções concorrentes e o impacto de fragmentação na árvore B-Tree (page splits)?
  > Escolhi o UUIDv7 porque ele combina a ordenação temporal (time-sorted) dos primeiros bits com a unicidade global e descentralizada das aplicações. 
  Em relação ao UUIDv4, que é totalmente aleatório e causa fragmentação severa de B-Tree (page splits) e estouro de buffer pool na inserção de registros massivos, o UUIDv7 garante que os novos registros sejam gravados sequencialmente no final das páginas de índice. Além disso, elimina a contenção de locks centrais no banco que ocorrem com Sequences/Identity durante picos de inserções concorrentes na Black Friday.

---

## 2. A Dualidade da Concorrência: Locks e Mitigações

Analise e defina a abordagem de concorrência para os dois fluxos críticos do negócio:

### 2.1 Mudança de Status do Pagamento (Baixa contenção por registro)

- **Abordagem de Lock:** (x) Lock Otimista (`@Version`) ou ( ) Lock Pessimista (`SELECT FOR UPDATE`)?
- **Justificativa:** Por que essa abordagem é a mais adequada para este cenário específico?
  > Como apenas a transação do próprio usuário ou o callback/webhook do gateway de pagamento alteram o status de um pedido específico, a chance de duas transações disputarem a escrita do mesmo registro simultaneamente é praticamente nula. O Lock Otimista (@Version) garante a integridade dos dados sem a sobrecarga de bloqueio de linhas no banco de dados, liberando recursos do servidor para atender outras conexões.

### 2.2 Controle de Estoque Concorrente (Alta contenção no mesmo registro)

- **Abordagem de Lock:** ( ) Lock Otimista (`@Version`) ou (x) Lock Pessimista (`SELECT FOR UPDATE`)?
- **Justificativa:** Centenas de usuários disputando os últimos itens na Black Friday. Como você garante que o saldo nunca fique negativo sem derrubar o banco?
  > Em picos como a Black Friday, o Lock Otimista falharia para quase 100% dos usuários concorrentes disputando o mesmo item, gerando uma tempestade de exceções (`OptimisticLockException`) e reprocessamentos inúteis. O Lock Pessimista garante que apenas uma transação atualize o estoque por vez, mantendo a consistência estrita da contagem de saldo sem risco de venda sem estoque (overselling).

### 2.3 Riscos e Mitigações

- **Análise Crítica:** Quais os riscos operacionais das abordagens de lock que você escolheu (ex: excesso de `OptimisticLockException` frustrando o cliente vs `ConnectionTimeoutException` e *deadlocks* enfileirando threads no Lock Pessimista)? Como sua arquitetura mitiga esses riscos?
  > O maior risco do Lock Pessimista em altíssima concorrência é o enfileiramento de transações no banco, causando exaustão e saturação do pool de conexões (HikariCP Timeout) e potenciais Deadlocks.
  > Para mitigar esse risco, adotaremos duas estratégias complementares:
  > 1. Pré-reserva de Estoque no Redis (In-Memory): A validação e o decremento do saldo são realizados primeiramente no Redis de forma atômica (via comandos `DECR` ou scripts Lua). Caso o saldo no Redis seja zero, a requisição é rejeitada imediatamente no nível da API (Fail-Fast), evitando que a chamada chegue ao PostgreSQL.
  > 2. Lock com Timeout/NOWAIT: No banco relacional, as consultas com Lock Pessimista utilizarão cláusulas de timeout (`SELECT ... FOR UPDATE NOWAIT`), garantindo que requisições que não consigam o lock imediatamente falhem rapidamente em vez de prender a thread no pool do HikariCP.

---

## 3. Troubleshooting do Incidente e Gestão de Índices

### 3.1 Investigação de Incidente em Produção

- **Cenário:** Em testes de carga para a Black Friday, o tempo de resposta p99 do Checkout subiu repentinamente para 4.8s e o pool de conexões (HikariCP) saturou. Os logs apontam que a consulta de pedidos mais frequente (`WHERE customer_id = ? AND order_status = 'processing'`) está demorando segundos para responder.
- **Plano de Diagnóstico:** Qual é o seu processo técnico passo a passo para investigar a causa raiz dessa lentidão diretamente no banco de dados? Quais comandos, métricas ou planos você analisaria para comprovar a hipótese antes de propor qualquer mudança?
  > 1. Análise do Plano de Execução: Executaremos o comando `EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE customer_id = '123' AND order_status = 'processing';` no banco de dados para checar a estratégia de busca (verificando a ocorrência de Sequential Scan vs. Index Scan), o custo total de I/O e a taxa de uso do Buffer Pool.
  > 2. Verificação de Saturação e Locks: Consultaremos a view `pg_stat_activity` para identificar se as sessões presas que estão saturando o pool HikariCP estão bloqueadas aguardando locks de transação (`wait_event_type = 'Lock'`) ou gargalos de gravação/leitura no disco (`wait_event_type = 'IO'`).
  > 3. Atualização de Estatísticas: Verificaremos se o auto-vacuum/analyze do PostgreSQL está atualizado para a tabela de pedidos (`pg_stat_user_tables`), garantindo que o planejador do banco esteja utilizando estatísticas recentes para a escolha do melhor plano de execução.

### 3.2 Decisão de Engenharia: A Abordagem de Índices

- **Dilema no Pull Request:** Para solucionar a lentidão, um desenvolvedor da equipe abriu um PR criando índices individuais para todas as colunas da tabela de pedidos que aparecem em filtros de consulta.
- **Seu Posicionamento:** Como responsável técnico, você aprovaria esse PR? Qual é a sua avaliação crítica sobre essa estratégia e quais impactos colaterais sistêmicos devem ser ponderados antes de aprovar novos índices?
  > Eu **reprovaria o Pull Request**.
  
  > Avaliação Crítica:
  > 1. Degradabilidade da Performance de Escrita: Índices individuais em tabelas de altíssima vazão de escrita (como a tabela de pedidos no Checkout) geram uma sobrecarga severa no disco e na CPU. A cada novo `INSERT` ou `UPDATE`, a árvore B-Tree de cada um dos índices precisa ser atualizada, o que deixaria as escritas na Black Friday extremamente lentas.
  > 2. Ineficiência para Filtros Compostos: Criar índices separados por coluna (um para `customer_id` e outro para `order_status`) obriga o banco a realizar leituras e cruzamentos de múltiplos índices (*Bitmap Index Scan*), o que é muito menos eficiente do que um único índice focado.
  
  > Solução Recomendada:
  > Substituir o PR pela criação de um único **Índice Composto (Multi-Coluna)** direcionado para a consulta mais frequente:
  > `CREATE INDEX CONCURRENTLY idx_orders_customer_status ON orders (customer_id, order_status);`
  
  > *(A flag `CONCURRENTLY` garante que o índice seja construído em produção sem travar a tabela para escritas durante a Black Friday).*
