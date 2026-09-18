# Como apresentar o projeto AWS Redis com Crossplane

## Objetivo da apresentação

Mostrar como uma instância Kubernetes simples vira um Redis gerenciado na AWS, explicar as abstrações Crossplane e demonstrar o ciclo de reconciliação até `READY=True`.

Mensagem central:

> O time declara o que precisa em uma `RedisCache`; a Composition traduz esse contrato para recursos AWS, e o Crossplane mantém o estado real alinhado ao estado desejado.

## Roteiro de 10 minutos

### 1. Contexto e arquitetura

Apresente o desenho:

```text
RedisCache
   |
   v
XRD namespaced: contrato do produto
   |
   v
Composition Pipeline + Function
   |
   +--> ElastiCache SubnetGroup
   +--> ElastiCache Cluster
```

Explique que o Docker Desktop fornece o cluster Kubernetes local. O Redis e o Subnet Group são recursos reais na AWS.

### 2. Fundamentos que precisam aparecer

- **Provider:** pacote que conhece a API da AWS.
- **Managed Resource (MR):** objeto que representa um recurso externo, como `Cluster` ou `SubnetGroup`.
- **XRD:** define o contrato da abstração `RedisCache`.
- **XR:** instância namespaced desse contrato.
- **Composition:** transforma o XR em MRs.
- **Function:** executa o pipeline de composição e os patches.
- **Reconciliation:** controller compara o estado desejado com o estado real.

Uma frase útil:

> XRD define o produto, Composition define a implementação e Provider executa a integração com a nuvem.

## Plano de estudo e execução

### Segunda-feira — fundamentos

Estude:

```powershell
kubectl get crd
kubectl get providers
kubectl get functions
```

Explique a diferença entre CRD Kubernetes e XRD Crossplane: ambos estendem a API, mas o XRD representa uma abstração de plataforma que pode compor recursos externos.

### Terça-feira — leitura guiada

Leia nesta ordem:

1. `crossplane/definitions/xrediscache.yaml`
2. `crossplane/compositions/rediscache.yaml`
3. `crossplane/claims/redis-claim.yaml`
4. `crossplane/providers/provider-config.yaml`

Checkpoint de dúvidas:

- Quais campos são fornecidos pelo consumidor?
- Quais campos são constantes da plataforma?
- Onde o nome do Subnet Group é derivado?
- Como o provider encontra as credenciais?
- O que significa `Synced` e o que significa `Ready`?

### Quarta-feira — desenho do XRD

Mostre a categorização do contrato:

- **Obrigatórios:** `clusterId`, `subnetIds`.
- **Ambiente:** `region`, `subnetIds`.
- **Capacidade:** `nodeType`, `numCacheNodes`, `engineVersion`.
- **Constantes:** engine `redis` e porta `6379` na Composition.

Destaque as validações:

- `clusterId` imutável.
- Pelo menos duas subnets.
- Região limitada às regiões suportadas pelo contrato.
- `numCacheNodes` limitado a 1 para esta PoC.

### Quinta-feira — Composition e cluster

Verifique o ambiente:

```powershell
kubectl config current-context
kubectl get nodes
kubectl get pods -n crossplane-system
kubectl get providers
kubectl get functions
```

Aplique a ordem:

```powershell
kubectl apply -f crossplane/providers/provider-config.yaml
kubectl apply -f crossplane/definitions/xrediscache.yaml
kubectl apply -f crossplane/compositions/rediscache.yaml
```

Valide o contrato:

```powershell
kubectl get xrd
kubectl get compositions
```

### Sexta-feira — instância real e integração

Aplique a instância:

```powershell
kubectl apply -f crossplane/claims/redis-claim.yaml
```

Acompanhe:

```powershell
kubectl get rediscache app-redis-cache -o wide
kubectl get cluster.elasticache.aws.upbound.io app-redis-prod -o wide
kubectl get subnetgroup.elasticache.aws.upbound.io app-redis-prod-subnet-group -o wide
```

O resultado esperado é:

```text
SYNCED=True
READY=True
EXTERNAL-NAME=app-redis-prod
```

Consulte o endpoint:

```powershell
$env:AWS_PAGER = ""
aws elasticache describe-cache-clusters `
  --region us-east-1 `
  --cache-cluster-id app-redis-prod `
  --show-cache-node-info `
  --query "CacheClusters[0].CacheNodes[0].Endpoint" `
  --output json
```

## Como explicar a demonstração

1. Mostre a claim e diga: “Este é o pedido do consumidor.”
2. Mostre a XRD e diga: “Este é o contrato validado pela plataforma.”
3. Mostre a Composition e diga: “Aqui está a implementação concreta na AWS.”
4. Mostre os MRs e diga: “O Crossplane criou os objetos que representam os recursos externos.”
5. Mostre `READY=True` e diga: “O estado desejado foi reconciliado com sucesso.”
6. Mostre o console AWS em `us-east-1` como evidência externa.

## Perguntas prováveis

### O Crossplane criou as subnets?

Não. As subnets já existiam na VPC. O Crossplane criou o Subnet Group do ElastiCache usando os IDs fornecidos.

### O Redis roda no Docker Desktop?

Não. O Kubernetes e os controllers rodam localmente no Docker Desktop, mas o Redis é provisionado na AWS.

### Por que existem duas subnets?

O Subnet Group usa subnets em zonas distintas para atender ao modelo de rede do ElastiCache e aumentar a resiliência da implantação.

### O que acontece se eu alterar a claim?

O Crossplane detecta a diferença e tenta atualizar o recurso externo, respeitando o schema, os patches e as políticas do provider.

### O que acontece se eu apagar a claim?

Com `deletionPolicy: Delete`, os managed resources podem ser removidos junto com o recurso externo. Isso deve ser explicado como comportamento de laboratório e revisado antes de produção.

### Por que não colocar credenciais no Git?

Porque o Git é persistente e auditável. O Secret deve ser criado por um mecanismo seguro, e em produção devem ser preferidas identidades sem chave estática.

## Troubleshooting para a apresentação

```powershell
kubectl describe rediscache app-redis-cache
kubectl describe cluster.elasticache.aws.upbound.io app-redis-prod
kubectl get events -A --sort-by=.lastTimestamp
kubectl logs -n crossplane-system -l pkg.crossplane.io/provider=provider-aws-elasticache
```

Interpretação rápida:

- `XRD Established=False`: contrato inválido ou ainda sendo registrado.
- `Synced=False`: erro de composição, provider ou credencial.
- `Synced=True` e `Ready=False`: reconciliando ou aguardando a AWS.
- `EXTERNAL-NAME` vazio: ainda não houve associação com o recurso AWS.
- `READY=True`: recurso pronto para consumo.

## Cuidados durante a apresentação

- Confirmar a conta e a região AWS antes da demo.
- Não mostrar access keys, Secret ou arquivos de credencial.
- Não executar `kubectl delete` ao vivo.
- Não usar `Reset Kubernetes cluster`.
- Confirmar que o recurso é de PoC e pode gerar custo.
- Manter o terminal pronto com comandos curtos, um por vez.

## Fechamento

Finalize com:

> O valor do Crossplane é oferecer uma interface Kubernetes padronizada para produtos de infraestrutura, escondendo detalhes da AWS sem perder o controle declarativo, auditável e reconciliável do GitOps.
