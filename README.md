# service-manager
Docker-compose service to manage other services/sites

## Запуск

Создайте `.env` из примера и задайте нужные домены:

```bash
cp .env.dist .env
docker compose up -d
```

Если запускаете без `.env`, Docker Compose возьмет значения по умолчанию из
`docker-compose.yml`.

## Reverse proxy и HTTPS

В составе поднимаются:

- `nginx-proxy` - автоматически находит контейнеры с `VIRTUAL_HOST` и
  проксирует их по HTTP/HTTPS;
- `acme-companion` - выпускает и обновляет ACME/Let's Encrypt сертификаты для
  контейнеров с `LETSENCRYPT_HOST`.

Для работы публичных сертификатов домены из `LETSENCRYPT_HOST` должны
резолвиться на этот сервер, а порты `80` и `443` должны быть доступны извне.
Для доменов вида `*.local` публичный Let's Encrypt сертификат не будет выпущен.

Минимальная конфигурация сервиса за прокси:

```yaml
environment:
  - VIRTUAL_HOST=app.example.com
  - VIRTUAL_PORT=8080
  - LETSENCRYPT_HOST=app.example.com
  - HTTPS_METHOD=noredirect
networks:
  - host_network
```

## Prometheus и Grafana

Интерфейсы доступны напрямую:

```text
http://localhost:3000   # Grafana
http://localhost:9090   # Prometheus
```

Если домены из `.env` резолвятся на этот сервер, интерфейсы также доступны через
прокси:

```text
http://grafana.local
http://prometheus.local
```

Логин Grafana по умолчанию:

```text
admin / admin
```

Datasource Prometheus в Grafana добавляется с URL:

```text
http://prometheus:9090
```

## Автообнаружение targets в Prometheus

Prometheus использует Docker service discovery и читает Docker API через
read-only `/var/run/docker.sock`. Он автоматически добавляет targets только для
контейнеров, которые:

- подключены к сети `proxy-network`;
- имеют label `prometheus.scrape=true`;
- имеют label `prometheus.port`.

Пример сервиса, который будет автоматически собираться Prometheus:

```yaml
services:
  app:
    image: example/app
    environment:
      - VIRTUAL_HOST=app.local
      - VIRTUAL_PORT=8080
    networks:
      - host_network
    labels:
      prometheus.scrape: "true"
      prometheus.job: "app"
      prometheus.host: "app.local"
      prometheus.port: "8080"
      prometheus.path: "/metrics"
```

Labels:

- `prometheus.scrape` - включает сбор метрик для контейнера;
- `prometheus.job` - значение label `job` в Prometheus;
- `prometheus.host` - человекочитаемый `instance`;
- `prometheus.port` - порт контейнера с метриками;
- `prometheus.path` - путь метрик, по умолчанию обычно `/metrics`;
- `prometheus.scheme` - опционально, например `https`.

После изменения labels или сетей перезапустите нужные сервисы:

```bash
docker compose up -d --force-recreate prometheus
```
