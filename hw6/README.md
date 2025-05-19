bargo-system/
├── .gitlab-ci.yml                         # Главный CI/CD pipeline
├── README.md                              # Общее описание проекта
├── api/
│   └── openapi/
│       ├── bargo-openapi.yaml             # OpenAPI 3.0 спецификация (v1)
│       └── examples/
│           ├── create_order.json
│           ├── order_response.json
│           └── hr_employees.json
├── apps/
│   ├── backend-api/                       # Сервис BarGO (основной backend)
│   │   ├── build.gradle.kts
│   │   ├── Dockerfile
│   │   ├── src/
│   │   │   ├── main/
│   │   │   │   ├── java/com/bargo/
│   │   │   │   │   ├── controller/        # /orders, /recipes
│   │   │   │   │   ├── service/           # Логика заказа, оплаты, рецептов
│   │   │   │   │   ├── security/          # JWT, RBAC
│   │   │   │   │   └── adapter/
│   │   │   │   │       ├── PaymentAdapter.java     # YooKassa интеграция
│   │   │   │   │   
│   │   │   │   └── resources/
│   │   │   │       ├── application.yml
│   │   │   │       └── logback.xml
│   │   │   └── test/
│   │   │       └── java/com/bargo/
│   │   │           └── ...
│   └── frontend-app/                     # React-клиент
│       ├── Dockerfile
│       ├── package.json
│       ├── public/
│       └── src/
│           ├── pages/
│           ├── components/
│           ├── services/
│           │   ├── api.js                 # Обёртка над fetch/axios
│           │   └── auth.js                # Работа с JWT
│           └── index.jsx
├── config/
│   ├── nginx/
│   │   └── nginx.conf                     # Reverse proxy для HTTPS, API
│   ├── flyway/
│   │   ├── conf/flyway.conf
│   │   └── sql/
│   │       ├── V1__init_schema.sql
│   │       └── V2__add_recipes.sql
│   └── auth/
│       ├── public.pem                     # Публичный ключ для JWT
│       └── private.pem                    # Приватный ключ
├── deployments/
│   ├── docker-compose.yml                 # Локальная сборка
│   ├── k8s/
│   │   ├── backend-deployment.yaml
│   │   ├── frontend-deployment.yaml
│   │   ├── db-deployment.yaml
│   │   ├── metabase-deployment.yaml
│   │   ├── secrets.yaml
│   │   └── ingress.yaml
│   └── README.md
├── docker/
│   ├── metabase/
│   │   └── Dockerfile
│   └── postgres/
│       └── init.sql                       # Предзаполнение схемы
├── scripts/
│   ├── deploy.sh                          # Ручной деплой через SSH
│   ├── migrate.sh                         # Запуск миграций через Flyway
│   └── jwt-gen.sh                         # Генерация тестовых JWT
└── tests/
    ├── postman/
    │   └── bargo-api.postman_collection.json
    └── integration/
        ├── payment_adapter_test.http
        └── order_flow_test.http
