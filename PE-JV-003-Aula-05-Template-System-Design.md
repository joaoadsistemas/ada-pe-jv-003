# Template de System Design: Aula 05 - Containers, CI/CD & Deploy sem Downtime

> **Contexto do Cenário: O Deploy que Quebrou o Checkout e o Rollback Impossível**
>
> Sua equipe está reestruturando a engenharia de release do serviço de **Checkout** após um incidente severo na Black Friday:
>
> 1. **O Incidente:** Para subir uma otimização urgente na Saga, o pipeline de produção recompilou a branch `main`, sobrescreveu a tag `checkout:latest` e iniciou a atualização. No boot da primeira instância nova, o **Flyway** renomeou a coluna `status_pagamento` para `estado_saga`.
> 2. **O Efeito em Cadeia:** Os 80% de servidores que ainda rodavam a versão anterior passaram a falhar imediatamente (`column "status_pagamento" does not exist`). Ao tentar acionar o rollback puxando `:latest`, a equipe descobriu que a imagem anterior havia sido sobrescrita e, mesmo restaurando um backup do binário antigo, ele continuou falhando porque o banco já estava alterado.
> 3. **A Carga:** 5.000 TPS ativos com múltiplas réplicas de aplicação conectadas ao mesmo banco PostgreSQL.
>
> **Instruções:** Responda às decisões arquiteturais abaixo fundamentando suas escolhas estritamente com base nos requisitos e restrições deste cenário. Documente suas decisões no repositório contínuo do seu projeto através de commits atômicos e bem descritivos.

---

## 1. Empacotamento, Superfície de Ataque e Promoção Imutável

### 1.1 Anatomia da Imagem e Redução de Superfície de Ataque

- **Construção vs Execução:** Como você estruturará o empacotamento do serviço de Checkout para produção? Especifique o que deve ser usado na etapa de compilação, o que deve (e o que não deve) entrar na imagem final de execução, com qual permissão de usuário o processo rodará e qual o trade-off da sua escolha de imagem base para a segurança e para o diagnóstico em produção.
  
  >Adotei o padrão **Multi-Stage Build** do Docker:
  >1. *Estágio de Construção (Builder):* Utiliza a imagem `maven:3.9-eclipse-temurin-21` para baixar dependências, executar testes unitários e compilar o arquivo `.jar`.
  >2. *Estágio de Execução (Runner):* Utiliza uma imagem minimalista `gcr.io/distroless/java21-debian12:nonroot`. Copiaremos exclusivamente o `.jar` final gerado. Código-fonte, ferramentas de compilação (Maven) e bibliotecas de teste são completamente descartados.
  >*Permissão de Usuário:* O processo rodará obrigatoriamente com um usuário sem privilégios (`USER nonroot:nonroot`), impedindo escalação de privilégios no sistema operacional do host em caso de RCE (Remote Code Execution).
 > *Trade-off da Imagem Base (Distroless):*
  >- **Vantagens:** Superfície de ataque minimalista sem gerenciadores de pacote ou shell (`bash`/`sh`), reduzindo drasticamente CVEs no SO e impedindo execução de scripts maliciosos.
  >- **Trade-off:** Não permite acesso interativo via terminal (`kubectl exec`) para troubleshooting direto. O diagnóstico em produção dependerá 100% de observabilidade externa (Logs em JSON via stdout, Métricas do Actuator/Prometheus e Tracing Distribuído).

### 1.2 Rastreabilidade, Identidade do Artefato e Parametrização

