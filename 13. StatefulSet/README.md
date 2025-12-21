# 📊 StatefulSet

## Descrição

Nesta pasta você encontrará exemplos de **StatefulSets** no Kubernetes, que gerenciam aplicações **com estado** (stateful) e precisam de identidades estáveis e armazenamento persistente.

## O que é um StatefulSet?

Um StatefulSet é um objeto Kubernetes que:
- **Gerencia Pods com estado**: diferente do Deployment (stateless)
- **Fornece identidades estáveis**: nomes previsíveis e permanentes para Pods
- **Garante ordem de deployment**: Pods são criados em ordem (pod-0, pod-1, pod-2...)
- **Garante ordem de deleção**: Pods são deletados em ordem reversa
- **Armazenamento persistente**: cada Pod tem seu próprio volume
- **Descoberta de serviço estável**: Pods acessíveis via DNS headless

## StatefulSet vs Deployment

| Aspecto | Deployment | StatefulSet |
|---------|-----------|-------------|
| **Identidade** | ❌ Efêmera | ✅ Estável (pod-0, pod-1...) |
| **Hostname** | Aleatório | Previsível |
| **Armazenamento** | Compartilhado | Individual (PVC por Pod) |
| **Ordem** | Paralela | Sequencial |
| **DNS** | Genérico | Headless + individual |
| **Caso de uso** | Web apps, APIs | Bancos, caches, filas |
| **Escalabilidade** | Rápida | Controlada |

## Quando usar StatefulSet?

✅ **Use StatefulSet para**:
- **Bancos de dados**: MySQL, PostgreSQL, MongoDB com replicação
- **Cache com estado**: Redis, Memcached em cluster
- **Message queues**: RabbitMQ, Kafka com partições
- **Distributed storage**: Elasticsearch, Cassandra
- **Aplicações que precisam de dados persistentes**

❌ **Use Deployment para**:
- Aplicações web stateless (nginx, Node.js API)
- Serviços sem estado
- Escalabilidade horizontal simples

## Conceitos-chave

### 1. **Identidade estável (Ordinal Index)**
Cada Pod tem um número determinístico:
```
my-sts-0
my-sts-1
my-sts-2
```

Permanece constante mesmo após reinicializações!

### 2. **DNS Headless Service**
StatefulSet **requer** um Headless Service (`clusterIP: None`):

```
my-sts-0.svc-sts.default.svc.cluster.local → IP do Pod-0
my-sts-1.svc-sts.default.svc.cluster.local → IP do Pod-1
```

### 3. **PersistentVolumeClaim (PVC) por Pod**
Cada Pod tem seu próprio volume via `volumeClaimTemplates`:

```
Pod-0 → PVC-0 → Volume-0
Pod-1 → PVC-1 → Volume-1
Pod-2 → PVC-2 → Volume-2
```

### 4. **Ordem de operações**
```
Deploy (criar):     pod-0 → pronto → pod-1 → pronto → pod-2 → pronto
Scale down (deletar): pod-2 deletado → pod-1 deletado → pod-0 deletado
```

## Arquivo nesta pasta

### `svc-sts.yaml`
Exemplo completo com StatefulSet + Headless Service:

#### Service (linhas 1-10)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: svc-sts
spec:
  ports:
  - port: 80
  clusterIP: None           # Headless Service!
  selector:
    app: nginx-app-pods
