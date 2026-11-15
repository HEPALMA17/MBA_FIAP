## 🟢 Nível 1 — Básico: Consolidando os Fundamentos

### Exercício 1.1 — Tipos de armazenamento

Para cada cenário, escolha **Object Storage**, **File Storage** ou **Block Storage** e justifique em uma frase.

| Cenário | Tipo | Justificativa |
| :--- | :--- | :--- |
| Hospedar imagens de produtos do e-commerce QC (5M de SKUs) | Object Storage (Amazon S3) | Suporta altíssimo volume e escalabilidade ilimitada com acesso direto via HTTP/CDN, sem exigir sistema de arquivos hierárquico. |
| Disco onde roda o sistema operacional de uma VM de banco | Block Storage (Amazon EBS) | Oferece baixíssima latência e alto desempenho de I/O atrelado diretamente a uma única instância virtual para o sistema operacional e dados ativos. |
| Pasta compartilhada entre 10 VMs de um time de DevOps | File Storage (Amazon EFS) | Permite montar o mesmo sistema de arquivos simultaneamente em múltiplas VMs (ex.: `/mnt/dados`) com controle de permissões POSIX. |
| Backup mensal de bancos de dados (retenção de 7 anos) | Object Storage (S3 Glacier Deep Archive) | Oferece custo extremamente reduzido para retenção de longo prazo e conformidade regulatória em arquivos acessados raramente. |
| Armazenamento de modelos `.pkl` do time de ML para servir | Object Storage (Amazon S3) | Facilita o download rápido via API/HTTP pelos serviços de inferência e permite versionamento nativo dos artefatos de modelo. |
| Dump diário de logs de aplicação para análise futura | Object Storage (S3 com Lifecycle) | Permite aplicar políticas de ciclo de vida (`Standard` → `Standard-IA` → `Glacier`) e realizar consultas serverless diretamente via Amazon Athena. |
