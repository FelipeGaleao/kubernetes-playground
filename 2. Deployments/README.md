# 🚀 Deployments

## Descrição

Nesta pasta você encontrará exemplos de **Deployments** no Kubernetes, que são o método recomendado para implantar e gerenciar aplicações.

## O que é um Deployment?

Um Deployment é um objeto Kubernetes de nível superior que gerencia **ReplicaSets** e **Pods**. Ele fornece declaratividade e automação para implantações, atualizações e rollbacks de aplicações.

## Características principais

- **Gerenciamento automático**: mantém o número desejado de Pods em execução
- **Atualizações contínuas**: suporta diferentes estratégias de atualização
- **Rollback automático**: permite voltar a versões anteriores facilmente
- **Health checks**: pode verificar a saúde dos Pods
- **Escalabilidade**: ajuste de `replicas` para escalar a aplicação

## Estratégias de atualização (Strategy)

### 1. **Rolling Update** (padrão)
- Substitui os Pods gradualmente, sem downtime
- Ideal para aplicações que precisam de alta disponibilidade

### 2. **Recreate**
- Destrói todos os Pods existentes antes de criar os novos
- Pode gerar downtime durante a atualização
- Usado quando a aplicação não suporta múltiplas versões simultâneas

### 3. **Blue/Green**
- Cria um novo Deployment (Green) enquanto o antigo (Blue) permanece ativo
- Permite testes antes de fazer a transição

### 4. **Canary**
- Atualização gradual com pequena porcentagem de Pods novos
- Monitora a saúde antes de continuar

### 5. **A/B Testing**
- Similar ao Canary, mas com múltiplas versões simultâneas
- Permite comparar comportamentos diferentes

## Arquivos nesta pasta

### `my-deployment.yaml`
Exemplo completo de Deployment com:
- Metadata com rótulos (`frontend`)
- Template de Pod com nginx
- Selector usando `matchLabels` para gerenciar Pods
- Estratégia de atualização: **Recreate**
- 1 réplica ativa

## Como usar

### Criar um Deployment
```bash
kubectl apply -f my-deployment.yaml
```

### Verificar Deployments
```bash
kubectl get deployments
```

### Ver ReplicaSets (gerenciados automaticamente)
```bash
kubectl get replicasets
```

### Ver Pods (criados pelo Deployment)
```bash
kubectl get pods
```

### Descrever um Deployment
```bash
kubectl describe deployment frontend-deployment
```

### Escalar a aplicação (aumentar réplicas)
```bash
kubectl scale deployment frontend-deployment --replicas=3
```

### Atualizar imagem de um container
```bash
kubectl set image deployment/frontend-deployment nginx-container=nginx:latest
```

### Ver histórico de atualizações
```bash
kubectl rollout history deployment/frontend-deployment
```

### Fazer rollback para versão anterior
```bash
kubectl rollout undo deployment/frontend-deployment
```

### Monitorar atualização em tempo real
```bash
kubectl rollout status deployment/frontend-deployment
```

## Diferenças: Pod vs ReplicaSet vs Deployment

| Aspecto | Pod | ReplicaSet | Deployment |
|---------|-----|-----------|-----------|
| **Auto-recuperação** | Não | Sim | Sim |
| **Escalabilidade** | Não | Sim | Sim |
| **Atualizações** | Manual | Manual | Automática |
| **Rollback** | Não | Não | Sim |
| **Recomendação** | Testes/debug | Raramente | ✅ Sempre |

## Próximos passos

- Explore **4. Services** para expor Deployments externamente
- Veja **5. Liveness Prob** para adicionar health checks aos Deployments
- Consulte **6. Resources** para definir limites de CPU e memória


