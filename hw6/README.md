# Структура репозитория `bar-management-system`

```txt
bar-management-system/
├── .gitlab-ci.yml                    # CI/CD‑конвейер
├── README.md                         # Описание проекта и инструкция по запуску
├── docs/
│   ├── deployment_diagram.puml       # PlantUML‑исходник диаграммы развёртывания
│   ├── 4C_model.puml                 # PlantUML‑исходник 4C‑диаграммы
│   ├── api_spec.yaml                 # OpenAPI 3.0 спецификация BarGo API
│   └── architecture.md               # ЗАР‑документы (Д5, Д6 и т.д.)
│
├── frontend/                         # Веб‑приложение (Web‑интерфейс)
│   ├── Dockerfile
│   ├── nginx.conf
│   ├── package.json
│   └── src/
│       └── ...
│
├── auth-comp/                        # Компонент авторизации (auth‑comp)
│   ├── Dockerfile
│   ├── src/                          # Spring Security + JWT
│   └── config/
│
├── order-waiter-comp/                # Управление заказами (order‑waiter‑comp)
│   ├── Dockerfile
│   ├── src/                          # REST‑контроллеры для /orders
│   └── config/
│
├── store-bartender-comp/             # Учёт ингредиентов (store‑bartender‑comp)
│   ├── Dockerfile
│   ├── src/                          # REST‑контроллеры для /recipes
│   └── config/
│
├── manage-admin-comp/                # CRUD‑админка (manage‑admin‑comp)
│   ├── Dockerfile
│   ├── src/                          # REST‑контроллеры /admin/…
│   └── config/
│
├── report-admin-comp/                # BI‑адаптер (report‑admin‑comp)
│   ├── Dockerfile
│   ├── src/                          # Spring + Metabase REST API клиент
│   └── config/
│
├── payment-comp/                     # Платёжный адаптер (payment‑comp)
│   ├── Dockerfile
│   ├── src/                          # REST‑клиент YooKassa
│   └── config/
│
├── bi-adapter/                       # (опционально) Metabase в Docker
│   ├── docker-compose.yml
│   └── env/                          # Конфиги окружения для BI
│
├── charts/                           # Helm‑чарты для Kubernetes
│   └── bargo/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── deployment.yaml
│           ├── service.yaml
│           └── ingress.yaml
│
├── infra/                            # Инфраструктурные манифесты
│   ├── k8s/                          # «Чистые» YAML‑манифесты K8s
│   └── terraform/                    # (опционально) Terraform
│
├── scripts/                          # Скрипты деплоя и отката
│   ├── deploy.sh
│   └── rollback.sh
│
└── backend/                          # (оставить только, если есть общее API)
    ├── Dockerfile
    └── src/                          # Общие утилиты, если order‑waiter/… разделены
