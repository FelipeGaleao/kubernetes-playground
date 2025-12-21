# 💾 Volumes

## Descrição

Nesta pasta você encontrará exemplos de **Volumes** no Kubernetes, que são mecanismos para persistir dados e compartilhar dados entre contêineres.

## Por que Volumes?

### Problema
- Contêineres são **efêmeros**: quando terminam, dados são perdidos
- Cada reinicialização cria um novo contêiner com filesystem limpo
- Dados em `/tmp`, `/var`, etc. não sobrevivem a reinicializações

### Solução
Volumes permitem:
- **Persistência de dados**: dados sobrevivem a reinicializações
- **Compartilhamento**: múltiplos contêineres acessam os mesmos dados
- **Performance**: diferentes tipos otimizados para diferentes casos

## Tipos de Volumes

### 1. **emptyDir**
Volume temporário criado quando o Pod inicia
- Dados **perdidos** quando o Pod termina
- **Compartilhado** entre contêineres no mesmo Pod
- Ideal para cache, temp files, shared state
- Storage: Node local (rápido)

```yaml
volumes:
  - name: cache-storage
    emptyDir: {}
```

**Ciclo de vida:**
```
Pod inicia → emptyDir criado → Pod roda → Pod termina → emptyDir deletado
```

### 2. **hostPath**
Monta diretório/arquivo do Node no Pod
- Dados **persistem** no Node
- ⚠️ Cuidado: Pod muda de Node = dados inacessíveis
- Ideal para: Kubernetes components, node-specific data
- Storage: Node local

```yaml
volumes:
  - name: my-hostpath-vol
    hostPath:
      path: /Users/mfelipemota/Downloads/volumes-persist
```

### 3. **configMap** e **secret**
Injetam dados de ConfigMaps/Secrets como volumes
- Dados: vêm de objetos Kubernetes
- Ideal para: configurações, credenciais
- Read-only (geralmente)

### 4. **PersistentVolume (PV)** e **PersistentVolumeClaim (PVC)**
Sistema robusto de armazenamento persistente
- **PV**: recurso de armazenamento no cluster
- **PVC**: requisição de armazenamento por um Pod
- Dados persistem além da vida do Pod/Node
- Suporte para diferentes tipos: local, NFS, AWS EBS, etc.

### 5. **nfs**
Network File System - compartilhado entre múltiplos Nodes
- Dados acessíveis de qualquer Node
- Ideal para: dados compartilhados, dados críticos
- Requer servidor NFS externo

### 6. **Outros tipos**
- **gcePersistentDisk**: Google Cloud
- **awsElasticBlockStore**: AWS EBS
- **azureDisk**: Azure
- **cinder**: OpenStack
- **fc**, **iscsi**, **rbd**: storage avançado

## Arquivos nesta pasta

### `volumes.yaml`
Exemplo com **emptyDir**:

```
Pod: redis-pod
Container: redis
Volume: cache-storage (emptyDir)
Mounting: /my-volume
```

**Características:**
- Volume temporário
- Redis acessa `/my-volume` no contêiner
- Dados perdidos quando Pod termina
- Perfeito para cache em memória

**Uso:**
```
Redis inicia → Cria cache em /my-volume
                Usuários acessam cache
Redis para    → Cache deletado
```

### `volumes-persist.yaml`
Exemplo com **hostPath**:

```
Pod: redis-pod
Container: redis
Volume: my-hostpath-vol (hostPath)
Path do Node: /Users/mfelipemota/Downloads/volumes-persist
Mounting: /my-data
```

**Características:**
- Volume persistente no Node
- Redis acessa `/my-data` no contêiner
- Dados persistem enquanto Node existir
- Dados acessíveis de fora do cluster

**Uso:**
```
Redis inicia → Acessa /my-data → Persiste em /Users/.../volumes-persist
Redis para   → Dados ainda existem no Node
Redis reinicia → Acessa mesmos dados
```

## Ciclos de vida de dados

### emptyDir
```
┌─────────────────┐
│   Pod inicia    │
└────────┬────────┘
         │
         ▼
┌─────────────────────┐
│ emptyDir criado     │
│ (vazio)             │
└────────┬────────────┘
         │
         ▼
┌──────────────────────┐
│ Contêineres acessam  │
│ Escrevem/leem dados  │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ Pod termina          │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ emptyDir deletado    │
│ Dados PERDIDOS ❌    │
└──────────────────────┘
```

