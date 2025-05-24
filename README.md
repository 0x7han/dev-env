# General Development Environment with Docker

Repository ini menyediakan **lingkungan pengembangan (dev env) universal, multi-stack, dan siap pakai** menggunakan Docker Compose.  
Lingkungan ini mendukung berbagai kombinasi project seperti web backend (PHP/Laravel, Node.js, Python, dsb), frontend (React, Vite, Next.js, dsb), hingga mobile/web Flutter — baik monorepo maupun multi-project.

---

## 🗂 Struktur Folder

```
.
├── apps/
│   ├── project-main/         # Contoh: Laravel + Vite (PHP + Node.js)
│   └── project-flutter/      # Contoh: Node.js backend + Flutter web frontend
├── docker-compose.yml
├── Dockerfile.laravel        # Untuk project Laravel (PHP + Node.js)
├── Dockerfile.flutter        # Untuk build Flutter web
├── Dockerfile.node           # (Opsional) Untuk Node.js murni
├── Caddyfile
└── README.md
```

---

## 🏗 Workflow & Perubahan

1. **General Purpose:**  
   Dev env ini tidak terbatas hanya untuk web app, tapi bisa untuk backend, frontend, bahkan mobile/web Flutter.
2. **Service utama (app_main) bisa jalan Laravel (PHP) + Node.js bersamaan** (misal Laravel + Vite).
3. **Setiap stack bisa punya Dockerfile sendiri** (`Dockerfile.laravel`, `Dockerfile.flutter`, dst).
4. **Caddy digunakan sebagai reverse proxy** untuk mengatur domain lokal setiap service (backend/frontend/misc).
5. **Semua project diletakkan di subfolder `apps/`** (bisa monorepo atau multi-project).
6. **Penambahan project stack baru tinggal tambahkan Dockerfile & service di compose, serta domain di Caddyfile.**

---

## 🐳 Contoh `docker-compose.yml`

```yaml name=docker-compose.yml
version: '3.8'

services:
  # Main Web App: Laravel + Vite (PHP + Node.js)
  app_main:
    build:
      context: .
      dockerfile: Dockerfile.laravel
    container_name: app_main
    working_dir: /app
    volumes:
      - ./apps/project-main:/app
    networks:
      - devnet
    depends_on:
      - postgres
      - redis
      - beanstalkd
      - minio
      - clickhouse

  # Node.js Backend untuk Project Flutter
  backend_node:
    image: node:20
    container_name: backend_node
    working_dir: /usr/src/app
    volumes:
      - ./apps/project-flutter/backend:/usr/src/app
    command: sh -c "npm install && npm run dev"
    networks:
      - devnet

  # Flutter Web Build
  flutter_web:
    build:
      context: .
      dockerfile: Dockerfile.flutter
    container_name: flutter_web
    working_dir: /app
    volumes:
      - ./apps/project-flutter:/app
    # Expose port jika ingin serve build result langsung
    # ports:
    #   - "8080:8080"
    networks:
      - devnet

  # --- Shared Services ---
  postgres:
    image: postgres:15
    container_name: postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: devdb
      POSTGRES_USER: devuser
      POSTGRES_PASSWORD: devpass
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - devnet

  redis:
    image: redis:7
    container_name: redis
    restart: unless-stopped
    networks:
      - devnet

  clickhouse:
    image: clickhouse/clickhouse-server:latest
    container_name: clickhouse
    restart: unless-stopped
    ports:
      - "8123:8123"
    volumes:
      - clickhouse_data:/var/lib/clickhouse
    networks:
      - devnet

  beanstalkd:
    image: schickling/beanstalkd:latest
    container_name: beanstalkd
    restart: unless-stopped
    networks:
      - devnet

  minio:
    image: minio/minio:latest
    container_name: minio
    command: server /data --console-address ":9001"
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    volumes:
      - minio_data:/data
    networks:
      - devnet

  adminer:
    image: adminer:4
    container_name: adminer
    restart: unless-stopped
    networks:
      - devnet

  # Caddy Reverse Proxy
  caddy:
    image: caddy:2
    container_name: caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config
    networks:
      - devnet
    depends_on:
      - app_main
      - backend_node
      - flutter_web
      - adminer
      - minio
      - clickhouse

networks:
  devnet:
    driver: bridge

volumes:
  postgres_data:
  clickhouse_data:
  minio_data:
  caddy_data:
  caddy_config:
```

