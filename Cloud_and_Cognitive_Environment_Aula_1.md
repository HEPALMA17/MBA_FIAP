## 🟢 Nível 1 — Básico: Consolidando os Fundamentos
### Exercício 1.1 — Mapeamento de modelos de serviço
Para cada serviço, identifique-se como IaaS, PaaS, SaaS ou FaaS. Justifique em uma frase.



| Serviço | Modelo (IaaS/PaaS/SaaS/FaaS) | Justificativa |
| :--- | :--- | :--- |
| Gmail | SaaS | Software pronto para uso final acessado pelo navegador, sem necessidade de gerenciar servidores ou código. |
| Amazon EC2 | IaaS | Fornece máquinas virtuais (computação e rede), mas exige que você gerencie o sistema operacional, atualizações e aplicações. |
| AWS Elastic Beanstalk (hospedar uma API) | PaaS | Abstrai o gerenciamento da infraestrutura e do sistema operacional, permitindo focar apenas no deploy do código da aplicação. |
| AWS Lambda | FaaS | Executa trechos de código sob demanda orientados a eventos, sem provisionamento de servidores e com cobrança por execução. |
| Amazon RDS para PostgreSQL | PaaS | Banco de dados gerenciado onde a AWS cuida do provisionamento, backups e patches, enquanto você administra apenas os dados e schemas. |
| Salesforce CRM | SaaS | Plataforma completa de gestão de clientes entregue pronta pela nuvem, sem gestão de infraestrutura pelo usuário. |
| Google Kubernetes Engine (GKE) | PaaS/IaaS (Híbrido) | O Google gerencia o control plane do Kubernetes (PaaS), mas você ainda configura e administra os nós de trabalho e pods (IaaS). |
| Amazon S3 | PaaS (Storage gerenciado) | Serviço de armazenamento de objetos totalmente gerenciado, no qual você controla apenas os buckets, dados e permissões de acesso. |
| Amazon Bedrock | SaaS / API como serviço | Disponibiliza modelos fundacionais de IA prontos para consumo direto via API, sem exigir gerenciamento de infraestrutura ou treinamento base. |
**


---

### Exercício 1.2 — Os 6 Rs na prática

Leia cada cenário e escolha o R de migração mais adequado (Rehost, Replatform, Refactor, Repurchase, Retire, Retain). Justifique.

* **Cenário A — Rehost (*Lift and Shift*):** Como o sistema é antigo (2008), não possui documentação, tem baixa disponibilidade de conhecimento técnico interno e a prioridade é migrar rapidamente para ganhar elasticidade, mover a máquina virtual diretamente para a nuvem (ex.: Amazon EC2) sem alterar o código é a abordagem mais segura e ágil.
* **Cenário B — Retire (Desativar):** O sistema possui baixíssima utilização (menos de 5 usuários ativos por mês) e dados raramente consultados; desativá-lo (arquivando os dados históricos caso haja exigência legal) elimina custos desnecessários de manutenção e infraestrutura.
* **Cenário C — Refactor / Re-architect (Refatorar):** A empresa decidiu reescrever a aplicação monolítica para uma arquitetura nativa em nuvem baseada em microsserviços, Kubernetes e eventos, visando escalabilidade e agilidade de entrega.
* **Cenário D — Repurchase (Recomprar):** Como um produto SaaS de mercado já atende 90% das necessidades por um custo menor do que manter um legado interno de 15 anos, substituir a solução própria por uma assinatura comercial é a escolha mais eficiente.
* **Cenário E — Retain (Manter):** Devido a restrições regulatórias e de conformidade impostas pelo Banco Central, os dados precisam permanecer em ambiente *on-premise*, mantendo a carga de trabalho localmente enquanto outras partes do ecossistema podem ser integradas via arquitetura híbrida.

---

### Exercício 1.3 — Calculando o impacto do SLA

Sistema de e-commerce com SLA de 99,9%.

a) Quantas horas de inatividade por ano? b) Se processa R$ 50.000/hora em vendas, qual o impacto financeiro máximo por ano? c) Para reduzir o impacto para menos de R$ 50.000/ano, qual SLA mínimo seria necessário?

> **Fórmulas úteis:** Downtime anual (horas) = 8.760 × (1 - SLA/100) | Downtime mensal (min) = 43.800 × (1 - SLA/100)

* **a) Horas de inatividade por ano (SLA 99,9%):**
  $$\text{Downtime anual} = 8.760 \times \left(1 - \frac{99,9}{100}\right) = 8.760 \times 0,001 = \mathbf{8,76\text{ horas/ano}}\text{ (aproximadamente 8h 45min 36s)}$$

* **b) Impacto financeiro máximo por ano:**
  $$\text{Impacto financeiro} = 8,76\text{ horas} \times \text{R\$ } 50.000/\text{hora} = \mathbf{\text{R\$ } 438.000,00/\text{ano}}$$

* **c) SLA mínimo para reduzir o impacto para menos de R$ 50.000/ano:**
  * Primeiro, calcula-se o limite máximo de horas de indisponibilidade permitido:
    $$\text{Downtime máximo} < \frac{\text{R\$ } 50.000}{\text{R\$ } 50.000/\text{hora}} \implies \mathbf{< 1\text{ hora/ano}}$$
  * Aplicando na fórmula de SLA anual:
    $$8.760 \times \left(1 - \frac{\text{SLA}}{100}\right) < 1 \implies 1 - \frac{\text{SLA}}{100} < \frac{1}{8.760} \approx 0,00011415$$
    $$\text{SLA} > (1 - 0,00011415) \times 100 = \mathbf{99,9886\%}$$
  * **Conclusão:** Seria necessário um SLA mínimo de **99,99%** ("quatro noves"), o que limitaria a inatividade anual a **0,876 horas** (cerca de 52 minutos e 33 segundos), gerando um impacto máximo de **R$ 43.800,00/ano**.
