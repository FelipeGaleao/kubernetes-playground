# 🚀 Kubernetes Playground

Chegou a hora de praticar Kubernetes!

Este projeto contém documentação completa e exemplos práticos de todos os conceitos principais do Kubernetes em Português.

## 📚 Índice de Pastas

### Core Concepts (Fundamentos)

1. **[1. Pods](./1.%20Pods/README.md)** 🐳
   - Unidade mínima do Kubernetes
   - Contêineres, namespaces, labels
   - ReplicaSet para gerenciar múltiplas réplicas

2. **[2. Deployments](./2.%20Deployments/README.md)** 🚀
   - Gerenciamento de aplicações
   - Estratégias de atualização (Rolling, Recreate, Blue/Green, Canary)
   - Scaling e rollback automático

3. **[3. Networking](./3.%20Networking/README.md)** 🌐
   - Comunicação entre Pods
   - Multi-container Pods
   - DNS interno do cluster

4. **[4. Services](./4.%20Services/README.md)** 🔌
   - ClusterIP, NodePort, LoadBalancer, ExternalName
   - Load balancing
   - Descoberta de serviço

### Advanced Features (Funcionalidades Avançadas)

5. **[5. Liveness Probes](./5.%20Liveness%20Prob/README.md)** ❤️
   - Health checks (Exec, HTTP, TCP)
   - Auto-recuperação de contêineres
   - Readiness e Startup Probes

6. **[6. Resources](./6.%20Resources/README.md)** 📊
   - Requests e Limits (CPU/Memória)
   - QoS Classes
   - Scheduling e eviction

7. **[7. Volumes](./7.%20Volumes/README.md)** 💾
   - emptyDir, hostPath, configMap, secret
   - PersistentVolume e PersistentVolumeClaim
   - Compartilhamento entre contêineres

### Controladores Especializados

8. **[8. DaemonSets](./8.%20DaemonSets/README.md)** 👹
   - Pod em cada Node
   - Monitoramento, logging, segurança
   - NodeSelector e Tolerations

9. **[9. Jobs](./9.%20Jobs/README.md)** 💼
   - Tarefas pontuais
   - Completions, parallelism, backoffLimit
   - Retry automático

10. **[10. CronJobs](./10.%20Cronjobs/README.md)** ⏰
    - Agendamento periódico (Cron syntax)
    - Histórico de execuções
    - Limpeza automática

### Configuration Management

11. **[11. ConfigMap](./11.%20ConfigMap/README.md)** ⚙️
    - Dados de configuração (não-sensíveis)
    - Injeção via variáveis de ambiente
    - Injeção via volumes

12. **[12. Secrets](./12.%20Secrets/README.md)** 🔐
    - Armazenamento seguro (senhas, tokens, certificados)
    - Tipos: Opaque, basic-auth, tls, ssh-auth, dockerconfigjson
    - Boas práticas de segurança

### Stateful Applications

13. **[13. StatefulSet](./13.%20StatefulSet/README.md)** 📊
    - Aplicações com estado
    - Identidades estáveis (pod-0, pod-1...)
    - Headless Service e armazenamento persistente
    - Bancos de dados, caches, message queues

### Service Discovery & Networking

14. **[14. Endpoints](./14.%20Endpoints/README.md)** 🔗
    - Lista de IPs para load balancing
    - Endpoints automáticos (com selector)
    - Endpoints manuais (integração externa)
    - Blue/Green deployment

15. **[15. Endpoints Slice](./15.%20Endpoints%20Slice/README.md)** 🔗
    - Versão moderna dos Endpoints
    - Escalabilidade melhorada
    - Auto-mirroring
    - Suporte IPv4/IPv6 (dual-stack)

## 🎯 Guia de Aprendizado

### Iniciante (Semana 1-2)
Comece por estes conceitos fundamentais:
1. Pods - entender unidade básica
2. Deployments - como executar aplicações
3. Services - como expor aplicações
4. ConfigMap - como gerenciar configurações

### Intermediário (Semana 3-4)
Aprofunde com:
5. Liveness Probes - saúde de containers
6. Resources - performance e limites
7. Volumes - dados persistentes
8. Secrets - dados sensíveis

