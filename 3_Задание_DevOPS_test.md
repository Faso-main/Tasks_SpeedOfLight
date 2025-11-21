# Полноценный диплой облачного решения 

### Суть задачи
- Вам дается базовый код приложения (frontend, backend, база данных), который необходимо самостоятельно упаковать в Docker-контейнеры, настроить инфраструктуру и автоматизировать процесс развертывания.
- Один за другим создаем файлы в корне проекта и выполняем настроку конфигурации решения.
- Идите по документу постепенно, тут много.
- Нейронка тут не нужна!!!.

### Ключевые компоненты

**Приложение:**
- **Frontend** - статические HTML/JS файлы
- **Backend** - Node.js/Express API сервер  
- **Database** - PostgreSQL с тестовыми данными

**Инфраструктура:**
- **Docker** - контейнеризация всех компонентов
- **Nginx** - reverse proxy и веб-сервер
- **Docker Compose** - оркестрация контейнеров

**Автоматизация:**
- **GitHub Actions** - CI/CD пайплайн
- **SSH-деплой** - автоматическое развертывание на VPS
- **Health-check** - проверка работоспособности после деплоя

### Основные требования

1. **Frontend** доступен по `http://SERVER_IP/`
2. **Backend API** отвечает на:
   - `GET /api/users` - список пользователей
   - `GET /api/health` - статус приложения
3. **PostgreSQL** инициализируется автоматически
4. **Nginx** проксирует запросы:
   - `/` → frontend
   - `/api/` → backend
5. **Docker Compose** запускает весь стек одной командой
6. **CI/CD** автоматически деплоит при каждом push в main ветку

### Что нужно сделать

1. Создать Dockerfile для каждого сервиса
2. Настроить docker-compose.yml для оркестрации
3. Развернуть на VPS с Ubuntu (можем дать)
4. Настроить автоматический деплой через GitHub Actions
5. Обеспечить health-check эндпоинты
6. Протестировать работоспособность

### Результат
Полностью рабочее приложение, доступное из интернета, с автоматизированным процессом обновления при изменении кода.

---

## Техническая реализация

### 1. Структура файлов проекта

```
cloud-web-app/
├── frontend/
│   ├── index.html
│   ├── app.js
│   └── Dockerfile
├── backend/
│   ├── server.js
│   ├── package.json
│   └── Dockerfile
├── db/
│   └── init.sql
├── nginx/
│   ├── nginx.conf
│   └── Dockerfile
├── docker-compose.yml
└── .github/
    └── workflows/
        └── deploy.yml
```

### 2. Конфигурационные файлы

#### 2.1 Frontend

**frontend/index.html**
```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Cloud Web App</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; }
        .user { border: 1px solid #ddd; padding: 10px; margin: 5px 0; }
        .error { color: red; }
    </style>
</head>
<body>
    <h1>Cloud Web Application</h1>
    <div id="users"></div>
    <script src="app.js"></script>
</body>
</html>
```

**frontend/app.js**
```javascript
async function loadUsers() {
    try {
        const response = await fetch('/api/users');
        if (!response.ok) throw new Error('Network response was not ok');
        
        const users = await response.json();
        const usersContainer = document.getElementById('users');
        
        usersContainer.innerHTML = users.map(user => 
            `<div class="user">
                <strong>${user.name}</strong><br>
                Email: ${user.email}<br>
                Создан: ${new Date(user.created_at).toLocaleDateString('ru-RU')}
            </div>`
        ).join('');
    } catch (error) {
        document.getElementById('users').innerHTML = 
            `<div class="error">Ошибка загрузки пользователей: ${error.message}</div>`;
    }
}

document.addEventListener('DOMContentLoaded', loadUsers);
```

**frontend/Dockerfile**
```dockerfile
FROM nginx:alpine
COPY . /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

#### 2.2 Backend

**backend/package.json**
```json
{
  "name": "cloud-web-app-backend",
  "version": "1.0.0",
  "description": "Backend for cloud web application",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "test": "echo \"No tests specified\" && exit 0"
  },
  "dependencies": {
    "express": "^4.18.2",
    "pg": "^8.11.3"
  }
}
```

**backend/server.js**
```javascript
const express = require('express');
const { Pool } = require('pg');
const app = express();
const port = 3000;

const pool = new Pool({
    user: 'user',
    host: 'db',
    database: 'appdb',
    password: 'password',
    port: 5432,
});

app.get('/api/health', (req, res) => {
    res.status(200).json({ status: 'OK', timestamp: new Date().toISOString() });
});

app.get('/api/users', async (req, res) => {
    try {
        const result = await pool.query('SELECT * FROM users ORDER BY id');
        res.json(result.rows);
    } catch (err) {
        console.error('Database error:', err);
        res.status(500).json({ error: 'Internal server error' });
    }
});

app.listen(port, '0.0.0.0', () => {
    console.log(`Backend server running on port ${port}`);
});
```

**backend/Dockerfile**
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["npm", "start"]
```

#### 2.3 База данных

**db/init.sql**
```sql
CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO users (name, email) VALUES 
('Иван Иванов', 'ivan@example.com'),
('Петр Петров', 'petr@example.com'),
('Мария Сидорова', 'maria@example.com')
ON CONFLICT (email) DO NOTHING;
```

#### 2.4 Nginx

**nginx/nginx.conf**
```nginx
events {
    worker_connections 1024;
}

http {
    server {
        listen 80;
        server_name _;

        location / {
            proxy_pass http://frontend:80;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        location /api/ {
            proxy_pass http://backend:3000;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}
```

