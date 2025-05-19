# Структура репозитория `bar-management-system`

```txt
bar-management-system/
├── .gitlab-ci.yml                # CI/CD‑конвейер (сборка, тесты, деплой)
├── README.md                     # Общее описание проекта
├── charts/                       # Helm‑чарты для Kubernetes
│   └── bargo/                    # Chart для BarGo API + фронтенда
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── deployment.yaml
│           ├── service.yaml
│           ├── ingress.yaml
│           └── configmap.yaml
│
├── frontend/                     # Web‑клиент (React/Vue)
│   ├── Dockerfile                # Сборка и упаковка SPA
│   ├── nginx.conf                # Конфигурация Nginx для статических файлов
│   ├── package.json
│   ├── public/
│   └── src/
│       ├── components/
│       └── index.js
│
├── backend/                      # BarGo API (Spring Boot)
│   ├── Dockerfile                # Сборка bar‑api.jar и запуск
│   ├── build.gradle / pom.xml
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/…             # исходники сервисов и адаптеров
│   │   │   └── resources/
│   │   │       └── application.yml
│   │   └── test/…
│   └── flyway/                   # Миграции БД
│       └── migrations/
│           ├── V1__init_schema.sql
│           ├── V2__add_order_table.sql
│           └── …
│
├── payment‑adapter/              # Микросервис–адаптер для YooKassa
│   ├── Dockerfile
│   ├── src/…                     # реализация REST‑клиента к YooKassa
│   └── config/
│
├── bi‑adapter/                   # Микросервис–адаптер для Metabase
│   ├── Dockerfile
│   ├── src/…                     # REST/SQ L‑клиент к Metabase
│   └── config/
│
├── infra/                        # Инфраструктурные манифесты
│   ├── k8s/                      # «чистый» мануал K8s без Helm
│   │   ├── backend-deployment.yaml
│   │   ├── frontend-deployment.yaml
│   │   └── mongo-config.yaml
│   └── terraform/                # (опционально) IaC Terraform скрипты
│       └── main.tf
│
├── metabase/                     # Авто‑деплой Metabase в Docker
│   ├── docker-compose.yml
│   └── env/                      # конфиги окружения (секреты через Vault)
│
├── scripts/                      # Утилиты и helper‑скрипты
│   ├── deploy.sh                 # вызов kubectl/helm по средам
│   └── rollback.sh
│
└── docs/
    ├── deployment_diagram.puml   # PlantUML исходник диаграммы развёртывания
    ├── api_spec.yaml             # OpenAPI‑спецификация BarGo API
    └── architecture.md           # Архитектурные документы (ЗАРы)