- **Da Homologação à Produção:** Como sua esteira garante que o binário executado em produção é exatamente o mesmo que passou nos testes de homologação? Explique como as configurações mudam entre ambientes (dev, staging, prod) sem recompilar o artefato e como você identifica a versão no registry para que um rollback nunca puxe um binário sobrescrito por engano.
  >1. *Identidade Imutável e Rastreabilidade:* A tag `:latest` será estritamente proibida em todos os ambientes. Cada imagem construída no pipeline receberá uma tag imutável baseada no Commit Hash do Git + Versionamento Semântico (ex: `checkout:v2.1.0-a1b2c3d`) e será referenciada pelo seu hash digest único SHA-256. Isso garante que a imagem promovida para produção seja matematicamente idêntica à testada em homologação e que um rollback puxe exatamente a imagem anterior sem riscos de sobreposição.
  >2. *Parametrização por Ambiente (Twelve-Factor App):* O artefato compilado (`.jar`) e a imagem do container são 100% agnósticos de ambiente. As configurações específicas (endpoints de banco de dados, credenciais, URLs do gateway) são injetadas em tempo de execução através de variáveis de ambiente gerenciadas por `ConfigMaps` e `Secrets` do Kubernetes, sem a necessidade de recompilar o código ou gerar novas imagens para cada ambiente.

---

## 2. Estratégia de Rollout e Migração de Banco sem Downtime

### 2.1 O Modelo de Rollout em Produção

- **A Escolha:** (x) Canary Release (Liberação Gradual por Percentual)  ( ) Blue-Green Deployment (Alternância de Ambientes)
- **Justificativa e Mitigação das Desvantagens do seu Modelo:**
  - *Se você escolheu Canary Release:* Como o tráfego evoluirá sob 5.000 TPS, quais sinais definirão o avanço ou a reversão (rollback) automática e como lidar com o tempo em que duas versões diferentes da API atendem clientes simultaneamente?
  - *Se você escolheu Blue-Green Deployment:* Como você gerenciará o custo de infraestrutura duplicada no pico da Black Friday, o aquecimento das novas instâncias (conexões/cache) antes da virada de chave e o risco de virar 100% do tráfego de uma só vez?
  > Para suportar a carga de 5.000 TPS na Black Friday, adotei o Canary Release automatizado via Argo Rollouts / Service Mesh. 
  >*Evolução do Tráfego:* A nova versão (v2) receberá progressivamente 2% -> 10% -> 25% -> 50% -> 100% do tráfego real.
  >*Gatilhos de Rollback Automático:* A esteira monitorará os SLAs da release. Caso ocorra aumento no tempo de resposta p99 (> 500ms), elevação da taxa de erros HTTP 5xx (> 1%) ou estouro no pool HikariCP, a versão v2 é cortada e o tráfego retorna 100% para a v1 instantaneamente.
  >*Coexistência de Versões:* Para suportar o período em que v1 e v2 atendem clientes simultaneamente, o banco de dados utilizará o padrão de evolução de schema sem quebra (Expand & Contract).

### 2.2 Evolução de Schema Sem Downtime e Plano de Rollback

- **Passo a Passo da Migração:** Descreva em etapas sequenciais como evoluir a coluna `status_pagamento` para `estado_saga` no PostgreSQL garantindo que as instâncias da versão antiga (`v1`) e da versão nova (`v2`) operem juntas sem erros de SQL.
  > 1. *Fase 1 (Expandir - Flyway v1.1):* Adicionar a nova coluna `estado_saga` na tabela de pedidos como opcional (`NULLable`). A versão antiga da aplicação (v1) continua utilizando a coluna `status_pagamento` sem interrupção.
  >2. *Fase 2 (Dupla Escrita - Deploy Canary v2):* A versão v2 é implantada e configurada para escrever simultaneamente em ambas as colunas (`status_pagamento` e `estado_saga`), mantendo-as sincronizadas (via código ou Triggers no PostgreSQL).
  >3. *Fase 3 (Backfill de Dados):* Executa-se um script assíncrono em lote para copiar os dados históricos legados de `status_pagamento` para `estado_saga` nos registros antigos.
  >4. *Fase 4 (Contract / Limpeza - Flyway v1.2):* Apenas após a v2 estar com 100% do tráfego consolidado e sem necessidade de rollback, uma nova migração remove a coluna antiga `status_pagamento`.
