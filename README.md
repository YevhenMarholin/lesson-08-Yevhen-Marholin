# lesson-08-Yevhen-Marholin

## Service Discovery (ClusterIP)

Було змінено тип Service з `NodePort` на `ClusterIP` для внутрішньої взаємодії в кластері.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: course-app-service
spec:
  type: ClusterIP
  selector:
    app: course-app
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
```

Застосування:

```bash
kubectl apply -f service.yaml
kubectl get svc
service/course-app-service created
NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
course-app-service   ClusterIP   10.96.149.31   <none>        8080/TCP   1s
kubernetes           ClusterIP   10.96.0.1      <none>        443/TCP    3m11s
```

---

## Health Checks (livenessProbe і readinessProbe)

До Deployment додано перевірки стану контейнера.

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

Застосування:

```bash
kubectl apply -f deployment.yaml
kubectl rollout status deployment/course-app
kubectl get pods
Waiting for deployment "course-app" rollout to finish: 0 out of 10 new replicas have been updated...
Waiting for deployment "course-app" rollout to finish: 0 out of 10 new replicas have been updated...
Waiting for deployment "course-app" rollout to finish: 0 out of 10 new replicas have been updated...
Waiting for deployment "course-app" rollout to finish: 0 out of 10 new replicas have been updated...
Waiting for deployment "course-app" rollout to finish: 0 of 10 updated replicas are available...
Waiting for deployment "course-app" rollout to finish: 1 of 10 updated replicas are available...
Waiting for deployment "course-app" rollout to finish: 2 of 10 updated replicas are available...
Waiting for deployment "course-app" rollout to finish: 3 of 10 updated replicas are available...
Waiting for deployment "course-app" rollout to finish: 4 of 10 updated replicas are available...
Waiting for deployment "course-app" rollout to finish: 5 of 10 updated replicas are available...
Waiting for deployment "course-app" rollout to finish: 6 of 10 updated replicas are available...
Waiting for deployment "course-app" rollout to finish: 7 of 10 updated replicas are available...
Waiting for deployment "course-app" rollout to finish: 8 of 10 updated replicas are available...
Waiting for deployment "course-app" rollout to finish: 9 of 10 updated replicas are available...
deployment "course-app" successfully rolled out
NAME                          READY   STATUS    RESTARTS   AGE
course-app-5bf7b45877-67xpk   1/1     Running   0          17s
course-app-5bf7b45877-7cgfr   1/1     Running   0          17s
course-app-5bf7b45877-7j8mb   1/1     Running   0          17s
course-app-5bf7b45877-kdxr4   1/1     Running   0          17s
course-app-5bf7b45877-n464l   1/1     Running   0          17s
course-app-5bf7b45877-nvvcv   1/1     Running   0          17s
course-app-5bf7b45877-r2rsm   1/1     Running   0          17s
course-app-5bf7b45877-smvjz   1/1     Running   0          17s
course-app-5bf7b45877-vswt9   1/1     Running   0          17s
course-app-5bf7b45877-zm7lg   1/1     Running   0          17s

```

---

## Ingress

Було створено Ingress для доступу до застосунку за доменом.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: course-app-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: course-app.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: course-app-service
                port:
                  number: 8080
```

Застосування:

```bash
kubectl apply -f ingress.yaml
kubectl get ingress

ingress.networking.k8s.io/course-app-ingress created
NAME                 CLASS   HOSTS              ADDRESS   PORTS   AGE
course-app-ingress   nginx   course-app.local             80      0s
```

Перевірка:

```bash
curl -H "Host: course-app.local" http://localhost/healthz
{"status":"ok"}
```

---

## Zero Downtime Test

Було перевірено, що при недоступності одного Pod трафік перенаправляється на інші.

Перевірка endpoints:

```bash
kubectl get endpoints course-app-service
NAME                 ENDPOINTS                                                        AGE
course-app-service   10.244.0.15:8080,10.244.0.16:8080,10.244.0.17:8080 + 7 more...   17m
```

Зміна label Pod (імітація відмови):

```bash
POD=$(kubectl get pod -l app=course-app -o jsonpath='{.items[0].metadata.name}')
kubectl label pod $POD app=broken --overwrite

pod/course-app-5bf7b45877-67xpk labeled
```

Перевірка:

```bash
kubectl get endpoints course-app-service

course-app-service   10.244.0.15:8080,10.244.0.16:8080,10.244.0.17:8080 + 7 more...   17m
```

Pod зникає зі списку endpoints.

Повернення Pod:

```bash
kubectl label pod $POD app=course-app --overwrite
pod/course-app-5bf7b45877-67xpk labeled
```

---

## HTTPS (TLS)

Було налаштовано HTTPS через самопідписаний сертифікат.

Створення сертифіката:

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key \
  -out tls.crt \
  -subj "/CN=course-app.local/O=course-app"
...+..........+.....+......+++++++++++++++++++++++++++++++++++++++*..+......+...................+++++++++++++++++++++++++++++++++++++++*....+........+.........+.......+.....+.............+......+.....+..........+.....+.......+...+..+...+....+...+..............................+..+.+..+....++++++
..+......+++++++++++++++++++++++++++++++++++++++*.......+......+....+.....+....+..+...+.......+...+..+....+...........+...+.+...+........+....+.....+++++++++++++++++++++++++++++++++++++++*..........+..+.....................+.......+..+.++++++
-----
```

Створення secret:

```bash
kubectl create secret tls course-app-tls --key tls.key --cert tls.crt
secret/course-app-tls created
```

Оновлений Ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: course-app-ingress
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - course-app.local
      secretName: course-app-tls
  rules:
    - host: course-app.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: course-app-service
                port:
                  number: 8080
```

Застосування:

```bash
kubectl apply -f ingress.yaml
ingress.networking.k8s.io/course-app-ingress configured
```

Перевірка:

```bash
Перевірка HTTPS виконувалась через `port-forward`, оскільки у kind/Codespaces порт `443` напряму не відкритий.

У першому терміналі:

```bash
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8443:443
```

В іншому терміналі:

```bash
curl -k -H "Host: course-app.local" https://localhost:8443/healthz
{"status":"ok"}
```

---

## Висновок

У межах домашнього завдання було:

- змінено Service на `ClusterIP`
- додано `livenessProbe` і `readinessProbe`
- налаштовано Ingress для доступу за доменом
- перевірено балансування трафіку при відмові Pod
- реалізовано HTTPS через TLS

Застосунок стабільно працює та коректно обробляє зовнішній трафік навіть при зміні стану окремих Pods.