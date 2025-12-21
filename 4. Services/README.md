# 🔌 Services

## Descrição

Nesta pasta você encontrará exemplos dos diferentes **tipos de Services** no Kubernetes. Services são abstrações que definem como acessar um conjunto lógico de Pods.

## O que é um Service?

Um Service é um objeto Kubernetes que fornece:
- **Descoberta de serviço**: encontrar e comunicar-se com Pods
- **Load balancing**: distribuir tráfego entre múltiplos Pods
- **IP estável**: Pods vêm e vão, mas o Service mantém um IP constante
- **Abstração**: aplicações não precisam saber sobre Pods específicos

## Tipos de Services

### 1. **ClusterIP** (padrão)
- Expõe o Service apenas dentro do cluster
- Útil para comunicação interna entre Pods
- **Acesso**: apenas de dentro do cluster

```
Cliente interno → ClusterIP → Load Balancer → Pods
```

### 2. **NodePort**
- Expõe o Service em uma porta fixa em cada Node
- Permite acesso externo ao cluster
- **Acesso**: `<NodeIP>:<NodePort>`
- **Porta**: entre 30000-32767

```
Cliente externo → Node IP:NodePort → Load Balancer → Pods
```

### 3. **LoadBalancer**
- Expõe o Service através de um load balancer externo
- Cria um IP externo (em provedores de nuvem)
- **Acesso**: via IP externo fornecido
- Cada Service cria um novo load balancer (pode ser caro)

```
Cliente externo → Load Balancer Externo → NodePort → Pods
```

### 4. **ExternalName**
- Mapeia o Service para um nome DNS externo
- Não fornece load balancing
- Útil para integração com serviços externos
- **Exemplo**: banco de dados externo

```
Cliente interno → ExternalName DNS → Serviço Externo
```

## Arquivos nesta pasta

### `clusterip-service.yaml`
- **Tipo**: ClusterIP
- **Propósito**: comunicação interna
- **Seletor**: pods com `type: web-app`
- **Ports**: HTTP na porta 86 → targetPort 8080
- Pod associado: Apache (80) + Tomcat (8080)

### `nodeport-service.yaml`
- **Tipo**: NodePort
- **Propósito**: acesso externo básico
- **NodePort**: porta fixa em cada Node
- **Seletor**: pods com `app: backend`
- Permite acessar a aplicação de fora do cluster

### `loadbalancer-service.yaml`
- **Tipo**: LoadBalancer
- **Propósito**: acesso externo com load balancing
- **IP Externo**: fornecido pelo provedor de nuvem
- Integração automática com infraestrutura de nuvem

### `externalname-service.yaml`
- **Tipo**: ExternalName
- **Propósito**: integração com serviços externos
- **Redirecionamento**: para um CNAME externo
- **Uso típico**: banco de dados em outro servidor

## Comparação de tipos

| Tipo | Escopo | Acesso | Caso de Uso |
|------|--------|--------|-----------|
| **ClusterIP** | Interno | DNS interno | Comunicação entre Pods |
| **NodePort** | Node | `<NodeIP>:<NodePort>` | Exposição básica |
| **LoadBalancer** | Externo | IP externo | Produção em nuvem |
| **ExternalName** | Externo | DNS | Serviços externos |

## Como usar

### Implantar um Service
```bash
kubectl apply -f clusterip-service.yaml
```

### Listar Services
```bash
kubectl get services
kubectl get svc  # forma abreviada
```

### Ver detalhes de um Service
```bash
kubectl describe service frontend-service
```

### Acessar ClusterIP (dentro do cluster)
```bash
# De dentro de um Pod
kubectl exec -it <pod-name> -- sh
curl http://frontend-service:86
```

### Acessar NodePort (de fora do cluster)
```bash
# Obter IP de um Node
kubectl get nodes -o wide

# Acessar via NodePort
curl http://<NodeIP>:<NodePort>
```

### Obter IP externo do LoadBalancer
```bash
kubectl get svc
# Na coluna EXTERNAL-IP, aguarde até aparecer um IP
```

### Acessar via LoadBalancer (de fora)
```bash
curl http://<EXTERNAL-IP>:<port>
```

### Editar um Service
```bash
kubectl edit service frontend-service
```

### Deletar um Service
```bash
kubectl delete service frontend-service
```

## Ports em um Service

```yaml
ports:
  - name: http                 # Nome descritivo
    port: 80                   # Porta do Service
    targetPort: 8080           # Porta no Pod/container
    protocol: TCP              # TCP ou UDP
```

- **port**: porta onde o Service escuta
- **targetPort**: porta para qual o tráfego é roteado no Pod
- **nodePort** (NodePort/LoadBalancer): porta no Node (30000-32767)

## Exemplos práticos

### ClusterIP para Redis
```bash
# Service acessa redis-pod na porta 6379
kubectl apply -f clusterip-service.yaml
# De outro Pod: redis-cli -h redis-service
```

### NodePort para acesso externo
```bash
kubectl apply -f nodeport-service.yaml
# De fora: curl http://192.168.1.10:31234
```

## Próximos passos

- Combine Services com **2. Deployments** para expor múltiplas réplicas
- Use **5. Liveness Prob** com Services para health checks
- Explore **Ingress** para roteamento HTTP avançado (não está nesta pasta)


