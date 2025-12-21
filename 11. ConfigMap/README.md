# ⚙️ ConfigMap

## Descrição

Nesta pasta você encontrará exemplos de **ConfigMaps** no Kubernetes, que armazenam dados de configuração em pares chave-valor.

## O que é um ConfigMap?

Um ConfigMap é um objeto Kubernetes que:
- **Armazena configurações**: dados não-sensíveis em formato chave-valor
- **Separa config de código**: configuração fora da imagem Docker
- **Injeta em Pods**: como variáveis de ambiente ou volumes
- **Permite atualizações**: mudanças sem reconstruir imagem
- **Suporta múltiplos formatos**: strings, arquivos, JSON, etc.

## Dados sensíveis vs Configuração

| Tipo | Exemplo | Onde | Objeto |
|------|---------|------|--------|
| **Configuração** | URLs, IPs, portas, temas | Visível | **ConfigMap** |
| **Segredo** | Senhas, tokens, certs | Secreto (criptografado) | **Secret** |

⚠️ **Importante**: ConfigMaps **não são secretos**! Não armazene senhas, tokens ou dados sensíveis aqui.

## Criando ConfigMaps

### 1. Declarativo (YAML)
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
data:
  key1: value1
  key2: value2
```

### 2. Imperativo (kubectl)
```bash
# De um arquivo
kubectl create configmap my-config --from-file=config.txt

# De literal
kubectl create configmap my-config --from-literal=key1=value1 --from-literal=key2=value2

# De um diretório
kubectl create configmap my-config --from-file=./config/
```

## Consumindo ConfigMaps

### 1. Como Variáveis de Ambiente
ConfigMap é convertido em variáveis de ambiente no Pod

### 2. Como Volume
ConfigMap é montado como arquivos no filesystem do Pod

## Arquivos nesta pasta

### `my-cm.yaml`
ConfigMap básico com dados estruturados:

```yaml
data:
  my.config.db: |
    database: mariadb
    database_uri: mariadb://localhost:3306
  test.field: hello
  test2: hello2.1
  test3: hello3.0
immutable: true
```

**Características:**
- Dados de conexão (database)
- Campo de teste (test.field)
- ConfigMap imutável (`immutable: true`)

### `my-cm-test-env.yaml`
ConfigMap + Pod que injeta como variáveis de ambiente:

```yaml
kind: ConfigMap
data:
  database: mysql
  database_uri: mysql://localhost:3306
  font.title: Arial Bold
  background-color: red
---
kind: Pod
containers:
  envFrom:
  - configMapRef:
      name: my-cm
```

**Resultado**: Variáveis no Pod:
```
$database = "mysql"
$database_uri = "mysql://localhost:3306"
$font_title = "Arial Bold"         # (pontos viram underscores)
$background_color = "red"
```

### `pod-cm-env.yaml`
Pod simples injetando ConfigMap como environment:

Usa `envFrom.configMapRef` para carregar todas as chaves como variáveis.

### `pod-cm-vol.yaml`
Pod que monta ConfigMap como volume (arquivos):

```yaml
volumeMounts:
- name: my-vol
  mountPath: "/etc/my-vol"
  readOnly: true

volumes:
- name: my-vol
  configMap:
    name: my-cm
```

**Resultado**: Arquivo no Pod:
```
/etc/my-vol/database → "mysql"
/etc/my-vol/database_uri → "mysql://localhost:3306"
/etc/my-vol/background-color → "red"
```

### `my-cm-test-command.yaml`
ConfigMap + Pod que executa comando com variáveis:

```yaml
command:
- "bin/sh"
- "-c"
- "echo My Database = $database - $database_uri"
```

**Saída**:
```
My Database = mysql - mysql://localhost:3306
```

### `my-app-file.txt`
Arquivo de exemplo que poderia ser carregado no ConfigMap.

## Como usar

### 1. Criar ConfigMap

#### Declarativo (YAML)
```bash
kubectl apply -f my-cm.yaml
```

#### Imperativo (direct)
```bash
# De literal
kubectl create configmap my-config --from-literal=key1=value1 --from-literal=key2=value2

# De arquivo
kubectl create configmap my-config --from-file=my-app-file.txt

# De diretório
kubectl create configmap my-config --from-file=./config-dir/
```

### 2. Listar ConfigMaps
```bash
kubectl get configmaps
kubectl get cm  # forma abreviada
```

### 3. Ver detalhes
```bash
kubectl describe configmap my-cm
kubectl get configmap my-cm -o yaml
```

### 4. Usar em Pod com variáveis de ambiente

#### Injetar todas as chaves
```yaml
envFrom:
- configMapRef:
    name: my-cm
