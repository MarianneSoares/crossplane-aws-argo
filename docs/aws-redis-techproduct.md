# AWS Redis — Guia de Uso

## 0. Quickstart de Integração

### O que é e quando usar

Este Tech Product provisiona um cluster Redis gerenciado pelo Amazon ElastiCache a partir de uma instância Kubernetes `RedisCache`. Use-o quando a aplicação precisa de cache, sessões ou dados temporários de baixa latência sem operar servidores Redis manualmente.

O fluxo é:

```text
RedisCache (Kubernetes)
  -> XRD namespaced
  -> Composition Pipeline
  -> SubnetGroup + Cluster do ElastiCache
```

### Onde encontrar os dados de conexão

Depois que o recurso estiver `READY=True`, consulte o endpoint na AWS ou no managed resource:

```powershell
kubectl get cluster.elasticache.aws.upbound.io app-redis-prod -o yaml
```

Na AWS CLI:

```powershell
$env:AWS_PAGER = ""
aws elasticache describe-cache-clusters `
  --region us-east-1 `
  --cache-cluster-id app-redis-prod `
  --show-cache-node-info `
  --query "CacheClusters[0].CacheNodes[0].Endpoint" `
  --output json
```

Não fixe o endpoint no código. Injete-o por Secret, variável de ambiente ou mecanismo de configuração da plataforma.

### Autenticação

O Crossplane usa um `ProviderConfig` AWS e o Secret Kubernetes `aws-creds`. A aplicação não deve receber a access key do Crossplane. Para workloads em produção, prefira IAM Roles for Service Accounts, workload identity ou outro mecanismo sem chave estática.

### Exemplo de conexão

**Node.js**

```js
import Redis from "ioredis";

const client = new Redis({
  host: process.env.REDIS_HOST,
  port: Number(process.env.REDIS_PORT || 6379),
  tls: process.env.REDIS_TLS === "true" ? {} : undefined,
});

await client.set("healthcheck", "ok", "EX", 60);
console.log(await client.get("healthcheck"));
```

**Python**

```python
import os
import redis

client = redis.Redis(
    host=os.environ["REDIS_HOST"],
    port=int(os.getenv("REDIS_PORT", "6379")),
    ssl=os.getenv("REDIS_TLS", "false").lower() == "true",
    decode_responses=True,
)

client.set("healthcheck", "ok", ex=60)
print(client.get("healthcheck"))
```

### Variáveis de ambiente esperadas

| Variável | Descrição |
|---|---|
| `REDIS_HOST` | Endpoint DNS do ElastiCache |
| `REDIS_PORT` | Porta do Redis, normalmente `6379` |
| `REDIS_TLS` | `true` quando TLS estiver habilitado |
| `REDIS_PASSWORD` | Senha, quando autenticação estiver configurada |

## 1. O que é o recurso

O Amazon ElastiCache for Redis é um serviço gerenciado para executar Redis na AWS. A Composition deste projeto cria um `SubnetGroup` usando subnets existentes e um `Cluster` Redis com o tipo de nó e versão definidos na instância Kubernetes.

O Crossplane gerencia o ciclo de vida declarado no Git/Kubernetes, enquanto a AWS executa o provisionamento físico.

## 2. Modelo de implantação

A instância atual é:

```yaml
apiVersion: cache.example.org/v1alpha1
kind: RedisCache
metadata:
  name: app-redis-cache
  namespace: default
spec:
  clusterId: app-redis-prod
  region: us-east-1
  nodeType: cache.t3.micro
  numCacheNodes: 1
  engineVersion: "7.1"
  subnetIds:
    - subnet-0f06a150456c0952c
    - subnet-04f8416c7fa2b67ca
```

As subnets já existem na VPC. O Crossplane cria o Subnet Group do ElastiCache; ele não cria VPCs ou subnets neste projeto.

## 3. Campos do contrato

| Categoria | Campo | Regra |
|---|---|---|
| Obrigatório | `clusterId` | Identificador imutável, minúsculo e limitado a 20 caracteres |
| Obrigatório | `subnetIds` | Pelo menos duas subnets |
| Ambiente | `region` | `us-east-1` ou `us-east-2`; padrão `us-east-1` |
| Ambiente | `subnetIds` | Devem existir na mesma conta, região e VPC |
| Capacidade | `nodeType` | Tipo de instância ElastiCache |
| Capacidade | `numCacheNodes` | Atualmente fixado entre 1 e 1 |
| Constante | `port` | `6379`, definido na Composition |
| Constante | `engine` | `redis`, definido na Composition |

## 4. Guardrails

- Não versionar `aws-credentials.ini`.
- Não expor chaves AWS em logs, commits ou documentação.
- Usar subnets privadas em zonas distintas.
- Restringir acesso de rede ao Redis via security groups e VPC.
- Habilitar TLS, autenticação e criptografia em repouso em ambientes reais.
- Aplicar tags e políticas de backup antes de produção.
- Confirmar `READY=True` antes de liberar o endpoint para consumidores.

## 5. Estados e troubleshooting

```powershell
kubectl get rediscache app-redis-cache -o wide
kubectl get cluster.elasticache.aws.upbound.io app-redis-prod -o wide
kubectl get subnetgroup.elasticache.aws.upbound.io app-redis-prod-subnet-group -o wide
kubectl describe cluster.elasticache.aws.upbound.io app-redis-prod
kubectl get events -A --sort-by=.lastTimestamp
```

Estados importantes:

- `Synced=True`: o Crossplane conseguiu reconciliar a configuração.
- `Ready=True`: o recurso externo está pronto para uso.
- `Ready=False`: consulte `status.conditions` e eventos.
- `EXTERNAL-NAME` preenchido: o managed resource foi associado ao recurso AWS.

| Erro | Causa provável | Ação |
|---|---|---|
| `InvalidSubnetID.NotFound` | ID não existe na conta/região | Listar subnets com `aws ec2 describe-subnets` |
| `Secret aws-creds not found` | Secret removido ou namespace incorreto | Recriar no namespace `crossplane-system` |
| `CacheSubnetGroupNotFoundFault` | Cluster tentou iniciar antes do Subnet Group | Aguardar reconciliação e verificar o Subnet Group |
| `READY=False` sem mensagem | Provider ou Function indisponível | Verificar pods, providers e eventos |
| Endpoint não acessível | VPC/security group/rede | Validar rota e regras de entrada |

## 6. Custos

O custo depende do tipo de nó, quantidade, região, horas de execução, backup e tráfego. `cache.t3.micro` é adequado para uma PoC, não para produção sem validar memória, throughput e disponibilidade. Consulte a calculadora oficial da AWS para comparar `us-east-1` e `us-east-2` antes de escolher a região.

## 7. Referências

- [Amazon ElastiCache](https://aws.amazon.com/elasticache/)
- [Crossplane](https://docs.crossplane.io/)
- [Provider AWS](https://marketplace.upbound.io/providers)
- [AWS CLI ElastiCache](https://docs.aws.amazon.com/cli/latest/reference/elasticache/)