- **Garantia de Rollback Imediato:** Caso a versão `v2` apresente degradação grave de performance com 20% do tráfego na Black Friday, qual é o procedimento exato de rollback e por que o seu desenho de banco de dados garante que a versão `v1` voltará a assumir 100% da carga sem quebrar nem perder dados gravados pela `v2`?
  > Se a v2 apresentar degradação grave com 20% do tráfego, o roteador corta a v2 e redireciona 100% das requisições de volta para as instâncias v1.
  >Como a coluna `status_pagamento` nunca foi renomeada nem removida do PostgreSQL e continuou sendo atualizada pelas instâncias v2 durante a dupla escrita, a versão v1 volta a atender 100% da carga sem falhas de SQL e sem perda das transações gravadas durante a janela de teste do Canary.

---

## 3. Guardrails de Segurança, Governança e Validação da Release

### 3.1 Gates de Segurança e Guardrails no Processo de Entrega

- **Guardrails do Commit ao Deploy:** Quais portões automatizados de qualidade e segurança devem proteger a sua esteira ao longo do fluxo de desenvolvimento (desde o Pull Request até a publicação e promoção da imagem)?
  > 1. *Análise Estática e Qualidade (SAST):* Análise automatizada via SonarQube bloqueando PRs com cobertura de testes abaixo de 80% ou dívidas técnicas críticas.
  >2. *Escaneamento de Imagens e Dependências (SCA / Container Scan):* Ferramentas automatizadas (Trivy / Grype) escaneiam a imagem gerada no build. O pipeline é abortado imediatamente se forem detectadas vulnerabilidades de segurança de nível Alto ou Crítico (CVEs).
  >3. *Assinatura do Artefato:* A imagem Docker é assinada digitalmente (via Cosign / Sigstore) no registry. O cluster Kubernetes só aceita rodar imagens devidamente assinadas pelo pipeline oficial.
- **Governança e Controle em Produção:** Quais *guardrails* de processo e arquitetura impedem que alterações manuais não rastreadas cheguem à produção e como equilibrar segurança rigorosa com agilidade de resposta durante incidentes críticos sem perder a trilha de auditoria?
  > 1. *Adoção de GitOps:* A infraestrutura e os deployments são 100% declarativos e gerenciados por GitOps (ex: ArgoCD). Acessos manuais diretos ao cluster (`kubectl apply` / `docker run`) são estritamente revogados em produção.
  >2. *Procedimento Emergencial (Break-Glass):* Em incidentes críticos na Black Friday, correções emergenciais devem seguir o fluxo de Hotfix via PR automatizado com aprovação rápida de dois engenheiros sêniores, mantendo o histórico de auditoria e commits imutáveis no Git.

### 3.2 Estratégia de Testes da Imagem e da Migração

- **Validação Pré-Tráfego:** Passar nos testes unitários durante o build do código não prova que a imagem final do container sobe nem que a migração de banco é retrocompatível.
- **Plano de Testes na Pipeline:** Descreva como você testará automaticamente a imagem construída e validará a coexistência da migração de banco de dados na esteira antes de liberar o deploy para o tráfego real de clientes.
  > 1. *Container Smoke Test & Healthcheck:* Antes de publicar a imagem no registry, o pipeline de CI executa o container em um ambiente isolado (via Testcontainers) e valida a inicialização dos endpoints de saúde (`/actuator/health` e `/actuator/liveness`), garantindo que a imagem boota perfeitamente como usuário non-root e sem dependências faltantes.
 > 2. *Teste de Retrocompatibilidade da Migração (Flyway Validation):* O pipeline dispara um teste de integração que aplica as migrações do Flyway em um banco PostgreSQL descartável. O teste executa chamadas de escrita e leitura simulando simultaneamente instâncias da versão antiga (v1) e da versão nova (v2). A imagem só é aprovada para o Canary se for comprovado que nenhuma instrução SQL da v1 falhou durante a execução com o novo schema.