---

## 🐘 Contoh `Dockerfile.laravel` (Laravel + Node.js)

```dockerfile name=Dockerfile.laravel
FROM dunglas/frankenphp:1.2.0-php8.2

# Install Node.js (for Vite, Laravel Mix, etc)
RUN curl -fsSL https://deb.nodesource.com/setup_20.x | bash - && \
    apt-get install -y nodejs npm && \
    npm install -g npm

WORKDIR /app

# Source code dimount via volume (tidak perlu copy di dev)

EXPOSE 80 5173 3000

CMD ["bash"]
```

---

## 🦋 Contoh `Dockerfile.flutter`

```dockerfile name=Dockerfile.flutter
FROM cirrusci/flutter:stable

WORKDIR /app

# Source code Flutter dimount via volume
# Untuk build otomatis:
RUN flutter pub get
RUN flutter build web

# Untuk serve hasil build, bisa pakai nginx, Caddy, atau container lain
```

---

## 🌐 Contoh `Caddyfile`

```
mainapp.local.dev {
    reverse_proxy app_main:80
}
nodebackend.local.dev {
    reverse_proxy backend_node:3000
}
flutterweb.local.dev {
    reverse_proxy flutter_web:8080
}
adminer.local.dev {
    reverse_proxy adminer:8080
}
minio.local.dev {
    reverse_proxy minio:9001
}
clickhouse.local.dev {
    reverse_proxy clickhouse:8123
}
```
**Edit domain/service sesuai kebutuhan project.**

---

## 💡 Menambah Project Baru (Contoh: Node.js + Flutter)

1. Buat folder project di `apps/`:
    - `apps/my-new-app/backend` (Node.js)  
    - `apps/my-new-app/frontend` (Flutter)
2. Tambahkan service baru di `docker-compose.yml`:
    - Untuk backend:  
      ```yaml
      myapp_backend:
        image: node:20
        working_dir: /usr/src/app
        volumes:
          - ./apps/my-new-app/backend:/usr/src/app
        command: sh -c "npm install && npm run dev"
        networks:
          - devnet
      ```
    - Untuk frontend (Flutter):  
      (bisa duplikasi dari `flutter_web`)
3. Tambahkan domain baru di `Caddyfile`:
    ```
    myapp-backend.local.dev {
        reverse_proxy myapp_backend:3000
    }
    myapp-frontend.local.dev {
        reverse_proxy myapp_frontend:8080
    }
    ```
4. Edit `/etc/hosts`:
    ```
    127.0.0.1 myapp-backend.local.dev
    127.0.0.1 myapp-frontend.local.dev
    ```
5. Jalankan:
    ```sh
    docker compose up -d --build
    ```

---

## 📝 ENV & Catatan

- **Semua konfigurasi ENV (seperti DB_HOST, REDIS_HOST, dsb) diatur di `.env` tiap project**.
- Gunakan service name untuk hostname DB/Redis/MinIO (e.g. `DB_HOST=postgres`, `REDIS_HOST=redis`).
- Hot reload support: file di-host akan langsung ke container via bind mount.
- Untuk produk baru/stack baru, cukup tambah Dockerfile & service.

---

## 📣 Credits

- [Caddy](https://caddyserver.com/)
- [FrankenPHP](https://frankenphp.dev/)
- [MinIO](https://min.io/)
- [ClickHouse](https://clickhouse.com/)
- [Beanstalkd](https://beanstalkd.github.io/)
- [Adminer](https://www.adminer.org/)
- [Laravel](https://laravel.com/)
- [Flutter](https://flutter.dev/)

---

Happy coding, for any project! 🚀