**nginx/Dockerfile**
```dockerfile
FROM nginx:alpine
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

#### 2.5 Docker Compose

**docker-compose.yml**
```yaml
version: '3.8'

services:
  nginx:
    build: ./nginx
    ports:
      - "80:80"
    depends_on:
      - frontend
      - backend
    networks:
      - app-network

  frontend:
    build: ./frontend
    networks:
      - app-network

  backend:
    build: ./backend
    environment:
      - DATABASE_URL=postgresql://user:password@db:5432/appdb
    depends_on:
      - db
    networks:
      - app-network

  db:
    image: postgres:15
    environment:
      - POSTGRES_DB=appdb
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
    volumes:
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

volumes:
  postgres_data:
```

### 3. Настройка VPS

#### 3.1 Подготовка сервера

```bash
# Подключение к VPS
ssh root@your-server-ip

# Обновление системы
apt update && apt upgrade -y

# Установка Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# Установка Docker Compose
apt install docker-compose-plugin -y

# Установка Git
apt install git -y

# Настройка прав
usermod -aG docker $USER
```

#### 3.2 Клонирование проекта

```bash
mkdir -p /opt/cloud-web-app
cd /opt/cloud-web-app
git clone https://github.com/your-username/cloud-web-app.git .
```

### 4. Настройка CI/CD

#### 4.1 Генерация SSH ключей

```bash
# На локальной машине
ssh-keygen -t ed25519 -C "github-actions" -f github-actions-key
```

#### 4.2 Настройка VPS для SSH доступа

```bash
# На VPS
mkdir -p ~/.ssh
cat github-actions-key.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

#### 4.3 Настройка GitHub Secrets

В репозитории GitHub: Settings → Secrets and variables → Actions → New repository secret:

- `SSH_HOST` - IP адрес VPS
- `SSH_USER` - root
- `SSH_KEY` - содержимое файла `github-actions-key` (приватный ключ)

#### 4.4 GitHub Actions Workflow

**.github/workflows/deploy.yml**
```yaml
name: Deploy to VPS

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
      
    - name: Test backend
      run: |
        cd backend
        npm install
        npm test
      
    - name: Deploy to VPS
      uses: appleboy/ssh-action@v1.0.3
      with:
        host: ${{ secrets.SSH_HOST }}
        username: ${{ secrets.SSH_USER }}
        key: ${{ secrets.SSH_KEY }}
        script: |
          cd /opt/cloud-web-app
          git pull origin main
          docker compose down --remove-orphans
          docker compose up -d --build
          sleep 10
          
          # Health checks
          curl -f http://localhost/api/health || exit 1
          curl -f http://localhost/api/users || exit 1
```

### 5. Запуск и проверка

#### 5.1 Первоначальный запуск

```bash
# На VPS
cd /opt/cloud-web-app
docker compose up -d --build

# Проверка статуса
docker compose ps

# Просмотр логов
docker compose logs -f backend
```

#### 5.2 Проверка работоспособности

```bash
# Health check
curl http://localhost/api/health

# Проверка пользователей
curl http://localhost/api/users

# Проверка через браузер
http://your-server-ip/
http://your-server-ip/api/users
```
---

### Требования к сдаче

#### GitHub репозиторий должен содержать:
- Все Dockerfile файлы
- docker-compose.yml
- Исходный код frontend/backend
- SQL скрипты инициализации БД
- GitHub Actions workflow

#### Скриншоты для демонстрации:
- Работающий frontend в браузере
- Успешный CI/CD pipeline в GitHub Actions
- Вывод `docker compose ps` на VPS
- Логи инициализации PostgreSQL

#### Рабочий деплой:
- Приложение доступно по внешнему IP
- Все эндпоинты отвечают корректно
- Автоматический деплой работает при push в main

### Возможные проблемы и решения

- **Проблема:** Контейнеры не могут подключиться друг к другу
- **Решение:** Проверить network в docker-compose.yml и настройки DNS

- **Проблема:** База данных не инициализируется
- **Решение:** Проверить монтирование init.sql и логи PostgreSQL

- **Проблема:** Nginx возвращает 502 ошибку
- **Решение:** Проверить, что backend контейнер запущен и слушает порт 3000

- **Проблема:** CI/CD пайплайн падает на health-check
- **Решение:** Увеличить sleep перед проверкой или добавить retry логику

### Дополнительные улучшения

- Добавить SSL сертификаты через Let's Encrypt.
- Настроить мониторинг и логирование.
- Добавить бэкапы базы данных.
- Реализовать стратегию blue-green деплоя.
- Настроить кэширование в Nginx.


## Срок выполнения
К 2.12.25 вы должны прислать решение.

## Полезные материалы
- [Много хорошего текста и ссылок про DevOPS](https://uproger.com/dorozhnaya-karta-devops-inzhenera-ot-middle-do-advanced/)
- [Коротко про Докер](https://habr.com/ru/companies/slurm/articles/515508/)
- [Докер (нужен будет VPN)](https://www.docker.com/)
- [Гитхаб и базовые команды)](https://github.com/Faso-main/README/blob/main/GitHub_bash.md)
- [Сыылка на диск с файлами для последнего задания)](https://drive.google.com/drive/folders/1MRLk57Df6NBK_ujR5jijt0De2wkliW_H?usp=sharing)


## Вопросы
Тут вообще все очень страшно на первый взгляд, если что, дадим виртуальный сервер и даже VPN можем, пишите.
Это задание побольше объемом, сделайте, что успеете.
По всем вопросам пишите в беседу или в личку.

