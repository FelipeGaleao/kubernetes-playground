# 🔐 Secrets

## Descrição

Nesta pasta você encontrará exemplos de **Secrets** no Kubernetes, que armazenam dados sensíveis como senhas, tokens e certificados de forma segura.

## O que é um Secret?

Um Secret é um objeto Kubernetes que:
- **Armazena dados sensíveis**: senhas, tokens, chaves SSH, certificados
- **Codifica em base64**: proteção básica contra leitura acidental
- **Integração com etcd**: pode ser criptografado em repouso (ativado por administrador)
- **Injeta em Pods**: como variáveis de ambiente ou volumes
- **Mantém segredo**: acesso controlado via RBAC

## Segurança: ConfigMap vs Secret

| Aspecto | ConfigMap | Secret |
|---------|-----------|--------|
| **Dados** | Configuração pública | Dados sensíveis |
| **Codificação** | Nenhuma | Base64 (por padrão) |
| **Criptografia** | ❌ Não | ✅ Sim (opcional) |
| **Acesso** | Qualquer um | RBAC controlado |
| **Exemplo** | URLs, portas | Senhas, tokens |
| **Tamanho máx** | 1 MB | 1 MB |

⚠️ **Importante**: Base64 **não é criptografia**! Qualquer um pode decodificar:
```bash
# Decodificar base64
echo "YWRtaW4=" | base64 -d
# Output: admin
```

Habilite encriptação em repouso no cluster para segurança real!

## Tipos de Secrets

### 1. **Opaque** (padrão)
Dados arbitrários (senhas, tokens, etc.)

```yaml
type: Opaque
data:
  password: TXktc2VjcmV0
```

### 2. **kubernetes.io/basic-auth**
Credenciais para autenticação básica (HTTP)

```yaml
type: kubernetes.io/basic-auth
data:
  username: YWRtaW4=
  password: UGFzcy0xMjM=
```

Campos esperados: `username`, `password`

### 3. **kubernetes.io/ssh-auth**
Chave SSH privada

```yaml
type: kubernetes.io/ssh-auth
data:
  ssh-privatekey: LSXXXXXX...
```

Campo esperado: `ssh-privatekey`

### 4. **kubernetes.io/tls**
Certificado TLS/SSL

```yaml
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTi... (base64 cert)
  tls.key: LS0tLS1CRUdJTi... (base64 key)
```

Campos esperados: `tls.crt`, `tls.key`

### 5. **kubernetes.io/dockercfg**
Configuração do Docker (deprecated)

### 6. **kubernetes.io/dockerconfigjson**
Credenciais privadas do Docker

```yaml
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: ey...
```

Usar para: pull images de registros privados

### 7. **kubernetes.io/service-account-token**
Token de Service Account (criado automaticamente)

### 8. **bootstrap.kubernetes.io/token**
Token de bootstrap para join de nodes

## Arquivos nesta pasta

### `my-secret.yaml`
Secret genérico com dados básicos:

```yaml
data:
  user: YWRtaW4=           # (base64: "admin")
  pass: TXktcGFzcy0xMjM=   # (base64: "My-pass-123")
stringData:
  my-database-name: mysql  # (plaintext - será codificado)
```

**Características:**
- Campo `data`: valores já em base64
- Campo `stringData`: valores em plaintext (Kubernetes codifica automaticamente)
- ⚠️ **Nota**: Combinar `data` e `stringData` com mesma chave → `stringData` vence

### `my-secret-basic-auth.yaml`
Secret com tipo `kubernetes.io/basic-auth`:

```yaml
type: kubernetes.io/basic-auth
data:
  username: YWRtaW4K
  password: UGFzcy0xMjMK
stringData:
  extra2: value2  # campos adicionais além dos obrigatórios
```

**Características:**
- Tipo específico para autenticação básica
- Campos padrão: `username`, `password`
- Campos extras permitidos

### `test-secret-vol-env.yaml`
Secret + Pod que injeta como **variáveis de ambiente** e **volume**:

```yaml
---
kind: Secret
data:
  user: YWRtaW4=
  pass: TXktcGFzcy0xMjM=
---
kind: Pod
envFrom:
  - secretRef:
      name: my-secret        # Todas as chaves como variáveis
volumeMounts:
  - name: my-vol
    mountPath: "/etc/my-vol"  # Chaves como arquivos
volumes:
  - name: my-vol
    secret:
      secretName: my-secret
```

**Resultado**: 
- Variáveis: `$user`, `$pass`
- Arquivos: `/etc/my-vol/user`, `/etc/my-vol/pass`

### `test-optional-secret.yaml`
Pod que torna Secret **opcional** (não falha se não existir):

```yaml
envFrom:
  - secretRef:
      name: my-secret
      optional: true    # Pod funciona mesmo sem o Secret
volumeMounts:
  - name: my-vol
    mountPath: "/etc/my-vol"
volumes:
  - name: my-vol
    secret:
      secretName: my-secret
      optional: true    # Volume será criado vazio se Secret não existir
```

**Comportamento:**
- Sem `optional: true` → Pod fica em `Pending` se Secret não existir
- Com `optional: true` → Pod inicia normalmente (sem variáveis/arquivos)

## Como usar

### 1. Criar Secret

#### Declarativo (YAML) - Base64
```bash
# Codificar valor
echo -n "my-password" | base64
# Output: bXktcGFzc3dvcmQ=

# Usar no YAML
kubectl apply -f my-secret.yaml
```

#### Declarativo (YAML) - Plaintext
```yaml
stringData:
  password: my-password  # Kubernetes codifica automaticamente
```

#### Imperativo (kubectl)
```bash
# De literal
kubectl create secret generic my-secret \
  --from-literal=username=admin \
  --from-literal=password=secret123

# De arquivo
kubectl create secret generic my-secret \
  --from-file=ssh-key=/path/to/id_rsa

# De diretório
kubectl create secret generic my-secret \
  --from-file=/path/to/certs/

# Basic auth
kubectl create secret basic-auth my-auth \
  --from-literal=username=admin \
  --from-literal=password=secret123
```

### 2. Listar Secrets
```bash
kubectl get secrets
kubectl get secret my-secret -o yaml
```

### 3. Ver valor (criptografado/base64)
```bash
kubectl get secret my-secret -o jsonpath='{.data.password}'
# Output: UGFzcy0xMjM=

# Decodificar
kubectl get secret my-secret -o jsonpath='{.data.password}' | base64 -d
# Output: Pass-123
```

### 4. Usar em Pod - Variáveis de Ambiente

#### Injetar todas as chaves
```yaml
envFrom:
- secretRef:
    name: my-secret
```

#### Injetar chaves específicas
```yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: my-secret
      key: password
```

### 5. Usar em Pod - Volume

```yaml
volumeMounts:
- name: secret-vol
  mountPath: /etc/secrets
  readOnly: true

volumes:
- name: secret-vol
  secret:
    secretName: my-secret
```

**Resultado**: Arquivos em `/etc/secrets/`:
```
/etc/secrets/user
/etc/secrets/pass
/etc/secrets/my-database-name
```

### 6. Acessar dados do Pod

#### Variáveis de ambiente
```bash
kubectl exec -it <pod-name> -- env | grep user
# Output: user=admin
```

#### Volume (arquivos)
```bash
kubectl exec -it <pod-name> -- cat /etc/secrets/user
# Output: admin
```

### 7. Atualizar Secret

#### Editar
```bash
kubectl edit secret my-secret
```

#### Delete e Recreate
```bash
kubectl delete secret my-secret
kubectl apply -f my-secret.yaml
```

⚠️ **Pods não se atualizam automaticamente!**

```bash
# Forçar rollout
kubectl rollout restart deployment <deployment-name>
```

### 8. Deletar Secret
```bash
kubectl delete secret my-secret
```

## Codificando dados em base64

### Encoder base64
```bash
# Terminal
echo -n "meu-password" | base64
# Output: bWV1LXBhc3N3b3Jk

# Online: https://www.base64encode.org/
```

### Decoder base64
```bash
# Terminal
echo "bWV1LXBhc3N3b3Jk" | base64 -d
# Output: meu-password

# Online: https://www.base64decode.org/
```

