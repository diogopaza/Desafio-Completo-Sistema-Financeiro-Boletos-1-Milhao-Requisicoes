# 💳 Desafio Completo — Sistema Financeiro (Boletos, Hexagonal, Escala)

Projeto novo e independente, sem relação de numeração com os outros 3 repositórios da série — inspirado no artigo [**"Como eu construiria uma aplicação financeira com Java, Spring e AWS para suportar 1 milhão de requests por minuto"**](https://www.linkedin.com/pulse/como-eu-construiria-uma-aplica%C3%A7%C3%A3o-financeira-com-java-cleyton-chagas-9dr1f/), de Cleyton Chagas. Diferente dos anteriores (que usam dado de pagamento fictício), este integra de verdade com a API de boletos da Tecnospeed — os DTOs precisam refletir o contrato real da API, não um formato inventado.

## 🎯 Sobre este desafio

O artigo original propõe uma arquitetura para suportar 1 milhão de requisições por minuto (~16.667 req/s), usando WAF, Route53, ECS/Fargate, Redis, mensageria, Idempotency Key, OpenTelemetry e testes de carga com JMeter. Aqui, cada peça vira um desafio prático — com duas diferenças importantes em relação ao artigo original:

1. **Arquitetura Hexagonal (Ports & Adapters)** — nunca usada antes na série. O domínio (regra de negócio) não conhece HTTP, JPA, RabbitMQ nem nenhuma tecnologia externa; cada integração é um adaptador que implementa uma porta definida pelo domínio.
2. **Boletos reais via API da Tecnospeed** (ambiente de homologação) — em vez de simular pagamentos com dado sintético, o tipo de pagamento "boleto" chama uma API real, com erros e comportamento reais.

### 💰 Sobre o "1 milhão de requisições por minuto" — expectativa honesta

Não vamos atingir 16.667 req/s de verdade, e é importante entender por quê antes de começar:

* Gerar essa carga de forma sustentada exige infraestrutura real de produção (múltiplas máquinas geradoras de carga, cluster real) — custaria dinheiro de nuvem de verdade, o que vai contra o princípio de custo zero que seguimos nos outros repositórios.
* **Bombardear a API real da Tecnospeed com esse volume seria irresponsável** — ambiente de homologação de fintech tem rate limit baixo; isso resultaria em bloqueio, não em teste.

O que vamos fazer, que é a distinção que o próprio artigo defende no fim ("arquitetura é a hipótese, teste de carga mostra o limite"):

* **Teste de carga (volume)** — Parte 8, roda contra o **seu próprio** `payment-service` local, usando um adaptador **stub** de boleto (sem bater na Tecnospeed). Mede o teto real da sua máquina, seja qual for — o número que sair é o dado valioso, não "1 milhão".
* **Teste de corretude (comportamento)** — Parte 9, poucas dezenas de chamadas **reais** à Tecnospeed, focado em provar que idempotência e tratamento de erro funcionam contra uma API real e imprevisível — volume não é o ponto aqui.

WAF e ECS/Fargate reais (serviços pagos até em ferramentas de emulação como LocalStack Pro) ficam só teóricos; Route53 é usado de verdade via LocalStack Community (grátis). WAF na prática vira um filtro de rate-limiting escrito por você; ECS/Fargate vira Kubernetes local (Minikube/Kind), reaproveitando o que já está planejado no CI/CD da Parte 2.

### 🔐 Credenciais

A credencial da API da Tecnospeed (mesmo de homologação) **nunca** vai pro Git — usar variável de ambiente ou `application-local.properties` (já no `.gitignore`).

---

## 🧩 PARTE 1 — Fundamentos de Arquitetura Hexagonal (Ports & Adapters)

### 🎯 Objetivo

Entender Hexagonal não só na teoria, mas reestruturando um `payment-service` mínimo em três camadas: domínio (puro, sem framework), portas (interfaces que o domínio define) e adaptadores (implementações concretas de infraestrutura).

### 🧪 Desafio

* Cria o pacote `domain` com a entidade `Payment` — **sem nenhuma anotação Spring/JPA** dentro dela
* Define as portas: uma porta de entrada (`CreatePaymentUseCase`, interface com o método que o mundo externo chama) e portas de saída (`PaymentRepositoryPort`, interface que o domínio usa pra persistir, sem saber que é Postgres)
* Implementa os adaptadores: um adaptador de entrada (`PaymentController`, REST, chama o `CreatePaymentUseCase`) e um adaptador de saída (`PaymentRepositoryAdapter`, implementa `PaymentRepositoryPort` usando Spring Data JPA por baixo)
* **Regra de dependência**: os adaptadores dependem do domínio (implementam as portas dele); o domínio nunca depende dos adaptadores

### 🚨 Regras

* O pacote `domain` não pode importar nada de `org.springframework.*` nem `jakarta.persistence.*` — se compilar com esses imports lá dentro, a regra foi quebrada
* Teste real: escreva um teste unitário do `domain` que roda **sem subir o contexto Spring** — se precisar de `@SpringBootTest` pra testar a regra de negócio pura, a separação não está completa

### ❓ Perguntas

1. Qual a diferença entre uma porta de entrada (*inbound port*) e uma porta de saída (*outbound port*)? Dê um exemplo de cada no seu código.
2. Por que o domínio depende da porta (interface) e não da implementação concreta — isso já apareceu antes na série, em qual princípio do SOLID?
3. Compare com a arquitetura em camadas que você já usa (`Controller` → `Service` → `Repository`, do repositório 1): qual é a diferença real entre "Service depende de Repository" e "domínio depende de uma Port implementada por um Repository Adapter"?
4. Se amanhã você trocar Postgres por MongoDB, o que muda no domínio?

### 🎯 Avaliação (0 a 10)

* Domínio genuinamente livre de dependência de framework (comprovado por teste sem contexto Spring)
* Portas e adaptadores corretamente separados, com a direção de dependência certa
* Entendimento do "porquê", não só da estrutura de pastas

---

## 💳 PARTE 2 — Payment Service completo (Idempotency Key + Boleto real via Tecnospeed)

### 🎯 Objetivo

Aplicar tudo que já foi aprendido em outro repositório da série (Idempotency Key, Strategy, Factory) dentro da estrutura hexagonal da Parte 1 — e, pela primeira vez, integrar com uma API financeira real.

### 🧪 Desafio

* Implementa o `CreatePaymentUseCase` com a checagem de Idempotency Key (mesma lógica do repositório 1, agora vivendo no domínio/aplicação, não no `Controller`)
* Cria uma porta de saída `BoletoGatewayPort` (ex: `emitirBoleto(dados): BoletoResponse`) e um adaptador `TecnospeedBoletoAdapter` que implementa essa porta chamando a API real de homologação
* Credencial da Tecnospeed via variável de ambiente/`application-local.properties`

### 🚨 Regras

* **A idempotência precisa proteger a chamada externa também, não só o banco**: se a mesma `Idempotency-Key` chegar 2x, a segunda vez não pode gerar um segundo boleto na Tecnospeed — só devolver o resultado já obtido na primeira chamada
* **Teste real obrigatório**: chamada real à Tecnospeed (homologação) criando um boleto; segunda chamada com a mesma `Idempotency-Key` prova que não foi gerado um boleto duplicado na Tecnospeed (não só no seu banco)
* Pelo menos 1 teste provocando um erro real da API (dado inválido) e provando que seu sistema trata isso sem quebrar

### ❓ Perguntas

1. Por que a porta `BoletoGatewayPort` é definida pelo domínio e implementada pelo adaptador, e não o contrário?
2. O que acontece se a Tecnospeed responder com timeout **depois** de ter criado o boleto do lado dela, mas antes da sua resposta chegar? Sua Idempotency Key resolve esse caso? Por quê ou por que não?
3. Como você decidiu id/timeout/retry pra essa chamada externa — reaproveitou algo do Circuit Breaker que já construiu no repositório 1?

### 🎯 Avaliação (0 a 10)

* Idempotência funcionando de ponta a ponta, incluindo proteção contra chamada externa duplicada
* Integração real com a Tecnospeed, com pelo menos 1 cenário de erro real tratado
* Entendimento do limite entre o que a Idempotency Key resolve e o que ela não resolve (timeout pós-processamento é um caso limite genuíno, sem solução trivial)

---

## 🏦 PARTE 3 — Account Service

### 🎯 Objetivo

Modelar o domínio de contas/saldo como um segundo serviço, e resolver comunicação síncrona entre serviços de forma resiliente.

### 🧪 Desafio

* `account-service` com saldo por conta, também em estrutura hexagonal
* `payment-service` valida saldo/existência de conta chamando o `account-service` via REST antes de processar um pagamento
* Aplica Circuit Breaker nessa chamada (mesma ferramenta do repositório 1) — se o `account-service` cair, o `payment-service` não trava esperando indefinidamente

### ❓ Perguntas

1. Por que essa chamada é síncrona (bloqueia a resposta) e não assíncrona via fila, diferente do fluxo de notificação que você já construiu?
2. O que o fallback do Circuit Breaker deveria fazer aqui — recusar o pagamento, ou aceitar sem validar saldo? Qual é o risco de cada escolha, sendo um sistema financeiro?

### 🎯 Avaliação (0 a 10)

* Comunicação síncrona funcional com Circuit Breaker real
* Justificativa consciente do trade-off síncrono vs assíncrono nesse caso específico

---

## 🌐 PARTE 4 — Proxy-BFF + Auth-ACL

### 🎯 Objetivo

Completar os 4 serviços do artigo original: um ponto de entrada único (BFF) e um serviço de autenticação/autorização.

### 🧪 Desafio

* `proxy-bff`: roteia requisições pros serviços internos (`payment-service`, `account-service`) e compõe a resposta quando necessário
* `auth-acl`: valida token (JWT) e decide se o usuário tem permissão pra operação solicitada

### ❓ Perguntas

1. Qual a diferença entre um BFF (*Backend for Frontend*) e um API Gateway genérico?
2. Autenticação e autorização deveriam viver só no `auth-acl`, ou cada serviço também precisa validar por conta própria? Qual o risco de confiar 100% num único ponto?

### 🎯 Avaliação (0 a 10)

* BFF roteando corretamente pros 2 serviços internos
* Auth funcional com JWT, decisão justificada sobre onde validar

---

## ⚡ PARTE 5 — Cache com Redis

### 🎯 Objetivo

Reduzir leitura desnecessária no banco pra dados consultados com frequência (ex: saldo, extrato).

### 🧪 Desafio

* Cacheia a consulta de saldo/extrato no `account-service` com Redis, TTL definido
* Invalida o cache quando o saldo muda (não deixa o dado ficar desatualizado depois de um pagamento)

### ❓ Perguntas

1. Num sistema financeiro, o que é seguro cachear e o que nunca deveria ser cacheado? Por quê?
2. Qual estratégia de invalidação você escolheu (TTL curto, invalidação ativa no evento de mudança, ou as duas) — e por quê essa e não outra?

### 🎯 Avaliação (0 a 10)

* Cache funcionando, com ganho de performance mensurável (antes/depois)
* Estratégia de invalidação correta — sem saldo desatualizado sendo servido

---

## 📨 PARTE 6 — Mensageria Assíncrona (Notificação + Analytics)

### 🎯 Objetivo

Desacoplar o que não precisa bloquear a resposta do pagamento — reaproveitando o RabbitMQ que você já domina do repositório 1, sem reensinar do zero.

### 🧪 Desafio

* `payment-service` publica um evento de "pagamento criado"
* Um consumidor simples de analytics (pode ser um serviço novo, mínimo) processa esse evento de forma assíncrona

### ❓ Perguntas

1. O que diferencia essa mensageria da fila de notificação do repositório 1 — é o mesmo padrão, ou tem alguma diferença por ser dado de analytics em vez de notificação?

### 🎯 Avaliação (0 a 10)

* Evento publicado e consumido de forma assíncrona, sem bloquear a resposta do `POST /payments`

---

## 🔭 PARTE 7 — Observabilidade com OpenTelemetry + OTEL Collector

### 🎯 Objetivo

Rastrear uma requisição de ponta a ponta através dos múltiplos serviços — sem acoplar a aplicação a uma ferramenta específica de observabilidade.

### 🧪 Desafio

* Instrumenta os serviços com o SDK do OpenTelemetry
* Sobe um OTEL Collector local (Docker) recebendo os traces
* Exporta pra uma ferramenta de visualização **gratuita e local** (ex: Jaeger via Docker) — nada de X-Ray/Datadog pago
* Prova visualmente um trace completo: `proxy-bff` → `auth-acl` → `payment-service` → `account-service`

### ❓ Perguntas

1. O que é um trace e o que é um span, e como eles se relacionam?
2. Como o contexto do trace (trace ID) é propagado entre serviços diferentes (o que viaja no HTTP pra isso funcionar)?
3. Por que instrumentar via OpenTelemetry (padrão aberto) em vez de usar o SDK proprietário de uma ferramenta específica direto?

### 🎯 Avaliação (0 a 10)

* Trace completo visível de ponta a ponta entre os 4 serviços
* Entendimento de trace/span/propagação de contexto

---

## 🚀 PARTE 8 — Teste de Carga Real (contra o próprio sistema)

### 🎯 Objetivo

Não é medir "1 milhão" — é achar o teto **real** da sua infraestrutura e empurrar esse teto o máximo possível, de forma iterativa: mede → identifica o gargalo → ajusta essa peça → mede de novo. O JMeter em si consegue gerar volume alto; o que limita é o lado do servidor (mesmo o artigo original mostra o p99 disparando perto de 17k req/s, com toda uma infraestrutura de cluster por trás — aqui, numa máquina só, o teto chega bem antes disso).

### 🧪 Desafio

* Cria um adaptador **stub** de `BoletoGatewayPort` só pra esse teste (não bate na Tecnospeed em volume)
* Monta no JMeter as 3 jornadas do artigo, com a mesma distribuição: 60% consulta, 25% pagamento, 15% transferência
* **Roda o JMeter numa máquina separada do `payment-service`** (o segundo computador), em **modo não-GUI** (`jmeter -n -t teste.jmx`) — GUI mode é pesado demais e mente sobre o resultado, e gerar carga na mesma máquina que recebe a carga contamina a medição
* Aplica carga com ramp-up gradual (aumento progressivo, não um salto direto pro máximo)
* **Processo iterativo de otimização** — depois da primeira rodada, ataca o gargalo que apareceu primeiro e mede de novo. Alavancas reais, nessa ordem de impacto provável:
  1. Tamanho do pool de conexão do HikariCP (o padrão do Spring Boot, 10, satura rápido)
  2. Separação leitura/escrita — a consulta (60%, cacheada via Redis da Parte 5) tende a parar de ser gargalo depois do cache esquentar; pagamento/transferência (40%, `INSERT` real com checagem de índice único) é onde o teto de verdade costuma aparecer
  3. Heap/GC da JVM — pausas de coleta aparecem no p99, não na média
* Documenta cada rodada: o que mudou, o que melhorou, o que não melhorou

### 🚨 Regras

* **Nunca gerar volume de carga contra a API real da Tecnospeed** — só contra o stub
* Números têm que ser medidos de verdade (JMeter rodando), não estimados
* Load generator e servidor em máquinas separadas — não vale medir os dois competindo pelo mesmo hardware

### ❓ Perguntas

1. Qual foi o teto real de throughput que você conseguiu depois de todas as rodadas de otimização — e quanto ele melhorou entre a primeira e a última rodada?
2. Em cada rodada, o que estava saturando primeiro (CPU, pool de conexão, banco, GC)? Como você identificou isso (que métrica/log te mostrou)?
3. Depois de otimizar tudo que dava pra otimizar numa única instância, o que faria o teto subir de verdade — mais tuning, ou mais instâncias (horizontal)? Por quê?
4. Por que usar um stub pro teste de volume e a API real só pra poucas chamadas de corretude (Parte 9) — o que aconteceria se você invertesse isso?

### 🎯 Avaliação (0 a 10)

* Teste de carga real, com jornadas realistas (não só `POST` bruto), load generator em máquina separada
* Pelo menos 2 rodadas de otimização documentadas (antes/depois de atacar um gargalo identificado)
* Números documentados com evidência (gráficos/tabelas do JMeter)
* Diagnóstico correto do gargalo real em cada rodada, e entendimento de quando tuning vertical para de ajudar

---

## ✅ PARTE 9 — Corretude com API Real da Tecnospeed (baixo volume)

### 🎯 Objetivo

Provar que idempotência e tratamento de erro funcionam contra uma API real e imprevisível — não contra dado sintético que sempre se comporta como esperado.

### 🧪 Desafio

* Algumas dezenas de chamadas reais à Tecnospeed (não centenas, não milhares)
* Pelo menos 1 cenário de sucesso completo (boleto emitido, idempotência provada)
* Pelo menos 1 cenário de erro proposital (dado inválido) — documenta a resposta real da API e como seu sistema reagiu

### ❓ Perguntas

1. O comportamento real da API bateu com o que você esperava pela documentação, ou teve alguma surpresa?
2. Que tipo de erro você não tinha tratado até testar contra a API real?

### 🎯 Avaliação (0 a 10)

* Evidência real de chamadas à Tecnospeed (sucesso e erro)
* Tratamento de erro real, não só do caminho feliz

---

## 📊 PARTE 10 — SLOs, Dashboards e p95/p99

### 🎯 Objetivo

Transformar os números da Parte 8 num SLO de verdade, com dashboard visível — igual ao raciocínio do artigo original.

### 🧪 Desafio

* Sobe Prometheus + Grafana local (Docker, grátis)
* Dashboard com throughput, latência p95/p99 e taxa de erro, alimentado pelos dados reais medidos
* Define um SLO (ex: "99% dos pagamentos devem responder em menos de X ms", com X baseado no que sua máquina realmente aguenta) e simula uma violação de propósito (sobrecarregando o sistema) pra ver o dashboard reagir

### ❓ Perguntas

1. Qual a diferença entre SLO e SLA?
2. Por que p99 é uma métrica mais útil que a média nesse contexto? Reproduza o raciocínio do artigo com seus próprios números.

### 🎯 Avaliação (0 a 10)

* Dashboard funcional com dados reais
* SLO definido de forma coerente com o que foi medido, não um número arbitrário

---

## 💵 PARTE EXTRA — Análise de Custo

### 🎯 Objetivo

Aplicar o raciocínio de custo do artigo aos **seus** números reais medidos na Parte 8 — não ao "1 milhão" teórico.

### 🧪 Desafio

* A partir do throughput real que sua máquina aguentou, projeta: quantas requisições isso representaria rodando 24h/dia por 30 dias
* Estima volume de dados trafegados (tamanho médio de payload × total de requisições) e volume de log gerado
* Reflete: se você precisasse escalar esse número real até chegar em 1 milhão/minuto, quantas instâncias a mais isso exigiria (proporcionalmente)?

### ❓ Perguntas

1. Pequenos detalhes de payload/log, na escala do artigo, viram quantos GB/TB por mês? Refaça a conta do artigo com seus próprios números de payload.
2. Por que "adicionar mais containers" sozinho não é uma estratégia de custo sustentável nessa escala?

### 🎯 Avaliação (0 a 10)

* Cálculo real, baseado nos números medidos, não em estimativa solta
* Entendimento de que custo é uma decisão de arquitetura, não só de infraestrutura
