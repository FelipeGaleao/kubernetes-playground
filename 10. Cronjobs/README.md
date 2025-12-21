# ⏰ CronJobs

## Descrição

Nesta pasta você encontrará exemplos de **CronJobs** no Kubernetes, que executam **Jobs em horários programados**, assim como `cron` em sistemas Unix.

## O que é um CronJob?

Um CronJob é um objeto Kubernetes que:
- **Cria Jobs automaticamente** em horários programados
- **Usa sintaxe cron** familiar: `* * * * *` (minuto hora dia mês dia-semana)
- **Executa tarefas periódicas**: backups, limpeza, relatórios
- **Gerencia histórico**: mantém registros de execuções bem/mal-sucedidas
- **Permite concorrência**: controla execução paralela e histórico

## Sintaxe Cron

```
* * * * *
│ │ │ │ │
│ │ │ │ └─── Dia da semana (0-6, 0=domingo)
│ │ │ └───── Mês (1-12)
│ │ └─────── Dia do mês (1-31)
│ └───────── Hora (0-23)
└─────────── Minuto (0-59)
```

### Exemplos

| Expressão | Significado |
|-----------|------------|
| `* * * * *` | A cada minuto |
| `*/5 * * * *` | A cada 5 minutos |
| `0 * * * *` | A cada hora (minuto 0) |
| `0 0 * * *` | Diariamente à meia-noite |
| `0 0 * * 0` | Toda segunda-feira à meia-noite |
| `0 2 * * 1-5` | De segunda a sexta às 2 da manhã |
| `0 0 1 * *` | Primeiro dia do mês à meia-noite |
| `0 12 15 * *` | Dia 15 ao meio-dia |

## Jobs vs CronJobs

### Job
- Executa **uma vez** um contêiner até completar
- Termina após sucesso ou falha
- Não se repete automaticamente

### CronJob
- Cria **múltiplos Jobs** em horários programados
- Cada execução é um Job separado
- **Periódico**: segue agendamento Cron

```
Cronograma:      │ 1min │ 2min │ 3min │ 4min │ 5min │
                 │      │      │      │      │      │
CronJob agenda:  │  Job │  Job │      │  Job │  Job │
                 │      │      │      │      │      │
```

## Arquivos nesta pasta

### `my-cronjob.yaml`
CronJob que executa script de teste:

**Configuração:**
- **Nome**: my-cj
- **Schedule**: `* * * * *` (a cada minuto - apenas para teste!)
- **Parallelism**: 5 (executa 5 Pods em paralelo)
- **Completions**: 10 (precisa completar 10 Pods)
- **failedJobsHistoryLimit**: 5 (mantém 5 Jobs falhados no histórico)
- **successfulJobsHistoryLimit**: 10 (mantém 10 Jobs bem-sucedidos)
- **Container**: busybox
- **Comando**: gera números aleatórios

**Comportamento:**
```
Minuto 1: Cria Job → 10 Pods com 5 paralelos → Sucesso → Job #1 completa
Minuto 2: Cria Job → 10 Pods com 5 paralelos → Sucesso → Job #2 completa
Minuto 3: Cria Job → 10 Pods com 5 paralelos → Sucesso → Job #3 completa
...
```

## Como usar

### Criar um CronJob
```bash
kubectl apply -f my-cronjob.yaml
```

### Verificar CronJobs
```bash
kubectl get cronjobs
kubectl get cj  # forma abreviada
```

### Ver próxima execução
```bash
kubectl describe cronjob my-cj
```

Procure por `Last Successful Time` e `Next Scheduled Time`

### Ver Jobs criados pelo CronJob
```bash
kubectl get jobs
kubectl get jobs -o wide
```

### Ver Pods criados pelos Jobs
```bash
kubectl get pods
```

### Monitorar em tempo real
```bash
kubectl get cronjobs -w  # watch mode
```

### Ver logs de um Pod
```bash
kubectl logs <pod-name>
```

### Ver status detalhado
```bash
kubectl describe cronjob my-cj
kubectl describe job <job-name>
```

### Deletar um CronJob
```bash
kubectl delete cronjob my-cj
```

⚠️ **Nota**: Deletar o CronJob não deleta os Jobs históricos automaticamente!

### Limpar Jobs históricos manualmente
```bash
kubectl delete jobs --all
```

## Configurações importantes

### `schedule`
Expressão cron para determinar quando executar

### `successfulJobsHistoryLimit`
Número de Jobs bem-sucedidos mantidos no histórico (padrão: 3)

### `failedJobsHistoryLimit`
Número de Jobs falhados mantidos no histórico (padrão: 1)

