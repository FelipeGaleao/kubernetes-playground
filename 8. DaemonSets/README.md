# 👹 DaemonSets

## Descrição

Nesta pasta você encontrará exemplos de **DaemonSets** no Kubernetes, que garantem que um Pod seja executado em **cada Node** do cluster.

## O que é um DaemonSet?

Um DaemonSet é um objeto Kubernetes que:
- **Executa um Pod em cada Node**: automaticamente cria e gerencia Pods em todos (ou alguns) Nodes
- **Gera novos Pods automaticamente**: quando novos Nodes são adicionados, o DaemonSet cria Pods neles
- **Remove Pods automaticamente**: quando Nodes são removidos, os Pods também são removidos
- **Garante presença**: é a ferramenta perfeita para executar algo em cada máquina

## Casos de Uso

### 1. **Monitoramento**
```
Executar agente de monitoramento (Prometheus, DataDog) em cada Node
Node 1: Prometheus Agent
Node 2: Prometheus Agent
Node 3: Prometheus Agent
```

### 2. **Logging**
```
Coletar logs de cada Node (Fluentd, Logstash)
Node 1: Fluentd
Node 2: Fluentd
Node 3: Fluentd
```

### 3. **Storage**
```
Plugin de storage em cada Node (Rook, Ceph)
Node 1: Storage Agent
Node 2: Storage Agent
Node 3: Storage Agent
```

### 4. **Redes**
```
Proxy de rede em cada Node (Calico, Weave)
Node 1: Network Plugin
Node 2: Network Plugin
Node 3: Network Plugin
```

### 5. **Segurança**
```
Agente de segurança em cada Node (Falco)
Node 1: Security Agent
Node 2: Security Agent
Node 3: Security Agent
```

## Diferenças: ReplicaSet vs DaemonSet

| Aspecto | ReplicaSet | DaemonSet |
|---------|-----------|----------|
| **Distribuição** | Qualquer Node | Cada Node |
| **Número de réplicas** | Configurável | = Número de Nodes |
| **Replicação** | Manual | Automática por Node |
| **Caso de uso** | Aplicações | Componentes do sistema |

## Arquivos nesta pasta

### `my-daemonset.yaml`
DaemonSet simples com nginx:
- **Nome**: my-daemonset
- **Container**: nginx
- **Labels**: `apps: my-app`, `tier: frontend`
- **Comportamento**: Executará um Pod nginx em CADA Node do cluster
- Sem `replicas` (não faz sentido em DaemonSets)

**Exemplo com 3 Nodes:**
```
Node 1 → Pod nginx (my-daemonset-abc123)
Node 2 → Pod nginx (my-daemonset-def456)
Node 3 → Pod nginx (my-daemonset-ghi789)
```

### `my-replicaset.yaml`
ReplicaSet para comparação:
- **Nome**: frontend-rs
- **Replicas**: 4
- **Container**: nginx
- **Distribuição**: 4 Pods distribuídos entre os Nodes (não garante 1 por Node)

## Como usar

### Criar um DaemonSet
```bash
kubectl apply -f my-daemonset.yaml
```

### Verificar DaemonSets
```bash
kubectl get daemonsets
kubectl get ds  # forma abreviada
```

### Ver Pods criados pelo DaemonSet
```bash
kubectl get pods -o wide
# Observe que tem um Pod em cada Node
```

### Descrever um DaemonSet
```bash
kubectl describe daemonset my-daemonset
```

Procure por:
- `Desired Number of Nodes Scheduled With a Ready Daemon Pod`
- `Current Number of Nodes Scheduled with Running Daemon Pod`
- `Number of Nodes Scheduled with Available Daemon Pod`

### Monitorar em tempo real
```bash
kubectl get ds -w  # watch mode
```

### Ver logs de um Pod do DaemonSet
```bash
kubectl logs <pod-name>
```

### Deletar um DaemonSet
```bash
kubectl delete daemonset my-daemonset
# Todos os Pods associados serão deletados automaticamente
```

### Ver quais Nodes têm Pods
```bash
kubectl get pods -o wide
```

## NodeSelector e Tolerations

### Executar DaemonSet em Nodes específicos

#### Com nodeSelector
```yaml
spec:
  template:
    spec:
      nodeSelector:
        disk: ssd  # Apenas em Nodes com este label
```

#### Com tolerations (para Nodes tainted)
```yaml
spec:
  template:
    spec:
      tolerations:
      - key: "dedicated"
        operator: "Equal"
        value: "logging"
        effect: "NoSchedule"
```

## Atualização de DaemonSet

### Estratégia de Update

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
```

- **RollingUpdate** (padrão): atualiza um Pod por vez
- **OnDelete**: atualiza apenas ao deletar o Pod

### Ver histórico de atualizações
```bash
kubectl rollout history daemonset my-daemonset
```

### Fazer rollback
```bash
kubectl rollout undo daemonset my-daemonset
```

## Casos práticos

### DaemonSet para Monitoramento Prometheus
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: prometheus-node-exporter
spec:
  template:
    spec:
      containers:
      - name: node-exporter
        image: prom/node-exporter:latest
        ports:
        - containerPort: 9100
  selector:
    matchLabels:
      app: node-exporter
```

### DaemonSet para Coleta de Logs
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
spec:
  template:
    spec:
      containers:
      - name: fluentd
        image: fluent/fluentd:latest
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
  selector:
    matchLabels:
      app: fluentd
```

## Status de um DaemonSet

Ao descrever, você verá estatísticas:

```
Desired Number of Nodes Scheduled With a Ready Daemon Pod: 3
Current Number of Nodes Scheduled with Running Daemon Pod: 3
Number of Nodes Scheduled with Available Daemon Pod: 3
Number of Nodes Misscheduled: 0
```

- **Desired**: Quantos Nodes deveriam ter o Pod
- **Current**: Quantos Nodes têm o Pod rodando
- **Available**: Quantos Pods estão prontos
- **Misscheduled**: Quantos Pods estão em Nodes errados

## Boas práticas

✅ **Faça**:
- Use para componentes do sistema
- Configure limits e requests apropriados
- Use nodeSelector para controlar placement
- Monitore saúde com readiness probes
- Teste em staging antes de produção

❌ **Não faça**:
- Não use para aplicações normais (use Deployments)
- Não deixe sem limits (pode sobrecarregar Nodes)
- Não execute no Node master sem toleration explícita
- Não ignore updateStrategy

## Próximos passos

- Combine com **4. Services** para expor Pods de monitoramento
- Use **nodeSelector** para placement inteligente
- Explore **Taints e Tolerations** para controle fino
- Compare com **StatefulSets** para aplicações com estado

