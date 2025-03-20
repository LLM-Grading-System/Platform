# LLM Grading

## Архитектура

```mermaid
graph LR
    %% C4 style classes c4model.com %%
    classDef person fill:#08427b,stroke:black,color:white;
    classDef container fill:#1168bd,stroke:black,color:white;
    classDef database fill:#1168bd,stroke:black,color:white;
    classDef software fill:#1168bd,stroke:black,color:white;
    classDef existing fill:#999999,stroke:black,color:white;
    classDef boundary fill:white,stroke:black,stroke-width:2px,stroke-dasharray: 5 5;
    classDef frame fill:white,stroke:black;


    %% nodes %%
    GitHub["GitHub API"]:::existing
    Mistral["Mistral AI API"]:::existing
    Admin((Admin)):::person
    Student((Student)):::person
    WebApp("Admin Panel Application <br>[Container: React]"):::container
    WebServer("Web Server <br>[Container: nginx]"):::container
    CoreAPI("Core API <br>[Container: FastAPI]"):::container
    Worker("LLM Grader <br>[Container: FastStream]"):::container
    CoreDB[("Core Data <br>[Container: PostgreSQL]")]:::database
    S3[("Submissions <br>[Container: Minio]")]:::database
    EventsQueue(["Events Queue <br>[Container: Kafka]"]):::container
    TGBot("Telegram Bot <br>[Container: Aiogram]"):::container

    %% connections and boundaries %%
    
    subgraph Legend [Containers]
        Student-.->|Uses| TGBot
        Admin-.->|Uses| WebApp
        
        
        subgraph Boundary["Boundary: System"]
            WebApp-.->|"Proxies requests <br> [HTTP/HTTPS]"| WebServer
            WebServer-.->|"Proxies requests <br> [HTTP/HTTPS]"| CoreAPI
            TGBot-.->|"Makes requests <br> [HTTP/HTTPS]"| CoreAPI
            CoreAPI-.->|"Makes requests <br> [HTTP/HTTPS]"| S3
            CoreAPI-.->|"Makes requests  <br> [TCP]"| CoreDB
            CoreAPI-.->|"Push events <br> [TCP]"| EventsQueue
            Worker-.->|"Makes requestts <br> [HTTP/HTTPS]"| CoreAPI
            Worker-.->|"Pull events <br> [TCP]"| EventsQueue
            TGBot-.->|"Pull events <br> [TCP]"| EventsQueue
        end
        class Boundary boundary
        GitHub
        Mistral
    end
    class Legend frame
```

## Запуск

### Переменные окружения

```env
# Core API
PLATFORM_ADMIN_USER=admin
PLATFORM_ADMIN_PASSWORD=password
# Postgres
POSTGRES_DB=grading
POSTGRES_USER=postgres
POSTGRES_PASSWORD=password
# Minio
MINIO_ROOT_USER=minio
MINIO_ROOT_PASSWORD=password
MINIO_ACCESS_KEY=<SET-LATER>
MINIO_SECRET_KEY=<SET-LATER>
# Kafka
KAFKA_BOOTSTRAP_SERVERS=localhost:29092
KAFKA_UI_ADMIN_LOGIN=admin
KAFKA_UI_ADMIN_PASSWORD=password
# Mistral API
MISTRAL_API_KEY=<SET-LATER>
MISTRAL_MODEL=codestral-latest
```

### Инструкция по изначальному запуску

1. Запустить minio `docker-compose up -d minio`
2. Зайти в админку Minio по http://localhost:9001/login под кредами MINIO_ROOT_USER/MINIO_ROOT_PASSWORD
3. Перейти по http://localhost:9001/access-keys и создать access key
4. Скопировать данные по созданному ключу в env MINIO_ACCESS_KEY, MINIO_SECRET_KEY
5. Запустить контейнеры для Kafka `docker-compose up -d zookeeper kafka kafka-ui`
6. Зайти в админку Kafka UI по http://localhost:8090/auth под кредами KAFKA_UI_ADMIN_LOGIN/KAFKA_UI_ADMIN_PASSWORD
7. Перейти по http://localhost:8090/ и убедиться, что кластер живой
8. Запустить остальные контейнеры `docker-compose up -d postgres api grader`
