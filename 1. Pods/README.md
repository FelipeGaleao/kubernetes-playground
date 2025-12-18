# 🐳 Pods

## Descrição

Nesta pasta você encontrará exemplos e configurações de **Pods** no Kubernetes.

## O que é um Pod?

Um Pod é a menor unidade implantável no Kubernetes. É uma abstração que encapsula um ou mais contêineres (na maioria das vezes, um único contêiner). Os Pods compartilham espaço de rede, permitindo que os contêineres se comuniquem através de localhost.

## Características

- **Unidade mínima do Kubernetes**: não é possível implantar contêineres diretamente, apenas Pods
- **Efêmero**: Pods são criados e destruídos dinamicamente
- **Compartilhamento de rede**: todos os contêineres dentro de um Pod compartilham o mesmo IP e namespace de rede
- **Armazenamento compartilhado**: Pods podem compartilhar volumes entre contêineres

## Arquivos nesta pasta

### `my-pod.yaml`
Exemplo de configuração que inclui:
- Definição de um **Namespace** personalizado
- Exemplo de **Pod** simples (comentado)
- **ReplicaSet** para gerenciar múltiplas réplicas de um Pod
  - Usa `matchLabels` para seleção de Pods
  - Configurado com `replicas: 0` (sem instâncias ativas)
  - Container: nginx

## Como usar

### Aplicar a configuração
```bash
kubectl apply -f my-pod.yaml
```

### Verificar Pods
```bash
kubectl get pods -n kubernetes-playground
```

### Ver detalhes de um Pod
```bash
kubectl describe pod <pod-name> -n kubernetes-playground
```

### Acessar logs
```bash
kubectl logs <pod-name> -n kubernetes-playground
```

## Próximos passos

- Explore a pasta **2. Deployments** para aprender sobre gerenciamento automático de Pods
- Veja **4. Services** para entender como expor Pods para a rede
- Consulte **5. Liveness Prob** para aprender sobre health checks

