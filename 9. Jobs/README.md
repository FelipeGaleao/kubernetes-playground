# 💼 Jobs

## Descrição

Nesta pasta você encontrará exemplos de **Jobs** no Kubernetes, que executam tarefas pontuais até completarem (não continuam rodando).

## O que é um Job?

Um Job é um objeto Kubernetes que:
- **Executa tarefas uma vez**: diferente de Deployments (contínuos)
- **Garante completação**: reexecuta até sucesso (por padrão)
- **Permite paralelismo**: múltiplos Pods executando em paralelo
- **Rastreia sucesso**: sabe quantas tasks completaram com sucesso
- **Oferece retentativas**: automáticas em caso de falha
- **Limpeza automática**: Pods são mantidos para logs, depois deletados

## Job vs Deployment vs CronJob

| Aspecto | Deployment | Job | CronJob |
|---------|-----------|-----|---------|
| **Execução** | Contínua | Uma vez | Periódica |
| **Reinicio** | Automático | Se falhar | Por agenda |
| **Resultado** | N/A | Sucesso/Falha | Múltiplos Jobs |
| **Exemplo** | Web app | Backup | Backup diário |
| **Pod status** | Running | Completed/Failed | Completed/Failed |

## Componentes de um Job

### 1. **Template** (como executar)
Define o Pod que será criado:

```yaml
template:
  spec:
    containers:
    - name: my-task
      image: busybox
      command: ["echo", "Hello"]
    restartPolicy: Never
```

### 2. **Completions** (quantos sucessos)
Número total de Pods que precisam completar com sucesso:

```yaml
completions: 10  # 10 Pods precisam completar
```

### 3. **Parallelism** (quanto ao mesmo tempo)
Quantos Pods executam em paralelo:

```yaml
parallelism: 5   # 5 Pods rodando simultaneamente
```

**Comportamento**:
```
completions: 10, parallelism: 5

T=0s:   Inicia 5 Pods (Pod-0, Pod-1, Pod-2, Pod-3, Pod-4)
T=3s:   Pod-0 completa → Inicia Pod-5
T=5s:   Pod-1 completa → Inicia Pod-6
T=7s:   Pod-2 completa → Inicia Pod-7
T=9s:   Pod-3 completa → Inicia Pod-8
T=11s:  Pod-4 completa → Inicia Pod-9
T=13s:  Pod-9 completa → Todos os 10 completaram ✅
```

### 4. **backoffLimit** (máximo de retentativas)
Quantidade máxima de reinicializações antes de falhar:

```yaml
backoffLimit: 3  # Tenta 3 vezes, depois falha permanentemente
```

### 5. **activeDeadlineSeconds** (timeout)
Tempo máximo para o Job completar:

```yaml
activeDeadlineSeconds: 600  # 10 minutos máximo
```

Se Job não completar em 600s, é cancelado.

### 6. **CompletionMode**
Como rastrear completação:

- **`NonIndexed`** (padrão): Pods são idênticos, sem índice
- **`Indexed`**: Pods recebem índice (0, 1, 2...) como variável

```yaml
completionMode: "Indexed"  # Pods sabem seu índice via JOB_COMPLETION_INDEX
```

## Arquivo nesta pasta

### `my-job.yaml`
Job completo com configuração avançada:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: my-job
spec:
  completions: 10              # Precisa completar 10 Pods
  completionMode: "Indexed"    # Cada Pod tem índice
  parallelism: 5               # 5 Pods em paralelo
  activeDeadlineSeconds: 15    # Timeout de 15 segundos
  template:
    metadata:
      name: my-job-pod
    spec:
      containers:
      - name: my-container-busybox
        image: busybox
        command:
        - "bin/sh"
        - "-c"
        - "for i in 1 2; do echo Lucky number $i = $((1 + RANDOM % 70)); done"
      restartPolicy: Never    # Não reiniciar, falha permanente