### Avançado (Semana 5-6)
Especialize-se em:
9. DaemonSets - tarefas em cada node
10. Jobs/CronJobs - tarefas periódicas
11. StatefulSet - aplicações com estado
12. Endpoints - descoberta de serviço

## 📝 Estrutura das Pastas

Cada pasta contém:
- **README.md** - Documentação completa em Português
- **arquivos .yaml** - Exemplos práticos
- Explicações de conceitos
- Tabelas comparativas
- Casos de uso reais
- Troubleshooting

## 🚀 Como usar este projeto

### 1. Ler a documentação
Cada pasta tem um README.md com explicação completa:
```bash
cat "1. Pods/README.md"
```

### 2. Aplicar os exemplos
```bash
kubectl apply -f "1. Pods/my-pod.yaml"
```

### 3. Verificar status
```bash
kubectl get pods
kubectl describe pod my-pod
```

### 4. Ver logs
```bash
kubectl logs my-pod
```

### 5. Limpar recursos
```bash
kubectl delete -f "1. Pods/my-pod.yaml"
```

## 📊 Comparações Rápidas

### Pod vs Deployment vs StatefulSet
- **Pod**: Unidade mínima, efêmera
- **Deployment**: Múltiplos Pods stateless, escalável
- **StatefulSet**: Múltiplos Pods com estado, identidade estável

### Service vs Endpoint
- **Service**: Abstração para acessar Pods
- **Endpoint**: Lista real de IPs para roteamento

### ConfigMap vs Secret
- **ConfigMap**: Dados públicos (configurações)
- **Secret**: Dados sensíveis (senhas, tokens)

### Job vs CronJob
- **Job**: Executa uma vez
- **CronJob**: Executa periodicamente

## 🔧 Comandos Úteis

```bash
# Listar recursos
kubectl get pods
kubectl get deployments
kubectl get services
kubectl get configmaps
kubectl get secrets

# Descrever recursos
kubectl describe pod <pod-name>
kubectl describe svc <service-name>

# Ver logs
kubectl logs <pod-name>
kubectl logs -f <pod-name>  # Follow

# Executar comando no Pod
kubectl exec -it <pod-name> -- bash

# Editar recurso
kubectl edit pod <pod-name>

# Deletar recurso
kubectl delete pod <pod-name>
kubectl delete -f arquivo.yaml

# Ver eventos
kubectl get events

# Ver uso de recursos
kubectl top nodes
kubectl top pods
```

## 📈 Progression Path

```
Iniciante:
  Pods → Deployments → Services → ConfigMap

Intermediário:
  + Secrets → Volumes → Resources → Probes

Avançado:
  + DaemonSets → Jobs/CronJobs → StatefulSet
  + Endpoints → Networking → Performance

Expert:
  + Operators → Custom Resources
  + Network Policies → RBAC → Admission Controllers
  + Monitoring → Logging → Tracing
```

## 🎓 Recursos Adicionais

- [Documentação oficial do Kubernetes](https://kubernetes.io/docs)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Kubernetes Architecture](https://kubernetes.io/docs/concepts/architecture/)

## 📝 Notas Importantes

⚠️ **Segurança**:
- Base64 em ConfigMap/Secrets é codificação, não criptografia
- Habilite encriptação em repouso no etcd
- Use RBAC para controle de acesso
- Não versione Secrets em plaintext no git

✅ **Boas Práticas**:
- Sempre defina resource requests/limits
- Use health checks (liveness/readiness probes)
- Versione seus manifestos no git
- Monitore saúde dos containers
- Documente decisões de arquitetura

## 📅 Atualizações

Última atualização: 2024
- ✅ 15 pastas com documentação completa
- ✅ 6500+ linhas de documentação em Português
- ✅ Exemplos práticos para cada conceito
- ✅ Tabelas comparativas e guias de troubleshooting

## 📄 Licença

Este projeto é livre para uso educacional.

---

**Comece agora**: Abra [1. Pods/README.md](./1.%20Pods/README.md) para começar! 🚀
