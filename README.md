## Установка кластера на чистом железе

https://help.reg.ru/support/servery-vps/oblachnyye-servery/ustanovka-programmnogo-obespecheniya/rukovodstvo-po-kubernetes

### Проблема 1

После установки кластера, будет периодически отвалиться flannel, 
на мастер-ноде невозможно выполнить команды, недоступен 6443 порт

Решение

![img.png](readme-png/img-1.png)

Затем выполнить:

```
sudo systemctl restart containerd
sudo systemctl restart kubelet
```

### Проблема 2

При попытке доступа в сеть из кластера (в примере попытка доступа к smtp) может возникнуть проблема из-за неверного dns:

![img.png](readme-png/img-2.png)

В примере к name добавляется .DOMAINS и получается неверный адрес

Необходимо удалить строчку из resolv.conf

```
search DOMAINS
```

![img.png](readme-png/img-3.png)

После редактирования сохранить и выполнить:

```
sudo systemctl restart kubelet
```

## Установка сервисов в кластер

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

Перейти в директорию [balancer](balancer)

```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx

kubectl apply -f ingress.yaml
```

Перейти в директорию [storage](storage)

```
kubectl apply -f secrets.yaml
kubectl apply -f postgres-service.yaml
```

Перейти в директорию [redis](redis)

```
kubectl apply -f redis-service.yaml
```

Перейти в директорию [pgadmin](pgadmin)

```
kubectl apply -f secrets.yaml
kubectl apply -f pgadmin-service.yaml
```

Перейти в директорию [other](other)

```
kubectl apply -f bot-secrets.yaml
kubectl apply -f sender-secrets.yaml
kubectl apply -f jwt-secrets.yaml
```

Перейти в директорию [minio](minio)

```
kubectl apply -f secrets.yaml
kubectl apply -f minio-service.yaml
```

Перейти в директорию [application](application)

```
kubectl apply -f roommate-service.yaml
```

## Режим debug

```
kubectl port-forward deployment.apps/roommate-app 5005:5005
```