## 📦 Bước 1: Cài đặt Rocket.Chat với Docker

### **Chuẩn bị VPS Ubuntu 22.04:**
```bash
# Update system
apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# Install Docker Compose
apt install docker-compose -y
```

### **Deploy Rocket.Chat:**
```bash
# Tạo thư mục project
mkdir rocketchat && cd rocketchat

# Tạo docker-compose.yml
nano docker-compose.yml
```

**File docker-compose.yml:**
```yaml
volumes:
  mongodb_data: { driver: local }

services:
  rocketchat:
    image: ${IMAGE:-registry.rocket.chat/rocketchat/rocket.chat}:${RELEASE:-latest}
    restart: always
    labels:
      traefik.enable: "true"
      traefik.http.routers.rocketchat.rule: Host(`${DOMAIN:-}`)
      traefik.http.routers.rocketchat.tls: "true"
      traefik.http.routers.rocketchat.entrypoints: https
      traefik.http.routers.rocketchat.tls.certresolver: le
    environment:
      MONGO_URL: "${MONGO_URL:-\
        mongodb://${MONGODB_ADVERTISED_HOSTNAME:-mongodb}:${MONGODB_INITIAL_PRIMARY_PORT_NUMBER:-27017}/\
        ${MONGODB_DATABASE:-rocketchat}?replicaSet=${MONGODB_REPLICA_SET_NAME:-rs0}}"
      MONGO_OPLOG_URL: "${MONGO_OPLOG_URL:\
        -mongodb://${MONGODB_ADVERTISED_HOSTNAME:-mongodb}:${MONGODB_INITIAL_PRIMARY_PORT_NUMBER:-27017}/\
        local?replicaSet=${MONGODB_REPLICA_SET_NAME:-rs0}}"
      ROOT_URL: ${ROOT_URL:-http://localhost:${HOST_PORT:-3000}}
      PORT: ${PORT:-3000}
      DEPLOY_METHOD: docker
      DEPLOY_PLATFORM: ${DEPLOY_PLATFORM:-}
      REG_TOKEN: ${REG_TOKEN:-}
    depends_on:
      - mongodb
    expose:
      - ${PORT:-3000}
    ports:
      - "${BIND_IP:-0.0.0.0}:${HOST_PORT:-3000}:${PORT:-3000}"

  mongodb:
    image: docker.io/bitnami/mongodb:${MONGODB_VERSION:-6.0}
    restart: always
    volumes:
      - ${MONGODB_HOST_PATH:-mongodb_data}:/bitnami/mongodb
    environment:
      MONGODB_REPLICA_SET_MODE: primary
      MONGODB_REPLICA_SET_NAME: ${MONGODB_REPLICA_SET_NAME:-rs0}
      MONGODB_PORT_NUMBER: ${MONGODB_PORT_NUMBER:-27017}
      MONGODB_INITIAL_PRIMARY_HOST: ${MONGODB_INITIAL_PRIMARY_HOST:-mongodb}
      MONGODB_INITIAL_PRIMARY_PORT_NUMBER: ${MONGODB_INITIAL_PRIMARY_PORT_NUMBER:-27017}
      MONGODB_ADVERTISED_HOSTNAME: ${MONGODB_ADVERTISED_HOSTNAME:-mongodb}
      MONGODB_ENABLE_JOURNAL: ${MONGODB_ENABLE_JOURNAL:-true}
      ALLOW_EMPTY_PASSWORD: ${ALLOW_EMPTY_PASSWORD:-yes}
```

### **Tạo file .env**
Ở đây mình chạy rocket-chat container phía sau 1 nginx reverse proxy nên bind_ip là 127.0.0.1. Các bác thay thế chỗ BIND_IP và ROOT_URP thành ip vps và url của dùng cho Rocket-chat nhé. 
```
### Rocket.Chat configuration

# Rocket.Chat version
# see:- https://github.com/RocketChat/Rocket.Chat/releases
#RELEASE=
# MongoDB endpoint (include ?replicaSet= parameter)
#MONGO_URL=
# MongoDB endpoint to the local database
#MONGO_OPLOG_URL=
# IP to bind the process to
BIND_IP=127.0.0.1
# URL used to access your Rocket.Chat instance
ROOT_URL=https://rocket-chat.zc-idc.net
# Port Rocket.Chat runs on (in-container)
#PORT=
# Port on the host to bind to
#HOST_PORT=

### MongoDB configuration
# MongoDB version/image tag
#MONGODB_VERSION=
# See:- https://hub.docker.com/r/bitnami/mongodb

# MongoDB local host path
# chown -R 1001:1001 /data/mongodb
#MONGODB_HOST_PATH="/data/mongodb"

### Traefik config (if enabled)
# Traefik version/image tag
#TRAEFIK_RELEASE=
# Domain for https (change ROOT_URL & BIND_IP accordingly)
#DOMAIN=
# Email for certificate notifications
#LETSENCRYPT_EMAIL=
```

### **Cấu hình mẫu Nginx reverse proxy:**
```nginx
# /etc/nginx/sites-available/rocketchat
server {
    listen 443 ssl http2;
    server_name rocket-chat.zc-idc.net;

    ssl_certificate /etc/letsencrypt/live/chat.yourcompany.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/chat.yourcompany.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Nginx-Proxy true;
        proxy_redirect off;
    }
}
```



# Start Rocket.Chat
```
# Nên chạy lệnh này khi deploy lần đầu để check log
docker compose up 
```
Sau khi chạy thành công thì có thể chạy ở mode daemon
```
docker-compose up -d
```
# Check logs
```
docker-compose logs -f <container-ID>
```

## ⚙️ Bước 2: Cấu hình ban đầu

### **Setup admin account:**

```
Truy cập https://<domain>
```

## 💬 Bước 3: Setup LiveChat cho website

### **Enable Livechat:**
```bash
Administration → Omnichannel → Settings:
- Enable Omnichannel: True
- Accept new omnichannel requests: True
- Show pre-registration form: True
- Show offline form: True
```

### **Tạo LiveChat departments:**
```bash
Administration → Omnichannel → Departments:

Sales Department:
- Name: Kinh doanh
- Email: sales@domain.com
- Show on registration page: Yes

Support Department:
- Name: Hỗ trợ kỹ thuật  
- Email: support@domain.com
- Show on registration page: Yes
```

### **Add LiveChat agents:**
```bash
Administration → Omnichannel → Agents:
- Add existing users làm agents
- Assign departments
- Set max chats per agent: 3-5
```

### **Embed LiveChat vào website:**
Đây là code mẫu, để lấy đúng code embed thì vào Omnichannel -> Live chat Installation
```html
<!-- Paste vào before </body> của website -->
<script type="text/javascript">
(function(w, d, s, u) {
    w.RocketChat = function(c) { w.RocketChat._.push(c) }; w.RocketChat._ = []; w.RocketChat.url = u;
    var h = d.getElementsByTagName(s)[0], j = d.createElement(s);
    j.async = true; j.src = 'https://chat.company.com/livechat/rocketchat-livechat.min.js?_=201903270000';
    h.parentNode.insertBefore(j, h);
})(window, document, 'script', 'https://chat.company.com/livechat');
</script>

<!-- Custom styling -->
<script type="text/javascript">
RocketChat(function() {
    this.setCustomField('company', 'Your Company Name');
    this.setTheme({
        color: '#1976d2', // Brand color
        fontColor: '#fff',
        iconColor: '#1976d2'
    });
});
</script>
```
