# Практична 10. Kubernetes — практична частина (REPORT)

**Дата перевірки:** 25.11.2026. **Статус:** рівні 60 / 75 / 90 / 100 виконано в Minikube. Fork: https://github.com/Uamixerz/k8s-demo. Власний сервіс: https://github.com/Uamixerz/webPolahod. Minikube v1.39.0, Kubernetes v1.37.0, driver docker.

Повні логи: [папка lab10](https://github.com/Uamixerz/webPolahod/blob/main/docs/evidence/lab10/). Команди для `k8s-demo` виконувалися з `external/k8s-demo` (clone `vivalafit/k8s-demo`, commit `76861b8`), для власного сервісу — з кореня репозиторію.

## 60 балів

### Локальний запуск

`npm install`, `npm run dev` ([лог](https://github.com/Uamixerz/webPolahod/blob/main/docs/evidence/lab10/01-local.txt)).

| Endpoint | Відповідь |
|---|---|
| /api/status | `{"status":"ok","appMode":"dev","version":"1.0.0","timestamp":…}` |
| /api/items?filter=a | `[{"id":1,"name":"Item A"}]` |
| /api/slow | 200 через 2 с, `{"status":"ok","delayMs":2000}` |
| /api/error | 200 `No error`; при `ERROR_MODE=always` — 500 (за кодом) |
| /healthz | `ok` |
| /ready | перші 5 с — 503 `not-ready`, далі 200 `ready` |

### Контейнер

```powershell
docker build -t k8s-demo-app:1.0.0 .
docker run --rm -p 127.0.0.1:8080:8080 -e SECRET_TOKEN=local k8s-demo-app:1.0.0
curl.exe http://localhost:8080/api/status
```

`{"status":"ok","appMode":"dev","version":"1.0.0","timestamp":"2026-11-24T23:32:21.716Z"}` ([лог](https://github.com/Uamixerz/webPolahod/blob/main/docs/evidence/lab10/02-docker.txt)).

### Minikube і розгортання

`minikube start --memory=4096 --cpus=2 --driver=docker`. Контекст `minikube`, нода `minikube` — **Ready** ([лог](https://github.com/Uamixerz/webPolahod/blob/main/docs/evidence/lab10/03-minikube.txt)). Образ зібрано в кластері: `minikube image build -t maxnavrotsky/k8s-demo-app:1.0.0 .`. Застосовано namespace, configmap, secret, deployment, service ([лог](https://github.com/Uamixerz/webPolahod/blob/main/docs/evidence/lab10/04-deploy.txt)):

```text
$ kubectl get deployment,rs,pods,svc -n k8s-demo
deployment.apps/k8s-demo-api   2/2     2            2           10s
replicaset.apps/k8s-demo-api-5c45559886   2         2         2       10s
pod/k8s-demo-api-5c45559886-t88md   1/1     Running   0          10s
pod/k8s-demo-api-5c45559886-vrgvf   1/1     Running   0          10s
service/k8s-demo-api   ClusterIP   10.105.163.226   <none>        80/TCP    9s
```

Port-forward `svc/k8s-demo-api 8080:80` повернув status і items ([лог](https://github.com/Uamixerz/webPolahod/blob/main/docs/evidence/lab10/05-port-forward.txt)). Запит іде через тунель API-сервера: kubectl за Service знаходить Pod і передає трафік безпосередньо в нього.

## 75 балів

Upstream уже має `APP_MODE: "prod"`, тому спочатку встановлено `dev`, потім повернуто `prod` ([лог](https://github.com/Uamixerz/webPolahod/blob/main/docs/evidence/lab10/06-configmap-probes.txt)).

| Крок | ConfigMap | env у Pod | /api/status |
|---|---|---|---|
| Після rollout restart | dev | dev | dev |
| `apply` з prod, той самий Pod | prod | **dev** | dev |
| Після rollout restart, новий Pod | prod | prod | prod |

**Чому значення не змінилося:** env копіюється в контейнер один раз, коли kubelet створює його. Оновлення ConfigMap не змінює середовище запущеного процесу.

**Роль rollout restart:** змінює анотацію шаблону, Deployment створює новий ReplicaSet і поступово замінює Pod-и; нові читають актуальний ConfigMap. **Рекомендований порядок:** змінити ConfigMap у git → `apply` → `rollout restart` → `rollout status` → перевірити новий Pod.

| Перевірка | Шлях | initialDelay | period | failureThreshold |
|---|---|---|---|---|
| Liveness | /healthz | 5 с | 10 с | 3 |
| Readiness | /ready | 3 с | 5 с | 3 |

Liveness визначає, чи треба перезапустити контейнер; readiness — чи можна надсилати йому трафік. Видно в подіях: `Readiness probe failed: … 503` у перші секунди без жодного restart.

## 90 балів

### Ingress

`minikube addons enable ingress`, `kubectl apply -f k8s/ingress.yaml -n k8s-demo`, `minikube tunnel` ([лог](https://github.com/Uamixerz/webPolahod/blob/main/docs/evidence/lab10/07-ingress.txt)). Host у маніфесті — `k8s-demo.local`.

```text
NAME               CLASS   HOSTS            ADDRESS        PORTS   AGE
k8s-demo-ingress   nginx   k8s-demo.local   192.168.49.2   80      77s

$ curl --resolve k8s-demo.local:80:127.0.0.1 http://k8s-demo.local/api/status
{"status":"ok","appMode":"prod","version":"1.0.0",…} [200]
```

`--resolve` підставляє IP замість запису в `hosts`. Для браузера додати від адміністратора рядок `127.0.0.1 k8s-demo.local polahod.local` у `C:\Windows\System32\drivers\etc\hosts`. **Шлях:** tunnel → Ingress controller → Ingress (правило host/path) → Service `k8s-demo-api` → готовий Pod. У логах controller видно обидва Pod-и: `10.244.0.11:8080` і `10.244.0.12:8080`.

### Debug

Помилок Ingress не було. Базові команди:

| Команда | Що показує |
|---|---|
| `kubectl describe pod/ingress …` | Probes, стан, backends, події |
| `kubectl get events -n k8s-demo --sort-by=.metadata.creationTimestamp` | Хронологія scheduling, pull, kill, probe failures |
| `kubectl logs -n ingress-nginx deploy/ingress-nginx-controller` | Який upstream отримав запит і код відповіді |
| `kubectl get endpointslices -n k8s-demo` | Чи є готові Pod-и за Service |

Реальна проблема під час розгортання власного сервісу: `deploy-k8s.ps1` передавав `deployment,svc,pods,ingress` як масив PowerShell, тому kubectl відповідав `NotFound`. Виправлено лапками.

### Самовідновлення

Видалено Pod `…-rhkpz`; ReplicaSet одразу створив `…-65h7c`, на першій перевірці (31 с) знову 2/2 Running ([лог](https://github.com/Uamixerz/webPolahod/blob/main/docs/evidence/lab10/08-self-healing.txt)). Deployment задає бажані 2 репліки, ReplicaSet порівнює фактичну кількість із бажаною і створює відсутні.

## 100 балів: власний сервіс «Полагодь»

ASP.NET Core API заявок на ремонт: `/api/repairs` (CRUD), `/api/status`, `/health`, `/ready`; поруч Blazor Web і PostgreSQL. Код: `src/Polahod.Api`. [Dockerfile](https://github.com/Uamixerz/webPolahod/blob/main/Dockerfile):

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0.202 AS build
WORKDIR /src
COPY . .
RUN dotnet restore src/Polahod.Api/Polahod.Api.csproj --locked-mode
RUN dotnet publish src/Polahod.Api -c Release --no-restore -o /out /p:UseAppHost=false
FROM mcr.microsoft.com/dotnet/aspnet:10.0
WORKDIR /app
COPY --from=build /out .
RUN mkdir /data && chown app:app /data
USER app
ENV ASPNETCORE_HTTP_PORTS=8080
ENV ConnectionStrings__Database="Data Source=/data/polahod.db"
EXPOSE 8080
ENTRYPOINT ["dotnet", "Polahod.Api.dll"]
```

Відповідь у контейнері — [docker-run.txt](https://github.com/Uamixerz/webPolahod/blob/main/docs/evidence/lab06/docker-run.txt): `GET /health` → `{"status":"ok"}`.

Маніфести — [`k8s`](https://github.com/Uamixerz/webPolahod/blob/main/k8s/), власний namespace `polahod`. Розгортання однією командою ([лог](https://github.com/Uamixerz/webPolahod/blob/main/docs/evidence/lab10/09-polahod-deploy.txt)):

```powershell
.\scripts\deploy-k8s.cmd
```

```text
deployment.apps/polahod-api   2/2     2            2
deployment.apps/polahod-db    1/1     1            1
deployment.apps/polahod-web   1/1     1            1
ingress.networking.k8s.io/polahod   nginx   polahod.local   192.168.49.2   80

$ curl http://localhost:8082/api/status     (port-forward svc/polahod-api 8082:80)
{"service":"polahod-api","mode":"k8s","secretConfigured":true} [200]
$ curl --resolve polahod.local:80:127.0.0.1 http://polahod.local/api/status   [200]
$ curl --resolve polahod.local:80:127.0.0.1 http://polahod.local/ (Blazor)    [200]
```

Повторний POST з тим самим `Idempotency-Key` дав той самий `id` ([лог](https://github.com/Uamixerz/webPolahod/blob/main/docs/evidence/lab10/10-polahod-access.txt)). `GET /items` повертає `[]`, бо в кластері вимкнено seed.

ConfigMap і Secret підключено через `envFrom`. `APP_MODE: k8s → k8s-prod`: до рестарту API відповідав `k8s`, після `rollout restart` — `k8s-prod`; заявка збереглася в PVC. Видалений API-Pod відновився ([лог](https://github.com/Uamixerz/webPolahod/blob/main/docs/evidence/lab10/11-polahod-config-heal.txt)).

## Контрольні запитання

1. **ConfigMap і Secret.** ConfigMap — звичайна конфігурація. Secret — чутливі дані з окремими правилами доступу; base64 — кодування, а не шифрування.
2. **Liveness і readiness.** Невдала liveness перезапускає контейнер, невдала readiness лише виводить Pod із Service.
3. **Шлях запиту.** port-forward: клієнт → kubectl → API server → Pod (Service лише для вибору Pod-а). `k8s-demo.local`: клієнт → tunnel → Ingress controller → правило Ingress → Service → Pod.
4. **Оновлення env із ConfigMap.** apply → rollout restart → rollout status → перевірка нового Pod-а.
5. **Міграція власного сервісу.** Створено образи API і Web, конфігурацію винесено в ConfigMap/Secret, SQLite замінено на PostgreSQL з PVC, додано probes, Service, Ingress. Відмінності від демо: стан у БД, два застосунки, порядок запуску (спочатку БД, одна репліка API для створення схеми, потім дві).
