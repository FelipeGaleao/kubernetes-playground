# 📊 Resources (Limites e Solicitações)

## Descrição

Nesta pasta você encontrará exemplos de como configurar **solicitações e limites de recursos** (CPU e memória) para seus contêineres no Kubernetes.

## O que são Requests e Limits?

### Requests (Solicitações)
A quantidade de recursos que **você garante** que o contêiner receberá
- Kubernetes usa isso para **scheduling** (decidir qual Node colocar o Pod)
- Se não houver recursos suficientes, o Pod permanecerá em `Pending`
- **Essencial para garantir performance previsível**

### Limits (Limites)
A quantidade máxima de recursos que o contêiner **pode usar**
- Se exceder, Kubernetes pode **throttle** (reduzir) ou matar o contêiner
- Protege o Node de ser sobrecarregado por um contêiner maligno/bugado
- **Essencial para segurança de vizinhança**

## Recursos gerenciáveis

### CPU
Unidade: **miliCPU (m)**
- 1 CPU = 1000m
- 500m = meia CPU
- 100m = 0.1 CPU

**Exemplos:**
- 1 CPU = `1` ou `1000m`
- Meia CPU = `500m` ou `0.5`
- Um décimo = `100m` ou `0.1`

### Memória
Unidade: **bytes** com sufixos
- Mi = Mebibyte (1024 × 1024 bytes)
- Gi = Gibibyte (1024 × 1024 × 1024 bytes)
- M = Megabyte (1000 × 1000 bytes)
- G = Gigabyte (1000 × 1000 × 1000 bytes)

**Exemplo:**
- 128Mi ≈ 128 Megabytes
- 512Mi ≈ 512 Megabytes
- 1Gi ≈ 1 Gigabyte

## Arquivos nesta pasta

### `resources-pod.yaml`
Exemplo com 2 contêineres com recursos diferentes:

**Apache Container:**
```
Requests:
  CPU: 500m (meia CPU)
  Memória: 128Mi

Limits:
  CPU: 1000m (1 CPU)
  Memória: 256Mi
```

**Redis Container:**
```
Requests:
  CPU: 400m
  Memória: 64Mi

Limits:
  CPU: 500m
  Memória: 128Mi
```

## Como usar

### Implantar o Pod
```bash
kubectl apply -f resources-pod.yaml
```

### Verificar recursos alocados
```bash
kubectl get pods
kubectl describe pod resources-pod
```

Procure pela seção `Requests` e `Limits`

### Monitorar uso real de recursos
```bash
kubectl top nodes        # Uso de recursos dos Nodes
kubectl top pods         # Uso de recursos dos Pods
kubectl top pods -n <namespace>  # Uso em um namespace específico
```

⚠️ **Nota**: Requer Metrics Server instalado no cluster

### Ver uso detalhado
```bash
kubectl describe node <node-name>
# Procure por "Allocated resources"
```

## Padrões de configuração

### Aplicações stateless (web)
```yaml
requests:
  cpu: "250m"
  memory: "256Mi"
limits:
  cpu: "500m"
  memory: "512Mi"
```

### Aplicações com banco de dados
```yaml
requests:
  cpu: "500m"
  memory: "512Mi"
limits:
  cpu: "2000m"
  memory: "1Gi"
```

### Aplicações CPU-intensive (processamento)
```yaml
requests:
  cpu: "1000m"
  memory: "512Mi"
limits:
  cpu: "2000m"
  memory: "1Gi"
```

### Aplicações memory-intensive
```yaml
requests:
  cpu: "500m"
  memory: "2Gi"
limits:
  cpu: "1000m"
  memory: "4Gi"
```

## QoS (Quality of Service) Classes

Kubernetes atribui classes de QoS baseado em requests/limits:

### 1. **Guaranteed** (Melhor)
- `requests == limits` para CPU e memória
- Nunca será evicted (removido) por falta de recursos
- Exemplo: todos os 3 campos iguais

### 2. **Burstable** (Médio)
- `requests < limits`
- Será evicted se Node ficar sem recursos
- Exemplo: requests menores que limits

### 3. **BestEffort** (Pior)
- Sem requests ou limits
- Será evicted primeiro em caso de contenção
- Exemplo: valores vazios

**Estratégia:**
```
Garantido > Burstable > BestEffort
```

## Scheduling (Agendamento)

### Como o Kubernetes escolhe um Node?

1. **Filtra Nodes**: remove Nodes sem recursos suficientes
2. **Classifica Nodes**: prioriza Nodes com mais espaço livre
3. **Binds Pod**: coloca o Pod no melhor Node

```
┌─────────────────┐
│   Seu Pod       │
│ Req: 500m CPU   │
│ Req: 256Mi RAM  │
└────────┬────────┘
         │
         ▼
┌─────────────────────────┐
│ Scheduler              │
│ Procura Node com:      │
│ - 500m CPU livres      │
│ - 256Mi RAM livres     │
└────────┬────────────────┘
         │
    ┌────┴────┐
    ▼         ▼
┌──────┐  ┌──────┐
│Node 1│  │Node 2│
│8 CPUs│  │4 CPUs│
│16 GB │  │8 GB  │
└──────┘  └──────┘
    │         ▼
    │    Not enough
    │    CPUs!
    │
    ▼
 Pod vai para Node 1
```

## Eviction (Remoção de Pods)

Quando Node fica sem recursos:

```
Ordem de remoção:
1. BestEffort pods (sem limits/requests)
2. Burstable pods (usando mais que requests)
3. Guaranteed pods (nunca!)
```

## Ferramentas úteis

### Ver recursos disponíveis no cluster
```bash
kubectl describe nodes
```

### Ver uso em tempo real
```bash
kubectl top pods
kubectl top nodes
```

### Estimar recursos necessários
```bash
# Rodar aplicação sem limites e monitorar
kubectl top pods --watch
```

### Limpar recursos
```bash
kubectl delete pod resources-pod
```

## Exemplos práticos

### Pod minimalista
```yaml
resources:
  requests:
    cpu: "100m"
    memory: "64Mi"
  limits:
    cpu: "200m"
    memory: "128Mi"
```

### Pod padrão
```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

### Pod pesado
```yaml
resources:
  requests:
    cpu: "1000m"
    memory: "2Gi"
  limits:
    cpu: "2000m"
    memory: "4Gi"
```

## Boas práticas

✅ **Faça**:
- Sempre defina requests e limits
- Monitore uso real de recursos
- Teste para encontrar valores apropriados
- Use QoS Guaranteed para aplicações críticas
- Revise e ajuste periodicamente

❌ **Não faça**:
- Não deixe em branco (BestEffort é arriscado)
- Não defina limits muito altos (desperdício)
- Não ignore Metrics Server (necessário para monitoramento)
- Não confie apenas em defaults

## Próximos passos

- Combine com **2. Deployments** para escalar baseado em recursos
- Use **Horizontal Pod Autoscaler (HPA)** para auto-scaling
- Explore **Vertical Pod Autoscaler (VPA)** para recomendações
- Veja **5. Liveness Prob** para garantir saúde sob pressão