```

#### Injetar chaves específicas
```yaml
env:
- name: DATABASE
  valueFrom:
    configMapKeyRef:
      name: my-cm
      key: database
```

### 5. Usar em Pod com volume
```yaml
volumeMounts:
- name: config
  mountPath: /etc/config

volumes:
- name: config
  configMap:
    name: my-cm
```

### 6. Acessar dados do Pod

#### Com variáveis de ambiente
```bash
kubectl exec -it <pod-name> -- env | grep database
```

#### Com volume
```bash
kubectl exec -it <pod-name> -- cat /etc/config/database
```

### 7. Deletar ConfigMap
```bash
kubectl delete configmap my-cm
```

## Nomes de chaves e variáveis

### Restrições de nomes
- **Chaves em arquivo**: podem ter qualquer caractere
- **Chaves como variáveis**: devem ser nomes válidos de shell

```yaml
data:
  database: mysql           # ✅ Válido
  database.host: localhost  # ✅ Válido em arquivo
  database-host: localhost  # ✅ Válido em arquivo
  "my key": value           # ✅ Válido em arquivo
```

Quando usados como variáveis de ambiente:
```bash
# Pontos e hífens viram underscores
database.host → $database_host
database-host → $database_host
```

## Tipos de dados

### String simples
```yaml
data:
  key: value
```

### Multi-linha (YAML pipe)
```yaml
data:
  config.json: |
    {
      "key": "value",
      "nested": {
        "data": "here"
      }
    }
```

### Arquivo completo
```bash
kubectl create configmap my-cm --from-file=config.json
```

## Atualizando ConfigMaps

### Método 1: Editar
```bash
kubectl edit configmap my-cm
```

### Método 2: Replace
```bash
kubectl apply -f my-cm.yaml --overwrite
```

### Método 3: Delete e Recreate
```bash
kubectl delete configmap my-cm
kubectl apply -f my-cm.yaml
```

⚠️ **Atualizar ConfigMap não atualiza Pods automaticamente!**

**Solução**: 
- Se via volume: remonte o Pod (delete + recreate)
- Se via env: reinicie o Deployment

```bash
# Forçar rollout para atualizar
kubectl rollout restart deployment <deployment-name>
```

## Configurações importantes

### `immutable`
Torna o ConfigMap imutável (não pode ser alterado após criação)

```yaml
immutable: true
```

**Benefício**: Impede mudanças acidentais e melhora performance.

### `readOnly` em volume
Monta ConfigMap em modo read-only

```yaml
volumeMounts:
- name: config
  mountPath: /etc/config
  readOnly: true
```

## Exemplos práticos

### ConfigMap com dados de banco

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-config
data:
  host: postgres.default.svc.cluster.local
  port: "5432"
  database: myapp
  user: appuser
```

### Pod usando ConfigMap
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: db-config
          key: host
    - name: DB_PORT
      valueFrom:
        configMapKeyRef:
          name: db-config
          key: port
```

### ConfigMap com arquivo de configuração
```bash
# Arquivo: nginx.conf
server {
  listen 80;
  location / {
    proxy_pass http://backend:8080;
  }
}

# Criar ConfigMap
kubectl create configmap nginx-config --from-file=nginx.conf

# Usar no Pod
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: config
      mountPath: /etc/nginx/conf.d
  volumes:
  - name: config
    configMap:
      name: nginx-config
```

## Comparação: ConfigMap vs Secret vs Deployment env

| Aspecto | ConfigMap | Secret | Env no Dockerfile |
|---------|-----------|--------|------------------|
| **Segurança** | ❌ Base64 | ✅ Criptografado | ❌ Na imagem |
| **Flexibilidade** | ✅ Reutilizável | ✅ Reutilizável | ❌ Fixo |
| **Atualizáveis** | ✅ Sim | ✅ Sim | ❌ Não |
| **Caso de uso** | Configuração | Credenciais | Padrão |

## Boas práticas

✅ **Faça**:
- Use ConfigMaps para dados não-sensíveis
- Use Secrets para dados sensíveis
- Separe configuração por ambiente (dev, staging, prod)
- Versionize ConfigMaps no git
- Use `immutable: true` para dados críticos
- Documente todas as chaves de configuração

❌ **Não faça**:
- Não armazene senhas em ConfigMap
- Não coloque dados grandes (limite: 1MB)
- Não confunda com Secrets
- Não esqueça que Pods não atualizam automaticamente
- Não deixe dados sensíveis em texto claro

## Próximos passos

- Explore **Secrets** para dados sensíveis
- Combine com **Deployments** para gerenciar configurações em escala
- Use **Kustomize** ou **Helm** para ConfigMaps complexos
- Implemente **ConfigMap watchers** para atualizações automáticas

