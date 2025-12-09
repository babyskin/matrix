# Matrix Synapse with Bridges

A complete self-hosted Matrix server with WhatsApp, Telegram, Discord, and Signal bridges.

## Prerequisites

- Docker & Docker Compose
- Domain name with DNS configured
- Ports 80, 443, 8448 open
- Minimum 2GB RAM

## Quick Start

### 1. Create Directory Structure

```bash
mkdir -p /opt/synapse
cd /opt/synapse
docker network create mautrix
```

### 2. Docker Compose File

Create `docker-compose.yml`:

```yaml
services:
  synapse:
    image: docker.io/matrixdotorg/synapse:latest
    restart: unless-stopped
    volumes:
      - ./synapse-data:/data
    depends_on:
      - db
    ports:
      - '127.0.0.1:8008:8008/tcp'
  
  db:
    image: docker.io/postgres:15-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: synapse_user
      POSTGRES_PASSWORD: your_secure_password
      POSTGRES_DB: postgres
      POSTGRES_INITDB_ARGS: "--encoding='UTF8' --lc-collate='C' --lc-ctype='C'"
    volumes:
      - ./postgres-data:/var/lib/postgresql/data

  mautrix-whatsapp:
    image: dock.mau.dev/mautrix/whatsapp:latest
    container_name: mautrix-whatsapp
    restart: unless-stopped
    depends_on:
      - db
      - synapse
    volumes:
      - ./mautrix-whatsapp:/data

  mautrix-telegram:
    image: dock.mau.dev/mautrix/telegram:latest
    container_name: mautrix-telegram
    restart: unless-stopped
    depends_on:
      - db
      - synapse
    volumes:
      - ./mautrix-telegram:/data
    ports:
      - 29317:29317

  mautrix-discord:
    image: dock.mau.dev/mautrix/discord:latest
    container_name: mautrix-discord
    restart: unless-stopped
    depends_on:
      - db
      - synapse
    volumes:
      - ./mautrix-discord:/data
    ports:
      - 29334:29334

  mautrix-signal:
    image: dock.mau.dev/mautrix/signal:latest
    container_name: mautrix-signal
    restart: unless-stopped
    depends_on:
      - db
      - synapse
      - signald
    volumes:
      - ./mautrix-signal:/data
    ports:
      - 29328:29328

  signald:
    image: docker.io/signald/signald:latest
    container_name: signald
    restart: unless-stopped
    volumes:
      - ./signald:/signald

networks:
  default:
    name: mautrix
    external: true
```

### 3. Generate Synapse Configuration

```bash
docker run -it --rm \
  -v ./synapse-data:/data \
  -e SYNAPSE_SERVER_NAME=yourdomain.com \
  -e SYNAPSE_REPORT_STATS=no \
  matrixdotorg/synapse:latest generate
```

### 4. Configure Synapse Database

Edit `synapse-data/homeserver.yaml`:

```yaml
database:
  name: psycopg2
  args:
    user: synapse_user
    password: your_secure_password
    database: synapse
    host: db
    cp_min: 5
    cp_max: 10
```

### 5. Create Databases

Start PostgreSQL first:

```bash
docker compose up -d db
sleep 10

docker exec -it synapse-db-1 psql -U synapse_user -d postgres <<EOF
CREATE DATABASE synapse ENCODING 'UTF8' LC_COLLATE='C' LC_CTYPE='C' template=template0 OWNER synapse_user;
CREATE DATABASE whatsapp ENCODING 'UTF8' LC_COLLATE='C' LC_CTYPE='C' template=template0 OWNER synapse_user;
CREATE DATABASE telegram ENCODING 'UTF8' LC_COLLATE='C' LC_CTYPE='C' template=template0 OWNER synapse_user;
CREATE DATABASE discord ENCODING 'UTF8' LC_COLLATE='C' LC_CTYPE='C' template=template0 OWNER synapse_user;
CREATE DATABASE signal ENCODING 'UTF8' LC_COLLATE='C' LC_CTYPE='C' template=template0 OWNER synapse_user;
\q
EOF
```

### 6. Start Synapse

```bash
docker compose up -d synapse
```

### 7. Create Admin User

```bash
docker exec -it synapse-synapse-1 register_new_matrix_user \
  http://localhost:8008 -c /data/homeserver.yaml
```

## Nginx Configuration

Install Nginx and Certbot:

```bash
apt update
apt install nginx certbot python3-certbot-nginx -y
```

Create `/etc/nginx/sites-available/matrix`:

```nginx
server {
    listen 80;
    server_name yourdomain.com;
    
    location / {
        return 200 "OK";
    }
}
```

Enable and get SSL certificate:

```bash
ln -s /etc/nginx/sites-available/matrix /etc/nginx/sites-enabled/
nginx -t
systemctl reload nginx

# Get SSL certificate
certbot --nginx -d yourdomain.com
```

Create final Nginx configuration at `/etc/nginx/sites-available/matrix`:

