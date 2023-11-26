## Установка кластера на чистом железе

Важно перед настройкой кластера отключить службу systemd-resolved.service, 
иначе будет проблема с DNS внутри кластера, будет невозможно получить доступ во внешнюю сеть:

При попытке доступа в сеть из кластера (в примере попытка доступа к smtp) может возникнуть проблема из-за неверного dns:

![img.png](readme-png/img-2.png)

В примере к name добавляется .DOMAINS и получается неверный адрес, 
при попытке доступа во внешнюю сеть из пода


Это происходит из-за файла /run/systemd/resolve/resolv.conf, 
который создает служба systemd-resolved.service


![img.png](readme-png/img-3.png)

**Чтобы этого недопустить нужно выполнить**:

```
sudo systemctl disable systemd-resolved
sudo systemctl stop systemd-resolved
rm  /etc/resolv.conf
echo 'nameserver 8.8.8.8' >  /etc/resolv.conf
```

И перезагрузить сервер, чтобы удалился файл /run/systemd/resolve/resolv.conf

```
reboot
```

Далее выполнить настройку кластера по инструкции:

**!!! После выполнения шага 9 "Как установить Containerd" нужно изменить конфигурацию 
/etc/containerd/config.toml, иначе будет отвалиться flannel (у всех подов CrashLoopBackOff),
на мастер-ноде невозможно выполнить команды, недоступен 6443 порт !!!**

![img.png](readme-png/img-1.png)

Затем выполнить:

```
sudo systemctl restart containerd
sudo systemctl restart kubelet
```

Редактор для редактирования конфигураций:

```
sudo apt-get install nano
```


Сама инструкция:

https://help.reg.ru/support/servery-vps/oblachnyye-servery/ustanovka-programmnogo-obespecheniya/rukovodstvo-po-kubernetes


## Установка сервисов в кластер

Установить helm

```
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

Перейти в директорию 

```
cd /.
```

Склонировать репозиторий с k8s манифестами:

```
git clone https://github.com/1Bitcoin/roommate-kubernetes.git
```

Перейти в директорию

```
cd /roommate-kubernetes/observability/pv
```

Монтируем тома для хранения логов, метрик


```
kubectl apply -f pv-loki-1.yaml
kubectl apply -f pv-loki-2.yaml
kubectl apply -f pv-loki-3.yaml
kubectl apply -f pv-loki-4.yaml
```

Установка подов grafana и prometheus:

Перейти в директорию

```
cd /roommate-kubernetes/observability
```

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

Перейти в директорию

```
cd /roommate-kubernetes/storage
```

```
kubectl apply -f secrets.yaml
kubectl apply -f postgres-service.yaml
```

Перейти в директорию

```
cd /roommate-kubernetes/redis
```

```
kubectl apply -f redis-service.yaml
```

Перейти в директорию

```
cd /roommate-kubernetes/pgadmin
```

```
kubectl apply -f secrets.yaml
kubectl apply -f pgadmin-service.yaml
```

Перейти в директорию

```
cd /roommate-kubernetes/other
```

```
kubectl apply -f bot-secrets.yaml
kubectl apply -f sender-secrets.yaml
kubectl apply -f jwt-secrets.yaml
```

Перейти в директорию

```
cd /roommate-kubernetes/minio
```

```
kubectl apply -f secrets.yaml
kubectl apply -f minio-service.yaml
```

Перейти в директорию

```
cd /roommate-kubernetes/application
```

```
kubectl apply -f roommate-service.yaml
```

Перейти в директорию

```
cd /roommate-kubernetes/frontend
```

```
kubectl apply -f frontend-service.yaml
```

## Установка nginx-ingress

Перейти в директорию

```
cd /roommate-kubernetes/balancer
```

```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx --set controller.service.externalIPs[0]=INPUT_EXTERNAL_IP --set controller.metrics.enabled=true --set-string controller.podAnnotations."prometheus\.io/scrape"="true" --set-string controller.podAnnotations."prometheus\.io/port"="10254"
kubectl apply -f ingress.yaml
```

## Установка cert-manager

Перейти в директорию

```
cd /roommate-kubernetes/balancer
```

Создать отдельный namespace для Cert-Manager:

```
kubectl create namespace cert-manager
```

Добавить helm-репозиторий Jetstack и обновить его:
```
helm repo add jetstack https://charts.jetstack.io
helm repo update
```


Установить Cert-Manager в отдельный namespace "cert-mamager". Актуальную версию уточнить в [ArtifactHub](https://artifacthub.io/packages/helm/cert-manager/cert-manager):

```
helm install cert-manager jetstack/cert-manager --namespace cert-manager --version v1.13.2 --set installCRDs=true
```


Создать объект типа ClusterIssuer, который будет запрашивать сертификаты у LetsEncrypt

Заполнить поле "email", на который будут приходить уведомления об окончании срока действия сертификатов

```
kubectl apply -f production_issuer.yaml
```

Создать объект типа Certificate. Данные в нем менять не нужно. 
При необходимости можно добавить еще dnsNames, на которые необходим HTTPS

```
kubectl apply -f certificate.yaml
```


Отредактировать манифест Ingress, чтобы связать CertManager и ClusterIssuer 
с хостами Ingress через Annotation: 

`annotations.cert-manager.io/cluster-issuer: letsencrypt-prod`

Значение `letsencrypt-prod` берется из манифеста `production_issuer.yaml`: `metadata.name`.  

Также нужно добавить раздел tls. Данные hosts и secretName нужно подставить из certificate.yaml

Затем выполнить:

```
kubectl apply -f ingress.yaml
```

Пример правильно настроенного на SSL ingress :
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
name: hello-kubernetes-ingress
annotations:
  kubernetes.io/ingress.class: nginx
  cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - hw1.your_domain
    - hw2.your_domain
    secretName: ingress-roommate-tls
  rules:
  - host: "hw1.your_domain_name"
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: service-first
            port:
              number: 80
  - host: "hw2.your_domain_name"
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: service-second
            port:
              number: 80
```



Статус сертификатов можно посмотреть через команду:

```
kubectl describe certificate roommate-certificate
```


## Режим debug

```
kubectl port-forward deployment.apps/roommate-app 5005:5005
```