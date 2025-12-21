# 🔗 Endpoints

## Descrição

Nesta pasta você encontrará exemplos de **Endpoints** e **EndpointSlices** no Kubernetes, que representam os IPs e portas reais dos Pods que um Service pode alcançar.

## O que é um Endpoint?

Um Endpoint é um objeto Kubernetes que:
- **Armazena lista de IPs**: IPs e portas dos Pods que um Service pode rotear
- **Criado automaticamente**: quando um Service com selector é criado
- **Atualizado dinamicamente**: conforme Pods são criados/deletados
- **Usado para roteamento**: Service usa Endpoints para saber para onde rotear tráfego
- **Permite integração externa**: Endpoints pode apontar para serviços fora do cluster

## Service vs Endpoint

```
┌──────────────────────────────────────────────────┐
│ Cliente (dentro ou fora do cluster)              │
└────────────────┬─────────────────────────────────┘
                 │ curl http://my-service:80
                 ▼
┌──────────────────────────────────────────────────┐
│ Service (my-service)                             │
│ - clusterIP: 10.0.0.5                           │
│ - port: 80                                       │
│ - selector: app: web                            │
└────────────────┬─────────────────────────────────┘
                 │ lookup Endpoints com selector
                 ▼
┌──────────────────────────────────────────────────┐
│ Endpoints (my-service)                           │
│ - 10.244.0.10:80 (Pod-A)                        │
│ - 10.244.0.11:80 (Pod-B)                        │
│ - 10.244.0.12:80 (Pod-C)                        │
└────────────────┬─────────────────────────────────┘
                 │ load balance entre
                 ▼
         Roteamento para Pod
```

## Tipos de Endpoints

### 1. **Endpoints (Legado)**
Objeto que agrupa todos os IPs de um Service

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: my-service
subsets:
  - addresses:
    - ip: 10.244.0.10
    - ip: 10.244.0.11
    ports:
    - port: 80
```

⚠️ **Nota**: Pode crescer muito em clusters grandes (limite de 1000 endereços)

### 2. **EndpointSlice (Recomendado)**
Versão mais nova que divide Endpoints em slices menores

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: my-service-1
  labels:
    kubernetes.io/service-name: my-service
endpoints:
  - addresses: ["10.244.0.10", "10.244.0.11"]
    ports:
      - name: http
        port: 80
```

**Vantagens**:
- Escalabilidade: 1000 endpoints por slice
- Performance: updates incrementais
- Padrão atual (desde Kubernetes 1.19+)

## Ciclo de vida

### Quando Endpoints são criados?

1. **Service COM selector** → Endpoints criados automaticamente
```yaml
kind: Service
spec:
  selector:
    app: web  # Busca Pods com este label
```
Kubernetes cria Endpoints automaticamente quando Pods com `app: web` existem.

2. **Service SEM selector** → Endpoints manuais necessários
```yaml
kind: Service
spec:
  # Sem selector!
```
Você precisa criar Endpoints manualmente para apontar para outro lugar.

### Atualização automática

```
Novo Pod criado com label "app: web"
         ▼
Kubernetes detecta
         ▼
Atualiza Endpoints
         ▼
IPs adicionados automaticamente
```

## Arquivos nesta pasta

### `my-endpoints-pods.yaml`
Dois Pods simples para teste:

```yaml
---
# Pod 1: Apache (comentário sugere 10.244.0.10)
apiVersion: v1
kind: Pod
metadata:
  name: pod-apache
spec:
  containers:
  - name: my-apache
    image: httpd

---
# Pod 2: Nginx (comentário sugere 10.244.0.11)
apiVersion: v1
kind: Pod
metadata:
  name: pod-nginx
spec:
  containers:
  - name: my-nginx
    image: nginx
```

**Propósito**: Fornece Pods que serão referenciados pelos Endpoints.

### `my-endpoints-test.yaml`
Exemplo de **Endpoints manual** + Service **sem selector**:

```yaml
---
# Endpoints manual com IP externo
apiVersion: v1
kind: Endpoints
metadata:
  name: my-endpoints-service
subsets:
  - addresses:
    - ip: 77.68.88.76  # IPS externo (fora do cluster)
    ports:
    - port: 80

---
# Service sem selector
apiVersion: v1
kind: Service
metadata:
  name: my-endpoints-service
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

**Conceito importante**:
- Service **sem selector** requer Endpoints manual
- Endpoints aponta para IP externo (77.68.88.76)
- Quando cliente acessa `my-endpoints-service:80`, tráfego vai para IP externo
- Permite integração com serviços fora do cluster (legados, SaaS, etc.)

**Casos de uso**:
- Integrar com banco de dados externo
- Serviço legado fora do cluster
- API de terceiro que você quer abstrair
- Durante migração de infraestrutura

### `my-endpoints-test-eps.yaml`
Exemplo de **EndpointSlice** (versão moderna):

```yaml
---
# Service sem selector
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

**Diferenças do Endpoints legado**:
- Dividido em 2 slices (escalabilidade)
- Label: `kubernetes.io/service-name` liga ao Service
- Campo `addressType: IPv4` especifica tipo
- API versão `discovery.k8s.io/v1` (moderno)

## Como usar

### 1. Ver Endpoints automáticos (com selector)
```bash
# Service COM selector cria Endpoints automaticamente
kubectl apply -f deployment.yaml  # deployment com selector

# Ver Endpoints criados
kubectl get endpoints
kubectl get ep  # forma abreviada

# Descrever Endpoints
kubectl describe endpoints my-service
```

**Saída esperada**:
```
Name: my-service
Subsets:
  Addresses:     10.244.0.10, 10.244.0.11, 10.244.0.12
  Ports:         80/TCP
```

### 2. Criar Endpoints manuais
```bash
# Para Service SEM selector
kubectl apply -f my-endpoints-test.yaml

# Verifica
kubectl get endpoints my-endpoints-service
```

### 3. Ver Endpoints em detalhes
```bash
kubectl describe endpoints my-endpoints-service

# Output:
# Name: my-endpoints-service
# Namespace: default
# Subsets:
#   Addresses: 77.68.88.76
#   Ports: 80/TCP
```

### 4. Editar Endpoints
```bash
# Editar manual
kubectl edit endpoints my-endpoints-service

# Adicionar/remover IPs conforme necessário
```

### 5. Ver EndpointSlices (moderno)
```bash
kubectl get endpointslices
kubectl describe endpointslice my-eps
```

### 6. Acessar serviço via Endpoints
```bash
# De um Pod
kubectl run -it debug --image=busybox -- sh

# Dentro do Pod
wget -O- http://my-endpoints-service:80

# Tráfego vai para IP definido em Endpoints
```

### 7. Monitorar Endpoints em tempo real
```bash
kubectl get endpoints -w

# Observe mudanças conforme Pods são criados/deletados
```

### 8. Comparar Service vs Endpoints
```bash
# Ver Service
kubectl get svc my-service

# Ver Endpoints associados
kubectl get endpoints my-service

# Descrever ambos
kubectl describe svc my-service
kubectl describe endpoints my-service
```

## Casos de uso

### 1. Integração com serviço externo (fora do cluster)

**Cenário**: Você tem um banco de dados em outro servidor

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  ports:
  - port: 5432
    targetPort: 5432

---
apiVersion: v1
kind: Endpoints
metadata:
  name: external-db
subsets:
  - addresses:
    - ip: 192.168.1.100  # IP do BD externo
    ports:
    - port: 5432
```

**Benefício**: Pods acessam como `external-db:5432` (abstração)

### 2. Load balancing entre Pods
Automático com Service + Selector:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web  # Endpoints criados automaticamente
  ports:
  - port: 80

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web  # Label que Service procura
    spec:
      containers:
      - name: web
        image: nginx
```

Kubernetes cria Endpoints com 3 IPs (um para cada Pod).

### 3. Blue/Green deployment

