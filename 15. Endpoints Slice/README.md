# 🔗 Endpoints Slice (Moderno)

## Descrição

Nesta pasta você encontrará exemplos de **EndpointSlices** no Kubernetes, que é a evolução moderna dos Endpoints com melhor escalabilidade e performance.

## O que é EndpointSlice?

Um EndpointSlice é um objeto Kubernetes que:
- **Substitui Endpoints legados**: API moderna (`discovery.k8s.io/v1`)
- **Escalabilidade**: divide endereços em slices de até 1000 por slice
- **Performance**: updates incrementais em vez de replace total
- **Padrão desde 1.19+**: recomendado para novos clusters
- **Compatível com Services**: transparente para usuários

## Endpoints vs EndpointSlice

| Aspecto | Endpoints (legado) | EndpointSlice (moderno) |
|---------|-------------------|------------------------|
| **API** | v1 | discovery.k8s.io/v1 |
| **Limite** | 1000 endereços | 1000 por slice (divisível) |
| **Performance** | Degrada com escala | Consistente |
| **Update** | Replace total | Incremental |
| **Criação** | Automática com Service | Automática com Service |
| **Kubernetes** | Todas versões | 1.16+ (GA em 1.19+) |
| **Status** | Deprecado | ✅ Recomendado |

## Quando usar EndpointSlice?

✅ **Use EndpointSlice para**:
- Kubernetes 1.19+
- Clusters com muitos Pods
- Serviços com centenas de endereços
- Performance e escalabilidade importantes

❌ **Endpoints legados ainda usados para**:
- Compatibilidade retroativa
- Integração com ferramentas antigas

## Comparação com Endpoints legados

### Endpoints (v1)
```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: my-service
subsets:
  - addresses:
    - ip: 10.244.0.10
    - ip: 10.244.0.11
    - ip: 10.244.0.12
    ports:
    - port: 80
```

**Problema**: Se tem 5000 endereços, um Endpoint com 5000 linhas! Atualizações lentas.

### EndpointSlice (discovery.k8s.io/v1)
```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: my-service-1
  labels:
    kubernetes.io/service-name: my-service
endpoints:
  - addresses: ["10.244.0.10", "10.244.0.11", ...]
ports:
  - name: http
    port: 80
---
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: my-service-2
  labels:
    kubernetes.io/service-name: my-service
endpoints:
  - addresses: ["10.244.0.1000", "10.244.0.1001", ...]
ports:
  - name: http
    port: 80
```

**Vantagem**: Dividido em múltiplas slices, cada uma com até 1000 endereços!

## Arquivos nesta pasta

### `my-endpoints-test-eps.yaml`
Exemplo de EndpointSlices manuais:

```yaml
---
# Service sem selector (requer EndpointSlices manuais)
apiVersion: v1
kind: Service
metadata:
  name: my-eps-service
spec:
  ports:
  - name: http
    port: 80

---
# EndpointSlice 1 (Apache)
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: my-eps
  labels:
    kubernetes.io/service-name: my-eps-service
addressType: IPv4
endpoints:
  - addresses: ["10.244.0.10"]  # Apache
ports:
  - name: http
    port: 80
    protocol: TCP

---
# EndpointSlice 2 (Nginx)
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: my-eps-2
  labels:
    kubernetes.io/service-name: my-eps-service
addressType: IPv4
endpoints:
  - addresses: ["10.244.0.11"]  # Nginx
ports:
  - name: http
    port: 80
    protocol: TCP
```

**Características**:
- 2 EndpointSlices separados (divisão lógica)
- Label vincula ao Service: `kubernetes.io/service-name`
- `addressType: IPv4` (também suporta IPv6)
- Endpoints em listas separadas para escalabilidade

### `test-auto-mirroring.yaml`
Demonstração de auto-mirroring (Kubernetes cria EndpointSlices):

```yaml
---
# Pod com label
apiVersion: v1
kind: Pod
metadata:
  name: pod-nginx
  labels:
    app: app.nginx
spec:
  containers:
  - name: my-nginx
    image: nginx

---
# Service com selector
apiVersion: v1
kind: Service
metadata:
  name: svc
spec:
  selector:
    app: app.nginx    # Auto-discovery de Pods
  ports:
  - name: http
    protocol: TCP
    port: 80
```