### `concurrencyPolicy`
Como lidar com execuções concorrentes:
- **`Allow`** (padrão): permite múltiplos Jobs em paralelo
- **`Forbid`**: não cria novo Job se anterior ainda está rodando
- **`Replace`**: cancela anterior e cria novo

```yaml
concurrencyPolicy: Forbid  # Exemplo
```

### `startingDeadlineSeconds`
Tempo máximo (em segundos) para iniciar uma execução faltante

### `suspend`
Pausa o agendamento sem deletar
```yaml
suspend: true  # Pausa
suspend: false # Retoma
```

## Exemplos práticos

### Backup diário às 2 da manhã
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-backup
spec:
  schedule: "0 2 * * *"  # 2 AM todo dia
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: backup-tool:latest
            command: ["/backup.sh"]
          restartPolicy: Never
```

### Limpeza de logs a cada 6 horas
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup-logs
spec:
  schedule: "0 */6 * * *"  # A cada 6 horas
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cleanup
            image: busybox
            command: ["rm", "-rf", "/var/log/old/*"]
          restartPolicy: Never
```

### Gerar relatório toda segunda-feira
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: weekly-report
spec:
  schedule: "0 9 * * 1"  # Segunda às 9 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: report
            image: report-generator:latest
            command: ["/generate_report.sh"]
          restartPolicy: Never
```

### Health check a cada 5 minutos
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: health-check
spec:
  schedule: "*/5 * * * *"  # A cada 5 minutos
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: healthcheck
            image: curl:latest
            command: ["curl", "https://example.com/health"]
          restartPolicy: Never
```

## Monitoramento de CronJob

### Ver histórico de execuções
```bash
kubectl describe cronjob my-cj

# Saída incluirá:
# Last Successful Time: ...
# Next Scheduled Time: ...
# Active Jobs: 0 ou 1
```

### Ver todos os Jobs de um CronJob
```bash
kubectl get jobs --selector=batch.kubernetes.io/cronjob-name=my-cj
```

### Contar Jobs bem-sucedidos vs falhados
```bash
# Bem-sucedidos
kubectl get jobs --field-selector=status.successful=1

# Falhados
kubectl get jobs --field-selector=status.failed=1
```

## Debugging

### CronJob não está executando

```bash
# 1. Verificar se está suspenso
kubectl describe cronjob my-cj
# Procure por: suspend: false

# 2. Verificar timezone do cluster
kubectl exec -it <pod-name> -- date

# 3. Ver eventos
kubectl get events --sort-by='.lastTimestamp'
```

### Job não está criando Pods

```bash
# 1. Descrever o Job
kubectl describe job <job-name>

# 2. Ver eventos do Job
kubectl get events | grep <job-name>

# 3. Verificar limites de recursos
kubectl top nodes
```

### Pausar temporariamente
```bash
kubectl patch cronjob my-cj -p '{"spec":{"suspend":true}}'

# Retomar
kubectl patch cronjob my-cj -p '{"spec":{"suspend":false}}'
```

## Limpeza e Housekeeping

### Manter histórico limpo

```bash
# Ver Jobs do CronJob
kubectl get jobs --selector=batch.kubernetes.io/cronjob-name=my-cj

# Deletar Jobs antigos manualmente
kubectl delete job <old-job-name>

# Usar histórico limits (configurado no YAML)
```

### Deletar CronJob e seu histórico
```bash
# Deletar CronJob (Jobs não são deletados)
kubectl delete cronjob my-cj

# Deletar Jobs órfãos
kubectl delete jobs --selector=batch.kubernetes.io/cronjob-name=my-cj
```

## Boas práticas

✅ **Faça**:
- Sempre defina um schedule claro e testado
- Use `concurrencyPolicy: Forbid` se Jobs são pesados
- Defina `successfulJobsHistoryLimit` apropriado
- Configure `activeDeadlineSeconds` em Jobs
- Teste a expressão cron antes de usar em produção
- Monitore falhas de execução
- Use `restartPolicy: Never` (Jobs já tratam retry)

❌ **Não faça**:
- Não use `* * * * *` em produção (a cada minuto!)
- Não deixe histórico ilimitado (consome storage)
- Não ignore `concurrencyPolicy`
- Não execute tarefas muito longas (configure timeout)
- Não esqueça de versionar o CronJob no git

## Próximos passos

- Compare com **9. Jobs** para entender execução única vs periódica
- Use **8. DaemonSets** para tarefas em cada Node
- Combine com **4. Services** para notificações
- Explore **Observabilidade** para monitorar CronJobs