```

**Características:**
- **Type**: implicitamente ClusterIP
- **clusterIP: None**: torna um Headless Service
- Seletor: pods com `app: nginx-app-pods`
- Porta 80 (HTTP)

#### StatefulSet (linhas 12-41)
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: my-sts
spec:
  serviceName: "svc-sts"           # Requer Headless Service
  selector:
    matchLabels:
      app: nginx-app-pods
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx-app-pods
    spec:
      containers:
      - name: my-container
        image: nginx:1.23.1
        volumeMounts:
        - name: my-pv
          mountPath: /usr/share/nginx/html
  
  volumeClaimTemplates:            # Cada Pod obtém seu PVC
  - metadata:
      name: my-pv
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

**Características:**
- **serviceName**: vincula ao Headless Service
- **replicas: 1**: cria 1 Pod (pod pode ser escalado)
- **volumeClaimTemplates**: template para criar PVCs automáticos
- Cada Pod obterá um PVC: `my-pv-0`, `my-pv-1`, etc.
- AccessMode `ReadWriteOnce`: apenas 1 Pod por volume

## Como usar

### 1. Criar StatefulSet + Service
```bash
kubectl apply -f svc-sts.yaml
```

### 2. Verificar StatefulSet
```bash
kubectl get statefulsets
kubectl get sts  # forma abreviada
```

### 3. Ver Pods criados
```bash
kubectl get pods
# Output:
# NAME        READY   STATUS    RESTARTS   AGE
# my-sts-0    1/1     Running   0          10s
```

Observe o nome determinístico `my-sts-0`!

### 4. Ver PVCs criados automaticamente
```bash
kubectl get pvc
# Output:
# NAME            STATUS   VOLUME
# my-pv-0         Bound    pvc-xxx
```

### 5. Descrever StatefulSet
```bash
kubectl describe sts my-sts

# Procure por:
# Pod Status:
#   Name: my-sts-0
#   Running and Ready
```

### 6. Acessar DNS do Pod
```bash
# De dentro de outro Pod
kubectl run -it debug --image=busybox --restart=Never -- sh

# Dentro do Pod:
nslookup my-sts-0.svc-sts
# Retorna o IP do Pod-0

nslookup svc-sts
# Retorna IPs de TODOS os Pods (A records)
```

### 7. Escalar StatefulSet
```bash
# Aumentar de 1 para 3 Pods
kubectl scale sts my-sts --replicas=3

# Cria: my-sts-0, my-sts-1, my-sts-2 (sequencialmente)
# Cria PVCs: my-pv-0, my-pv-1, my-pv-2 (automático)
```

### 8. Monitorar durante escalagem
```bash
kubectl get pods -w
# Observe a ordem: pod-0 → pronto → pod-1 → pronto → pod-2 → pronto
```

### 9. Deletar Pod específico
```bash
kubectl delete pod my-sts-1

# StatefulSet recria automaticamente com mesmo nome/volume!
# Pod volta como my-sts-1 (não my-sts-99)
```

### 10. Scale down
```bash
kubectl scale sts my-sts --replicas=1

# Deleta em ordem reversa: my-sts-2 → my-sts-1 (mantém my-sts-0)
```

### 11. Acessar dados no Pod
```bash
kubectl exec -it my-sts-0 -- ls /usr/share/nginx/html

# Dados persistem mesmo após Pod ser deletado e recriado!
```

### 12. Deletar StatefulSet (sem deletar PVCs)
```bash
# Conserva PVCs (dados não são perdidos)
kubectl delete sts my-sts

# Deletar também os PVCs
kubectl delete sts,pvc -l app=nginx-app-pods
```

## Headless Service

### O que é?
Um Headless Service (`clusterIP: None`) que:
- **Não fornece IP ClusterIP**: sem load balancing
- **Retorna IPs de Pods individuais**: DNS resolve para Pod específico
- **Necessário para StatefulSet**: permite descoberta de Pod por nome

### Por que precisa?

```
Com ClusterIP (Deployment):
meu-app.default.svc.cluster.local → 10.0.0.5 (load balancer)
                                 → Pod-A, Pod-B, Pod-C (round-robin)

Com Headless (StatefulSet):
meu-app.default.svc.cluster.local → 10.0.0.10 (Pod-0)
                                  → 10.0.0.11 (Pod-1)
                                  → 10.0.0.12 (Pod-2)

my-sts-0.svc-sts.default → 10.0.0.10 (Pod-0 específico)
my-sts-1.svc-sts.default → 10.0.0.11 (Pod-1 específico)
```

### Criar Headless Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  clusterIP: None        # Torna headless
  selector:
    app: my-app
  ports:
  - port: 80
```

## Armazenamento Persistente (PVC)

### volumeClaimTemplates
Cria um PVC para **cada Pod**:

```yaml
volumeClaimTemplates:
- metadata:
    name: data
  spec:
    accessModes: ["ReadWriteOnce"]
    resources:
      requests:
        storage: 10Gi
```

