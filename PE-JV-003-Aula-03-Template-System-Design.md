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

- **A Escolha para o Checkout:** ( ) Relacional Puro (PostgreSQL) ( ) Persistência Poliglota / Híbrida
- **Justificativa Arquitetural:** Avalie o Teorema CAP e o modelo de consistência (ACID vs BASE). Por que a sua escolha atende aos requisitos do Checkout sem comprometer a integridade financeira?
  > [Sua resposta aqui]
- **Além do Modelo Documental (Ecossistema NoSQL):** A persistência poliglota vai muito além da dicotomia "Relacional vs MongoDB". Considerando as necessidades complementares de uma plataforma de e-commerce (como carrinhos efêmeros, busca textual de produtos, telemetria/logs, auditoria temporal ou análise de conexões/fraude), cite e comente pelo menos outros dois tipos/famílias de bancos NoSQL que poderiam coexistir na arquitetura, indicando para quais casos de uso específicos cada um seria a ferramenta ideal e por quê.
  > [Sua resposta aqui]

### 1.2 Estratégia da Chave Primária

- **A Escolha:** ( ) Sequence / Identity ( ) UUIDv4 ( ) UUIDv7 / TSID
- **Impacto no Armazenamento:** Por que essa estratégia foi escolhida considerando inserções concorrentes e o impacto de fragmentação na árvore B-Tree (page splits)?
  > [Sua resposta aqui]

---

## 2. A Dualidade da Concorrência: Locks e Mitigações

Analise e defina a abordagem de concorrência para os dois fluxos críticos do negócio:

### 2.1 Mudança de Status do Pagamento (Baixa contenção por registro)

- **Abordagem de Lock:** ( ) Lock Otimista (`@Version`) ou ( ) Lock Pessimista (`SELECT FOR UPDATE`)?
- **Justificativa:** Por que essa abordagem é a mais adequada para este cenário específico?
  > [Sua resposta aqui]

### 2.2 Controle de Estoque Concorrente (Alta contenção no mesmo registro)

- **Abordagem de Lock:** ( ) Lock Otimista (`@Version`) ou ( ) Lock Pessimista (`SELECT FOR UPDATE`)?
- **Justificativa:** Centenas de usuários disputando os últimos itens na Black Friday. Como você garante que o saldo nunca fique negativo sem derrubar o banco?
  > [Sua resposta aqui]

### 2.3 Riscos e Mitigações

- **Análise Crítica:** Quais os riscos operacionais das abordagens de lock que você escolheu (ex: excesso de `OptimisticLockException` frustrando o cliente vs `ConnectionTimeoutException` e *deadlocks* enfileirando threads no Lock Pessimista)? Como sua arquitetura mitiga esses riscos?
  > [Sua resposta aqui]

---

## 3. Troubleshooting do Incidente e Gestão de Índices

### 3.1 Investigação de Incidente em Produção

- **Cenário:** Em testes de carga para a Black Friday, o tempo de resposta p99 do Checkout subiu repentinamente para 4.8s e o pool de conexões (HikariCP) saturou. Os logs apontam que a consulta de pedidos mais frequente (`WHERE customer_id = ? AND order_status = 'processing'`) está demorando segundos para responder.
- **Plano de Diagnóstico:** Qual é o seu processo técnico passo a passo para investigar a causa raiz dessa lentidão diretamente no banco de dados? Quais comandos, métricas ou planos você analisaria para comprovar a hipótese antes de propor qualquer mudança?
  > [Sua resposta aqui]

### 3.2 Decisão de Engenharia: A Abordagem de Índices

- **Dilema no Pull Request:** Para solucionar a lentidão, um desenvolvedor da equipe abriu um PR criando índices individuais para todas as colunas da tabela de pedidos que aparecem em filtros de consulta.
- **Seu Posicionamento:** Como responsável técnico, você aprovaria esse PR? Qual é a sua avaliação crítica sobre essa estratégia e quais impactos colaterais sistêmicos devem ser ponderados antes de aprovar novos índices?
  > [Sua resposta aqui]
