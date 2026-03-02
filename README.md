<<<<<<< HEAD
# rk_proekt
=======
# Гүлдерді онлайн тапсырыс беру

Қазақстан бойынша гүлдерді онлайн тапсырыс беру қызметі. Full-stack жоба: NestJS backend, Next.js frontend, PostgreSQL.

## Дереккөздер

- **[data.egov.kz](https://data.egov.kz)** – Қазақстан Республикасының ашық деректер порталы. Қалалар/әкімшілік бірліктер тізімін алуға болады.
- **[egov.kz](https://egov.kz)** – Ресми мемлекеттік портал, Open Data бөлімі data.egov.kz-ке сілтеме жасайды.

## Құрылым

```
├── apps/
│   ├── backend/     # NestJS API
│   └── web/         # Next.js frontend
├── docker-compose.yml
├── docker-compose.prod.yml
└── README.md
```

## Технологиялар

- **Backend:** NestJS, Prisma, PostgreSQL, JWT, Swagger, class-validator
- **Frontend:** Next.js 16, React 19, TailwindCSS, react-hook-form, zod
- **DB:** PostgreSQL 16, Prisma ORM

## Жергілікті іске қосу

### 1. PostgreSQL-ді іске қосу

```bash
docker compose up -d
```

Бұл postgres контейнерін 5433 портында іске қосады.

### 2. Backend орнату және іске қосу

```bash
cd apps/backend
cp .env.example .env
# .env-ті тексеріңіз: DATABASE_URL localhost:5433
npm install
npx prisma migrate dev --name init   # Бірінші рет
npm run db:seed                     # Тест деректерін қосу
npm run start:dev
```

Backend: http://localhost:3001  
Swagger: http://localhost:3001/api/docs

### 3. Frontend іске қосу

```bash
cd apps/web
cp .env.example .env.local
# NEXT_PUBLIC_API_URL=http://localhost:3001
npm install
npm run dev
```

Frontend: http://localhost:3000

### Түйінді деректер (seed)

- **Admin:** admin@example.kz / Admin123! – **өндірісте міндетті түрде өзгертіңіз!**
- **Қолданушылар:** user1@example.kz … user1500@example.kz / User123!
- **Қалалар:** prisma/seed-data/cities-kz.json (Қазақстан қалалары)

---

## Production деплой (Ubuntu VPS)

### Алғышарттар

- Ubuntu 22.04/24.04 VPS
- Git, Docker, Docker Compose орнатылған
- Домен (мысалы: gulder.example.kz)

### 1. Репозиторийді клондау

```bash
git clone <your-repo-url> gulder
cd gulder
```

### 2. Өңделген ортадағы айнымалыларды орнату

`apps/backend/.env`:

```env
DATABASE_URL="postgresql://postgres:КҮШТІ_ҚҰПИЯ_СӨЗ@postgres:5432/gulder_db"
JWT_SECRET="өте-ұзын-және-кепілді-жеке-кілік"
JWT_EXPIRES_IN="7d"
PORT=3001
NODE_ENV=production
```

Жоба түбінде `.env` (docker-compose.prod үшін):

```env
POSTGRES_USER=postgres
POSTGRES_PASSWORD=КҮШТІ_ҚҰПИЯ_СӨЗ
POSTGRES_DB=gulder_db
JWT_SECRET=өте-ұзын-және-кепілді-жеке-кілік
JWT_EXPIRES_IN=7d
NEXT_PUBLIC_API_URL=https://gulder.example.kz/api
```

### 3a. Docker Compose арқылы (nginx контейнерімен)

Жобада `docker/nginx.conf` және `docker-compose.prod.yml` бар. Nginx арқылы:

```bash
docker compose -f docker-compose.prod.yml up -d
```

Порт 80 арқылы кіруге болады.

### 3b. Жүйелік Nginx орнату (HTTPS үшін)

```bash
sudo apt update
sudo apt install nginx -y
```

Доменіңізді `docker/nginx.conf`-та `server_name`-ге қойыңыз, немесе VPS-та жеке Nginx орнатып, `/api`-ны backend:3001-ге, қалғанын frontend:3000-ге бағыттаңыз.

`/etc/nginx/sites-available/gulder` (жүйелік Nginx кезінде):

```nginx
server {
    listen 80;
    server_name gulder.example.kz;

    location /api {
        proxy_pass http://127.0.0.1:3001;  # docker ports: 3001:3001 болғанда
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/gulder /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

### 4. Docker Compose арқылы іске қосу

```bash
# Миграция үшін алдымен postgres іске қосылады
docker compose -f docker-compose.prod.yml up -d postgres
sleep 10

# Қалған сервистер
docker compose -f docker-compose.prod.yml up -d
```

### 5. HTTPS (Let's Encrypt)

```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d gulder.example.kz
```

### 6. Қауіпсіздік

- **ufw:** `sudo ufw allow 80,443,22 && sudo ufw enable`
- **PostgreSQL:** Сыртқа порт ашылмайды (тек Docker желісінде)
- **Admin пароль:** admin@example.kz парольін міндетті түрде өзгертіңіз

### 7. Seed (бірінші іске қосу кезінде)

```bash
cd apps/backend
npm run db:seed
```

---

## API

- **Health:** GET /health
- **Swagger:** GET /api/docs
- **Auth:** POST /api/auth/login, POST /api/auth/register
- **Products:** GET /api/products, GET /api/products/featured, GET /api/products/:slug
- **Categories:** GET /api/categories
- **Delivery:** GET /api/delivery/cities
- **Orders:** POST /api/orders, GET /api/orders/:id
- **Admin:** /api/admin/* (JWT, ADMIN/MANAGER рөлі қажет)
- **Analytics:** /api/analytics (JWT, ADMIN/MANAGER)

---

## Барлық интерфейс қазақ тілінде

Сайт, админка, қате хабарламалар, валидация мәтіндері – бәрі қазақ тілінде. Тек техникалық код/айнымалылар ағылшынша.
>>>>>>> c6eb30e (Initial commit)
