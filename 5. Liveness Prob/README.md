# ❤️ Liveness Probes

## Descrição

Nesta pasta você encontrará exemplos de **Liveness Probes** no Kubernetes, que verificam se um contêiner está vivo e em bom funcionamento.

## O que é um Liveness Probe?

Um Liveness Probe é um mecanismo de health check que verifica periodicamente se um contêiner está íntegro. Se o probe falhar, Kubernetes reinicia o contêiner automaticamente.

## Filosofia

> "Se o seu contêiner está travado, não será capaz de servir requisições. Ninguém irá reiniciá-lo. Você precisa do Kubernetes para detectar isso e reiniciar o contêiner para você."

## Tipos de Probes (Health Checks)

### 1. **Exec Probe**
Executa um comando dentro do contêiner
```yaml
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
```
- ✅ Melhor para: verificações customizadas
- ❌ Cuidado: comandos lentos afetam performance

### 2. **HTTP Probe**
Faz uma requisição HTTP
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
```
- ✅ Melhor para: aplicações web
- ✅ Simples e eficiente

### 3. **TCP Socket Probe**
Verifica se uma porta está aberta
```yaml
livenessProbe:
  tcpSocket:
    port: 3306
```
- ✅ Melhor para: serviços que ouvem em portas
- ✅ Rápido

## Configurações importantes

### `initialDelaySeconds`
Aguarda esse tempo antes de iniciar os health checks
- Permite que a aplicação inicie adequadamente
- Padrão: 0 segundos
- Exemplo: 30 segundos para aplicações lentas

### `periodSeconds`
Intervalo entre health checks
- Padrão: 10 segundos
- Menor = mais frequente (mais recursos), Maior = menos responsivo

### `timeoutSeconds`
Tempo máximo para o probe responder
- Padrão: 1 segundo
- Se exceder, é considerado falha

### `successThreshold`
Quantos sucessos consecutivos para marcar como "saudável"
- Padrão: 1
- Apenas 1 sucesso é necessário após falha

### `failureThreshold`
Quantas falhas consecutivas antes de reiniciar o contêiner
- Padrão: 3
- Após 3 falhas = reiniciar

## Arquivos nesta pasta

### `livenessprob.yaml`
Exemplo completo com Exec Probe:

**Configuração:**
- **Tipo**: Exec Probe
- **Comando**: `cat /tmp/healthy`
- **initialDelaySeconds**: 5 segundos (aguarda inicialização)
- **periodSeconds**: 5 segundos (verifica a cada 5 segundos)
- **failureThreshold**: 3 (reinicia após 3 falhas)

**Comportamento do Container:**
```bash
touch /tmp/healthy     # Cria arquivo (pod "vivo")
sleep 30               # Aguarda 30 segundos
rm -f /tmp/healthy     # Remove arquivo (pod "morto")
sleep 600              # Aguarda 10 minutos (será reiniciado)
```

**Linha do tempo:**
1. **t=0s**: Pod inicia
2. **t=5s**: Primeiro health check (arquivo existe ✅)
3. **t=10s**: Segundo health check (arquivo existe ✅)
4. **t=30s**: Arquivo é removido
5. **t=35s**: Terceiro health check (arquivo não existe ❌)
6. **t=40s**: Quarto health check falha (❌ - 1ª falha)
7. **t=45s**: Quinto health check falha (❌ - 2ª falha)
8. **t=50s**: Sexto health check falha (❌ - 3ª falha = reinicia!)
9. **t=50s+**: Kubernetes reinicia o contêiner automaticamente

## Como usar

### Implantar o Pod com Liveness Probe
```bash
kubectl apply -f livenessprob.yaml
```

### Monitorar o Pod
```bash
kubectl get pods
kubectl get pods -w  # watch mode (monitoramento em tempo real)
```

### Ver logs do Pod
```bash
kubectl logs liveness-pod
```

### Ver eventos (reinicializações)
```bash
kubectl describe pod liveness-pod
```

Procure por:
- `State`: Running, Waiting, Terminated
- `Last State`: Mostra reinicializações
- `Restart Count`: Quantas vezes foi reiniciado
- `Events`: Histórico de eventos

### Acessar o contêiner
```bash
kubectl exec -it liveness-pod -- /bin/sh
```

### Deletar o Pod
```bash
kubectl delete pod liveness-pod
```

## Exemplo com HTTP Probe (aplicação web)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  containers:
  - name: web-app
    image: nginx
    livenessProbe:
      httpGet:
        path: /health
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 5
      failureThreshold: 3
```

## Exemplo com TCP Probe (banco de dados)

```yaml
livenessProbe:
  tcpSocket:
    port: 3306  # MySQL
  initialDelaySeconds: 5
  periodSeconds: 10
```

## Tipos de Probes relacionados

### Readiness Probe
Verifica se o contêiner está pronto para receber tráfego
- Não reinicia (apenas remove do load balancer)
- Útil para inicializações lentas

### Startup Probe
Verifica se o contêiner inicializou correctamente
- Aplica-se apenas na inicialização
- Bloqueadores outras probes até sucesso

## Boas práticas

✅ **Faça**:
- Use probes apropriadas para sua aplicação
- Configure timeouts e delays adequados
- Monitore a taxa de reinicializações
- Teste os comandos de probe localmente

❌ **Não faça**:
- Não use probes lentas/pesadas
- Não use as mesmas probes em liveness e readiness
- Não reinicie por razões que o probe não pode detectar
- Não use probes muito agressivos (alta carga)

## Debugging

### Ver status do probe
```bash
kubectl describe pod liveness-pod
```

### Executar o comando manualmente
```bash
kubectl exec liveness-pod -- cat /tmp/healthy
```

### Ver eventos e reinicializações
```bash
kubectl logs liveness-pod --previous  # logs antes do crash
```

## Próximos passos

- Combine com **2. Deployments** para adicionar probes a múltiplas réplicas
- Use **Readiness Probes** em conjunto com **4. Services**
- Explore **Startup Probes** para aplicações que iniciam lentamente

