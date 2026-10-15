### **Cenário**

A API de Checkout precisa aguentar **10.000 usuários simultâneos** e chama um **Parceiro de Pagamento externo** (Cielo, Stripe) instável. O cliente pode reenviar o pedido se demorar, mas o dinheiro só pode sair uma vez e uma falha nunca pode virar compra aprovada.

---

### **1. Como garantir que uma mesma compra não seja cobrada duas vezes?**

* **Chave de Idempotência:** O app/site gera um código único (`UUID`) para o pedido na hora que o usuário clica em "Comprar".
* **Bloqueio no Redis:** Quando o Checkout recebe o pedido, ele grava esse código no cache (Redis). Se receber outra tentativa com o mesmo código:
* **Em andamento:** Rejeita a tentativa duplicada ou avisa que já está processando.
* **Já concluído:** Retorna a resposta que já estava salva sem chamar o banco/gateway de novo.


* **Repasse ao Gateway:** O Checkout envia essa mesma chave no cabeçalho da requisição para o parceiro de pagamento, garantindo que ele também não processe a cobrança duas vezes.

---

### **2. Se a cobrança for feita, mas a conexão cair antes da resposta, como tratar?**

* **Status Inicial Pendente:** O pedido é criado no banco como `PENDENTE`. Ele **nunca** é marcado como `APROVADO` de forma automática na dúvida.
* **Webhook do Parceiro:** Quando a conexão cai, esperamos o parceiro avisar o nosso Checkout via *Webhook* (notificação assíncrona) confirmando se aprovou ou recusou.
* **Job de Reconciliação (Fallback):** Se o webhook falhar, um processo em segundo plano (*Worker*) consulta periodicamente a API do parceiro usando a chave do pedido para confirmar o status real e atualizar o banco.
* **Polling no App:** O aplicativo exibe "Processando pagamento" e fica consultando o Checkout até que o status mude para aprovado ou recusado.

---

### **3. Como evitar que a lentidão do parceiro derrube o Checkout inteiro (10k usuários)?**

* **Arquitetura Assíncrona (Filas):** A API do Checkout **não chama o parceiro na hora**. Ela recebe o pedido, salva como `PENDENTE`, joga em uma fila (RabbitMQ/Kafka) e responde imediatamente `202 Processando` em milissegundos, liberando a API para atender os outros usuários.
* **Workers Dedicados:** Processos em segundo plano consomem a fila e chamam o parceiro de pagamento em um ritmo controlado.
* **Circuit Breaker (Disjuntor):** Se o parceiro começar a falhar demais ou ficar muito lento, o *Circuit Breaker* "abre" e cancela chamadas futuras instantaneamente antes que travem a aplicação.
* **Timeouts Curtos e Backoff:** Define-se um limite baixo de espera (ex: 3s). Se expirar, o pedido volta para a fila para tentar novamente mais tarde com intervalos crescentes (*Exponential Backoff*).