**Resultado com replicas: 3**:
```
Pod my-sts-0 → PVC data-0 → PV persistente
Pod my-sts-1 → PVC data-1 → PV persistente
Pod my-sts-2 → PVC data-2 → PV persistente
```

### AccessModes

| Mode | Significado | Uso |
|------|-----------|-----|
| **ReadWriteOnce (RWO)** | 1 Pod lê+escreve | Padrão, mais comum |
| **ReadOnlyMany (ROX)** | Múltiplos Pods lêem | Dados compartilhados |
| **ReadWriteMany (RWX)** | Múltiplos Pods lêem+escrevem | NFS, storage compartilhado |

⚠️ **Nota**: Nem todos os storage providers suportam todos os modes!

## Exemplos práticos

### Redis StatefulSet com replicação
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: redis
  replicas: 3
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 5Gi
```

### MySQL StatefulSet
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 20Gi
```

### Elasticsearch StatefulSet
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch
spec:
  serviceName: elasticsearch
  replicas: 3
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:8.0.0
        env:
        - name: discovery.seed_hosts
          value: "elasticsearch-0,elasticsearch-1,elasticsearch-2"
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 30Gi
```

## Ordem de operações detalhada

### Deploy (criar com replicas: 3)
```
T=0s:   Cria pod-0, aguarda ready
T=5s:   pod-0 ready ✅
T=5s:   Cria pod-1, aguarda ready
T=10s:  pod-1 ready ✅
T=10s:  Cria pod-2, aguarda ready
T=15s:  pod-2 ready ✅
T=15s:  Todos prontos ✅
```

### Scale down (replicas: 3 → 1)
```
T=0s:   Delete pod-2, aguarda finalização
T=5s:   pod-2 deletado ✅
T=5s:   Delete pod-1, aguarda finalização
T=10s:  pod-1 deletado ✅
T=10s:  Apenas pod-0 resta ✅
```

### Rebuild (recreate pod)
```
# Delete pod-1
kubectl delete pod my-sts-1

T=0s:   Pod deletado
T=0s:   StatefulSet recria automaticamente
T=5s:   my-sts-1 cria com mesmo nome e PVC!
```

## Boas práticas

✅ **Faça**:
- Use StatefulSet para aplicações com estado
- Sempre configure Headless Service
- Defina `podManagementPolicy: Parallel` para paralelização (se apropriado)
- Use PVC templates para persistência automática
- Monitore saúde com readiness probes
- Configure recursos (requests/limits)
- Documente ordem de inicialização esperada

❌ **Não faça**:
- Não use StatefulSet para aplicações stateless
- Não esqueça do Headless Service
- Não deletar PVCs manualmente (dados perdidos!)
- Não mude `serviceName` após criação
- Não ignore ordem de deployment
- Não use `replicas: 0` por tempo prolongado

## Troubleshooting

### StatefulSet não está criando Pods
```bash
# 1. Verificar StatefulSet
kubectl describe sts my-sts

# 2. Verificar Service
kubectl get svc svc-sts

# 3. Ver eventos
kubectl get events --sort-by='.lastTimestamp'
```

### PVC não está sendo criado
```bash
# 1. Verificar StorageClass
kubectl get storageclass

# 2. Descrever StatefulSet
kubectl describe sts my-sts

# 3. Verificar recursos de storage
kubectl get pv
kubectl get pvc
```

### Pod fica em Pending
```bash
# 1. Verificar PVC associado
kubectl get pvc

# 2. Descrever PVC
kubectl describe pvc my-pv-0

# 3. Verificar nós disponíveis
kubectl get nodes
```

### Dados perdidos após deletar
```bash
# Cuidado: deletar StatefulSet deleta Pods mas NÃO deleta PVCs
# Os dados ainda estão lá!

# Recuperar: recriar StatefulSet com mesmo nome
kubectl apply -f svc-sts.yaml

# Pods serão recriados com mesmos nomes e volumes!
```

## Próximos passos

- Combine com **4. Services** para Headless Services
- Use **6. Resources** para definir limites
- Explore **Affinity rules** para distribuição de Pods
- Implemente **backup de dados** de PVCs
- Configure **Operators** para gerenciar StatefulSets complexos (Prometheus, MySQL Operator)

