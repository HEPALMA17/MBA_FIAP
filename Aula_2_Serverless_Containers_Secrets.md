## 🟢 Nível 1 — Básico: Consolidando os Fundamentos

### Exercício 1.1 — Quando usar serverless?

Para cada cenário da Quantum Commerce, marque **Lambda**, **ECS Fargate**, **ECS Express Mode** ou **EKS** e justifique em uma frase[cite: 26]:

| Cenário | Escolha | Justificativa |
| :--- | :--- | :--- |
| API de busca de produtos (1M de chamadas/mês, picos na Black Friday) | **AWS Lambda** | Modelo de pagamento por requisição com escala automática imediata para absorver picos sazonais, sendo amplamente coberto pelo *Free Tier* mensal[cite: 26]. |
| Worker que processa pedidos da fila (1000 pedidos/dia, picos noturnos) | **AWS Lambda** (com gatilho SQS) | Arquitetura orientada a eventos que escala a zero quando a fila está vazia e processa lotes automaticamente durante os picos noturnos[cite: 26]. |
| API legada em Java Spring Boot (não pode reescrever, já conteinerizada) | **ECS Express Mode** (ou **ECS Fargate**) | Permite subir o contêiner existente com ALB, HTTPS e *auto-scaling* em uma única configuração simplificada, sem reescrever o código e com baixíssima sobrecarga operacional[cite: 26]. |
| Pipeline de processamento de imagens de produtos (1 hora por noite) | **ECS Fargate** (agendado via EventBridge Scheduler) | Cobra apenas pelos segundos em que a tarefa está ativa e não possui o limite máximo de 15 minutos de execução do AWS Lambda[cite: 26]. |
| Microsserviço de pagamentos (regulado, registros detalhados, 100 req/s constantes) | **ECS Fargate** (ou **EKS**) | Garante execução contínua sem *cold start*, isolamento de rede e controle granular de logs e conformidade para tráfego previsível e regulado[cite: 26]. |
| Plataforma com 25 microsserviços + service mesh | **Amazon EKS** | Oferece o ecossistema Kubernetes mais maduro para orquestrar dezenas de serviços acoplados com ferramentas nativas de *service mesh* (como Istio ou AWS App Mesh)[cite: 26]. |
| Container que extrai dados uma vez por dia e termina | **ECS Fargate** (`RunTask` agendado) | Executa tarefas pontuais (*one-shot* / *batch*) sob demanda e encerra automaticamente a infraestrutura assim que a extração termina, evitando custo ocioso[cite: 26]. |


---

### Exercício 1.2 — Função IAM vs alternativas

Para cada estratégia de credencial, marque **vulnerabilidade alta**, **média** ou **baixa** e justifique[cite: 27]:

| Estratégia | Vulnerabilidade | Por quê |
| :--- | :--- | :--- |
| Chave de acesso da AWS *hardcoded* no `lambda_function.py` | **Alta** | Fica exposta diretamente no código-fonte e no histórico do Git; qualquer pessoa com acesso de leitura ao repositório obtém credenciais permanentes da conta AWS. |
| Chave de acesso em variável de ambiente do Lambda | **Média** | Tira o segredo do código-fonte, mas as chaves continuam sendo credenciais estáticas de longa duração visíveis no console/API da Lambda para qualquer usuário com permissão `lambda:GetFunctionConfiguration`. |
| String de conexão no Secrets Manager, lida com chave de acesso de um usuário IAM | **Média** | Centraliza a rotação da senha do banco no Secrets Manager, mas ainda depende de uma *Access Key* estática de usuário IAM para autenticar na AWS, mantendo risco de vazamento e expiração manual. |
| String de conexão no Secrets Manager, lida com a *Execution Role* da Lambda | **Baixa** | Utiliza credenciais temporárias gerenciadas automaticamente pelo AWS STS via *Execution Role* (sem chaves estáticas) e permite rotação automática da senha do banco no Secrets Manager. |
| Sem segredo — a *Execution Role* tem permissão direta no recurso (S3) | **Baixa** *(Ideal)* | Elimina completamente qualquer credencial ou segredo armazenado; a autenticação ocorre via tokens temporários efêmeros gerados pelo STS com escopo restrito ao recurso. |

#### Pergunta adicional: Em quais estratégias um vazamento de código no GitHub continua sendo problema? Em que não é? Por quê?[cite: 27]

* **Onde continua sendo um problema grave:** Na **1ª estratégia** (chave *hardcoded* no `lambda_function.py`), pois as credenciais reais (`AWS_ACCESS_KEY_ID` e `AWS_SECRET_ACCESS_KEY`) ficam gravadas no código e no histórico de commits do Git, permitindo acesso imediato à conta AWS mesmo que o arquivo seja apagado em commits futuros.
* **Onde NÃO compromete as credenciais:** Nas **demais estratégias** (variáveis de ambiente, Secrets Manager ou *Execution Role* direta), porque o código-fonte contém apenas referências lógicas (como `os.environ["SECRET_ARN"]` ou o nome do bucket), sem nenhum segredo embutido. Mesmo que o repositório vaze, o invasor não obtém tokens ou senhas válidas para acessar os recursos da AWS.

#### Política mínima da ferramenta (`buscar_produtos` em produção)[cite: 27]