```nginx
server {
    listen 80;
    server_name yourdomain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name yourdomain.com;
    
    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    
    location /.well-known/matrix/server {
        return 200 '{"m.server": "yourdomain.com:8448"}';
        default_type application/json;
        add_header Access-Control-Allow-Origin *;
    }
    
    location /.well-known/matrix/client {
        return 200 '{"m.homeserver": {"base_url": "https://yourdomain.com"}}';
        default_type application/json;
        add_header Access-Control-Allow-Origin *;
    }
    
    location /_matrix {
        proxy_pass http://127.0.0.1:8008;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_http_version 1.1;
        client_max_body_size 50M;
    }
}

server {
    listen 8448 ssl http2;
    server_name yourdomain.com;
    
    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    
    location /_matrix {
        proxy_pass http://127.0.0.1:8008;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        client_max_body_size 50M;
    }
}
```

Test and reload:

```bash
nginx -t
systemctl reload nginx
```

**Test your Matrix server:**

```bash
curl https://yourdomain.com/_matrix/client/versions
```

You should see a JSON response with Matrix versions.

## Bridge Configuration

### WhatsApp Bridge

```bash
# Generate config
docker run --rm --network=mautrix -v $PWD/mautrix-whatsapp:/data:z \
  dock.mau.dev/mautrix/whatsapp:latest

# Edit config
nano mautrix-whatsapp/config.yaml
```

Required changes:
```yaml
homeserver:
    address: https://yourdomain.com
    domain: yourdomain.com

appservice:
    address: http://mautrix-whatsapp:29318
    hostname: 0.0.0.0
    port: 29318

database:
    type: postgres
    uri: postgres://synapse_user:your_secure_password@db/whatsapp?sslmode=disable

bridge:
    permissions:
        "*": relay
        "yourdomain.com": user
        "@admin:yourdomain.com": admin
```

Generate registration and add to Synapse:

```bash
docker run --rm --network=mautrix -v $PWD/mautrix-whatsapp:/data:z \
  dock.mau.dev/mautrix/whatsapp:latest

cp mautrix-whatsapp/registration.yaml synapse-data/whatsapp-registration.yaml
chown 991:991 synapse-data/whatsapp-registration.yaml

# Add to synapse-data/homeserver.yaml:
echo "app_service_config_files:" >> synapse-data/homeserver.yaml
echo "  - /data/whatsapp-registration.yaml" >> synapse-data/homeserver.yaml

docker restart synapse-synapse-1
docker compose up -d mautrix-whatsapp
```

### Telegram Bridge

Get API credentials from https://my.telegram.org/apps

```bash
docker run --rm --network=mautrix -v $PWD/mautrix-telegram:/data:z \
  dock.mau.dev/mautrix/telegram:latest

nano mautrix-telegram/config.yaml
```

Required changes:
```yaml
homeserver:
    address: https://yourdomain.com
    domain: yourdomain.com

appservice:
    address: http://mautrix-telegram:29317
    hostname: 0.0.0.0
    port: 29317

database:
    type: postgres
    uri: postgres://synapse_user:your_secure_password@db/telegram?sslmode=disable

telegram:
    api_id: YOUR_API_ID
    api_hash: YOUR_API_HASH

bridge:
    permissions:
        "*": relaybot
        "yourdomain.com": user
        "@admin:yourdomain.com": admin
```

Finalize:

```bash
docker run --rm --network=mautrix -v $PWD/mautrix-telegram:/data:z \
  dock.mau.dev/mautrix/telegram:latest

cp mautrix-telegram/registration.yaml synapse-data/telegram-registration.yaml
chown 991:991 synapse-data/telegram-registration.yaml

# Add to homeserver.yaml
nano synapse-data/homeserver.yaml
# Add: - /data/telegram-registration.yaml

docker restart synapse-synapse-1
docker compose up -d mautrix-telegram
```

### Discord Bridge

```bash
docker run --rm --network=mautrix -v $PWD/mautrix-discord:/data:z \
  dock.mau.dev/mautrix/discord:latest

nano mautrix-discord/config.yaml
```

Required changes:
```yaml
homeserver:
    address: https://yourdomain.com
    domain: yourdomain.com

appservice:
    address: http://mautrix-discord:29334
    hostname: 0.0.0.0
    port: 29334
    
    database:
        type: postgres
        uri: postgres://synapse_user:your_secure_password@db/discord?sslmode=disable

bridge:
    permissions:
        "*": relay
        "yourdomain.com": user
        "@admin:yourdomain.com": admin
```

Finalize:

```bash
docker run --rm --network=mautrix -v $PWD/mautrix-discord:/data:z \
  dock.mau.dev/mautrix/discord:latest

cp mautrix-discord/registration.yaml synapse-data/discord-registration.yaml
chown 991:991 synapse-data/discord-registration.yaml

# Add to homeserver.yaml
nano synapse-data/homeserver.yaml
# Add: - /data/discord-registration.yaml

docker restart synapse-synapse-1
docker compose up -d mautrix-discord
```

## Using the Bridges

### WhatsApp
1. Message `@whatsappbot:yourdomain.com`
2. Send: `login`
3. Scan QR code with WhatsApp

### Telegram
1. Message `@telegrambot:yourdomain.com`
2. Send: `login`
3. Enter phone number and verification code

### Discord
1. Message `@discordbot:yourdomain.com`
2. Send: `login-qr`
3. Scan QR code with Discord mobile

### Signal
1. Message `@signalbot:yourdomain.com`
2. Send: `link`
3. Scan QR code with Signal



## Resources

- [Matrix Documentation](https://matrix.org/docs/)
- [Synapse Documentation](https://element-hq.github.io/synapse/)
- [Mautrix Bridges](https://docs.mau.fi/)
- [Element Client](https://element.io/)
