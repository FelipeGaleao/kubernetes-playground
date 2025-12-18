# 🌐 Networking

## Descrição

Nesta pasta você encontrará exemplos de configuração de rede entre Pods no Kubernetes, incluindo múltiplos contêineres que se comunicam.

## Conceitos de Networking

### Comunicação entre Pods
- **Cada Pod tem um IP único**: todos os Pods em um cluster podem se comunicar entre si
- **Sem restrições nativas**: por padrão, qualquer Pod pode comunicar com qualquer outro
- **Network Policies**: podem ser usadas para restringir a comunicação (não implementada aqui)

### Descoberta de Serviços
- **DNS interno**: Kubernetes fornece DNS para descobrir serviços por nome
- **Nomes de serviço**: `<service-name>.<namespace>.svc.cluster.local`

## Arquivos nesta pasta

### `redis-pod.yaml`
- **Nome**: redis-pod
- **Container**: Redis (banco de dados in-memory)
- **Rótulo**: `apps: backend`
- **Propósito**: Exemplo de Pod backend para armazenamento de dados
- **Porta padrão**: 6379

### `tomcat-pod.yaml`
- **Nome**: tomcat-pod
- **Container**: Apache Tomcat (servidor de aplicação Java)
- **Rótulo**: `apps: app-java`
- **Propósito**: Exemplo de Pod executando aplicação Java
- **Porta padrão**: 8080

## Casos de Uso

### 1. Multi-container Pod (co-located)
Um Pod com múltiplos contêineres que precisam trabalhar juntos:
- **Sidecar**: contêiner auxiliar que fornece funcionalidades (logging, monitoring)
- **Proxy**: contêiner que processa tráfego para a aplicação principal

### 2. Pods diferentes comunicando-se
Redis e Tomcat podem ser implantados em Pods diferentes e se comunicarem via DNS:
```
tomcat-pod → resolve redis-pod.default.svc.cluster.local → redis-pod IP:6379
```

## Como usar

### Implantar os Pods
```bash
kubectl apply -f redis-pod.yaml
kubectl apply -f tomcat-pod.yaml
```

### Verificar Pods
```bash
kubectl get pods
```

### Testar conectividade entre Pods

#### Executar um comando dentro do Pod Tomcat
```bash
kubectl exec -it tomcat-pod -- /bin/bash
```

#### De dentro do Pod, testar conexão com Redis
```bash
apt-get update && apt-get install -y redis-tools
redis-cli -h redis-pod ping
```

### Ver logs
```bash
kubectl logs redis-pod
kubectl logs tomcat-pod
```

### Acessar IPs dos Pods
```bash
kubectl get pods -o wide
```

## Arquitetura de exemplo

```
┌─────────────────┐         ┌─────────────────┐
│   tomcat-pod    │         │   redis-pod     │
│                 │         │                 │
│  Tomcat:8080    │────────→│  Redis:6379     │
│                 │         │                 │
│ (app-java)      │         │ (backend)       │
└─────────────────┘         └─────────────────┘
        │                           │
        └───────────┬───────────────┘
                    │
            Kubernetes Network
            (Rede interna do cluster)
```

## DNS Interno do Kubernetes

O DNS resolve automaticamente:
- `<pod-name>.<namespace>.pod.cluster.local` → IP do Pod
- `<service-name>.<namespace>.svc.cluster.local` → IP do Serviço

## Próximos passos

- Explore **4. Services** para expor Pods e criar descoberta de serviços
- Veja **clusterip-service.yaml** para permitir comunicação via nome de serviço
- Consulte documentação de **Network Policies** para segurança de rede

