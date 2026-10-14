Cenário. A API de Checkout precisa aguentar 10.000 usuários simultâneos e, para fechar cada compra, chama um Parceiro de Pagamento externo (ex.: Cielo, Stripe) que é instável: às vezes fica lento, às vezes devolve erro, às vezes derruba a conexão. O cliente (app ou site) pode reenviar o mesmo pedido quando a resposta demora. O dinheiro só pode sair uma vez, e uma falha nunca pode aparecer como compra aprovada.

1 -> O cliente pode reenviar o mesmo pedido várias vezes se o pagamento demorar. Como você garantiria que uma mesma compra não seja processada ou cobrada duas vezes?

2 -> Se a API de pagamento processar a cobrança, mas a conexão cair antes de o Checkout receber a resposta, como você trataria essa situação sem aprovar uma compra que não foi confirmada?

3 -> Com 10.000 usuários simultâneos e um parceiro de pagamento instável, como você projetaria a API para evitar que a lentidão ou indisponibilidade do parceiro derrube o Checkout inteiro?