## Segurança - Boas práticas

### ✅ Faça

- **Habilite encriptação em repouso**: configure `EncryptionConfiguration` no apiserver
- **Use RBAC**: restrinja acesso a Secrets por role
- **Use Sealed Secrets ou External Secrets**: para dados realmente sensíveis
- **Não versione Secrets no git**: use tools como `sealed-secrets` ou `sops`
- **Rotação regular**: atualize senhas periodicamente
- **Auditoria**: monitore acessos a Secrets
- **Use stringData em desenvolvimento**: mais legível que base64
- **Documente secrets**: mantém registro do que cada Secret faz

### ❌ Não faça

- Não confie apenas em base64 (é codificação, não criptografia)
- Não armazene Secrets no git em plaintext
- Não compartilhe Secrets entre ambientes (dev/staging/prod)
- Não deixe Secrets em logs ou output
- Não crie Secrets manualmente sem documentação
- Não use nomes genéricos (seja específico: `db-password`, não `secret1`)

## Exemplos práticos

### Secret com credenciais de banco de dados

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  username: dbuser
  password: super-secret-password
  host: postgres.default.svc.cluster.local
  port: "5432"
  database: myapp
```

### Pod usando Secret
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
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
```

### Secret com certificado TLS
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-cert
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTi... (cert em base64)
  tls.key: LS0tLS1CRUdJTi... (key em base64)
```

### Secret com chave SSH
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ssh-key
type: kubernetes.io/ssh-auth
stringData:
  ssh-privatekey: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    b3BlbnNzaC1rZXktdjEA...
    -----END OPENSSH PRIVATE KEY-----
```

### Secret para registro privado de Docker
```bash
# Criar Secret
kubectl create secret docker-registry my-registry \
  --docker-server=registry.example.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=myemail@example.com

# Usar em Deployment
spec:
  imagePullSecrets:
  - name: my-registry
  containers:
  - name: app
    image: registry.example.com/myapp:latest
```

## Ferramentas de segurança avançada

### Sealed Secrets
Criptografa Secrets para versionar no git com segurança

```bash
# Instalar
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.18.0/controller.yaml

# Criar Sealed Secret
echo -n mypassword | kubectl create secret generic mysecret --dry-run=client --from-file=/dev/stdin -o yaml | kubeseal -o yaml > mysealedsecret.yaml

# Aplicar
kubectl apply -f mysealedsecret.yaml
```

### External Secrets Operator
Integra com gerenciadores externos (AWS Secrets Manager, Vault, etc.)

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-secret
spec:
  secretStoreRef:
    name: aws-secrets
    kind: SecretStore
  target:
    name: my-secret-k8s
    creationPolicy: Owner
  data:
  - secretKey: password
    remoteRef:
      key: prod/db/password
```

### HashiCorp Vault
Gerenciador centralizado de segredos

```bash
# Injeta automaticamente Secrets de Vault em Pods
vault-injector:
  inject: "true"
  secret-volume-mount-path: "/vault/secrets"
```

## Troubleshooting

### Secret não está disponível para Pod
```bash
# 1. Verificar se Secret existe
kubectl get secret my-secret

# 2. Verificar namespace
kubectl get secret my-secret -n <namespace>

# 3. Descrever Pod
kubectl describe pod <pod-name>

# 4. Ver eventos
kubectl get events
```

### Pod está em Pending
```bash
# Se secret é obrigatório (sem optional: true)
# Criar o Secret primeiro
kubectl apply -f my-secret.yaml
```

### Valores não aparecem como variáveis
```bash
# Verificar Pod
kubectl exec -it <pod-name> -- env | grep -i secret

# Verificar se Secret foi injetado
kubectl exec -it <pod-name> -- cat /etc/secrets/mykey
```

## Próximos passos

- Compare com **11. ConfigMap** para dados não-sensíveis
- Use **Sealed Secrets** para versionamento seguro
- Implemente **RBAC** para controle de acesso
- Explore **External Secrets** para integração com provedores
- Configure **encriptação em repouso** no etcd