### hostPath
```
┌──────────────────┐
│  Node filesystem │
│ /downloads/vols  │
└────────┬─────────┘
         │
    ┌────┴─────┐
    ▼          ▼
 Antes     Depois
 (vazio)   (com dados)
         │
         ▼
┌──────────────────────┐
│ Pod monta hostPath   │
│ em /my-data          │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ Redis acessa dados   │
│ Escreve em /my-data  │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ Pod termina          │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ Dados PERSISTEM ✅   │
│ em /downloads/vols   │
└──────────────────────┘
```

## Como usar

### emptyDir

#### Aplicar configuração
```bash
kubectl apply -f volumes.yaml
```

#### Acessar o Pod
```bash
kubectl exec -it redis-pod -- redis-cli
> SET mykey myvalue
> GET mykey
```

#### Ver volume montado
```bash
kubectl exec -it redis-pod -- ls -la /my-volume
```

#### Deletar e reiniciar (dados perdidos)
```bash
kubectl delete pod redis-pod
kubectl apply -f volumes.yaml
kubectl exec -it redis-pod -- redis-cli
# mykey não mais existe!
```

### hostPath

#### Criar diretório no Node
```bash
mkdir -p ~/Downloads/volumes-persist
```

#### Aplicar configuração
```bash
kubectl apply -f volumes-persist.yaml
```

#### Escrever dados do Pod
```bash
kubectl exec -it redis-pod -- redis-cli
> SET persistent_key persistent_value
> SAVE
```

#### Ver dados no filesystem
```bash
ls -la ~/Downloads/volumes-persist/
cat ~/Downloads/volumes-persist/dump.rdb
```

#### Deletar Pod e verificar dados
```bash
kubectl delete pod redis-pod
ls -la ~/Downloads/volumes-persist/  # Dados ainda lá!
```

#### Aplicar novamente
```bash
kubectl apply -f volumes-persist.yaml
kubectl exec -it redis-pod -- redis-cli
> KEYS *  # Dados recuperados!
```

## Compartilhamento entre contêineres

Múltiplos contêineres no mesmo Pod compartilham volumes:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-volume-pod
spec:
  containers:
  - name: writer
    image: busybox
    volumeMounts:
    - name: shared-data
      mountPath: /tmp/shared
    command: ['sh', '-c', 'while true; do date > /tmp/shared/timestamp; sleep 1; done']
  
  - name: reader
    image: busybox
    volumeMounts:
    - name: shared-data
      mountPath: /tmp/shared
    command: ['sh', '-c', 'while true; do cat /tmp/shared/timestamp; sleep 1; done']
  
  volumes:
  - name: shared-data
    emptyDir: {}
```

- **writer**: escreve timestamp em `/tmp/shared/timestamp`
- **reader**: lê arquivo do mesmo caminho
- Sincronização via filesystem

## Comparação de tipos

| Tipo | Persistência | Compartilhado | Caso de Uso |
|------|--------------|---------------|-----------|
| **emptyDir** | ❌ Temporário | Contêineres | Cache, temp files |
| **hostPath** | ✅ Node | Pods no Node | Logs, config local |
| **PV/PVC** | ✅ Cluster | Vários Pods | Dados críticos |
| **configMap** | ✅ Cluster | Config | Configurações |
| **secret** | ✅ Cluster | Credenciais | Senhas, tokens |
| **nfs** | ✅ Cluster | Qualquer Node | Compartilhado |

## Boas práticas

✅ **Faça**:
- Use PV/PVC para dados críticos
- Use emptyDir para compartilhamento temporário
- Monitore espaço em disco
- Implemente backup de dados persistentes
- Use readiness/liveness probes com volumes

❌ **Não faça**:
- Não use hostPath para data crítica (não é portável)
- Não confie em emptyDir para dados importantes
- Não suponha que dados existem após reinicializações
- Não esqueça de criar diretórios para hostPath

## Próximos passos

- Explore **PersistentVolumes (PV)** e **PersistentVolumeClaims (PVC)**
- Combine com **2. Deployments** para gerenciar volumes em escala
- Implemente backup e disaster recovery
- Veja **StatefulSets** para aplicações com estado (bancos de dados)


