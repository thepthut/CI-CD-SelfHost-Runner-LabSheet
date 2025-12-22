# ใบงาน: การ Deploy แอปพลิเคชันด้วย GitHub Actions และ Self-Hosted Runner
## วัตถุประสงค์

1. เพื่อเข้าใจแนวคิดและหลักการทำงานของ Self-Hosted Runner แบบ Pull-based Model
2. เพื่อสามารถติดตั้งและกำหนดค่า Self-Hosted Runner บนเครื่อง local
3. เพื่อเข้าใจกระบวนการ Polling และการสื่อสารระหว่าง Runner กับ GitHub
4. เพื่อสร้าง CI/CD Pipeline สำหรับ Deploy แอปพลิเคชันไปยัง on-premise server
5. เพื่อเรียนรู้การตั้งค่า Reverse Proxy ด้วย Nginx สำหรับ Production Environment

## ทฤษฎีที่เกี่ยวข้อง
### 1. Self-Hosted Runner คืออะไร
    Self-Hosted Runner คือเครื่อง server ที่เราติดตั้งและดูแลเอง ซึ่งทำหน้าที่รัน GitHub Actions workflows โดยใช้กลไก Pull-based (Polling) ในการรับงานจาก GitHub แทนที่จะใช้ GitHub-hosted runners (Cloud Runner ของ GitHub)
### จุดเด่นของ Pull-based Model:
- Runner เป็นฝ่าย ดึง (Pull) งานจาก GitHub ไม่ใช่ GitHub ส่ง (Push) งานมา
- Runner ทำการ Polling (ตรวจสอบเป็นระยะ) ไปที่ GitHub API
- ไม่ต้องเปิด Port ให้โลกภายนอกเข้าถึง
- ไม่ต้องมี Static IP Address

### 2. สถาปัตยกรรมการทำงาน (Architecture)
```txt
┌─────────────────────────────────────────────────────────────────────┐
│                        GitHub Cloud Platform                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────┐      ┌───────────────┐      ┌─────────────────┐   │
│  │  Repository  │      │    Actions    │      │   Job Queue     │   │
│  │   (Code)     │─────>│   Workflow    │─────>│ (Pending Jobs)  │   │
│  └──────────────┘      └───────────────┘      └─────────────────┘   │
│                                                         ▲           │
│                                                         │           │
└─────────────────────────────────────────────────────────┼─────────-─┘
                                                          │
                                                          │
                           Firewall (No Inbound Rules)    │
                           ═══════════════════════════════│═══
                                                          │
                                      HTTPS Polling       │
                                   (Outbound Connection)  │
                                                          │
                              1. "Any jobs for me?"       │
                          ┌───────────────────────────────┘
                          │
                          │  2. Response: Job Details
                          │     or "No jobs yet"
                          ▼
                  ┌─────────────────────┐
                  │   Self-Hosted       │ ← Runs on Your Local Machine
                  │      Runner         │   (Windows/Mac/Linux)
                  │   (Agent Process)   │
                  └─────────────────────┘
                          │
                          │ 3. Clone Repository
                          │ 4. Execute Steps
                          │ 5. Report Status
                          ▼
                  ┌─────────────────────┐
                  │  Local Deployment   │
                  │  Docker Compose     │
                  │  ├── App Container  │
                  │  └── Nginx Proxy    │
                  └─────────────────────┘
```
### 3. ขั้นตอนการทำงานโดยละเอียด
**ขั้นตอนที่ 1: Developer Push Code**

```txt
Developer → git push → GitHub Repository
```

- นักพัฒนา push code ขึ้น GitHub Repository (เช่น branch **main**)

**ขั้นตอนที่ 2: Workflow Triggered**
```
GitHub → Detect Push Event → Create Workflow Run → Generate Job
```

