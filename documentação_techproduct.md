# OCI Queue — Guia de Uso

## 0. Quickstart de Integração

### O que é / quando usar
Fila de mensagens totalmente gerenciada da OCI. Use para desacoplar produtor e consumidor
(ex.: um serviço grava um evento, outro processa depois, sem chamada síncrona direta).

### Onde encontrar os dados de conexão
Após criar o recurso pelo Platform Code, acesse o card de detalhe do recurso — os campos
`resourceName` e `cloudResourceId` (endpoint da fila na OCI) ficam disponíveis ali.
Não copie o endpoint manualmente para configuração fixa da aplicação — leia sempre do
detalhe do recurso ou de uma variável de ambiente injetada no deploy.

### Como autenticar
Nunca use chave de API estática. A aplicação autentica via **Instance Principal** (quando
roda no OKE) ou **Resource Principal** (quando roda em outro serviço gerenciado da OCI) —
a identidade já vem com a role de acesso à fila atribuída pelo Platform Code na criação.

### Snippet de código mínimo

**Node.js (NestJS)**
```ts
import { QueueClient } from 'oci-queue';
import { InstancePrincipalsAuthenticationDetailsProviderBuilder } from 'oci-common';

const provider = await new InstancePrincipalsAuthenticationDetailsProviderBuilder().build();
const client = new QueueClient({ authenticationDetailsProvider: provider });

// Enviar mensagem
await client.putMessages({
  queueId: process.env.OCI_QUEUE_ID,
  putMessagesDetails: { messages: [{ content: JSON.stringify({ evento: 'pedido_criado' }) }] }
});

// Consumir mensagem
const { getMessages } = await client.getMessages({ queueId: process.env.OCI_QUEUE_ID });
```

**Java (Spring)**
```java
InstancePrincipalsAuthenticationDetailsProvider provider =
    InstancePrincipalsAuthenticationDetailsProvider.builder().build();
QueueClient client = QueueClient.builder().build(provider);

client.putMessages(PutMessagesRequest.builder()
    .queueId(System.getenv("OCI_QUEUE_ID"))
    .putMessagesDetails(PutMessagesDetails.builder()
        .messages(List.of(PutMessagesDetailsEntry.builder()
            .content("{\"evento\":\"pedido_criado\"}").build()))
        .build())
    .build());
```
> ⚠️ Exemplo ilustrativo — validar a assinatura exata dos métodos contra a versão do SDK
> instalada no momento de escrever a doc real (a API pode ter mudado).

### Variáveis de ambiente esperadas
| Variável | Descrição |
|---|---|
| `OCI_QUEUE_ID` | ID da fila, obtido no card de detalhe do recurso no Platform Code |

### Erros comuns de integração
| Erro | Causa provável |
|---|---|
| `404 QueueNotFound` | `OCI_QUEUE_ID` errado/de outro ambiente (confundir dev/test/prod) |
| `401/403 NotAuthorizedOrNotFound` | Identidade (Instance/Resource Principal) sem a policy de acesso à fila — confirmar com o time de plataforma se o recurso foi criado corretamente |
| Mensagem nunca é consumida | Falta de long polling / timeout de visibilidade mal configurado — ver documentação completa, seção de configurações técnicas |

---

## 1. O que é o OCI Queue
[Descrição do serviço, análoga à documentação oficial da Oracle — 1 parágrafo + link para
a doc oficial. Não repetir a Seção 0 aqui.]

### 1.1 Principais características
[Lista de características do serviço — mesmo padrão das páginas atuais.]

## 2. Cenário de uso no ambiente Telefônica
[Quando este Tech Product é a escolha certa no contexto da empresa, link para eventual
matriz de decisão comparando com alternativas.]

## 3. Modelo de implantação no ambiente Telefônica
### 3.1 Definição das configurações técnicas
[Tabela dev/test/prod: escalabilidade, disponibilidade, tier, backup, etc.]

### 3.2 Guardrails
[Características obrigatórias: criptografia em trânsito/repouso, exposição pública,
tags, etc.]

## 4. Taxonomia dos componentes
[Padrão de nomenclatura dos recursos gerados por este Tech Product.]

## 5. Autenticação e autorização
### 5.1 Acesso sistêmico
[Como a identidade gerenciada da sigla acessa o recurso — Managed Identity/Instance
Principal, roles atribuídas.]

### 5.2 Acesso de usuário
[Tabela de permissões por perfil/ambiente, Control Plane vs. Data Plane, se aplicável.]

## 6. Observabilidade
[Link para a página central de observabilidade de Tech Products do provider aplicável.]

## 7. Estimativa de custos
[Como usar a calculadora de preços do provider (Azure/OCI) para este Tech Product
específico, parâmetros recomendados.]

## 8. Referências
[Links para documentação oficial do provider, matriz de decisão, arquitetura de
referência.]