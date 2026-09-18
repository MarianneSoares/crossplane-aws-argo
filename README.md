# crossplane-aws-argo

Provisiona infraestrutura AWS (bucket S3 + cache Redis via ElastiCache) usando
[Crossplane](https://www.crossplane.io/) (provider-family-aws) com entrega GitOps via [ArgoCD](https://argo-cd.readthedocs.io/).

## Estrutura

```
crossplane/
  providers/        # Provider packages + ProviderConfig (credenciais AWS)
  definitions/       # XRDs (Composite Resource Definitions)
  compositions/       # Compositions que implementam as XRDs
  claims/             # Claims (o que o time de app realmente pede)
argocd/
  applications/       # Applications filhas (app-of-apps)
  projects/           # AppProject do Argo CD
  app-of-apps.yaml     # Application raiz
```

Ordem de aplicação (controlada por `sync-wave` no Argo CD):

1. `providers` (wave -2): instala `provider-family-aws`, `provider-aws-s3`, `provider-aws-elasticache` e o `ProviderConfig`.
2. `definitions` (wave -1): instala as XRDs.
3. `compositions` (wave 0): instala as Compositions.
4. `claims` (wave 1): cria o bucket S3 e o Redis de fato.

## Pré-requisitos

- Cluster Kubernetes com [Crossplane](https://docs.crossplane.io/latest/software/install/) instalado (`crossplane-system` namespace).
- [Argo CD](https://argo-cd.readthedocs.io/en/stable/getting_started/) instalado no cluster.
- Uma credencial AWS (access key/secret) com permissão para criar buckets S3 e clusters ElastiCache.
- Este repositório publicado em um Git remoto (Argo CD sincroniza a partir de um repo Git).

## Configurando as credenciais AWS

**Não** commit suas credenciais. Crie o secret manualmente no cluster antes de sincronizar o Argo CD:

```bash
kubectl create namespace crossplane-system --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret generic aws-creds \
  -n crossplane-system \
  --from-file=creds=./aws-credentials.ini
```

Onde `aws-credentials.ini` tem o formato:

```ini
[default]
aws_access_key_id = AKIA...
aws_secret_access_key = ...
```

## Instalando o Argo CD apontando para este repo

Edite `argocd/app-of-apps.yaml` e ajuste `spec.source.repoURL` para a URL do seu repositório Git, então aplique:

```bash
kubectl apply -f argocd/projects/appproject.yaml
kubectl apply -f argocd/app-of-apps.yaml
```

O Argo CD vai então criar e sincronizar as demais Applications automaticamente (app-of-apps).

## Ajustando os recursos

- Edite `crossplane/claims/bucket-claim.yaml` para o nome do bucket, região e tags.
- Edite `crossplane/claims/redis-claim.yaml` para o tamanho/tipo de node do Redis, número de nós e região.

## Versões dos providers

Os manifests em `crossplane/providers/provider-family-aws.yaml` fixam uma versão (`v1`/`v1.x`) dos
pacotes `provider-family-aws`, `provider-aws-s3` e `provider-aws-elasticache`. Verifique as versões mais
recentes em https://marketplace.upbound.io/providers e atualize os campos `spec.package` conforme necessário.