```yaml
# Service aponta para versão "blue"
apiVersion: v1
kind: Endpoints
metadata:
  name: my-app
subsets:
  - addresses:
    - ip: 10.244.0.10  # Versão BLUE (atual)
    - ip: 10.244.0.11
    ports:
    - port: 80

# Para switch para green, edite Endpoints:
# - ip: 10.244.0.20  # Versão GREEN (nova)
# - ip: 10.244.0.21
```

Tráfego pode ser switchado atualizando Endpoints!

### 4. Migração gradual

```yaml
# Fase 1: Apenas serviço antigo
Endpoints:
  - 192.168.1.50  # Sistema legado

# Fase 2: Gradualmente adicionar novo
Endpoints:
  - 192.168.1.50  # 75% tráfego
  - 10.244.0.10   # 25% tráfego (novo)

# Fase 3: Apenas sistema novo
Endpoints:
  - 10.244.0.10   # 100% tráfego
```

## Monitoramento e Debugging

### Verificar por que Pods não aparecem nos Endpoints
```bash
# 1. Listar Pods com labels
kubectl get pods --show-labels

# 2. Ver selector do Service
kubectl get svc my-service -o yaml | grep -A 5 selector

# 3. Labels devem corresponder!
# Se Service busca "app: web", Pods precisam ter label "app: web"
```

### Endpoints vazios?
```bash
# 1. Service tem selector?
kubectl get svc my-service -o yaml

# 2. Se tem selector, por que Pods não aparecem?
kubectl get pods -L app

# 3. Verificar se Pods estão ready
kubectl get pods -o wide

# 4. Se Pods estão em outro namespace
kubectl get pods -n <namespace>
```

### Endpoints com IPs errados
```bash
# 1. Descrever para ver IPs
kubectl describe endpoints my-service

# 2. Comparar com Pods reais
kubectl get pods -o wide

# 3. Pode levar tempo para atualizar (geralmente <1s)
```

## Performance: Endpoints vs EndpointSlice

| Aspecto | Endpoints | EndpointSlice |
|---------|-----------|---|
| **Limite** | 1000 endereços | 100 por slice |
| **Performance** | Degrada com tamanho | Consistente |
| **Update** | Tudo de uma vez | Incremental |
| **Escalabilidade** | ❌ Limitada | ✅ Ótima |
| **API versão** | v1 (legado) | discovery.k8s.io/v1 |
| **Kubernetes** | Todas | 1.19+ |

**Recomendação**: Use EndpointSlice em novos projetos!

## Boas práticas

✅ **Faça**:
- Use Services com **selector** sempre que possível (Endpoints automáticos)
- Use **EndpointSlice** para novos projetos
- Use Endpoints manuais apenas para integração com **externos**
- Documente por que Endpoints manuais são necessários
- Monitore saúde de Endpoints externos
- Versione Endpoints manuais no git

❌ **Não faça**:
- Não edite Endpoints automáticos manualmente (serão sobrescritos)
- Não use Endpoints manuais para Pods (use Service com selector)
- Não deixe Endpoints estáticos apontando para Pods dinâmicos
- Não misture tipos de endereços (IPv4/IPv6) em um slice

## Troubleshooting avançado

### Endpoint não está recebendo tráfego
```bash
# 1. Verificar se Endpoint existe
kubectl describe endpoints my-service

# 2. Verificar se Service aponta ao Endpoint
kubectl describe svc my-service

# 3. Testar conectividade
kubectl exec -it <pod> -- curl http://my-service:80

# 4. Verificar iptables (networking)
kubectl exec -it <pod> -- iptables -t nat -L -n
```

### EndpointSlice não está sendo usado
```bash
# Kubernetes preferem EndpointSlice, mas Endpoints ainda funcionam
# Se ambos existem, EndpointSlice é prioridade

kubectl get endpointslices
kubectl get endpoints

# Você pode deletar Endpoints legados
kubectl delete endpoints my-service
```

## Próximos passos

- Combine com **4. Services** para criar abstrações
- Use com **External Name Service** para serviços externos
- Explore **Ingress** para roteamento HTTP avançado
- Monitore com **Prometheus** para health de Endpoints
- Implemente **Health Checker** para Endpoints externos