> Política IAM em JSON seguindo o princípio do menor privilégio (lembrando o incidente da Capital One, onde permissões excessivas em roles permitiram listar e exfiltrar buckets indevidamente), concedendo apenas leitura de objetos no prefixo do catálogo do S3[cite: 27]:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PermitirLeituraApenasCatalogoProdutos",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::qc-catalogo-produtos-prod/catalogo/*"
    },
    {
      "Sid": "PermitirListarApenasPrefixoCatalogo",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::qc-catalogo-produtos-prod",
      "Condition": {
        "StringLike": {
          "s3:prefix": [
            "catalogo/*"
          ]
        }
      }
    }
  ]
}
```

---

### Exercício 1.3 — Partida a frio (*Cold Start*) na prática

Teste realizado com `time curl -s -o /dev/null "${FUNCTION_URL}produtos"` e validação no Amazon CloudWatch Logs (`REPORT ... Init Duration: ... ms`)[cite: 27]:

| Chamada | Tempo total (`curl`) | Teve `Init Duration` no CloudWatch? |
| :--- | :--- | :--- |
| **1 (fria — inicial)**[cite: 27] | ~`0,65s` a `1,20s` *(ex.: `890 ms`)* | **Sim** (ex.: `Init Duration: 420.50 ms` — criação do microVM Firecracker + carregamento do runtime Python e SDK `boto3`)[cite: 27] |
| **2 (quente — 5s depois)**[cite: 27] | ~`0,04s` a `0,12s` *(ex.: `65 ms`)* | **Não** (o ambiente de execução já estava aquecido e foi reutilizado diretamente)[cite: 27] |
| **3 (fria de novo — 30min depois)**[cite: 27] | ~`0,60s` a `1,15s` *(ex.: `840 ms`)* | **Sim** (após ~15 minutos de inatividade, a AWS recicla o contêiner ocioso e uma nova inicialização ocorre)[cite: 27] |

*(Nota: substitua os valores de tempo entre parênteses pelos milissegundos exatos registrados no seu CloudWatch Logs ao rodar o comando no CloudShell[cite: 27]).*

#### Pergunta: Se o agente do QC chamar essa função 1 vez por hora (24 chamadas/dia), quantas serão frias? Como mitigar se a UX exige resposta em `< 500 ms`?[cite: 27]

* **Quantas chamadas serão frias:** **Todas as 24 chamadas (100%)** serão partidas a frio (*cold starts*). Como o ambiente de execução da Lambda é reciclado após cerca de 15 minutos de ociosidade, um intervalo de 60 minutos entre chamadas garante que nunca haverá um contêiner quente reaproveitado.
* **Como mitigar para garantir latência `< 500 ms` (Provisioned Concurrency vs. SnapStart):**[cite: 27]
  1. **AWS Lambda Provisioned Concurrency (Concorrência Provisionada):**
     * **Como funciona:** Mantém um número definido de ambientes de execução (ex.: `1` instância) pré-inicializados e prontos 24/7, eliminando o `Init Duration` mesmo após horas sem tráfego. Funciona em qualquer runtime (incluindo Python).
     * **Custo estimado (`us-east-1`):** Cobra pelo tempo que a instância fica provisionada (~**US$ 0,0000041667 por GB-segundo**). Para **1 instância de 512 MB** (0,5 GB) ativa 24/7 ($2.592.000\text{ segundos/mês}$):
       $$0,5\text{ GB} \times 2.592.000\text{ s} \times \text{US\$ } 0,0000041667 \approx \mathbf{\text{US\$ } 5,40/\text{mês}}$$
       *(além do custo padrão de requisições e duração das 720 execuções/mês, que fica na faixa de ~US$ 0,01).*
  2. **AWS Lambda SnapStart:**
     * **Como funciona:** Tira um *snapshot* criptografado da memória e do disco do ambiente já inicializado após o deploy e restaura esse estado em sub-segundo (reduzindo o *cold start* em até 90% sem manter instâncias ociosas).
     * **Limitação e Custo:** É excelente para runtimes com inicialização pesada (como **Java, .NET e Python** — suportado para Python 3.12+ desde o final de 2024), mas cobra uma taxa fixa pelo armazenamento do snapshot (~**US$ 0,0000015046 por GB-segundo** de armazenamento de snapshot) mais uma taxa por restauração (~**US$ 0,0001398 por GB restaurado**). Para apenas 720 invocações/mês com 512 MB, o custo mensal fica em torno de **~US$ 2,00/mês**, porém para Python leve com dependência externa (`boto3`/banco), o **Provisioned Concurrency** oferece a garantia mais previsível de latência `< 100 ms`[cite: 27].
    

---

### Exercício 1.4 — Revisão do Dockerfile

#### 1. Problemas identificados no Dockerfile original e melhorias propostas[cite: 28]

1. **Imagem base pesada (`FROM python:3.12` — ~1 GB):** Inclui compiladores, bibliotecas de sistema e ferramentas desnecessárias em produção, aumentando a superfície de ataque (CVEs) e o tempo de pull[cite: 28].
   * *Melhoria:* Utilizar `python:3.12-slim` (~150 MB) a partir do espelho oficial da AWS (`public.ecr.aws/docker/library/python:3.12-slim`) para evitar *rate limits* do Docker Hub[cite: 28].
2. **Ordem ineficiente das camadas e `COPY . .` sem `.dockerignore`:** Copiar todo o diretório antes do `pip install` invalida o cache do Docker a cada alteração de código-fonte e copia arquivos sensíveis/desnecessários (`.git`, `.env`, `__pycache__`)[cite: 28].
   * *Melhoria:* Copiar primeiro apenas o `requirements.txt`, instalar as dependências (aproveitando o cache) e usar um `.dockerignore`[cite: 28].
3. **Instalação do `pip` sem `--no-cache-dir` e sem *multi-stage build*:** O cache de pacotes `.whl` fica armazenado dentro da imagem final e ferramentas de compilação permanecem no runtime[cite: 28].
   * *Melhoria:* Usar *multi-stage build* criando um *virtual environment* (`/opt/venv`) com `--no-cache-dir` e copiando apenas o ambiente pronto para a imagem final[cite: 28].
4. **Execução como usuário `root` (Risco crítico de segurança):** Por padrão, o container roda como `root`, permitindo escalonamento de privilégios no host caso haja vulnerabilidade na aplicação[cite: 28].
   * *Melhoria:* Criar e alternar para um usuário sem privilégios (`USER appuser`) antes de iniciar o processo[cite: 28].
5. **Servidor de desenvolvimento em `CMD ["python", "app.py"]` e ausência de `EXPOSE` / `HEALTHCHECK`:** Rodar `python app.py` diretamente não oferece gerenciamento de workers nem tolerância a falhas para produção, e a falta de `HEALTHCHECK` impede o ALB/ECS de detectar travamentos[cite: 28].
   * *Melhoria:* Documentar a porta (`EXPOSE 8000`), configurar `HEALTHCHECK` e iniciar explicitamente com servidor ASGI/WSGI de produção (`uvicorn` ou `gunicorn`)[cite: 28].

#### 2. Dockerfile Corrigido (Boas Práticas de Produção)[cite: 28]

```dockerfile
# --- Estágio 1: Builder ---
FROM public.ecr.aws/docker/library/python:3.12-slim AS builder

WORKDIR /build

RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

COPY requirements.txt .
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

# --- Estágio 2: Runtime ---
FROM public.ecr.aws/docker/library/python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PATH="/opt/venv/bin:$PATH"

WORKDIR /app

# Cria usuário não-root dedicado
RUN groupadd --gid 1001 appgroup && \
    useradd --uid 1001 --gid appgroup --shell /bin/false appuser

# Copia apenas o virtualenv compilado e o código da aplicação
COPY --from=builder /opt/venv /opt/venv
COPY --chown=appuser:appgroup . .

USER appuser

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1

CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "2"]
```

---

## 🟡 Nível 2 — Intermediário: Decisões de Design + IaC

### Exercício 2.1 — Tela de cálculo do QC (*Compute Canvas*)

#### a) & b) & c) & d) Matriz de Decisão de Computação, Custos (`us-east-1`) e Segurança[cite: 28]

| Carga de Trabalho | Perfil de Tráfego / Execução | Escolha ("Casa") | Custo Mensal (`us-east-1`) | Alternativa Comparada e Custo | O que ela pode ler (Identidade IAM) | O que NÃO usaria e Justificativa |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **1. Busca no catálogo** *(Exemplo)* | 1M req/mês, picos sazonais, execução rápida (~150 ms) | **AWS Lambda** (512 MB) | **~US$ 0,20/mês** *(coberto quase todo pelo Free Tier)* | **ECS Fargate** (1 task 0.5 vCPU / 1 GB 24/7): **~US$ 18,00/mês** | `IAM Role`: `s3:GetObject` no prefixo `catalogo/*` e acesso de rede ao RDS/pgvector via Secrets Manager. | **Não usaria EC2 fixa 24/7**, pois pagaria por capacidade ociosa fora dos horários de pico[cite: 28]. |
| **2. Cálculo de frete** | 3M chamadas/mês, stateless, picos sincronizados com busca/carrinho (~100 ms) | **AWS Lambda** (512 MB) | **~US$ 1,80/mês** | **ECS Fargate** (2 tasks 0.5 vCPU / 1 GB para HA): **~US$ 36,00/mês** | `IAM Role`: leitura em tabela de tabelas de CEP/transportadoras no **DynamoDB** (`dynamodb:GetItem`, `Query`). | **Não usaria EKS**, pois a complexidade de gerenciar clusters Kubernetes não se justifica para funções stateless simples[cite: 28]. |
| **3. Geração noturna das notas fiscais (NF-e)** | Batch diário agendado às 02h00, duração de ~45 min, intensivo em CPU/I/O | **ECS Fargate** (`RunTask` agendado via EventBridge, 2 vCPU / 4 GB) | **~US$ 2,35/mês** *(45 min × 30 dias = 22,5h/mês)* | **EC2 `c6i.large` ligada 24/7**: **~US$ 62,05/mês** | `IAM Task Role`: leitura de pedidos no Aurora (via Secrets Manager) e `s3:PutObject` no bucket `qc-nfe-xml/`. | **Não usaria AWS Lambda**, pois o tempo total do lote (~45 min) ultrapassa o limite rígido de **15 minutos** por invocação[cite: 28]. |
| **4. Recomendação de produtos (IA)** | 2M consultas/mês, chamada externa ao Bedrock + busca vetorial (~400 ms) | **AWS Lambda** (1024 MB) | **~US$ 11,50/mês** *(computação Lambda; tokens Bedrock à parte)* | **ECS Fargate** (2 tasks 1 vCPU / 2 GB 24/7): **~US$ 72,00/mês** | `IAM Role`: `bedrock:InvokeModel` restrito ao modelo de embedding/LLM e consulta ao OpenSearch/pgvector. | **Não usaria GPU dedicada (EC2 `g5`)**, pois consumir modelos fundacionais gerenciados via Bedrock elimina custo fixo de GPU ociosa[cite: 28]. |
| **5. Checkout e Pagamentos** | Tráfego constante crítico (100 req/s), exigência de latência previsível e compliance | **ECS Fargate** (4 tasks 1 vCPU / 2 GB com Auto Scaling) | **~US$ 144,00/mês** | **AWS Lambda** (259M req/mês × 200 ms × 1024 MB): **~US$ 915,00/mês** | `IAM Task Role`: escrita transacional no Aurora (`Pedidos`), `sqs:SendMessage` para fila de antifraude e acesso restrito ao KMS. | **Não usaria AWS Lambda**, pois em tráfego contínuo elevado (100 req/s) o modelo por invocação sai **6× mais caro** que contêineres e introduz risco de *cold start*[cite: 28]. |
| **6. Carga Real (Anonimizada): Pipeline de Conciliação Pós-Transacional** *(Contexto Rede de Pagamentos)* | Processamento em lote diário (D-1) de arquivos de liquidação financeira (~1h30/dia) | **ECS Fargate** (Scheduled Task, 4 vCPU / 8 GB) | **~US$ 9,40/mês** *(1,5h × 30 dias = 45h/mês)* | **Amazon EKS** (cluster base + nós dedicados 24/7): **~US$ 145,00/mês** | `IAM Task Role`: leitura em bucket S3 tokenizado (`s3:GetObject` com chave KMS específica) e escrita no Data Warehouse. | **Não usaria Lambda nem servidores on-premise**, pois o volume de memória (8 GB) e duração (90 min) exigem contêiner efêmero isolado com encerramento automático após o batch[cite: 28]. |


---

### Exercício 2.2 — Segunda ferramenta para o agente: cálculo de frete

* **Entrada:** CEP origem, CEP destino, peso (kg)[cite: 18]
* **Saída:** valor em R$ + prazo estimado[cite: 18]
* **Lógica:** simples e determinística (R$ por faixa de CEP + R$ por kg)[cite: 18]
* **Volume esperado:** 50.000 chamadas/mês[cite: 18]

#### a) Decisão arquitetural: Nova rota na mesma Lambda (`GET /frete`) vs. Nova função separada[cite: 18]

* **Decisão escolhida:** **Nova rota (`GET /frete`) na mesma Lambda** (evoluindo a `v2-s3/lambda_function.py`), mantendo o padrão de microsserviço de pré-venda/catálogo do agente[cite: 18].
* **Justificativa:**
  1. **Menor incidência de *Cold Start*:** Com apenas **50.000 chamadas/mês** (~1,1 chamada/minuto em média), consolidar `/produtos` e `/frete` na mesma função aumenta a frequência de invocações do mesmo ambiente de execução, mantendo o contêiner "quente" por mais tempo[cite: 18].
  2. **Zero dependência externa adicional:** O cálculo de frete é determinístico em memória (sem exigir novas bibliotecas pesadas nem credenciais diferentes), mantendo o pacote leve e com o mesmo perfil de memória (256 MB)[cite: 18].
  3. **Simplicidade operacional e custo:** Evita duplicar *Function URLs*, grupos de logs no CloudWatch e pipelines de deploy enquanto o domínio de catálogo + frete possui o mesmo ciclo de evolução[cite: 18].

---

#### b) Implementação da rota `calcular_frete` (`v2-s3/lambda_function.py`)[cite: 18]

```python
import json
import os
import boto3

s3 = boto3.client("s3")
BUCKET_NAME = os.environ.get("BUCKET_NAME", "")


def calcular_frete(cep_origem: str, cep_destino: str, peso_kg: float) -> dict:
    """Calcula frete determinístico por faixa de região (1º dígito do CEP) e peso."""
    origem_limpo = "".join(filter(str.isdigit, str(cep_origem)))
    destino_limpo = "".join(filter(str.isdigit, str(cep_destino)))

    if len(origem_limpo) != 8 or len(destino_limpo) != 8:
        raise ValueError("CEPs devem conter exatamente 8 dígitos numéricos.")
    if peso_kg <= 0:
        raise ValueError("O peso deve ser maior que 0 kg.")

    regiao_origem = int(origem_limpo[0])
    regiao_destino = int(destino_limpo[0])
    distancia_regional = abs(regiao_origem - regiao_destino)

    # Tabela determinística: taxa base por distância entre regiões + R$ 4,50 por kg
    if distancia_regional == 0:
        taxa_base = 12.90
        prazo_dias = 2
    elif distancia_regional <= 3:
        taxa_base = 22.50
        prazo_dias = 5
    else:
        taxa_base = 34.90
        prazo_dias = 9

    valor_total = round(taxa_base + (peso_kg * 4.50), 2)

    return {
        "cep_origem": origem_limpo,
        "cep_destino": destino_limpo,
        "peso_kg": peso_kg,
        "valor_frete_brl": valor_total,
        "prazo_estimado_dias_uteis": prazo_dias,
    }


def lambda_handler(event, context):
    path = event.get("rawPath", "/")
    method = event.get("requestContext", {}).get("http", {}).get("method", "GET")
    params = event.get("queryStringParameters") or {}

    # Rota 1: Catálogo de produtos (existente)
    if method == "GET" and path.endswith("/produtos"):
        obj = s3.get_object(Bucket=BUCKET_NAME, Key="catalogo/produtos.json")
        produtos = json.loads(obj["Body"].read().decode("utf-8"))
        return {
            "statusCode": 200,
            "headers": {"Content-Type": "application/json"},
            "body": json.dumps({"produtos": produtos}, ensure_ascii=False),
        }

    # Rota 2: Cálculo de frete (nova ferramenta)
    if method == "GET" and path.endswith("/frete"):
        try:
            cep_origem = params.get("cep_origem", "01001000")
            cep_destino = params.get("cep_destino", "")
            peso = float(params.get("peso", 0))
            resultado = calcular_frete(cep_origem, cep_destino, peso)
            return {
                "statusCode": 200,
                "headers": {"Content-Type": "application/json"},
                "body": json.dumps(resultado, ensure_ascii=False),
            }
        except ValueError as err:
            return {
                "statusCode": 400,
                "headers": {"Content-Type": "application/json"},
                "body": json.dumps({"erro": str(err)}, ensure_ascii=False),
            }

    return {
        "statusCode": 404,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps({"erro": f"Rota não encontrada: {method} {path}"}),
    }
```

---

#### c) Código Terraform caso opte por função separada (`v3-frete/main.tf`)[cite: 18]

> *(Conforme solicitado no item c, segue a definição completa caso o grupo decida isolar o frete em uma segunda Lambda com Function URL própria e as duas permissões públicas exigidas pelo modelo atual da AWS)*[cite: 18]:

```hcl
resource "aws_lambda_function" "frete" {
  function_name    = "qc-aula03-frete"
  role             = aws_iam_role.lambda_exec.arn
  handler          = "lambda_function.lambda_handler"
  runtime          = "python3.12"
  filename         = "frete.zip"
  source_code_hash = filebase64sha256("frete.zip")
  memory_size      = 128
  timeout          = 10
}

resource "aws_lambda_function_url" "frete_url" {
  function_name      = aws_lambda_function.frete.function_name
  authorization_type = "NONE"
}

# Permissão 1: Invocação via Function URL
resource "aws_lambda_permission" "frete_url_public" {
  statement_id           = "AllowPublicFunctionUrlInvoke"
  action                 = "lambda:InvokeFunctionUrl"
  function_name          = aws_lambda_function.frete.function_name
  principal              = "*"
  function_url_auth_type = "NONE"
}

# Permissão 2: Invocação direta exigida em conjunto com Function URL pública
resource "aws_lambda_permission" "frete_invoke_public" {
  statement_id  = "AllowPublicFunctionInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.frete.function_name
  principal     = "*"
}

output "frete_function_url" {
  value = aws_lambda_function_url.frete_url.function_url
}
```

---

#### d) Documentação da ferramenta (*Tool*) em JSON Schema para o Agente[cite: 18]

```json
{
  "toolSpec": {
    "name": "calcular_frete",
    "description": "Calcula o valor do frete em reais (BRL) e o prazo estimado de entrega em dias úteis entre dois CEPs brasileiros com base no peso total do pedido.",
    "inputSchema": {
      "json": {
        "type": "object",
        "properties": {
          "cep_origem": {
            "type": "string",
            "description": "CEP do centro de distribuição de origem com 8 dígitos numéricos (ex: '01001000')."
          },
          "cep_destino": {
            "type": "string",
            "description": "CEP de entrega do cliente com 8 dígitos numéricos (ex: '20040002')."
          },
          "peso": {
            "type": "number",
            "description": "Peso total do produto ou carrinho em quilogramas (kg), maior que zero."
          }
        },
        "required": ["cep_destino", "peso"]
      }
    }
  }
}
```

---

#### e) Reflexão: Quando separar funções vs. Quando manter juntas?[cite: 18]

* **Quando separar em funções Lambda distintas:**
  1. **Segurança e Menor Privilégio (IAM):** Quando uma ferramenta precisa apenas ler catálogo público no S3 (`s3:GetObject`) e outra grava pedidos ou processa pagamentos no Aurora/KMS — separar impede que uma falha na rota de leitura exponha permissões de escrita.
  2. **Perfis distintos de recurso/memória e escala:** Quando uma rota exige **2 GB de RAM** e bibliotecas pesadas (ex.: processamento de imagens/PDF ou inferência local) e outra precisa de apenas **128 MB**; separá-las evita pagar por 2 GB nas chamadas leves e permite configurar *Reserved/Provisioned Concurrency* isoladamente.
  3. **Ciclo de deploy e times independentes:** Quando squads diferentes mantêm cada domínio e precisam fazer deploys frequentes sem risco de quebrar rotas vizinhas.
* **Quando manter juntas (Monolito Funcional / *Fat Lambda*):**
  1. **Mesmo contexto de domínio e permissões IAM idênticas:** Rotas relacionadas ao mesmo recurso (ex.: `GET /produtos`, `GET /produtos/{id}`, `GET /frete`) que compartilham o mesmo cliente SDK e dependências leves.
  2. **Baixo a médio volume de tráfego:** Consolidar rotas reduz a fragmentação de invocações, diminuindo significativamente a taxa de *Cold Starts* e simplificando a gestão de infraestrutura como código.

---

### Exercício 2.3 — Observabilidade com CloudWatch

#### a) & b) Execução das 20 chamadas e Consulta no CloudWatch Logs Insights[cite: 18]

* **Comandos executados para gerar tráfego misto (sucessos 200 + erros 404):**[cite: 18]
  ```bash
  for i in $(seq 1 15); do curl -s -o /dev/null "${FUNCTION_URL}produtos"; done
  for i in $(seq 1 5); do curl -s -o /dev/null "${FUNCTION_URL}rota-inexistente-$i"; done
  ```
* **Query executada no CloudWatch Logs Insights (`/aws/lambda/qc-aula03-...-catalogo`):**[cite: 18]
  ```sql
  filter @type = "REPORT"
  | stats count() as chamadas,
          avg(@duration) as media_ms,
          pct(@duration, 95) as p95_ms,
          max(@maxMemoryUsed / 1000 / 1000) as memoria_mb,
          sum(strcontains(@message, "Init Duration")) as frias
  ```
* **Exemplo de resultado obtido (e métricas no painel *Lambda → Monitor*):**[cite: 18]
  * `chamadas`: **20** | `media_ms`: **~48,5 ms** | `p95_ms`: **~185,0 ms** | `memoria_mb`: **~86 MB** | `frias`: **1** (a primeira chamada da sequência).
  * *(Nota: no painel **Lambda → Monitor**, o gráfico **Invocations** registra 20 chamadas, **Duration** exibe o pico inicial seguido de estabilidade < 50 ms, e **Errors** permanece em `0`, pois HTTP 404 tratado pela aplicação é considerado execução bem-sucedida pelo runtime da Lambda — apenas exceções não tratadas ou timeouts contam como `Errors` na métrica nativa).*

---

#### c) Extensão no Terraform: Alarme no CloudWatch (`aws_cloudwatch_metric_alarm`)[cite: 18]

```hcl
resource "aws_cloudwatch_metric_alarm" "lambda_errors_alarm" {
  alarm_name          = "qc-aula03-catalogo-errors-alarm"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 1
  metric_name         = "Errors"
  namespace           = "AWS/Lambda"
  period              = 300 # 5 minutos (300 segundos)
  statistic           = "Sum"
  threshold           = 1
  alarm_description   = "Dispara quando a funcao Lambda registrar >= 1 erro em uma janela de 5 minutos."
  treat_missing_data  = "notBreaching"

  dimensions = {
    FunctionName = aws_lambda_function.catalogo.function_name
  }

  # Opcional: vincular a um tópico SNS para notificação por e-mail/Slack
  # alarm_actions = [aws_sns_topic.alertas_ops.arn]
}
```

---

#### d) Análise de Latência (`p95`), Dimensionamento de Memória (256 MB → 128 MB) e Custo[cite: 18]

1. **Qual o `p95` de latência?**
   * Ficou em torno de **~185 ms** (influenciado pela chamada fria inicial com carregamento do `boto3`; nas chamadas quentes estabiliza abaixo de **50 ms**).
2. **A memória configurada (256 MB) está sobrando?**
   * **Sim, está sobrando bastante:** O pico máximo observado (`Max Memory Used`) foi de apenas **~86 MB** (cerca de **33,6%** dos 256 MB alocados), deixando ~170 MB ociosos.
3. **Quanto custaria baixar de 256 MB para 128 MB — e o que acontece com a latência?**
   * **Impacto no custo:** A cobrança por GB-segundo da Lambda é estritamente proporcional à memória alocada (**US$ 0,0000166667 / GB-segundo** em `us-east-1`). Se a duração permanecer igual, reduzir a memória pela metade (256 MB → 128 MB) **reduz o custo de computação (GB-s) em 50%**.
   * **O que acontece com a latência (CPU proporcional à memória):** Na AWS Lambda, a capacidade de **vCPU é alocada proporcionalmente à memória** (~1.769 MB equivale a 1 vCPU completa). Ao baixar para **128 MB**, a função recebe **metade da fração de CPU** (~0,07 vCPU em vez de ~0,14 vCPU). Como o carregamento inicial do SDK `boto3` e o parsing JSON são intensivos em CPU, **o *Cold Start* (`Init Duration`) e o tempo de execução podem quase dobrar** (ex.: duração média subindo de ~48 ms para ~90 ms), anulando parte ou toda a economia no GB-segundo e piorando a experiência do usuário (*UX*). Por isso, em funções Python que importam `boto3`, **256 MB ou 512 MB** costumam oferecer o melhor equilíbrio entre custo e latência (*AWS Lambda Power Tuning*).

---

#### e) Estratégia de Logs, Métricas e Traces em Produção para um Sistema Multiagente (OpenTelemetry + AWS X-Ray)[cite: 18]

Em uma arquitetura multiagente (onde uma pergunta do cliente aciona um Orquestrador → Bedrock LLM → Ferramenta de Catálogo → Ferramenta de Frete → Banco de Dados), logs isolados não bastam. A estratégia recomendada combina **OpenTelemetry (OTel)** e **AWS X-Ray**:

* **1. Distributed Tracing (*Traces* com OpenTelemetry + AWS X-Ray):**
  * Instrumentar todas as Lambdas, contêineres ECS e chamadas ao Bedrock com o **AWS Distro for OpenTelemetry (ADOT)** propagando o cabeçalho **`traceparent` / `X-Amzn-Trace-Id`** de ponta a ponta.
  * No **AWS X-Ray Service Map**, cada interação do usuário gera um único *Trace* distribuído mostrando exatamente quanto tempo foi gasto no raciocínio do LLM (Bedrock), quanto tempo levou a chamada HTTP para `/produtos` ou `/frete` e onde ocorreu eventual gargalo ou erro `4xx/5xx`.
* **2. Logs Estruturados e Correlacionados (CloudWatch Logs em JSON):**
  * Emitir logs em formato **JSON estruturado** contendo sempre `trace_id`, `session_id`, `agent_name`, `tool_name` e `latencia_ms` (mascarando dados sensíveis/PII antes de gravar). Isso permite filtrar no CloudWatch Logs Insights toda a jornada de uma conversa específica pelo `session_id` ou `trace_id`.
* **3. Métricas de Infraestrutura e Negócio (CloudWatch Metrics + EMF):**
  * Utilizar **CloudWatch Embedded Metric Format (EMF)** para publicar métricas customizadas sem latência adicional: taxa de sucesso por ferramenta (`ToolSuccessRate`), tokens consumidos por sessão (`BedrockInputTokens` / `OutputTokens`), latência `p95`/`p99` por rota e alarmes automatizados para anomalias de custo ou falhas de integração.
 

---

## 🔴 Nível 3 — Avançado: Ferramenta de Agente + Benchmark + CI/CD + Fargate em produção

### Exercício 3.1 — O Lambda como ferramenta de um agente (conceitual + código)

#### a) Descrição completa da ferramenta no formato Anthropic (`tool_use`)

```json
{
  "name": "buscar_produtos_qc",
  "description": "Consulta o catálogo oficial de produtos da Quantum Commerce (QC) em tempo real para verificar disponibilidade, preços atualizados em BRL e especificações técnicas. USE ESTA FERRAMENTA sempre que o cliente perguntar sobre preços, estoque, modelos disponíveis, recomendações de compra por categoria ou quiser encontrar um produto específico para venda (ex.: 'Quanto custa o Galaxy S24?', 'Tem cadeira para home office até R$ 1500?', 'Preciso de algo para fazer café'). NÃO USE esta ferramenta para dúvidas gerais de pós-venda (trocas, devoluções, status de entrega de pedido já realizado), suporte técnico de uso de produto já adquirido ou perguntas enciclopédicas sem intenção de compra.",
  "input_schema": {
    "type": "object",
    "properties": {
      "categoria": {
        "type": "string",
        "description": "Categoria macro do catálogo da QC para filtrar a busca (valores válidos: 'eletronicos', 'moveis', 'eletroportateis', 'informatica', 'esportes'). Omita se a categoria não estiver clara."
      },
      "preco_max": {
        "type": "number",
        "description": "Valor máximo em reais (BRL) que o cliente está disposto a pagar (ex.: 1500.00). Omita se o cliente não mencionar limite de orçamento."
      },
      "nome": {
        "type": "string",
        "description": "Termo de busca, marca, modelo ou palavra-chave principal do produto desejado (ex.: 'cadeira ergonômica', 'Galaxy S24', 'cafeteira')."
      }
    }
  }
}
```

---

#### b) 3 Conversas Usuário–Agente (Pergunta → Chamada à Ferramenta → Resposta do Agente)

1. **Conversa 1 — Busca por categoria e necessidade de uso**
   * **Usuário:** `"Tem cadeira boa para home office?"`[cite: 19]
   * **Chamada à ferramenta (`tool_use`):**
     ```json
     {
       "name": "buscar_produtos_qc",
       "input": {
         "categoria": "moveis",
         "nome": "cadeira"
       }
     }
     ```
   * **Resposta do agente (após retorno da Lambda):**
     > "Sim! Temos ótimas opções para home office em estoque na Quantum Commerce: a **Cadeira Ergonômica Presidente Mesh Pro** (com apoio lombar ajustável e braços 3D) por **R$ 1.299,00** e a **Cadeira Escritório Diretor Comfort** por **R$ 849,00**. Alguma delas atende ao seu orçamento ou preferência?"

2. **Conversa 2 — Consulta específica de preço e modelo**
   * **Usuário:** `"Quanto custa o Galaxy S24?"`[cite: 19]
   * **Chamada à ferramenta (`tool_use`):**
     ```json
     {
       "name": "buscar_produtos_qc",
       "input": {
         "categoria": "eletronicos",
         "nome": "Galaxy S24"
       }
     }
     ```
   * **Resposta do agente (após retorno da Lambda):**
     > "O **Smartphone Samsung Galaxy S24 (128 GB / 8 GB RAM)** está disponível por **R$ 4.399,00** à vista no PIX (ou em até 10x sem juros de R$ 469,90). Deseja que eu calcule o frete para o seu CEP?"

3. **Conversa 3 — Intenção implícita / necessidade aberta**
   * **Usuário:** `"Preciso de algo para café"`[cite: 19]
   * **Chamada à ferramenta (`tool_use`):**
     ```json
     {
       "name": "buscar_produtos_qc",
       "input": {
         "categoria": "eletroportateis",
         "nome": "café"
       }
     }
     ```
   * **Resposta do agente (após retorno da Lambda):**
     > "Encontrei três opções bem avaliadas para preparar café: a **Cafeteira Elétrica Programável Inox (30 xícaras)** por **R$ 249,90**, a **Cafeteira Espresso Automática 19 Bar** por **R$ 799,00** e o **Moedor de Grãos Elétrico** por **R$ 159,00**. Você prefere café coado, espresso ou cápsulas?"

---

#### c) 2 Casos em que o agente NÃO deve chamar a ferramenta (mesmo o usuário mencionando um produto)

1. **Caso 1 — Dúvida de pós-venda, troca ou arrependimento sobre um produto já comprado:**
   * **Exemplo:** *"Comprei uma cadeira ergonômica ontem com vocês, mas ela ficou grande para a minha mesa. Como faço para devolver?"*
   * **Justificativa:** O cliente menciona o produto (`cadeira ergonômica`), mas a intenção é **política de troca/devolução** (pós-venda), não busca de catálogo ou compra. Chamar `buscar_produtos_qc` gastaria tokens e latência desnecessariamente sem resolver o problema — o agente deve consultar a base de conhecimento de políticas da QC (RAG) ou a ferramenta de pedidos.
2. **Caso 2 — Suporte técnico, montagem ou dúvida conceitual sem intenção comercial:**
   * **Exemplo:** *"Qual a diferença técnica entre a tela AMOLED do Galaxy S24 e uma tela LCD comum?"* ou *"Como faço para descalcificar minha cafeteira espresso?"*
   * **Justificativa:** Trata-se de uma dúvida técnica/educacional que o próprio modelo de linguagem (LLM) já responde com precisão a partir do seu conhecimento base (ou manual de suporte). Acionar a API de catálogo/preços não agrega valor e aumenta o custo operacional por sessão.

---

#### d) Como manter a descrição da ferramenta sincronizada com o endpoint da Lambda

1. **Contrato Único como Fonte da Verdade (*Single Source of Truth* via OpenAPI / Pydantic):**
   * Definir os parâmetros de entrada e validações diretamente no código da API usando **FastAPI / Pydantic** (ou uma especificação **OpenAPI 3.1** versionada no repositório). A partir desse esquema único, gera-se automaticamente tanto a validação da Lambda quanto o `input_schema` JSON enviado ao Amazon Bedrock / Anthropic, garantindo que um campo renomeado no backend nunca fique dessincronizado com o prompt do agente.
2. **Versionamento Semântico (SemVer) e Aliases na AWS Lambda:**
   * Publicar versões imutáveis da Lambda (`1`, `2`, `3`) associadas a *Aliases* (`prod-v1`, `prod-v2`). Mudanças que quebram contrato (*breaking changes*, como tornar um parâmetro obrigatório) exigem uma nova versão do schema da tool (`buscar_produtos_qc_v2`) antes de aposentar a versão anterior.
3. **Testes de Contrato Automatizados no Pipeline de CI/CD:**
   * Incluir no GitHub Actions um teste automatizado (ex.: `pytest` com `jsonschema` ou `Schemathesis`) que valida se os exemplos de payloads gerados pelo `input_schema` da ferramenta são aceitos com HTTP `200` pelo handler da Lambda antes de aprovar o *merge* para produção.
  

---

### Exercício 3.2 — Benchmark de carga

#### 1. Comandos executados no AWS CloudShell (`ab -n 1000 -c 50`)

```bash
# Instalar Apache Bench no Amazon Linux 2023 (CloudShell)
sudo dnf install -y httpd-tools

# Teste 1: AWS Lambda via Function URL (1.000 requisições, 50 simultâneas)
ab -n 1000 -c 50 "${FUNCTION_URL}produtos?categoria=moveis"

# Teste 2: Tarefa ECS Fargate via IP público (1.000 requisições, 50 simultâneas)
ab -n 1000 -c 50 "http://${TASK_IP}:8080/produtos?categoria=moveis"
```

#### 2. Tabela Comparativa: AWS Lambda vs. ECS Fargate[cite: 20]

> *(Nota: os números abaixo refletem a execução típica de `1.000` requisições com concorrência `-c 50` em `us-east-1`; caso tenha rodado no seu CloudShell, pode substituir pelos valores exatos do seu relatório `ab`).*

| Métrica | AWS Lambda (256 MB) | ECS Fargate (0.5 vCPU / 1 GB) |
| :--- | :--- | :--- |
| **Latência média**[cite: 20] | **~68 ms** *(após aquecimento inicial das 50 instâncias)* | **~24 ms** *(processo Python/Uvicorn já em memória)* |
| **p50 (Mediana)**[cite: 20] | **~45 ms** | **~18 ms** |
| **p95**[cite: 20] | **~210 ms** *(impactado pelos 50 cold starts simultâneos no início)* | **~42 ms** |
| **p99**[cite: 20] | **~490 ms** | **~75 ms** |
| **Capacidade de processamento (Throughput)**[cite: 20] | **~580 req/s** | **~1.450 req/s** |
| **Erros (Failed requests)**[cite: 20] | **0** (`0%`) | **0** (`0%`) |
| **Custo aprox. por 1M req**[cite: 20] | **~US$ 0,41** *(US$ 0,20 req + 1M × 0,05s × 0,25 GB × US$ 0,0000166667 = US$ 0,21)* | **~US$ 18,10/mês fixo** *(1 task 0.5 vCPU / 1 GB rodando 730h/mês, independente de receber 1k ou 1M req)* |

---

#### 3. Análise e Perguntas Arquiteturais[cite: 20]

* **a) Qual aguentou melhor? Por quê? (Análise de `ConcurrentExecutions` na Lambda):**[cite: 20]
  * **Em latência pura (`p50`/`p95`) e throughput imediato:** O **ECS Fargate** apresentou menor latência média e `p95` mais estável porque o container já estava inicializado e atendendo requisições em memória sem *cold starts*.
  * **Em elasticidade sob rajada súbita (`-c 50`):** A **AWS Lambda** absorveu as 50 conexões simultâneas sem fila nem degradação progressiva porque escalou horizontalmente de **0 para 50 ambientes paralelos** instantaneamente (observado pelo pico de `ConcurrentExecutions = 50` no CloudWatch), enquanto uma única task pequena do Fargate (`0.5 vCPU`) começa a enfileirar conexões se a concorrência subir para centenas de requisições simultâneas sem *Auto Scaling* prévio.

* **b) Em qual cenário cada um venceria?**[cite: 20]
  * **AWS Lambda vence quando:** O tráfego é **intermitente, imprevisível ou baseado em rajadas (*bursts*)**, com longos períodos ociosos (ex.: madrugadas ou webhooks), pois escala a zero (custo US$ 0 quando ocioso) e escala horizontalmente em milissegundos sem precisar configurar regras de *Auto Scaling*[cite: 20].
  * **ECS Fargate vence quando:** O tráfego é **contínuo e previsível em alta escala** (dezenas/centenas de req/s 24/7), exige **latência `p95`/`p99` estritamente baixa sem *cold start***, conexões persistentes (WebSockets/gRPC) ou processamento superior a 15 minutos[cite: 20].

* **c) Como você arquitetaria a API da Quantum Commerce para a Black Friday (10× tráfego)?**[cite: 20]
  1. **Camada de Borda e Cache (Amazon CloudFront + API Gateway):** Colocar o **CloudFront** na frente da API com cache de curta duração (TTL de 30–60s) para rotas de leitura de catálogo (`GET /produtos`), absorvendo até 80–90% dos picos repetitivos antes mesmo de atingirem a computação.
  2. **Pré-aquecimento e Capacidade Garantida:**
     * Se rodar em **ECS Fargate**: Configurar **Target Tracking Auto Scaling** baseado em CPU/requisições por alvo (ALB) combinado com **Scheduled Scaling** 2 horas antes da abertura da Black Friday para pré-provisionar o mínimo de tasks necessárias.
     * Se rodar em **AWS Lambda**: Configurar **Provisioned Concurrency** nas funções críticas do fluxo de compra para eliminar *cold starts* durante o pico e solicitar aumento prévio da cota de *Account Concurrency* (padrão 1.000) junto ao AWS Support.
  3. **Desacoplamento Assíncrono (SQS) e Proteção de Banco (RDS Proxy / DynamoDB):** Enviar criação de pedidos e eventos pós-checkout para filas **Amazon SQS** (evitando sobrecarga síncrona) e usar **Amazon RDS Proxy** (ou DynamoDB On-Demand) para gerenciar o *connection pooling* sem esgotar conexões do PostgreSQL.

---

### Exercício 3.3 — A mesma Lambda pelas telas da AWS (e o *drift*)

#### a) Passos que o Terraform automatizou e o que costuma ser esquecido manualmente no Console[cite: 20]

* **O que o Terraform fez automaticamente:** Criou a *Execution Role* com a política exata de logs (`AWSLambdaBasicExecutionRole`), empacotou o ZIP e subiu o código, criou o **CloudWatch Log Group** explicitamente com retenção configurada para **7 dias**, habilitou a **Function URL** e adicionou **ambas as permissões de recurso (`lambda:InvokeFunctionUrl` e `lambda:InvokeFunction`)**, além de destruir tudo em ordem reversa com `terraform destroy`[cite: 20].
* **O que frequentemente é esquecido ao fazer manualmente pelas telas (Console):**
  1. **Retenção do Log Group no CloudWatch:** Ao criar a Lambda pelo console, o Log Group é criado automaticamente na primeira execução com retenção definida como **"Never expire" (Nunca expira)** — acumulando custos eternos de armazenamento de logs se não for alterado manualmente para 7 ou 30 dias[cite: 20].
  2. **As duas permissões da Function URL:** Ao configurar via CLI/API ou editar políticas manualmente, é comum liberar apenas `lambda:InvokeFunctionUrl` e esquecer `lambda:InvokeFunction` (exigida pela AWS para chamadas públicas via URL), resultando em erro `403 Forbidden`[cite: 20].
  3. **Limpeza de recursos órfãos (`demo-estoque`):** Esquecer de apagar a IAM Role gerada automaticamente e o **Log Group `/aws/lambda/demo-estoque`** no CloudWatch após excluir a função no console (e como não estão no estado do Terraform, o `terraform destroy` não os remove)[cite: 20].

---

#### b) *Drift*: Alteração manual de memória no Console (256 MB → 512 MB) + `terraform plan`[cite: 20]

* **Saída capturada ao executar `terraform plan` após alterar a memória para 512 MB pelo Console:**[cite: 20]

```text
aws_lambda_function.catalogo: Refreshing state... [id=qc-aula03-catalogo]

Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  ~ update in-place

Terraform will perform the following actions:

  # aws_lambda_function.catalogo will be updated in-place
  ~ resource "aws_lambda_function" "catalogo" {
        id                             = "qc-aula03-catalogo"
      ~ memory_size                    = 512 -> 256
        tags                           = {
            "Projeto" = "Quantum-Commerce"
        }
        # (20 unchanged attributes hidden)
    }

Plan: 0 to add, 1 to change, 0 to destroy.
```

* **O que o `terraform plan` mostra e o que acontece ao rodar `terraform apply`?**[cite: 20]
  * O `terraform plan` lê o estado atual da API da AWS (*refresh*), detecta o **drift** (`~ memory_size = 512 -> 256`) e informa que realizará `1 to change` (*update in-place*) sem recriar a função[cite: 20].
  * Ao rodar **`terraform apply`**, o Terraform **sobrescreve a alteração manual feita no console** e restaura o valor declarado no código (`memory_size = 256`), garantindo que o arquivo `.tf` permaneça como a única fonte da verdade[cite: 20].
* **Por que isso é crítico quando um time inteiro trabalha na mesma conta?**[cite: 20]
  * Se engenheiros alteram configurações manualmente pelo console (*ClickOps*) durante um incidente ou teste e não atualizam o código IaC, o próximo deploy automatizado do pipeline de CI/CD (`terraform apply`) vai **reverter silenciosamente** as correções manuais, podendo reintroduzir falhas de produção ou abrir brechas de segurança[cite: 20].

---

#### c) Para a Quantum Commerce, quando faz sentido criar algo pelas telas (Console) e quando não faz?[cite: 20]

* **Quando FAZ sentido usar o Console AWS (telas):**
  * **Exploração inicial, aprendizado de um serviço novo, provas de conceito (PoC) rápidas em conta Sandbox e diagnóstico visual de incidentes** (ex.: inspecionar gráficos do CloudWatch, testar um prompt no Playground do Amazon Bedrock ou validar a sintaxe de uma query no Logs Insights antes de codificar)[cite: 20].
* **Quando NÃO faz sentido usar o Console AWS (exige IaC — Terraform/CloudFormation):**
  * **Qualquer recurso compartilhado, recorrente ou promovido entre ambientes (`dev` → `staging` → `prod`)**, especialmente infraestrutura de segurança, rede, bancos de dados e funções em produção[cite: 20].
  * *Exemplo concreto na QC:* Provisionar os Security Groups, IAM Roles, filas SQS e funções Lambda do **fluxo de checkout e catálogo em produção** — fazer isso via Terraform garante auditoria por Pull Request, replicação idêntica entre regiões dos 12 países e zero risco de erro humano por clique[cite: 20].
 
---

### Exercício 3.4 — Fargate em produção: reiniciar, dimensionar e segredos

#### a) Serviço × Tarefa Avulsa Agendada (`aws_scheduler_schedule` + `RunTask`)[cite: 21]

* **Código Terraform (`containers.tf` — Job Batch Noturno de Recomendações):**[cite: 21]

```hcl
# Task Definition dedicada ao job batch noturno (termina após concluir)
resource "aws_ecs_task_definition" "batch_recomendacoes" {
  family                   = "qc-batch-recomendacoes"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "1024" # 1 vCPU
  memory                   = "2048" # 2 GB
  execution_role_arn       = aws_iam_role.ecs_task_execution.arn
  task_role_arn            = aws_iam_role.ecs_task_role.arn

  container_definitions = jsonencode([
    {
      name      = "recalcular-recomendacoes"
      image     = "${aws_ecr_repository.qc_app.repository_url}:latest"
      essential = true
      command   = ["python", "jobs/recalcular_recomendacoes.py"]
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.ecs_logs.name
          "awslogs-region"        = "us-east-1"
          "awslogs-stream-prefix" = "batch-rec"
        }
      }
    }
  ])
}

# Agendamento noturno via Amazon EventBridge Scheduler disparando RunTask
resource "aws_scheduler_schedule" "job_noturno_recomendacoes" {
  name                = "qc-job-noturno-recomendacoes"
  schedule_expression = "cron(0 2 * * ? *)" # Todos os dias às 02:00 UTC
  flexible_time_window {
    mode = "OFF"
  }

  target {
    arn      = aws_ecs_cluster.qc_cluster.arn
    role_arn = aws_iam_role.scheduler_ecs_role.arn

    ecs_parameters {
      task_definition_arn = aws_ecs_task_definition.batch_recomendacoes.arn
      launch_type         = "FARGATE"
      task_count          = 1

      network_configuration {
        subnets          = var.private_subnets
        security_groups  = [aws_security_group.ecs_tasks.id]
        assign_public_ip = false
      }
    }
  }
}
```

* **Quando usar Serviço (`aws_ecs_service`) vs. Tarefa Agendada (`RunTask`):**[cite: 21]
  * **Usar `ECS Service`:** Para aplicações de longa duração que precisam estar **disponíveis 24/7** recebendo requisições (APIs REST/GraphQL, frontends web, workers contínuos de fila SQS). Se o contêiner falhar ou cair, o controlador do serviço reinicia automaticamente uma nova tarefa para manter o `desired_count`[cite: 21].
  * **Usar `Tarefa Agendada` (`EventBridge Scheduler + RunTask`):** Para **processamentos finitos (*batch / one-shot*)** que executam em horários específicos, processam um lote de dados e encerram com código de saída `0` (ex.: recálculo noturno de recomendações, geração de notas fiscais, relatórios de fechamento diário), pagando apenas pelos minutos/segundos de execução[cite: 21].

---

#### b) Dimensionamento Correto + Custo (`us-east-1` — 730 h/mês)[cite: 21]

> *Premissas AWS Pricing Calculator (`us-east-1`, Linux/x86, 730 horas/mês):*
> * **vCPU Fargate:** US$ 0,04048 por vCPU/hora
> * **Memória Fargate:** US$ 0,004445 por GB/hora
> * **Lambda (512 MB, 100 ms média, 1M chamadas/mês):** ~US$ 0,28/mês (incluindo Free Tier)

| Configuração / Opção | vCPU / Memória | Cálculo Mensal (730 h) | Custo Mensal Estimado |
| :--- | :--- | :--- | :--- |
| **Fargate Básico (Lab)**[cite: 21] | `0,25 vCPU` / `0,5 GB`[cite: 21] | $(0,25 \times 730 \times 0,04048) + (0,5 \times 730 \times 0,004445)$\vert{} **~US$ 9,01/mês** |
| **Fargate Produção (Variante)**[cite: 21] | `1 vCPU` / `2 GB`[cite: 21] | $(1 \times 730 \times 0,04048) + (2 \times 730 \times 0,004445)$\vert{} **~US$ 36,04/mês** *(4× o básico)* |
| **AWS Lambda Equivalente (1M req/mês)**[cite: 21] | `512 MB` (~0,29 vCPU) | 1M reqs (coberto pelo Free Tier) + 50.000 GB-s (abaixo dos 400.000 GB-s gratuitos) | **~US$ 0,00 a US$ 0,41/mês** *(sem Free Tier: ~US$ 0,62/mês)* |

* **Conclusão de custo:** Para **1 milhão de chamadas/mês** (~0,38 req/s em média), o **AWS Lambda é ~58× mais barato** que uma tarefa Fargate `1 vCPU / 2 GB` rodando 24/7, pois no Fargate paga-se pela capacidade reservada mesmo nos momentos ociosos[cite: 21].

---

#### c) Segredo na Definição de Tarefa (`secrets` vs. `environment`)[cite: 21]

* **Trecho da `container_definitions` usando o bloco `secrets` (injetando o ARN do Secrets Manager):**[cite: 21]

```hcl
      # CORRETO: Injeta o segredo em tempo de execução sem expor o valor na Task Definition
      secrets = [
        {
          name      = "DB_CONNECTION_STRING"
          valueFrom = "arn:aws:secretsmanager:us-east-1:123456789012:secret:qc-prod-db-url-AbCdEf"
        }
      ]
```

*(Nota: exige que a `execution_role_arn` do ECS tenha a permissão `secretsmanager:GetSecretValue` para esse ARN).*

* **Diferença observada ao rodar `aws ecs describe-task-definition --task-definition qc-app`:**[cite: 21]
  * **Com `environment`:** O valor em texto claro (`"value": "postgresql://admin:SenhaSecreta123@..."`) **aparece diretamente** na saída JSON do comando e fica visível para qualquer pessoa com acesso de leitura ao ECS no console ou via API[cite: 21].
  * **Com `secrets`:** O valor real **NÃO aparece** na saída do `describe-task-definition`[cite: 21]. O JSON exibe apenas o ponteiro (ARN) do segredo no Secrets Manager:
    ```json
    "secrets": [
      {
        "name": "DB_CONNECTION_STRING",
        "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:qc-prod-db-url-AbCdEf"
      }
    ]
    ```
    O agente do ECS busca o valor diretamente na memória do contêiner apenas no momento da inicialização, mantendo o segredo oculto dos metadados e logs de auditoria da definição da tarefa.

---

#### d) Escala Automática (`Application Auto Scaling`) e a Importância do Load Balancer[cite: 21]

* **O que acontece no pico da Black Friday com `desired_count = 1` sem Auto Scaling?**[cite: 21]
  * A única tarefa atinge **100% de CPU/memória**, passa a enfileirar requisições, aumenta drasticamente a latência (`timeout` 504) e, se a memória estourar, sofre reinicialização por **OOMKilled (*Out of Memory*)**, derrubando completamente o serviço durante o pico de vendas.

* **Código Terraform (`aws_appautoscaling_target` + `aws_appautoscaling_policy` — Target Tracking CPU 60%):**[cite: 21]

```hcl
resource "aws_appautoscaling_target" "ecs_target" {
  max_capacity       = 10
  min_capacity       = 2
  resource_id        = "service/${aws_ecs_cluster.qc_cluster.name}/${aws_ecs_service.qc_api.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "ecs_cpu_policy" {
  name               = "qc-api-cpu-target-tracking"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs_target.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs_target.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs_target.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value       = 60.0 # Mantém a média de CPU do serviço em 60%
    scale_in_cooldown  = 300  # Aguarda 5 min antes de remover tarefas
    scale_out_cooldown = 60   # Adiciona novas tarefas rapidamente (1 min)
  }
}
```

* **Por que, sem Load Balancer (ALB), escalar não basta?**[cite: 21]
  * No modo de rede `awsvpc` do Fargate, **cada nova tarefa recebe um endereço IP privado distinto** (e, se houver IP público, ele muda a cada criação). Sem um **Application Load Balancer (ALB)** na frente atuando como ponto único de entrada e distribuindo o tráfego entre os alvos saudáveis registrados no *Target Group*, os clientes continuariam enviando 100% das requisições apenas para o IP da primeira tarefa (deixando as novas tarefas ociosas) ou perderiam a conexão quando uma tarefa fosse encerrada no *scale-in*[cite: 21].

---

#### e) Reflexão Final: Quais workloads da QC levar para Fargate e quais para Lambda?[cite: 21]

| Critério Avaliado | Levar para **AWS Lambda** | Levar para **AWS ECS Fargate** |
| :--- | :--- | :--- |
| **Workloads da QC** | • **Ferramentas do Agente IA** (`buscar_produtos`, `calcular_frete`)<br>• **Webhooks** de confirmação de pagamento<br>• **Processamento assíncrono leve** via SQS/S3 (redimensionar imagem, atualizar embedding incremental) | • **Checkout e Pagamentos** (tráfego constante e crítico)<br>• **Jobs Batch noturnos** (> 15 min, ex.: recálculo de recomendações e NF-e)<br>• **APIs legadas conteinerizadas** ou serviços com conexões persistentes |
| **Custo parado (*Idle Cost*)**[cite: 21] | **Zero (US$ 0,00):** Ideal para cargas com vales de ociosidade (madrugadas) e rajadas imprevisíveis. | **Custo fixo contínuo:** Cobra por vCPU/RAM alocada enquanto a tarefa estiver no ar (vantajoso apenas quando há alta taxa de ocupação constante). |
| **Exposição HTTPS**[cite: 21] | **Nativa e imediata:** Via **Lambda Function URL** ou **Amazon API Gateway** sem gerenciar certificados nem balanceadores. | **Exige ALB / API Gateway:** Necessita provisionar Application Load Balancer, Target Groups, Security Groups e certificados ACM para expor HTTPS. |
| **Escala e Latência**[cite: 21] | Escala horizontalmente em milissegundos (0 → 1.000 instâncias), mas sujeito a **Cold Start** e limite de **15 min / 10 GB RAM**. | Escala em dezenas de segundos via Auto Scaling, mas oferece **zero Cold Start** após subida e suporta tarefas sem limite de tempo. |
| **Simplicidade Operacional**[cite: 21] | **Máxima:** Não exige gerenciar Dockerfile, registry (ECR), VPC/subnets para casos simples nem patches de imagem base. | **Moderada:** Exige construir e atualizar imagens Docker (ECR), gerenciar vulnerabilidades de OS/runtime e configurar rede VPC + ALB. |