**Conceito**: Quando Service tem selector:
1. Kubernetes detecta Pods com matching labels
2. Kubernetes **cria EndpointSlices automaticamente**
3. Você não precisa criar manualmente!

**Auto-mirroring** = conversão automática de Endpoints para EndpointSlices.

## Como usar

### 1. Criar EndpointSlice manual
```bash
kubectl apply -f my-endpoints-test-eps.yaml
```

### 2. Ver EndpointSlices
```bash
kubectl get endpointslices
kubectl get eps  # forma abreviada

# Output:
# NAME             ADDRESSTYPE   ENDPOINTS
# my-eps           IPv4          1
# my-eps-2         IPv4          1
```

### 3. Descrever EndpointSlice
```bash
kubectl describe endpointslice my-eps

# Output:
# Name: my-eps
# Labels: kubernetes.io/service-name=my-eps-service
# Address Type: IPv4
# Endpoints:
#   - 10.244.0.10:80
# Ports:
#   - Name: http
#     Port: 80
```

### 4. Ver EndpointSlices automáticas (com selector)
```bash
# Aplicar Service COM selector
kubectl apply -f test-auto-mirroring.yaml

# Ver EndpointSlices criadas automaticamente
kubectl get endpointslices

# Kubernetes criou automaticamente!
```

### 5. Editar EndpointSlice
```bash
kubectl edit endpointslice my-eps

# Adicionar/remover endereços
```

### 6. Monitorar mudanças
```bash
kubectl get endpointslices -w

# Observe mudanças em tempo real conforme Pods são criados/deletados
```

### 7. Deletar EndpointSlice
```bash
kubectl delete endpointslice my-eps
```

⚠️ **Cuidado**: Se EndpointSlice é automática (tem selector), será recriada automaticamente!

## Estrutura detalhada

### Metadata
```yaml
metadata:
  name: my-service-1
  namespace: default
  labels:
    kubernetes.io/service-name: my-service  # Vincula ao Service
```

### AddressType
Tipo de endereço suportado:

```yaml
addressType: IPv4    # ou IPv6 (ou ambos)
```

### Endpoints
Lista de endereços:

```yaml
endpoints:
- addresses: ["10.244.0.10", "10.244.0.11"]
  hostname: ["pod-0", "pod-1"]              # Opcional
  targetRef: &ref                           # Opcional
    kind: Pod
    name: my-pod
  conditions:
    ready: true                             # Pronto?
    serving: true                           # Servindo?
    terminating: false                      # Terminando?
```

### Ports
Portas disponíveis:

```yaml
ports:
- name: http
  port: 80
  protocol: TCP                    # TCP ou UDP
  appProtocol: "http"             # Opcional
```

## Casos de uso

### 1. Serviço com muitos Pods (escalado)
```yaml
# Service com 5000 Pods seria 1 Endpoint gigante
# Com EndpointSlice: 5 slices de 1000 cada

# Criado automaticamente por Kubernetes!
```

### 2. Integração com múltiplos backends
```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: my-app-backend
spec:
  ports:
  - port: 8080

---
# EndpointSlice 1: Datacenter 1
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: my-app-dc1
  labels:
    kubernetes.io/service-name: my-app-backend
endpoints:
  - addresses: ["192.168.1.100", "192.168.1.101", ...]

---
# EndpointSlice 2: Datacenter 2
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: my-app-dc2
  labels:
    kubernetes.io/service-name: my-app-backend
endpoints:
  - addresses: ["192.168.2.100", "192.168.2.101", ...]
```

### 3. Dual-stack (IPv4 + IPv6)
```yaml
---
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: my-service-v4
  labels:
    kubernetes.io/service-name: my-service
addressType: IPv4
endpoints:
  - addresses: ["10.244.0.10", "10.244.0.11"]

---
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: my-service-v6
  labels:
    kubernetes.io/service-name: my-service
addressType: IPv6
endpoints:
  - addresses: ["2001:db8::1", "2001:db8::2"]
```

