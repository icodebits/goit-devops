# Проєкт Terraform: AWS Інфраструктура (lesson-7)

Цей проєкт створює базову інфраструктуру AWS з використанням Terraform.

Він охоплює:
- Зберігання стейтів у S3 з блокуванням через DynamoDB
- Побудову мережевої інфраструктури (VPC)
- Створення репозиторію ECR для зберігання Docker-образів
- Розгортання EKS-кластера з автоскейлінгом
- Деплой Django-додатка в Kubernetes через Helm

---

## Структура проєкту
```bash
lesson-7/
│
├── main.tf                         # Головний файл для підключення модулів
├── backend.tf                      # Налаштування бекенду для стейтів (S3 + DynamoDB)
├── outputs.tf                      # Загальне виведення ресурсів
│
├── modules/                        # Каталог з усіма модулями
│   │
│   ├── s3-backend/                 # Модуль для S3 та DynamoDB
│   │   ├── s3.tf                   # Створення S3-бакета
│   │   ├── dynamodb.tf             # Створення DynamoDB
│   │   ├── variables.tf            # Змінні для S3
│   │   └── outputs.tf              # Виведення інформації про S3 та DynamoDB
│   │
│   ├── vpc/                        # Модуль для VPC
│   │   ├── vpc.tf                  # Створення VPC, підмереж, Internet Gateway
│   │   ├── routes.tf               # Налаштування маршрутизації
│   │   ├── variables.tf            # Змінні для VPC
│   │   └── outputs.tf              # Виведення інформації про VPC
│   │
│   └── ecr/                        # Модуль для ECR
│       ├── ecr.tf                  # Створення ECR репозиторію
│       ├── variables.tf            # Змінні для ECR
│       └── outputs.tf              # Виведення URL репозиторію ECR
│
├── charts/                         # Каталог з Helm-каталогами    
│   └── django-app/                 # Helm-каталог для Django-проєкту
│       ├── Chart.yaml              # Метадані проєкту
│       ├── values.yaml             # Змінні для Django-проєкту
│       └── templates/              # Шаблони для Django-проєкту
│           ├── configmap.yaml      # Конфігмап
│           ├── deployment.yaml     # Депломент
│           ├── hpa.yaml            # Горизонтальне масштабування
│           └── service.yaml        # Сервіс
│
└── README.md                       # Документація проєкту
```



---

## Використання

### 1. Ініціалізація проєкту

```bash
terraform init
```
❗️Перед цим слід вручну створити S3-бакет і DynamoDB-таблицю, які вказані у backend.tf

### 2. Перегляд плану
```bash
terraform plan
```

### 3. Створення інфраструктури
```bash
terraform apply
```

### 4. Видалення інфраструктури
```bash
terraform destroy
```

### 5. Оновлення kubeconfig для підключення до EKS-кластера
```bash
aws eks update-kubeconfig --region us-east-1 --name eks-cluster-demo
```

### 6. Перевірити з'єднання до EKS-кластера
```bash
kubectl get nodes
```
або

```bash
kubectl cluster-info
```

### 7. Отримати ECR URL з Terraform
```bash
export ECR_URL=$(terraform output -raw ecr_repository_url)
```

### 8. Зібрати Docker-образ
```bash
cd ../doker-django-app/django
docker build -t l7-dj-app .
```

### 9. Тегувати образ
```bash
docker tag l7-dj-app:latest $ECR_URL
```

### 10. Login в ECR
```bash
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin $ECR_URL
```

### 11. Пушити образ в ECR
```bash
docker push $ECR_URL
```

### 12.  Перейти в корінь Helm
```bash
cd charts
```

### 13. (опціонально) перевірити шаблони:
```bash
helm template django-app ./django-app
```

### 14. Додати у кластер:
```bash
helm install django-app ./django-app
```

### 15. Перевірити статус:
```bash
helm status django-app
```

### 16. Перевірити EKS pods
```bash
kubectl get pods
```
або

```bash
kubectl get deployment django-app

```

## Огляд модулів

### s3-backend
    Створює S3-бакет з увімкненим версіюванням
    Створює DynamoDB-таблицю з атрибутом LockID для блокування стейт-файлу
    Виводить ARN та URL бакета, назву таблиці
### vpc
    Створює VPC з CIDR-блоком
    3 публічні та 3 приватні підмережі
    Internet Gateway для публічних підмереж
    NAT Gateway (можна додати)
    Route Tables та асоціації
### ecr
    Створює ECR репозиторій
    Автоматичне сканування образів при пуші (scan_on_push)
    Виводить URL репозиторію
### eks
    Створює кластер EKS з доступом до API
    Створює IAM-ролі для кластеру та нодів
    Додає Node Group зі змінним розміром (autoscaling)
    Параметри: instance_type, desired_size, max_size, min_size
    Виводить ім’я та endpoint кластера, IAM-роль вузлів

## Параметри, які можна змінювати

    Ім’я S3-бакета для state-файлів (bucket_name)
    Ім’я DynamoDB-таблиці (table_name)
    CIDR блок VPC та CIDR підмереж
    Ім’я ECR репозиторію (ecr_name)


## Helm-чарт django-app

### configmap.yaml
    Містить ключі з .env файлу (наприклад POSTGRES_HOST, POSTGRES_PORT тощо). Вони додаються у контейнер через envFrom.

### deployment.yaml
    Створює Pod з образом Django (з ECR)
    Підключає змінні середовища через ConfigMap
    Працює на port 8000 (вказаний у Django)
    Параметри образу конфігуруються у values.yaml

### service.yaml
    Тип: LoadBalancer
    Проксіює порт 80 до 8000 всередині контейнера
    Дозволяє зовнішній доступ до Django через DNS

### hpa.yaml
    Масштабує кількість pod’ів у межах 2–6
    Базується на CPU utilization > 70%