```

**Características**:
- Executa script que gera números aleatórios
- 10 Pods em total (5 em paralelo)
- Timeout: 15 segundos (curto, para teste)
- CompletionMode Indexed (Pods sabem seu índice)
- RestartPolicy Never (sem retry automático)

**Timeline esperada**:
```
T=0s:   Inicia 5 Pods
T=3s:   1º Pod completa → Inicia Pod-5
T=6s:   2º Pod completa → Inicia Pod-6
T=9s:   3º Pod completa → Inicia Pod-7
T=12s:  4º Pod completa → Inicia Pod-8
T=15s:  5º Pod completa → Inicia Pod-9 (timeout! Job falha)
        OU
T=15s:  Todos completam ✅
```

## Como usar

### 1. Criar um Job
```bash
kubectl apply -f my-job.yaml
```

### 2. Verificar Jobs
```bash
kubectl get jobs
kubectl get job my-job -o wide
```

**Colunas importantes**:
- `COMPLETIONS`: 5/10 (5 concluído, 10 total)
- `DURATION`: tempo que está rodando
- `AGE`: tempo desde criação

### 3. Ver Pods do Job
```bash
kubectl get pods --selector=batch.kubernetes.io/job-name=my-job

# Output:
# NAME            READY   STATUS      RESTARTS
# my-job-xxxxx    0/1     Completed   0
# my-job-yyyyy    0/1     Running     0
# my-job-zzzzz    0/1     Running     0
```

### 4. Ver logs
```bash
# Todos os logs
kubectl logs --selector=batch.kubernetes.io/job-name=my-job

# Log específico
kubectl logs my-job-xxxxx

# Follow (tempo real)
kubectl logs -f my-job-xxxxx
```

### 5. Descrever Job
```bash
kubectl describe job my-job

# Procure por:
# Status: Complete (ou Failed)
# Events: mostra progresso
```

### 6. Monitorar em tempo real
```bash
kubectl get jobs -w

# Output mostra mudanças em tempo real
```

### 7. Deletar Job
```bash
# Deleta Job e Pods
kubectl delete job my-job

# Deleta sem deletar Pods (para inspecionar logs)
kubectl delete job my-job --cascade=orphan
```

### 8. Cancelar Job em progresso
```bash
kubectl delete job my-job

# Cancela execução, deleta Pods
```

## RestartPolicy

Defines como Pods são gerenciados quando falham:

| Policy | Comportamento | Caso de uso |
|--------|---------------|-----------|
| **Never** | Pod falha permanentemente | Processamento de dados |
| **OnFailure** | Reinicia container no Pod | Tarefas com retry |
| **Always** | ❌ Não permitido em Jobs | (Erro se especificar) |

### Never (recomendado)
```yaml
restartPolicy: Never

# Se falhar → Pod falha permanentemente
# Job cria novo Pod para retry
```

### OnFailure
```yaml
restartPolicy: OnFailure

# Se falhar → Reinicia container no mesmo Pod
# Menos Pods criados, mais overhead por Pod
```

## Completion Modes

### NonIndexed (padrão)
Pods são idênticos, sem identificação:

```yaml
completionMode: "NonIndexed"

# Pods: my-job-0, my-job-1, my-job-2, ...
# Nenhum sabe seu índice
```

### Indexed
Pods recebem variável de ambiente com índice:

```yaml
completionMode: "Indexed"

# Pods: my-job-0, my-job-1, my-job-2, ...
# Cada Pod tem: $JOB_COMPLETION_INDEX
```

**Útil para distribuir trabalho**:
```bash
# Pod-0 processa dados[0-999]
# Pod-1 processa dados[1000-1999]
# Pod-2 processa dados[2000-2999]
```

## Exemplos práticos

### Job simples (sem parallelismo)
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: simple-job
spec:
  completions: 1      # 1 Pod precisa completar
  parallelism: 1      # 1 Pod por vez
  template:
    spec:
      containers:
      - name: task
        image: alpine
        command: ["echo", "Hello World"]
      restartPolicy: Never
```