- GitHub ตรวจจับเหตุการณ์ (push, pull request, schedule)
- สร้าง Workflow Run ตามไฟล์ **.github/workflows/*.yml**
- สร้าง Job และเก็บไว้ใน Job Queue
- Job จะมี metadata: repository URL, branch, commit SHA, steps ที่ต้องทำ

**ขั้นตอนที่ 3: Runner Polling Loop**
```
Runner → HTTPS GET → GitHub API → Poll for Jobs
                                      ↓
                              Check Job Queue
                                      ↓
                         Match with Runner Labels
```
**Runner ทำงานเป็น Loop:**
```
javascript// Simplified Polling Logic
while (runner.isActive) {
  // ส่ง request ไปที่ GitHub API ทุก 1-2 วินาที
  const response = await fetch('https://api.github.com/actions/v1/jobs', {
    method: 'GET',
    headers: {
      'Authorization': `Bearer ${RUNNER_TOKEN}`,
      'Accept': 'application/json'
    },
    body: JSON.stringify({
      runnerId: 'runner-12345',
      runnerName: 'my-local-runner',
      labels: ['self-hosted', 'macOS', 'X64'],
      timeout: 60  // Long-polling timeout
    })
  });
  
  if (response.hasJob) {
    await executeJob(response.job);
  }
  
  // ถ้าไม่มี job ก็ polling ต่อ
}
```

**Long-Polling Technique:**
- Runner เปิด HTTP connection และรอ (block) สูงสุด 60 วินาที
- ถ้ามี job ใหม่ GitHub จะ respond ทันที
- ถ้าไม่มี job ใน 60 วินาที GitHub จะ respond "no jobs" แล้ว Runner polling ใหม่
- ทำให้ได้รับ job แทบจะทันทีโดยไม่ต้องส่ง request บ่อยเกินไป

#### ขั้นตอนที่ 4: Job Assignment (Response from GitHub)
```
GitHub API → HTTP Response → Job Details (JSON)
```
**ตัวอย่าง Response:**
```
json{
  "jobId": "job_abc123",
  "repositoryUrl": "https://github.com/user/nodejs-cicd-selfhosted",
  "ref": "refs/heads/main",
  "sha": "a1b2c3d4e5f6...",
  "workflowFile": ".github/workflows/deploy.yml",
  "steps": [
    {
      "id": "step_1",
      "name": "Checkout Code",
      "uses": "actions/checkout@v4"
    },
    {
      "id": "step_2",
      "name": "Build",
      "run": "docker-compose build"
    }
  ],
  "secrets": {
    "DATABASE_URL": "***encrypted***"
  }
}
```

#### ขั้นตอนที่ 5: Execute on Local Machine

**Runner Process:**
  1. Clone repository
  2. Checkout specific commit (SHA)
  3. Set environment variables
  4. Execute each step sequentially
  5. Stream logs back to GitHub
**Runner ทำงานบนเครื่อง local:**

```bash
# 1. Clone repository to _work directory
git clone https://github.com/user/nodejs-cicd-selfhosted.git
cd nodejs-cicd-selfhosted
git checkout a1b2c3d4e5f6

# 2. Execute steps
docker-compose down
docker-compose build --no-cache
docker-compose up -d

# 3. Verify deployment
curl http://localhost:8080/health
```

#### ขั้นตอนที่ 6: Real-time Status Reporting
```
Runner → HTTPS POST → GitHub API → Update Job Status
```
**Runner ส่ง updates ตลอดเวลา:**

- เมื่อเริ่ม step ใหม่
- ขณะรัน step (streaming logs)
- เมื่อ step เสร็จ (success/failure)
- เมื่อ job เสร็จทั้งหมด

```javascript
// Example status updates
await github.updateJobStatus({
  jobId: 'job_abc123',
  stepId: 'step_1',
  status: 'in_progress',
  logs: 'Cloning repository...'
});

await github.updateJobStatus({
  jobId: 'job_abc123',
  stepId: 'step_1',
  status: 'completed',
  conclusion: 'success'
});
```

#### ขั้นตอนที่ 7: Service Running
```
Docker Compose → Start Containers → Application Running
                                            ↓
                              http://localhost:8080
```

### 4. ทำไม Pull-based Model ปลอดภัยกว่า Push-based

#### เปรียบเทียบ 2 Models:

**❌ Push-based Model (ไม่ปลอดภัย):**
```
GitHub → Send Job → Internet → Your Firewall → Port 22/8080 → Runner
```
**ปัญหา:**
- ต้องเปิด Port ให้โลกภายนอกเข้าถึง
- ต้องมี Static IP Address
- ต้อง Configure Port Forwarding บน Router
- ต้อง Whitelist GitHub IP ranges
- มี Attack Surface กว้าง
- ถ้า credentials รั่วไหล ใครก็เข้ามาได้


**✅ Pull-based Model (ปลอดภัย):**
```
Runner → Poll Jobs → Internet → GitHub API
         (Outbound)
```
**ข้อดี:**
- ไม่ต้องเปิด Port inbound
- ไม่ต้องมี Static IP
- ไม่ต้อง Configure Router
- Firewall ส่วนใหญ่อนุญาต outbound traffic อยู่แล้ว
- แม้ GitHub ถูก compromise ก็ไม่สามารถส่งคำสั่งมาได้โดยตรง
- Runner เป็นฝ่ายเลือกว่าจะรับ job ไหน

#### ตัวอย่างการโจมตีที่ป้องกันได้:

**Scenario 1: Attacker พยายาม Push Malicious Job**
```
Push Model (Vulnerable):
Attacker → Forge Request → Your IP:Port → Runner Execute
                                             ↓
                                      System Compromised ❌
```
**Pull Model (Safe):**
```
Attacker → Cannot Connect → Runner behind Firewall ✅
Runner → Only Polls from Trusted GitHub API ✅
```

**Scenario 2: GitHub Account Compromised**

**Push Model:**
```
Attacker with GitHub Access → Push Malicious Job → Force Execute ❌
```
**Pull Model:**
```
Attacker with GitHub Access → Push Malicious Job → Queue
                                                       ↓
Runner → Polls → Sees Malicious Job → Can be configured to reject ✅
Admin → Can stop Runner before it pulls the job ✅
```

### 5. ความแตกต่างระหว่าง GitHub-Hosted และ Self-Hosted Runner

| Feature | GitHub-Hosted Runner | Self-Hosted Runner |
|---------|---------------------|-------------------|
| **การดูแลรักษา** | GitHub ดูแลให้ | ต้องดูแลเอง |
| **Connection Model** | N/A (GitHub's infrastructure) | **Pull-based (Polling)** |
| **Network Requirements** | N/A | Only outbound HTTPS |
| **Firewall Configuration** | N/A | No inbound rules needed |
| **Static IP Required** | No | **No (Dynamic IP OK)** |
| **Port Opening** | N/A | **None required** |
| **ความปลอดภัย** | Isolated environment | ขึ้นกับการตั้งค่า |
| **เข้าถึง Local Resources** | ไม่ได้ | ได้ (Database, Files) |
| **ค่าใช้จ่าย** | มีโควต้า/เสียเงิน | ใช้ทรัพยากรของเรา |
| **ความเร็ว** | ขึ้นกับ network | เร็วกว่า (local) |
| **Public Repository** | ใช้ได้ปลอดภัย | **ไม่แนะนำเด็ดขาด** |

### 6. Communication Flow แบบละเอียด
```
Sequence Diagram:

Developer    GitHub         Job Queue      Runner          Local Server
   │            │                │            │                  │
   │──push─────>│                │            │                  │
   │            │                │            │                  │
   │            │──create job───>│            │                  │
   │            │                │            │                  │
   │            │                │<───poll────│                  │
   │            │                │            │                  │
   │            │                │──job info─>│                  │
   │            │                │            │                  │
   │            │<───────status update────────│                  │
   │            │                │            │                  │
   │            │                │            │──clone repo─────>│
   │            │                │            │                  │
   │            │                │            │──build & deploy─>│
   │            │                │            │                  │
   │            │<───────status update────────│                  │
   │            │                │            │                  │
   │            │                │            │<──health check───│
   │            │                │            │                  │
   │            │<───────final status─────────│                  │
   │            │                │            │                  │
   │            │                │<───poll────│ (continues...)   │
```

### 7. Docker Compose และ Multi-Container Application

**Docker Compose** เป็นเครื่องมือสำหรับกำหนดและรัน multi-container Docker application โดยใช้ไฟล์ YAML เดียว

**ข้อดี:**
- จัดการหลาย containers พร้อมกันด้วยคำสั่งเดียว
- กำหนด network และ volume ได้ง่าย
- Reproducible environment
- Version control สำหรับ infrastructure

### 8. Reverse Proxy Pattern

**Reverse Proxy (Nginx)** คือ server ที่อยู่ระหว่าง client และ application:
```
Internet User → Nginx (Port 80/443) → Application (Port 3000)
                  ↑
              Front Door
         (SSL, Caching, Security)
```
**ประโยชน์:**
- SSL/TLS termination
- Load balancing
- Static file serving
- Security headers
- Hide internal architecture

**อุปกรณ์และเครื่องมือที่ใช้**
**Software Requirements**

1. Docker Desktop Version: 4.25.0 or later 
2. Git Version: 2.40.0 or later
3. Text Editor/IDE
4. Web Browser
5. GitHub Account **ต้องเป็น Private Repository (สำคัญมาก!)**

**Hardware Requirements**

- **RAM:** อย่างน้อย 8 GB (แนะนำ 16 GB)
- **Storage:** พื้นที่ว่างอย่างน้อย 10 GB
- **CPU:** 2 cores ขึ้นไป (แนะนำ 4 cores)
- **Network:** เชื่อมต่อ Internet (ความเร็วอย่างน้อย 10 Mbps)

**Network Requirements**
**สำหรับ Self-Hosted Runner:**
- **Outbound HTTPS (Port 443):** ต้องสามารถเชื่อมต่อไปยัง ```*.github.com```, ```*.githubusercontent.com```
- **No Inbound Ports Required:** ไม่ต้องเปิด port รับจากภายนอก
- **Firewall:** อนุญาต outbound traffic (default ของ firewall ส่วนใหญ่)
- **Proxy Support:** หากมี corporate proxy สามารถ configure ได้

### ขั้นตอนการทดลอง
#### ส่วนที่ 1: เตรียม Project และ Repository
#### 1.1 สร้าง GitHub Repository (Private)

⚠️ **คำเตือนสำคัญ:** ต้องเลือก **Private Repository** เท่านั้น เพราะ Self-Hosted Runner ไม่ปลอดภัยกับ Public Repository


1. เข้าสู่ GitHub และสร้าง repository ใหม่
- ไปที่ https://github.com/new
- Repository name: ```nodejs-cicd-selfhosted```
- Description: ```CI/CD with Self-Hosted Runner Demo - Pull-based Model```
- **เลือก "Private"** ⚠️ (สำคัญมาก!)
- ✅ เลือก "Add a README file"
- คลิก Create repository

2. Clone repository มาที่เครื่อง local

```bash
# เปิด Terminal (macOS/Linux) หรือ PowerShell (Windows)
cd ~/Documents  # หรือ folder ที่ต้องการ
git clone https://github.com/YOUR_USERNAME/nodejs-cicd-selfhosted.git
cd nodejs-cicd-selfhosted
```
#### 1.2 สร้างโครงสร้าง Project

1. สร้างไฟล์ ```package.json```

```json
{
  "name": "nodejs-cicd-selfhosted",
  "version": "1.0.0",
  "description": "CI/CD Demo with Self-Hosted Runner (Pull-based Model)",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "keywords": ["cicd", "docker", "self-hosted", "github-actions"],
  "author": "",
  "license": "MIT",
  "dependencies": {
    "express": "^4.18.2",
    "dotenv": "^16.3.1"
  },
  "devDependencies": {
    "nodemon": "^3.0.2"
  }
}
```
2. สร้างไฟล์ ```server.js```

```javascript
const express = require('express');
const app = express();
require('dotenv').config();

const PORT = process.env.PORT || 3000;
const VERSION = process.env.VERSION || '1.0.0';
const HOSTNAME = require('os').hostname();

// Middleware
app.use(express.json());

// Root endpoint
app.get('/', (req, res) => {
  res.json({
    message: '🚀 Hello from CI/CD Demo with Self-Hosted Runner!',
    model: 'Pull-based (Polling)',
    version: VERSION,
    hostname: HOSTNAME,
    timestamp: new Date().toISOString(),
    environment: process.env.NODE_ENV || 'development',
    uptime: `${Math.floor(process.uptime())} seconds`
  });
});

// Health check endpoint
app.get('/health', (req, res) => {
  res.json({
    status: 'healthy',
    uptime: process.uptime(),
    timestamp: new Date().toISOString(),
    memoryUsage: process.memoryUsage(),
    version: VERSION
  });
});

// Info endpoint
app.get('/info', (req, res) => {
  res.json({
    nodejs: process.version,
    platform: process.platform,
    arch: process.arch,
    hostname: HOSTNAME,
    environment: process.env.NODE_ENV,
    version: VERSION
  });
});

// 404 handler
app.use((req, res) => {
  res.status(404).json({
    error: 'Not Found',
    path: req.path,
    method: req.method
  });
});

// Error handler
app.use((err, req, res, next) => {
  console.error('Error:', err);
  res.status(500).json({
    error: 'Internal Server Error',
    message: err.message
  });
});

// Start server
app.listen(PORT, '0.0.0.0', () => {
  console.log('═══════════════════════════════════════');
  console.log('🚀 Server started successfully!');
  console.log('═══════════════════════════════════════');
  console.log(`📦 Version: ${VERSION}`);
  console.log(`🌍 Environment: ${process.env.NODE_ENV || 'development'}`);
  console.log(`🖥️  Hostname: ${HOSTNAME}`);
  console.log(`🔗 URL: http://localhost:${PORT}`);
  console.log(`❤️  Health: http://localhost:${PORT}/health`);
  console.log(`ℹ️  Info: http://localhost:${PORT}/info`);
  console.log('═══════════════════════════════════════');
});

// Graceful shutdown
process.on('SIGTERM', () => {
  console.log('SIGTERM received, shutting down gracefully...');
  process.exit(0);
});

process.on('SIGINT', () => {
  console.log('SIGINT received, shutting down gracefully...');
  process.exit(0);
});
```
3. สร้างไฟล์ ```.env```

```bash
PORT=3000
VERSION=1.0.0
NODE_ENV=production
```
4. สร้างไฟล์ ```.gitignore```

```bash
# Dependencies
node_modules/

# Environment
.env
.env.local
.env.*.local

# Logs
*.log
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# OS
.DS_Store
Thumbs.db

# IDE
.vscode/
.idea/
*.swp
*.swo

# Docker
.docker/

# Build
dist/
build/
```
#### 1.3 สร้าง Dockerfile (Multi-stage Build)
**สร้างไฟล์** ```Dockerfile```
```dockerfile
# ═══════════════════════════════════════════════════════════
# Stage 1: Dependencies
# ═══════════════════════════════════════════════════════════
FROM node:18-alpine AS dependencies

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install production dependencies only
RUN npm ci --only=production && \
    npm cache clean --force

# ═══════════════════════════════════════════════════════════
# Stage 2: Production
# ═══════════════════════════════════════════════════════════
FROM node:18-alpine AS production

# Add metadata
LABEL maintainer="your-email@example.com"
LABEL description="Node.js App with Self-Hosted Runner CI/CD"
LABEL version="1.0.0"

WORKDIR /app

# Copy dependencies from builder stage
COPY --from=dependencies /app/node_modules ./node_modules

# Copy application code
COPY . .

# Create non-root user for security
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001 && \
    chown -R nodejs:nodejs /app

# Switch to non-root user
USER nodejs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"

# Start application
CMD ["node", "server.js"]
```

#### 1.4 สร้าง Docker Compose Configuration
**สร้างไฟล์** ```docker-compose.yml```
```yaml
version: '3.8'

services:
  # Application Service
  app:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        NODE_ENV: production
    container_name: nodejs-selfhosted-app
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - PORT=3000
      - VERSION=${VERSION:-1.0.0}
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "node", "-e", "require('http').get('http://localhost:3000/health', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # Nginx Reverse Proxy
  nginx:
    image: nginx:alpine
    container_name: nginx-selfhosted-proxy
    restart: unless-stopped
    ports:
      - "8080:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      app:
        condition: service_healthy
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

networks:
  app-network:
    driver: bridge
    name: selfhosted-network
```

#### b1.5 สร้าง Nginx Configuration
**สร้างไฟล์** ```nginx.conf```
```nginx
# Global settings
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
    use epoll;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Logging format
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    # Performance
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml text/javascript 
               application/json application/javascript application/xml+rss;

    # Upstream backend
    upstream app_backend {
        server app:3000 max_fails=3 fail_timeout=30s;
        keepalive 32;
    }

    # Server block
    server {
        listen 80;
        server_name localhost;

        # Security headers
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-XSS-Protection "1; mode=block" always;
        add_header Referrer-Policy "strict-origin-when-cross-origin" always;

        # Main location
        location / {
            proxy_pass http://app_backend;
            proxy_http_version 1.1;
            
            # WebSocket support
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            
            # Proxy headers
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Forwarded-Host $host;
            proxy_set_header X-Forwarded-Port $server_port;
            
            # Timeouts
            proxy_connect_timeout 60s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;
            
            # Buffering
            proxy_buffering on;
            proxy_buffer_size 4k;
            proxy_buffers 8 4k;
            proxy_busy_buffers_size 8k;
            
            # Cache bypass
            proxy_cache_bypass $http_upgrade;
        }

        # Health check endpoint (no logging)
        location /health {
            proxy_pass http://app_backend/health;
            access_log off;
            
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_connect_timeout 5s;
            proxy_send_timeout 5s;
            proxy_read_timeout 5s;
        }

        # Info endpoint
        location /info {
            proxy_pass http://app_backend/info;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
        }

        # Error pages
        error_page 404 /404.html;
        error_page 500 502 503 504 /50x.html;
        
        location = /50x.html {
            return 500 '{"error": "Backend service unavailable"}';
            add_header Content-Type application/json;
        }
    }
}
```
#### 1.6 สร้างไฟล์ README.md
**สร้างไฟล์** ```README.md```
```markdown
# Node.js CI/CD with Self-Hosted Runner

Demo project สำหรับเรียนรู้การใช้ GitHub Actions Self-Hosted Runner แบบ Pull-based Model

## 🏗️ Architecture

Runner (Local) → Polling → GitHub API → Pull Jobs → Execute Locally

## 🚀 Features

- Pull-based Self-Hosted Runner (ไม่ต้องเปิด port)
- Docker Compose สำหรับ Multi-container
- Nginx Reverse Proxy
- Health Checks
- Automated Deployment

## 📦 Requirements

- Docker Desktop
- GitHub Account (Private Repository)
- 8GB RAM minimum

## 🔧 Local Development
```bash
# Install dependencies
npm install

# Run locally
npm start

# Access application
http://localhost:3000
```

## 🐳 Docker Compose
```bash
# Build and run
docker-compose up -d

# View logs
docker-compose logs -f

# Stop
docker-compose down
```

## 🔐 Security Notes

- ⚠️ Use only with **Private Repository**
- Runner uses **Pull-based model** (no inbound ports)
- Secrets stored in GitHub Secrets
- Non-root user in container

## 📚 Learn More

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Self-Hosted Runners](https://docs.github.com/en/actions/hosting-your-own-runners)

#### 1.7 Commit และ Push Code ครั้งแรก
```bash
# Stage all files
git add .

# Commit
git commit -m "Initial project setup with pull-based architecture"

# Push to GitHub
git push origin main
```
**ส่วนที่ 2: ติดตั้ง Self-Hosted Runner**

💡 หมายเหตุ: Runner จะใช้ Pull-based model ทำการ Polling ไปยัง GitHub API เพื่อรับงาน ไม่ใช่ GitHub Push งานมา

**2.1 เข้าสู่การตั้งค่า Runner**
1. ไปที่ GitHub repository ของคุณ
2. คลิก Settings (แท็บด้านบน)
3. ในเมนูด้านซ้าย เลือก Actions → Runners
4. คลิกปุ่ม New self-hosted runner
5. เลือก Operating System:
    - macOS: สำหรับ Mac
    - Linux: สำหรับ Windows ที่ใช้ WSL
    - Windows: สำหรับ Windows แบบ native

**2.2 ติดตั้งบน macOS**
1. เปิด Terminal:

```bash
# สร้าง folder สำหรับ runner
mkdir -p ~/actions-runner && cd ~/actions-runner
```
2. Download runner package (คัดลอกคำสั่งจากหน้า GitHub - version อาจแตกต่างจากตัวอย่าง):

```bash
# Download (ดูเวอร์ชันล่าสุดจากหน้า GitHub)
curl -o actions-runner-osx-x64-2.311.0.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-osx-x64-2.311.0.tar.gz

# Optional: Validate the hash (คัดลอก hash จาก GitHub)
echo "EXPECTED_SHA256_HASH  actions-runner-osx-x64-2.311.0.tar.gz" | shasum -a 256 -c

# Extract
tar xzf ./actions-runner-osx-x64-2.311.0.tar.gz
```
3. Configure runner (คัดลอกคำสั่งจาก GitHub ซึ่งจะมี token แนบมาด้วย):

```bash
# คำสั่งจะคล้ายนี้ (token จะแตกต่างกัน)
./config.sh --url https://github.com/YOUR_USERNAME/nodejs-cicd-selfhosted \
            --token YOUR_REGISTRATION_TOKEN

# ===============================================
# การตอบคำถามระหว่าง Configuration:
# ===============================================

# Enter the name of the runner [default: hostname]:
# → กด Enter (ใช้ชื่อเดิม) หรือตั้งชื่อ เช่น "my-macbook-runner"

# Enter any additional labels (ex. label-1,label-2):
# → กด Enter (ไม่ต้องใส่ label เพิ่ม)

# Enter name of work folder [default: _work]:
# → กด Enter (ใช้ค่า default)

# ===============================================
# Output ที่ควรเห็น:
# ===============================================
# √ Runner successfully added
# √ Runner connection is good
# √ Settings Saved.
```
💡 **สิ่งที่เกิดขึ้นหลัง config:**
- Runner ลงทะเบียนกับ GitHub โดยใช้ token
- สร้าง credentials ใน .credentials file (encrypted)
- สร้าง work folder _work สำหรับ clone repositories
- Runner พร้อมเริ่ม polling jobs

4. รัน Runner (มี 2 วิธี):
**วิธีที่ 1: รันแบบ Interactive (สำหรับทดสอบ)**
```bash
# รันโดยตรง (เห็น output real-time)
./run.sh

# Output ที่ควรเห็น:
# √ Connected to GitHub
# Listening for Jobs
# 
# [รอ polling...]
```
**วิธีที่ 2: รันเป็น Background Service (แนะนำ)**
```bash
# ติดตั้งเป็น service
sudo ./svc.sh install

# เริ่มต้น service
sudo ./svc.sh start

# ตรวจสอบสถานะ
sudo ./svc.sh status

# Output:
# status: active (running) ✅
```
**ดู Logs ของ Runner:**
```bash
# Logs ของ runner เก็บอยู่ที่
tail -f _diag/Runner_*.log

# ควรเห็นข้อความประมาณนี้:
# [2024-01-15 10:00:00Z] Runner is online
# [2024-01-15 10:00:01Z] Listening for Jobs...
# [2024-01-15 10:00:02Z] Polling GitHub API...
```
**2.3 ติดตั้งบน Windows (WSL)**

1. เปิด PowerShell หรือ Windows Terminal (**Run as Administrator**)
2. ติดตั้ง/เปิดใช้งาน WSL:

```powershell
# ตรวจสอบว่ามี WSL หรือไม่
wsl --list

# ถ้ายังไม่มี ให้ติดตั้ง
wsl --install

# รีสตาร์ทเครื่อง
```
3. เปิด WSL (Ubuntu) terminal:

```bash
# สร้าง folder
mkdir -p ~/actions-runner && cd ~/actions-runner
```
4. Download และ extract:
```bash
# Download (version อาจแตกต่าง - ดูจาก GitHub)
curl -o actions-runner-linux-x64-2.311.0.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-linux-x64-2.311.0.tar.gz

# Validate (optional)
echo "EXPECTED_SHA256  actions-runner-linux-x64-2.311.0.tar.gz" | shasum -a 256 -c

# Extract
tar xzf ./actions-runner-linux-x64-2.311.0.tar.gz
```
5. Configure:
```bash
# คัดลอกคำสั่งจาก GitHub (มี token)
./config.sh --url https://github.com/YOUR_USERNAME/nodejs-cicd-selfhosted \
            --token YOUR_TOKEN

# ตอบคำถามเหมือนใน macOS
```
6. รันเป็น service:
```bash
# ติดตั้ง
sudo ./svc.sh install

# เริ่มต้น
sudo ./svc.sh start

# ตรวจสอบ
sudo ./svc.sh status
```
**ติดตั้ง Dependencies สำหรับ WSL:**
```bash
# อัปเดต packages
sudo apt update && sudo apt upgrade -y

# ติดตั้ง dependencies ที่จำเป็น
sudo apt install -y curl git wget

# ตรวจสอบว่า Docker Desktop for Windows เปิด WSL integration
```
#### 2.4 ตรวจสอบว่า Runner Online และ Polling

1. **ตรวจสอบใน GitHub UI:**
- ไปที่ **Settings → Actions → Runners**
- จะเห็น runner แสดงด้วย:
    - สถานะสีเขียว (Idle) = พร้อมรับงาน ✅
    - สถานะสีเทา (Offline) = ยังไม่ online ❌
- ถ้าเป็นสีเขียว แสดงว่า runner กำลัง polling อยู่
  
  ### บันทึกรูปผลการทดลอง
  ```
  บันทึกรูปหน้า Runners โดยคัดลอกให้เห็น Account และ Repository
  ```
  
2. ตรวจสอบจาก Logs:
```bash
# ดู runner logs
cd ~/actions-runner
tail -f _diag/Runner_*.log

# ควรเห็นข้อความประมาณ:
# [timestamp] Runner.Listener: Runner is online
# [timestamp] Runner.Worker: Listening for Jobs...
# [timestamp] Runner.Worker: Polling for jobs...
# [timestamp] Runner.Worker: No jobs available, waiting...
```

3. **ทดสอบ Polling Cycle:**
เมื่อ runner ทำงาน มันจะ:
- Poll ทุก 1-2 วินาที (long-polling)
- ถ้าไม่มี job จะเห็น log "waiting for jobs"
- ถ้ามี job ใหม่ จะเห็น log "job assigned"

#### 2.5 โครงสร้าง Runner Directory Structure
```
~/actions-runner/
├── _diag/                    # Diagnostic logs
│   ├── Runner_*.log         # Main runner logs
│   ├── Worker_*.log         # Worker process logs
│   └── ...
├── _work/                    # Working directory
│   └── (repositories will be cloned here)
├── bin/                      # Runner binaries
├── externals/               # External tools (git, node, etc.)
├── .credentials             # Encrypted credentials (don't touch!)
├── .runner                  # Runner configuration
├── config.sh                # Configuration script
├── run.sh                   # Run runner interactively
└── svc.sh                   # Service management script
```
**สำคัญ:**
- ห้าม แก้ไขไฟล์ ```.credentials``
- ห้าม commit folder ```_work`` เข้า git
- ถ้าต้องการลบ runner: ใช้ ```./config.sh remove --token YOUR_TOKEN```

### ส่วนที่ 3: สร้าง GitHub Actions Workflow
#### 3.1 สร้าง Workflow File
1. สร้าง directory structure:

```bash
# สร้างโฟลเดอร์ในโปรเจกต์ 
mkdir -p .github/workflows
```
2. สร้างไฟล์ ```.github/workflows/deploy.yml```

```yaml
name: Deploy to Self-Hosted Server (Pull-based)

# Trigger events
on:
  push:
    branches: 
      - main
  workflow_dispatch:  # Manual trigger
    inputs:
      environment:
        description: 'Deployment Environment'
        required: false
        default: 'production'

# Environment variables
env:
  VERSION: "1.0.${{ github.run_number }}"
  NODE_ENV: production

jobs:
  deploy:
    name: 🔄 Pull & Deploy Application
    runs-on: self-hosted  # 🔑 Runner จะ poll และรับ job นี้
    
    steps:
      # ================================================================
      # Step 1: Checkout Code
      # Runner จะ clone repository มาที่ _work directory
      # ================================================================
      - name: 📥 Checkout Code
        uses: actions/checkout@v4
        with:
          fetch-depth: 1  # Shallow clone (เร็วกว่า)

      # ================================================================
      # Step 2: Display Runner Information
      # แสดงข้อมูล runner ที่กำลังรัน job
      # ================================================================
      - name: 📊 Runner Information
        run: |
          echo "════════════════════════════════════════"
          echo "🖥️  Runner Information"
          echo "════════════════════════════════════════"
          echo "Runner Name: ${RUNNER_NAME}"
          echo "Runner OS: ${RUNNER_OS}"
          echo "Runner Arch: ${RUNNER_ARCH}"
          echo "Workflow: ${{ github.workflow }}"
          echo "Run Number: ${{ github.run_number }}"
          echo "Run ID: ${{ github.run_id }}"
          echo "════════════════════════════════════════"

      # ================================================================
      # Step 3: Set Version
      # กำหนด version สำหรับ deployment
      # ================================================================
      - name: 🏷️  Set Application Version
        run: |
          VERSION="1.0.${{ github.run_number }}"
          echo "VERSION=${VERSION}" >> $GITHUB_ENV
          echo "📦 Deploying Version: ${VERSION}"

      # ================================================================
      # Step 4: Cleanup Old Resources
      # ลบ containers และ images เก่า
      # ================================================================
      - name: 🧹 Cleanup Old Resources
        run: |
          echo "🧹 Stopping old containers..."
          docker-compose down --remove-orphans || true
          
          echo "🗑️  Removing unused images..."
          docker system prune -af --volumes || true
          
          echo "✅ Cleanup completed"

      # ================================================================
      # Step 5: Create Environment File
      # สร้าง .env file จาก secrets
      # ================================================================
      - name: 🔧 Create Environment File
        run: |
          cat > .env << EOF
          PORT=3000
          VERSION=${{ env.VERSION }}
          NODE_ENV=production
          # Add more secrets from GitHub Secrets here
          # DATABASE_URL=${{ secrets.DATABASE_URL }}
          # API_KEY=${{ secrets.API_KEY }}
          EOF
          
          echo "✅ Environment file created"

      # ================================================================
      # Step 6: Build Docker Images
      # Build images ใหม่ด้วย no-cache
      # ================================================================
      - name: 🔨 Build Docker Images
        run: |
          echo "🔨 Building Docker images..."
          docker-compose build --no-cache --pull
          
          echo "📦 Listing built images:"
          docker images | grep selfhosted || true
          
          echo "✅ Build completed"
        env:
          VERSION: ${{ env.VERSION }}

      # ================================================================
      # Step 7: Start Services
      # Start containers ด้วย docker-compose
      # ================================================================
      - name: 🚀 Start Services
        run: |
          echo "🚀 Starting services..."
          docker-compose up -d
          
          echo "📊 Container status:"
          docker-compose ps
          
          echo "✅ Services started"
        env:
          VERSION: ${{ env.VERSION }}

      # ================================================================
      # Step 8: Wait for Application
      # รอให้ application พร้อม
      # ================================================================
      - name: ⏳ Wait for Application Startup
        run: |
          echo "⏳ Waiting for application to start..."
          sleep 15
          
          echo "📊 Checking container health..."
          docker-compose ps

      # ================================================================
      # Step 9: Health Check
      # ตรวจสอบว่า application ทำงานได้
      # ================================================================
      - name: 🏥 Health Check
        run: |
          echo "🏥 Performing health check..."
          
          max_attempts=10
          attempt=0
          
          while [ $attempt -lt $max_attempts ]; do
            echo "Attempt $((attempt + 1))/$max_attempts..."
            
            # Check direct app
            if curl -f http://localhost:3000/health; then
              echo "✅ Direct app health check passed!"
              break
            fi
            
            # Check through nginx
            if curl -f http://localhost:8080/health; then
              echo "✅ Nginx proxy health check passed!"
              break
            fi
            
            attempt=$((attempt + 1))
            
            if [ $attempt -eq $max_attempts ]; then
              echo "❌ Health check failed after $max_attempts attempts"
              echo "📋 Application logs:"
              docker-compose logs --tail=50 app
              echo "📋 Nginx logs:"
              docker-compose logs --tail=50 nginx
              exit 1
            fi
            
            echo "⏳ Waiting 5 seconds before retry..."
            sleep 5
          done
          
          echo "✅ Health check passed!"

      # ================================================================
      # Step 10: Run Verification Tests
      # ทดสอบ endpoints ต่างๆ
      # ================================================================
      - name: 🧪 Run Verification Tests
        run: |
          echo "🧪 Running verification tests..."
          
          # Test root endpoint
          echo "Testing GET /"
          response=$(curl -s http://localhost:8080/)
          echo "Response: $response"
          echo "$response" | grep -q "Pull-based" || (echo "❌ Root endpoint test failed" && exit 1)
          
          # Test health endpoint
          echo "Testing GET /health"
          curl -f http://localhost:8080/health || (echo "❌ Health endpoint test failed" && exit 1)
          
          # Test info endpoint
          echo "Testing GET /info"
          curl -f http://localhost:8080/info || (echo "❌ Info endpoint test failed" && exit 1)
          
          echo "✅ All tests passed!"

      # ================================================================
      # Step 11: Display Container Information
      # แสดงข้อมูล containers ที่กำลังรัน
      # ================================================================
      - name: 📊 Display Container Information
        if: always()
        run: |
          echo "════════════════════════════════════════"
          echo "📦 Running Containers"
          echo "════════════════════════════════════════"
          docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
          echo ""
          
          echo "════════════════════════════════════════"
          echo "🔍 Container Details"
          echo "════════════════════════════════════════"
          docker-compose ps
          echo ""
          
          echo "════════════════════════════════════════"
          echo "💾 Disk Usage"
          echo "════════════════════════════════════════"
          docker system df
          echo ""

      # ================================================================
      # Step 12: Display Application Logs
      # แสดง logs ของ application
      # ================================================================
      - name: 📋 Display Application Logs
        if: always()
        run: |
          echo "════════════════════════════════════════"
          echo "📋 Application Logs (Last 30 lines)"
          echo "════════════════════════════════════════"
          docker-compose logs --tail=30 app
          echo ""
          
          echo "════════════════════════════════════════"
          echo "📋 Nginx Logs (Last 30 lines)"
          echo "════════════════════════════════════════"
          docker-compose logs --tail=30 nginx

      # ================================================================
      # Step 13: Deployment Summary
      # สรุปผลการ deploy
      # ================================================================
      - name: 🎉 Deployment Summary
        if: success()
        run: |
          echo "════════════════════════════════════════"
          echo "✅ Deployment Successful!"
          echo "════════════════════════════════════════"
          echo "📦 Version: ${{ env.VERSION }}"
          echo "🌍 Environment: production"
          echo "🔗 Direct URL: http://localhost:3000"
          echo "🔗 Proxy URL: http://localhost:8080"
          echo "❤️  Health: http://localhost:8080/health"
          echo "ℹ️  Info: http://localhost:8080/info"
          echo "════════════════════════════════════════"
          echo ""
          echo "🔄 Runner Model: Pull-based (Polling)"
          echo "🖥️  Runner: ${RUNNER_NAME}"
          echo "⏱️  Duration: ${{ job.status }}"
          echo "════════════════════════════════════════"

      # ================================================================
      # Step 14: Cleanup on Failure
      # กรณี deploy ล้มเหลว ให้ cleanup
      # ================================================================
      - name: 🔴 Cleanup on Failure
        if: failure()
        run: |
          echo "🔴 Deployment failed! Cleaning up..."
          docker-compose down --remove-orphans || true
          echo "📋 Final logs:"
          docker-compose logs --tail=100 || true
```
#### 3.2 อธิบาย Workflow แบบละเอียด
**การทำงานของ Workflow:**
1. **Developer Push Code** → GitHub Repository
2. **GitHub** สร้าง job และเก็บใน queue
3. **Runner** (บนเครื่อง local) polling ไปที่ GitHub API
4. **GitHub API** ตอบกลับด้วย job details
5. **Runner** ดึง code และรัน steps ตามที่กำหนด
6. **Runner** ส่ง status updates กลับไปที่ GitHub real-time

**จุดสำคัญ:**

- ```runs-on: self-hosted`` → บอกให้ GitHub queue job สำหรับ self-hosted runner
- Runner จะ **poll** และเมื่อเจอ job ที่ match ก็จะรับมาทำ
- **ไม่มี** การ push job จาก GitHub มาที่ runner

#### 3.3 Commit และ Push Workflow
```bash
git add .github/workflows/deploy.yml
git commit -m "Add CI/CD workflow with pull-based self-hosted runner"
git push origin main
```
**สิ่งที่จะเกิดขึ้น**
1. Code ถูก push ไป GitHub
2. Workflow trigger (event: push to main)
3. GitHub สร้าง job
4. Runner (บนเครื่องคุณ) polling และรับ job
5. Runner เริ่มทำ deployment

### ส่วนที่ 4: ทดสอบการ Deploy และติดตาม Polling (15 นาที)
#### 4.1 ดู Runner Polling Real-time
**ในขณะที่ push code ให้เปิด Terminal ดู runner logs:**
```bash
cd ~/actions-runner
tail -f _diag/Runner_*.log

# คุณจะเห็นประมาณนี้:
# [timestamp] Polling for jobs...
# [timestamp] No jobs available, waiting...
# [timestamp] Polling for jobs...
# [timestamp] Job assigned! Job ID: 12345
# [timestamp] Job details received
# [timestamp] Starting job execution...
```

#### 4.2 ติดตามผ่าน GitHub UI

1. ไปที่ repository บน GitHub
2. คลิกแท็บ **Actions**
3. คุณจะเห็น:
   - Workflow run ใหม่ปรากฏขึ้น
   - สถานะ: **Queued** → **In Progress** → **Success/Failure**
   - สัญลักษณ์: 🟡 (pending) → 🔵 (running) → 🟢 (success) หรือ 🔴 (failure)

4. คลิกเข้าไปดู workflow run:
   - เห็น jobs ทั้งหมด
   - คลิกที่ job name เพื่อดู steps
   - แต่ละ step แสดง logs แบบ real-time

**ตัวอย่าง Timeline:**
```
00:00 - Job queued
00:01 - Runner picks up job (polling)
00:02 - Checkout code
00:03 - Build Docker images
00:05 - Start services
00:06 - Health check
00:07 - Deployment success ✅
```
#### 4.3 ตรวจสอบ Application บนเครื่อง Local
1. **ตรวจสอบ Containers:**

```bash
# ดู running containers
docker ps

# Output ที่ควรเห็น:
# CONTAINER ID   IMAGE                    STATUS         PORTS
# abc123         nodejs-selfhosted-app    Up 2 mins      0.0.0.0:3000->3000/tcp
# def456         nginx:alpine             Up 2 mins      0.0.0.0:8080->80/tcp

# ดู container details
docker-compose ps

# Output:
#          Name                    State      Ports
# -------------------------------------------------------
# nodejs-selfhosted-app   Up (healthy)  0.0.0.0:3000->3000/tcp
# nginx-selfhosted-proxy  Up (healthy)  0.0.0.0:8080->80/tcp
```
2. **ทดสอบผ่าน Direct App (Port 3000):**

```bash
# Test root endpoint
curl http://localhost:3000

# Expected Output:
# {
#   "message": "🚀 Hello from CI/CD Demo with Self-Hosted Runner!",
#   "model": "Pull-based (Polling)",
#   "version": "1.0.1",
#   "hostname": "abc123def456",
#   "timestamp": "2024-01-15T10:30:00.000Z",
#   "environment": "production",
#   "uptime": "120 seconds"
# }

# Test health endpoint
curl http://localhost:3000/health

# Test info endpoint
curl http://localhost:3000/info
```
3. **ทดสอบผ่าน Nginx Reverse Proxy (Port 8080):**

```bash
# Test through proxy
curl http://localhost:8080

# Test health
curl http://localhost:8080/health

# Test info
curl http://localhost:8080/info

# Test with pretty print
curl -s http://localhost:8080 | jq .
```
4. **เปิด Web Browser:**

เปิดเบราว์เซอร์และไปที่:
- http://localhost:8080 (ผ่าน Nginx)
- http://localhost:8080/health
- http://localhost:8080/info

ควรเห็น JSON response 

#### 4.4 ดู Application Logs
```bash
# Logs ของ application
docker-compose logs app

# Logs ของ nginx
docker-compose logs nginx

# Follow logs real-time
docker-compose logs -f

# Logs 100 บรรทัดล่าสุด
docker-compose logs --tail=100

# Logs ของ container เฉพาะ
docker logs nodejs-selfhosted-app
```
#### 4.5 ตรวจสอบ Resource Usage
```bash
# ดู resource usage real-time
docker stats

# Output:
# CONTAINER           CPU %    MEM USAGE / LIMIT    MEM %    NET I/O
# nodejs-selfhosted   0.5%     50MiB / 8GiB        0.6%     1.2kB / 850B
# nginx-selfhosted    0.1%     10MiB / 8GiB        0.1%     850B / 1.2kB

# ดู disk usage
docker system df

# Output:
# TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
# Images          5         2         1.2GB     800MB (66%)
# Containers      2         2         50MB      0B (0%)
# Local Volumes   1         1         100MB     0B (0%)
```
### ส่วนที่ 5: ทดสอบการ Update Application (20 นาที)
#### 5.1 สังเกต Polling Behavior
**ก่อนแก้ไข code ให้เปิด Terminal ดู runner logs:**
```bash
cd ~/actions-runner
tail -f _diag/Runner_*.log

# Runner กำลัง polling อย่างต่อเนื่อง:
# [10:30:00] Polling for jobs... (no jobs)
# [10:30:02] Polling for jobs... (no jobs)
# [10:30:04] Polling for jobs... (no jobs)
```
Runner จะ poll ทุก 1-2 วินาที หรือใช้ long-polling (รอสูงสุด 60 วินาที)
#### 5.2 แก้ไข Code
1. เปิดไฟล์ ```server.js```

```javascript
// แก้ไข root endpoint
app.get('/', (req, res) => {
  res.json({
    message: '🎉 Updated! Now with Pull-based Runner Architecture!',
    model: 'Pull-based (Polling)',
    description: 'Runner actively polls GitHub API for new jobs',
    security: 'No inbound ports required',
    version: VERSION,
    hostname: HOSTNAME,
    timestamp: new Date().toISOString(),
    environment: process.env.NODE_ENV || 'development',
    uptime: `${Math.floor(process.uptime())} seconds`,
    deployCount: parseInt(VERSION.split('.')[2] || '0')
  });
});

// เพิ่ม endpoint ใหม่สำหรับอธิบาย architecture
app.get('/architecture', (req, res) => {
  res.json({
    title: 'Pull-based Architecture',
    description: 'Self-Hosted Runner uses polling to get jobs',
    flow: [
      '1. Developer pushes code to GitHub',
      '2. GitHub creates job and adds to queue',
      '3. Runner polls GitHub API periodically',
      '4. GitHub responds with job details',
      '5. Runner clones code and executes locally',
      '6. Runner reports status back to GitHub'
    ],
    advantages: [
      'No inbound firewall rules needed',
      'No static IP required',
      'Works with dynamic IP',
      'More secure (runner initiates connection)',
      'Can access local resources (databases, files)'
    ],
    polling: {
      method: 'Long-polling with timeout',
      interval: '1-2 seconds or 60s timeout',
      protocol: 'HTTPS',
      endpoint: 'https://api.github.com/actions/v1/jobs'
    }
  });
});
```
#### 5.3 Commit และ Push (เริ่มกระบวนการ Pull)
```bash
git add server.js
git commit -m "Update: Add architecture endpoint and improve messaging"
git push origin main
```

**สิ่งที่เกิดขึ้นทันที:**
```
Terminal 1 (Runner Logs):                 Terminal 2 (Docker Logs):
═══════════════════════════              ═══════════════════════════
[10:35:01] Polling...                    
[10:35:02] Polling...                    
[10:35:03] Job assigned! 🎯              
[10:35:03] Downloading job details...    
[10:35:04] Cloning repository...         
[10:35:05] Starting workflow...          
[10:35:10] Building Docker images...     app | Building...
[10:35:20] Starting services...          app | Starting server...
[10:35:25] Health check...               nginx | Server ready
[10:35:30] ✅ Deployment success!        
[10:35:31] Polling... (continues)        
```

#### 5.4 ดู Real-time Deployment

**ใน GitHub Actions UI คุณจะเห็น:**

1. **Workflow กำลังรัน:**
```
   🔵 Deploy to Self-Hosted Server (Pull-based)
      🔄 Pull & Deploy Application
         ├─ ✅ Checkout Code
         ├─ ⏳ Build Docker Images (in progress...)
         ├─ ⏱️  Start Services (pending)
         └─ ⏱️  Health Check (pending)
```
2. **Logs Streaming Real-time:**
- จะเห็น output จาก runner ทันที
- Logs streaming แบบ real-time (ไม่ต้องรีเฟรช)
- สี: output ปกติ (ขาว), errors (แดง), warnings (เหลือง)

#### 5.5 ตรวจสอบการ Deploy ใหม่
1. **รอให้ workflow เสร็จ:**

```bash
# ใน Terminal ดู runner logs
# [10:35:35] Job completed successfully
# [10:35:36] Polling for jobs... (back to normal)
```
2. **ทดสอบ endpoints ใหม่:**

```bash
# Test updated root endpoint
curl http://localhost:8080

# Expected: เห็นข้อความใหม่ "🎉 Updated!"

# Test new architecture endpoint
curl http://localhost:8080/architecture | jq .

# Expected: JSON อธิบาย pull-based architecture
```
3. **ตรวจสอบ version:**

```bash
# Check version number
curl -s http://localhost:8080 | jq '.version'

# Output: "1.0.2" (เพิ่มขึ้นจาก run ที่แล้ว)

# Check deploy count
curl -s http://localhost:8080 | jq '.deployCount'

# Output: 2
```

4. **เปิด Browser ทดสอบ:**
```
http://localhost:8080                # Root endpoint (ข้อความใหม่)
http://localhost:8080/architecture   # Architecture info
http://localhost:8080/health         # Health check
http://localhost:8080/info           # Server info
```
### ส่วนที่ 6: Troubleshooting และ Monitoring (15 นาที)
#### 6.1 คำสั่งที่มีประโยชน์
**Docker Management:**
```bash
# ดูสถานะ containers
docker-compose ps

# Restart services
docker-compose restart

# Stop services
docker-compose down

# Start services
docker-compose up -d

# Rebuild และ restart
docker-compose up -d --build

# ดู logs
docker-compose logs -f

# ดู logs ของ service เดียว
docker-compose logs -f app

# ดู resource usage real-time
docker stats

# ลบ everything และ start ใหม่
docker-compose down -v
docker-compose up -d --build
```
**Runner Management:**
```bash
# ไปที่ runner directory
cd ~/actions-runner

# ดูสถานะ runner
./svc.sh status    # (macOS/Linux)

# Restart runner
./svc.sh stop
./svc.sh start

# ดู runner logs
tail -f _diag/Runner_*.log

# ดู worker logs
tail -f _diag/Worker_*.log

# ดู logs ย้อนหลัง 100 บรรทัด
tail -100 _diag/Runner_*.log
```
**Network Debugging:**
```bash
# ตรวจสอบ ports ที่เปิดอยู่
# macOS/Linux:
lsof -i :3000
lsof -i :8080

# Windows (PowerShell):
netstat -ano | findstr :3000
netstat -ano | findstr :8080

# ทดสอบ connectivity
curl -v http://localhost:3000
curl -v http://localhost:8080

# ตรวจสอบ DNS resolution
nslookup api.github.com

# Test HTTPS connectivity ไปยัง GitHub
curl -v https://api.github.com
```
**Cleanup**
```bash
# ลบ containers และ networks
docker-compose down

# ลบ containers, networks, volumes
docker-compose down -v

# ลบ images เก่าที่ไม่ใช้
docker system prune -a

# ลบทั้งหมด (ระวัง!)
docker system prune -a --volumes

# ดู disk usage
docker system df
```
### 6.2 แก้ปัญหาที่พบบ่อย
**ปัญหา 1: Runner ไม่ Online (สีเทาใน GitHub)**
```bash
# ตรวจสอบว่า runner process ทำงานหรือไม่
ps aux | grep Runner.Listener

# ถ้าไม่เจอ ให้ start runner
cd ~/actions-runner
./svc.sh start

# ดู logs เพื่อหา error
tail -50 _diag/Runner_*.log

# ปัญหาที่พบบ่อย:
# - ไม่มี internet connection
# - Firewall block outbound HTTPS
# - Credentials หมดอายุ (ต้อง re-configure)
```
**Solution:**
```bash
# Test connectivity ไปยัง GitHub
curl -I https://api.github.com

# ถ้า credentials หมดอายุ
./config.sh remove --token OLD_TOKEN
./config.sh --url YOUR_REPO_URL --token NEW_TOKEN
./svc.sh install
./svc.sh start
```
**ปัญหา 2: Job Queue แต่ Runner ไม่รับ**
```bash
# ตรวจสอบ runner labels
cat .runner

# Output ควรมี labels ที่ match กับ workflow
# เช่น: "self-hosted", "macOS", "X64"

# ถ้า labels ไม่ตรง workflow จะไม่ assign job ให้

# ดู runner logs
tail -f _diag/Runner_*.log

# ถ้าเห็น "No matching jobs" แสดงว่า labels ไม่ตรง
```
**Solution:**
```bash
# Re-configure runner ด้วย labels ที่ถูกต้อง
./config.sh remove --token TOKEN
./config.sh --url URL --token TOKEN --labels self-hosted,macOS,X64
ปัญหา 3: Container ไม่ Start
bash# ดู logs ของ container
docker-compose logs app

# ตรวจสอบ port conflicts
lsof -i :3000  # macOS/Linux
netstat -ano | findstr :3000  # Windows

# ถ้า port ถูกใช้ ให้ปิด process นั้น
# หรือเปลี่ยน port ใน docker-compose.yml

# ตรวจสอบว่า Docker Desktop ทำงาน
docker info
```
**Solution:**
```bash
# Kill process ที่ใช้ port (macOS/Linux)
lsof -ti :3000 | xargs kill -9

# หรือเปลี่ยน port
# แก้ไขใน docker-compose.yml:
# ports:
#   - "3001:3000"  # เปลี่ยนจาก 3000:3000
```
**ปัญหา 4: Health Check ล้มเหลว**
```bash
# ตรวจสอบว่า app ทำงานหรือไม่
docker exec nodejs-selfhosted-app curl http://localhost:3000/health

# ถ้า error "command not found: curl"
docker exec nodejs-selfhosted-app wget -O- http://localhost:3000/health

# หรือใช้ node
docker exec nodejs-selfhosted-app node -e "require('http').get('http://localhost:3000/health', (r) => console.log(r.statusCode))"

# ดู application logs
docker-compose logs app

# ปัญหาที่พบบ่อย:
# - App ยังไม่เริ่มทำงาน (ต้องรอ startup time)
# - App crash ทันทีหลัง start
# - Port binding ไม่ถูกต้อง
```
**Solution:**
```bash
# เพิ่ม startup time ใน workflow
# ใน deploy.yml:
# - name: Wait for Application
#   run: sleep 30  # เพิ่มเวลารอ

# หรือทำ retry loop
# while ! curl -f http://localhost:3000/health; do
#   sleep 5
# done
```
**ปัญหา 5: Build ล้มเหลว (Out of Memory)**
```bash
# ตรวจสอบ memory usage
docker stats

# เพิ่ม memory limit ใน docker-compose.yml
# services:
#   app:
#     deploy:
#       resources:
#         limits:
#           memory: 1G

# หรือ cleanup ก่อน build
docker system prune -af
docker-compose build --no-cache
ปัญหา 6: Workflow Stuck (ไม่มีการ Update)
bash# Refresh GitHub Actions page
# ถ้ายัง stuck:

# ตรวจสอบ runner logs
cd ~/actions-runner
tail -f _diag/Runner_*.log

# ถ้าเห็น "Job started" แต่ไม่มี progress:
# - Runner อาจ hang
# - Network ขาด
# - Process killed โดย OS

# Solution: Restart runner
./svc.sh stop
./svc.sh start
```
**ปัญหา 7: Permission Denied Errors**
```bash
# ถ้าเจอ "permission denied" ใน runner logs

# ตรวจสอบ ownership ของ _work directory
ls -la ~/actions-runner/_work

# แก้ไข permissions
cd ~/actions-runner
chmod -R 755 _work

# หรือ run runner ด้วย user ที่ถูกต้อง
# (อย่า run เป็น root!)
```

#### 6.3 Monitoring Dashboard
**สร้าง Simple Monitoring Script:**
**สร้างไฟล์** ```monitor.sh```
```bash
#!/bin/bash

# Colors
GREEN='\033[0;32m'
RED='\033[0;31m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

echo "════════════════════════════════════════"
echo "🔍 Self-Hosted Runner Monitor"
echo "════════════════════════════════════════"
echo ""

# Check Runner Status
echo "📊 Runner Status:"
cd ~/actions-runner
if ./svc.sh status | grep -q "active"; then
    echo -e "${GREEN}✅ Runner: Online${NC}"
else
    echo -e "${RED}❌ Runner: Offline${NC}"
fi
echo ""

# Check Containers
echo "🐳 Container Status:"
docker-compose ps --format "table {{.Name}}\t{{.Status}}"
echo ""

# Check Application Health
echo "🏥 Application Health:"
if curl -sf http://localhost:8080/health > /dev/null 2>&1; then
    echo -e "${GREEN}✅ Application: Healthy${NC}"
    curl -s http://localhost:8080 | jq -r '.version' | xargs echo "📦 Version:"
else
    echo -e "${RED}❌ Application: Unhealthy${NC}"
fi
echo ""

# Resource Usage
echo "💻 Resource Usage:"
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
echo ""

# Recent Logs
echo "📋 Recent Logs (last 5 lines):"
docker-compose logs --tail=5
```
**ใช้งาน:**
```bash
# Make executable
chmod +x monitor.sh

# Run
./monitor.sh

# Run continuously (every 10 seconds)
watch -n 10 ./monitor.sh
```

### ส่วนที่ 7: Security Best Practices (10 นาที)

#### 7.1 การใช้ GitHub Secrets

> **⚠️ สำคัญ:** ห้าม hard-code secrets ในโค้ดหรือ workflow

1. **เพิ่ม Secrets ใน GitHub:**
   - ไปที่ **Settings** → **Secrets and variables** → **Actions**
   - คลิก **New repository secret**
   - เพิ่ม secrets:
```
Name: DATABASE_URL
Value: postgresql://user:password@localhost:5432/mydb

Name: API_KEY
Value: sk-abc123xyz789

Name: JWT_SECRET
Value: your-super-secret-key
```
2. **ใช้ Secrets ใน Workflow:**
แก้ไข ```.github/workflows/deploy.yml```
```yaml
- name: 🔧 Create Environment File
  run: |
    cat > .env << EOF
    PORT=3000
    VERSION=${{ env.VERSION }}
    NODE_ENV=production
    DATABASE_URL=${{ secrets.DATABASE_URL }}
    API_KEY=${{ secrets.API_KEY }}
    JWT_SECRET=${{ secrets.JWT_SECRET }}
    EOF
    
    echo "✅ Environment file created (secrets hidden)"
```
**สิ่งที่ GitHub จะทำ:**
- Secrets จะถูก encrypt ที่ rest
- ใน logs secrets จะแสดงเป็น ***
- ไม่มีใครเห็น secrets จริงๆ ได้

#### 7.2 ข้อควรระวังด้านความปลอดภัย Pull-based Model
1. **ใช้เฉพาะ Private Repository ⚠️**
```bash
# ❌ อันตราย - Public Repository
# ใครก็ตาม fork repository
# สร้าง Pull Request ที่มี malicious code
# Code นั้นจะรันบนเครื่องคุณ!

# ✅ ปลอดภัย - Private Repository
# เฉพาะ collaborators ที่คุณเชื่อถือเท่านั้น
# ที่สามารถ push code
```
2. **Limit Runner Permissions:**
```bash
# ✅ รัน runner ด้วย user ที่มี permissions จำกัด
# ❌ อย่า run runner เป็น root/administrator

# ตรวจสอบว่า runner ไม่ใช่ root
ps aux | grep Runner.Listener | grep -v root
```
3. **ใช้ Runner Groups (GitHub Enterprise):**
```yaml
# กำหนด runner group สำหรับ sensitive repositories
jobs:
  deploy:
    runs-on: [self-hosted, production, trusted]
```
4. **Network Isolation:**
```bash
# ถ้าเป็นไปได้ ให้ runner อยู่ใน isolated network
# - ไม่สามารถเข้าถึง internal systems โดยตรง
# - ใช้ VPN หรือ bastion host
# - Limit outbound connections
```
5. **Audit และ Logging:**
```bash
# เปิด logging สำหรับ runner activities
# Monitor:
# - Jobs executed
# - Code cloned
# - Commands run
# - Network connections

# ดู runner logs เป็นประจำ
cd ~/actions-runner
cat _diag/Runner_*.log | grep -i error
cat _diag/Runner_*.log | grep -i warning
```
#### 7.3 Regular Maintenance
1. **Update Runner:**
```bash
# ทุก 2-3 เดือน ควร update runner

cd ~/actions-runner

# Stop runner
./svc.sh stop

# Remove old version
./config.sh remove --token YOUR_REMOVAL_TOKEN

# Download new version (ดู version ใหม่จาก GitHub)
curl -o actions-runner-osx-x64-NEW.tar.gz -L \
  https://github.com/actions/runner/releases/download/vNEW/...

# Extract และ configure ใหม่
tar xzf actions-runner-osx-x64-NEW.tar.gz
./config.sh --url YOUR_URL --token NEW_TOKEN

# Start
./svc.sh install
./svc.sh start
```
2. **Cleanup Docker Resources:**
```bash
# รัน cleanup script อัตโนมัติ

# สร้างไฟล์ cleanup.sh
cat > cleanup.sh << 'EOF'
#!/bin/bash
echo "🧹 Cleaning up Docker resources..."

# Remove stopped containers
docker container prune -f

# Remove unused images
docker image prune -af

# Remove unused volumes
docker volume prune -f

# Remove unused networks
docker network prune -f

echo "✅ Cleanup completed!"
EOF

chmod +x cleanup.sh

# รัน weekly ด้วย cron
# crontab -e
# 0 2 * * 0 /path/to/cleanup.sh
```
3. **Monitor Disk Space:**
```bash
# Check disk usage
df -h

# Check Docker disk usage
docker system df

# ถ้าเหลือพื้นที่น้อย (<5GB):
docker system prune -a --volumes
```

## ผลการทดลอง

### สิ่งที่ควรได้จากการทดลอง

1. **GitHub Repository:**
   - ✅ มี workflow file ใน `.github/workflows/deploy.yml`
   - ✅ เข้าใจว่า workflow จะถูก **queue** รอ runner มา **poll**
   - ✅ มี commit history แสดงการ deploy หลายครั้ง
   - ✅ Actions tab แสดง successful runs พร้อม logs

2. **Self-Hosted Runner:**
   - ✅ Runner แสดงสถานะ "Idle" สีเขียวบน GitHub
   - ✅ เข้าใจว่า runner ทำการ **polling** ไปยัง GitHub API
   - ✅ Service ทำงานอยู่บนเครื่อง local อย่างต่อเนื่อง
   - ✅ สามารถดู polling logs ได้

3. **Pull-based Architecture:**
   - ✅ ไม่ต้องเปิด port inbound
   - ✅ ไม่ต้องมี static IP
   - ✅ Runner เป็นฝ่าย initiate connection
   - ✅ ใช้ long-polling technique

4. **Application Deployment:**
   - ✅ Containers ทำงานบน Docker Desktop
   - ✅ เข้าถึงได้ผ่าน http://localhost:8080
   - ✅ Health checks ผ่านทุกครั้ง
   - ✅ Nginx reverse proxy ทำงานถูกต้อง

5. **Monitoring & Troubleshooting:**
   - ✅ เห็น logs ทั้งจาก runner และ Docker
   - ✅ สามารถ debug ปัญหาได้
   - ✅ เข้าใจ workflow ของการ deploy

### ตัวอย่าง Evidence ที่ควรมี

**1. Screenshot จาก GitHub Actions:**
```
✅ Deploy to Self-Hosted Server (Pull-based)
   Duration: 2m 35s
   
   Jobs:
   🔄 Pull & Deploy Application
      ✅ All steps completed successfully
```

**2. Screenshot จาก Runner Settings:**
```
Self-hosted runners
Status: Idle (สีเขียว)
Name: my-macbook-runner
Labels: self-hosted, macOS, X64
Last Activity: Just now
```
**3. Browser Screenshot:**
```json
{
  "message": "🎉 Updated! Now with Pull-based Runner Architecture!",
  "model": "Pull-based (Polling)",
  "version": "1.0.5",
  "security": "No inbound ports required"
}
```
**4. Terminal Output:**
```bash
$ docker ps
CONTAINER ID   IMAGE                    STATUS
abc123def456   nodejs-selfhosted-app    Up 10 minutes (healthy)
ghi789jkl012   nginx:alpine             Up 10 minutes (healthy)

$ curl http://localhost:8080/health
{"status":"healthy","uptime":600}
```

**5. Runner Logs:**
```
[2024-01-15 10:30:00] Runner is online
[2024-01-15 10:30:01] Polling for jobs...
[2024-01-15 10:30:15] Job assigned: deploy (ID: abc123)
[2024-01-15 10:32:45] Job completed: success
[2024-01-15 10:32:46] Polling for jobs...
```
## คำถามท้ายบท
### คำถามทบทวน
**1.  Pull-based Model ของ Self-Hosted Runner คืออะไร มีข้อดีอย่างไร**
<details>
<summary>คำตอบ</summary>

</details>

**2. ทำไม Pull-based ปลอดภัยกว่า Push-based**
<details>
<summary>คำตอบ</summary>

</details>

**3. Long-polling คืออะไร และทำไม Runner ใช้เทคนิคนี้**
<details>
<summary>คำตอบ</summary>

</details>

**4. ทำไมห้ามใช้ Self-Hosted Runner กับ Public Repository**
<details>
<summary>คำตอบ</summary>
</details>

**5. อธิบายความแตกต่างระหว่าง Polling และ Webhook**
<details>
<summary>คำตอบ</summary>
</details>

**6. หากต้องการ deploy ไปหลาย servers พร้อมกัน ควรทำอย่างไร**
<details>
<summary>แนวทาง</summary>
</details>

**7. จะ implement Rollback mechanism อย่างไร**
<details>
<summary>แนวทาง</summary>
</details>
**8.  Best practices สำหรับการจัดการ Runner credentials และ secrets ควรทำอย่างไร**
<details>
<summary>แนวทาง</summary>
</details>
