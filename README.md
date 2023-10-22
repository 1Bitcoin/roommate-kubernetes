## Установка сервисов в кластер

loki stack

Перейти в директорию [observability](observability)

Монтируем тома для хранения логов, метрик

```
kubectl apply -f pv-loki-1.yaml
kubectl apply -f pv-loki-2.yaml
kubectl apply -f pv-loki-3.yaml
kubectl apply -f pv-loki-4.yaml
```

Установка подов

```
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm search repo loki
helm install --values values.yaml loki grafana/loki-stack
```

Дождаться ошибки CrashLoopBackOff на поде loki-prometheus-server

Необходимо дать полные права на PVC loki-prometheus-server, затем удалить job

```
kubectl apply -f job.yaml
```

После того как ошибка пропала, нужно удалить job

```
kubectl delete job prometheus-chown-0
```

!!!После выполнения job нужно удалить pod loki-prometheus-server, чтобы он пересоздался!!!!

Чтобы получить данные для входа в графану:

```
kubectl get secret loki-grafana -o go-template='{{range $k,$v := .data}}{{printf "%s: " $k}}{{if not $v}}{{$v}}{{else}}{{$v | base64decode}}{{end}}{{"\n"}}{{end}}'
```

Чтобы удалить loki-stack

```
helm uninstall loki
```

Postgres

Перейти в директорию [storage](storage)

```
kubectl apply -f secrets.yaml
kubectl apply -f postgres-service.yaml
```

Redis

Перейти в директорию [redis](redis)

```
kubectl apply -f redis-service.yaml
```

Pgadmin

Перейти в директорию [pgadmin](pgadmin)

```
kubectl apply -f secrets.yaml
kubectl apply -f pgadmin-service.yaml
```

Other

Перейти в директорию [other](other)

```
kubectl apply -f bot-secrets.yaml
kubectl apply -f sender-secrets.yaml
```

Minio

Перейти в директорию [minio](minio)

```
kubectl apply -f secrets.yaml
kubectl apply -f minio-service.yaml
```

Приложение

Выполнить сборку приложения TODO!!!

Перейти в директорию [application](application)

```
kubectl apply -f roommate-service.yaml
```

## Режим debug

```
kubectl port-forward deployment.apps/roommate-app 5005:5005
```