### Job com retry automático
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: retry-job
spec:
  completions: 1
  parallelism: 1
  backoffLimit: 5        # Tenta 5 vezes
  template:
    spec:
      containers:
      - name: task
        image: alpine
        command:
        - "sh"
        - "-c"
        - "if [ $RANDOM -lt 20000 ]; then echo 'fail'; exit 1; else echo 'success'; fi"
      restartPolicy: OnFailure    # Reinicia container no Pod
```

### Job paralelo com Indexed
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-indexed-job
spec:
  completions: 100
  parallelism: 10
  completionMode: "Indexed"
  template:
    spec:
      containers:
      - name: worker
        image: python:3.9
        command:
        - "python"
        - "-c"
        - "print(f'Processing batch {os.environ.get(\"JOB_COMPLETION_INDEX\")}')"
      restartPolicy: Never
```

### Job com processamento de arquivo
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: file-processor
spec:
  completions: 5
  parallelism: 2
  template:
    spec:
      containers:
      - name: processor
        image: myapp:latest
        command: ["./process.sh"]
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: data-pvc
      restartPolicy: Never
```

## Status do Job

### Sucesso ✅
```
Status: Complete
Succeeded: 10/10
```

Todos os Pods completaram com sucesso.

### Falha ❌
```
Status: Failed
Failed: 1
Reason: BackoffLimitExceeded
```

Excedeu limite de retentativas.

### Cancelado ⏹️
```
Status: Failed
Reason: DeadlineExceeded
```

Excedeu tempo máximo (activeDeadlineSeconds).

### Em progresso ⏳
```
Status: Active
Succeeded: 3/10
Active: 5
```

3 Pods completaram, 5 rodando.

## TTL (Time To Live)

Limpeza automática de Jobs completados:

```yaml
spec:
  ttlSecondsAfterFinished: 3600  # Delete 1 hora após completar
```

Útil para evitar acúmulo de Jobs antigos.

## Troubleshooting

### Job não está progredindo
```bash
# 1. Ver Pods
kubectl get pods --selector=batch.kubernetes.io/job-name=my-job

# 2. Descrever Job
kubectl describe job my-job

# 3. Ver logs
kubectl logs <pod-name>

# 4. Verificar eventos
kubectl get events --sort-by='.lastTimestamp'
```

### Job está demorando muito
```bash
# 1. Aumentar parallelism
kubectl patch job my-job -p '{"spec":{"parallelism":10}}'

# 2. Verificar recursos disponíveis
kubectl top nodes
kubectl top pods

# 3. Ver se Pods estão em Pending
kubectl get pods
```

### Job completou mas Pods ainda existem
```bash
# Pods são mantidos por padrão para inspecionar logs
# Para limpeza automática, use ttlSecondsAfterFinished

# Deletar manualmente
kubectl delete pods --selector=batch.kubernetes.io/job-name=my-job
```

## Boas práticas

✅ **Faça**:
- Use Jobs para **tarefas pontuais**
- Configure **activeDeadlineSeconds** para evitar travamentos
- Use **backoffLimit** apropriado (não muito alto)
- Monitore **logs** durante execução
- Use **Indexed** para distribuir trabalho entre Pods
- Defina **ttlSecondsAfterFinished** para limpeza automática
- Configure **requests/limits** de recursos

❌ **Não faça**:
- Não use Job para aplicações contínuas (use Deployment)
- Não deixe **backoffLimit** muito alto
- Não ignore **activeDeadlineSeconds**
- Não acumule Jobs antigos (use TTL)
- Não deixe Pods com `restartPolicy: Always` em Jobs

## Próximos passos

- Compare com **10. CronJobs** para agendamento periódico
- Use **8. DaemonSets** para tarefas em cada Node
- Combine com **4. Services** para Job distribuído
- Explore **Batch API** para WorkQueues
- Implemente **Job operators** para lógica complexa

