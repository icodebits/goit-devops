# Проєкт Terraform: AWS Інфраструктура (lesson-7)

Цей проєкт створює базову інфраструктуру AWS з використанням Terraform.

Він охоплює:
- Зберігання стейтів у S3 з блокуванням через DynamoDB
- Побудову мережевої інфраструктури (VPC)
- Створення репозиторію ECR для зберігання Docker-образів

---

## Структура проєкту
```bash
lesson-7/
│
├── main.tf                  # Головний файл для підключення модулів
├── backend.tf               # Налаштування бекенду для стейтів (S3 + DynamoDB)
├── outputs.tf               # Загальне виведення ресурсів
│
├── modules/                 # Каталог з усіма модулями
│   │
│   ├── s3-backend/          # Модуль для S3 та DynamoDB
│   │   ├── s3.tf            # Створення S3-бакета
│   │   ├── dynamodb.tf      # Створення DynamoDB
│   │   ├── variables.tf     # Змінні для S3
│   │   └── outputs.tf       # Виведення інформації про S3 та DynamoDB
│   │
│   ├── vpc/                 # Модуль для VPC
│   │   ├── vpc.tf           # Створення VPC, підмереж, Internet Gateway
│   │   ├── routes.tf        # Налаштування маршрутизації
│   │   ├── variables.tf     # Змінні для VPC
│   │   └── outputs.tf       # Виведення інформації про VPC
│   │
│   └── ecr/                 # Модуль для ECR
│       ├── ecr.tf           # Створення ECR репозиторію
│       ├── variables.tf     # Змінні для ECR
│       └── outputs.tf       # Виведення URL репозиторію ECR
│
└── README.md                # Документація проєкту
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

### 10. Пушити образ в ECR
```bash
docker push $ECR_URL
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

## Параметри, які можна змінювати

    Ім’я S3-бакета для state-файлів (bucket_name)
    Ім’я DynamoDB-таблиці (table_name)
    CIDR блок VPC та CIDR підмереж
    Ім’я ECR репозиторію (ecr_name)