## Performance: Endpoints vs EndpointSlice

### Endpoints (legado) - 5000 endereços
```
1 Endpoints com 5000 linhas
Update: Recria tudo
Tamanho: ~500KB no etcd
Performance: ❌ Lenta
```

### EndpointSlice (moderno) - 5000 endereços
```
5 EndpointSlices com 1000 cada
Update: Apenas slice afetada
Tamanho: ~50KB por slice no etcd
Performance: ✅ Rápida
```

**Impacto**:
- Menos contention no etcd
- Updates mais rápidos
- Melhor performance de API
- Escalabilidade até 1M+ endereços

## Auto-mirroring (conversão automática)

Desde Kubernetes 1.21+, Endpoints são **automaticamente convertidos para EndpointSlices**:

```
Service COM selector
      ▼
Kubernetes cria automaticamente
      ▼
EndpointSlices (moderno)
      ▼
Endpoints legados (para compatibilidade)
```

Você não precisa fazer nada! É transparente.

## Condições de Endpoint

EndpointSlice suporta condições para cada endpoint:

### Ready
O Pod está pronto para receber tráfego?

```yaml
conditions:
  ready: true    # Passou readiness probe
```

### Serving
O Pod está servindo (inclui condições customizadas)?

```yaml
conditions:
  serving: true  # Pode incluir lógica customizada
```

### Terminating
O Pod está terminando?

```yaml
conditions:
  terminating: true  # Graceful shutdown em progresso
```

## Observabilidade

### Ver quais EndpointSlices existem
```bash
kubectl get endpointslices -o wide

# Mostra Service name e quantidade de endpoints
```

### Monitorar mudanças
```bash
# Observe criação/deleção em tempo real
watch kubectl get endpointslices
```

### Métricas Prometheus
```
# Endereços por slice
endpoint_slice_endpoints{service="my-service"} 100

# Total de slices
endpoint_slices{service="my-service"} 5
```

## Troubleshooting

### EndpointSlice não está sendo criada
```bash
# 1. Verificar Service
kubectl get svc my-service

# 2. Verificar se tem selector
kubectl get svc my-service -o yaml | grep selector

# 3. Verificar Pods com labels
kubectl get pods --show-labels

# 4. Labels devem corresponder ao selector!
```

### EndpointSlice vazia
```bash
# 1. Service tem selector?
kubectl describe svc my-service

# 2. Pods estão com labels corretos?
kubectl get pods -L app

# 3. Pods estão rodando?
kubectl get pods
```

### Muitas EndpointSlices
```bash
# Esperado para serviços com muitos Pods
# Limite: 1000 endpoints por slice

# Exemplo: 5000 Pods = 5 slices (normal)

kubectl get endpointslices | wc -l
```

## Boas práticas

✅ **Faça**:
- Use **EndpointSlice** para novos clusters (1.19+)
- Deixe Kubernetes criar automaticamente (com selector)
- Crie EndpointSlices manuais para **integração externa**
- Monitore quantidade de slices para performance
- Use labels corretos para vincular ao Service

❌ **Não faça**:
- Não edite EndpointSlices automáticas (serão sobrescritas)
- Não confunda com Endpoints (são coisas diferentes)
- Não deixe EndpointSlices órfãs (remova se Service deletado)
- Não misture tipos de endereço em uma slice (IPv4 vs IPv6)

## Migração de Endpoints para EndpointSlice

Se tem cluster antigo:

```bash
# 1. Atualizar Kubernetes para 1.19+
# 2. Kubernetes converte automaticamente
# 3. Endpoints legados continuam funcionando
# 4. Novo código usa EndpointSlices

# Ver ambas
kubectl get endpoints
kubectl get endpointslices
```

## Próximos passos

- Compare com **14. Endpoints** (legado)
- Use com **4. Services** para descoberta
- Monitore com **Prometheus** para health
- Explore **kube-proxy** para roteamento
- Estude **networking** para compreender fluxo completo

