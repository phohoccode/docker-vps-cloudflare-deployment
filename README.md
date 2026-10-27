# HƯỚNG DẪN DEPLOY ỨNG DỤNG DOCKER LÊN VPS VÀ QUẢN LÝ DOMAIN BẰNG CLOUDFLARE (A–Z)

> **Phiên bản:** 1.0 — 28/10/2026  
> **Đối tượng:** Người đã có domain, có VPS Ubuntu và có mã nguồn ứng dụng (hoặc image) chạy bằng Docker.  
> **Mục tiêu:** Truy cập ứng dụng qua `https://example.com`, tự khởi động khi VPS reboot, dùng Cloudflare DNS/Proxy, HTTPS hợp lệ, có cách update, rollback và backup.  
> **Phương án chính:** **Cloudflare → Nginx (chạy trực tiếp trên VPS) → Docker Compose → ứng dụng**, phù hợp cho một VPS chạy một hoặc nhiều website.

---

## MỤC LỤC

1. [Kiến trúc và thuật ngữ](#1-kiến-trúc-và-thuật-ngữ)
2. [Chuẩn bị trước khi deploy](#2-chuẩn-bị-trước-khi-deploy)
3. [Đăng nhập và thiết lập bảo mật VPS](#3-đăng-nhập-và-thiết-lập-bảo-mật-vps)
4. [Cài Docker Engine và Docker Compose](#4-cài-docker-engine-và-docker-compose)
5. [Cấu hình domain trên Cloudflare](#5-cấu-hình-domain-trên-cloudflare)
6. [Đưa source code và biến môi trường lên VPS](#6-đưa-source-code-và-biến-môi-trường-lên-vps)
7. [Tạo/chỉnh Dockerfile và Docker Compose](#7-tạochỉnh-dockerfile-và-docker-compose)
8. [Build và chạy ứng dụng bằng Docker](#8-build-và-chạy-ứng-dụng-bằng-docker)
9. [Cài Nginx làm reverse proxy](#9-cài-nginx-làm-reverse-proxy)
10. [Cài HTTPS bằng Let's Encrypt và Certbot](#10-cài-https-bằng-lets-encrypt-và-certbot)
11. [Bật Cloudflare Proxy, Full (strict) và HTTPS](#11-bật-cloudflare-proxy-full-strict-và-https)
12. [Kiểm tra trước khi đưa website vào sử dụng](#12-kiểm-tra-trước-khi-đưa-website-vào-sử-dụng)
13. [Quy trình update code trên VPS](#13-quy-trình-update-code-trên-vps)
14. [Rollback nếu bản cập nhật bị lỗi](#14-rollback-nếu-bản-cập-nhật-bị-lỗi)
15. [Database, migrations và dữ liệu cần lưu](#15-database-migrations-và-dữ-liệu-cần-lưu)
16. [Backup, log, bảo trì và giám sát](#16-backup-log-bảo-trì-và-giám-sát)
17. [Bảo mật production và Cloudflare](#17-bảo-mật-production-và-cloudflare)
18. [Khắc phục các lỗi thường gặp](#18-khắc-phục-các-lỗi-thường-gặp)
19. [Bảng lệnh quản trị nhanh](#19-bảng-lệnh-quản-trị-nhanh)
20. [Checklist nghiệm thu production](#20-checklist-nghiệm-thu-production)
21. [Tài liệu chính thức](#21-tài-liệu-chính-thức)

---

## 1. Kiến trúc và thuật ngữ

```text
           Người dùng (trình duyệt/điện thoại)
                            |
                       HTTPS :443
                            |
                  CLOUDFLARE (DNS + Proxy)
                            |
                      HTTPS :443
                            |
               VPS Ubuntu (Public IP)
                            |
                Nginx (port 80/443)
                            |
                  http://127.0.0.1:3000
                            |
                   Docker Compose
                            |
                   Container ứng dụng
                            |
             Database / API / Storage (nếu có)
```

- **VPS:** máy chủ Linux đang chạy trên Internet, có IP công khai.
- **Domain:** tên miền đã mua, ví dụ `example.com`.
- **DNS:** bản ghi liên kết domain với địa chỉ IP của VPS.
- **Cloudflare Proxy (đám mây màu cam):** người dùng kết nối qua hạ tầng Cloudflare, không kết nối thẳng tới IP VPS theo luồng DNS thông thường.
- **Docker image:** gói chứa ứng dụng và các dependency để chạy.
- **Container:** tiến trình ứng dụng được chạy từ image.
- **Docker Compose:** cấu hình và quản lý một hay nhiều container.
- **Nginx:** web server nhận request tại cổng 80/443 và chuyển tới ứng dụng trên cổng 3000.
- **Certbot:** công cụ lấy và gia hạn chứng chỉ HTTPS từ Let's Encrypt.

**Quy tắc quan trọng:** Chỉ mở cổng 80, 443 và SSH cần thiết từ Internet. Cổng ứng dụng `3000` phải bind vào `127.0.0.1`, tránh công khai ngoài Internet.

### Các giá trị cần thay trong toàn bộ tài liệu

| Ký hiệu | Ví dụ | Ý nghĩa |
|---|---|---|
| `example.com` | `mydomain.com` | Domain thực tế |
| `www.example.com` | `www.mydomain.com` | Domain có `www` (tùy chọn) |
| `203.0.113.10` | *IP VPS của bạn* | IP IPv4 công khai; IP trong tài liệu chỉ là ví dụ |
| `ubuntu` | `ubuntu` hoặc tên user của bạn | Tài khoản SSH |
| `myapp` | `qr-ordering` | Tên dự án, container và thư mục |
| `/opt/myapp/app` | `/opt/qr-ordering/app` | Thư mục source trên VPS |
| `3000` | cổng thực tế của ứng dụng | Cổng HTTP của container |
| `OWNER/REPO` | chủ sở hữu/repo trên GitHub | Đường dẫn Git repository |

> **Giả định:** Các lệnh được thử theo quy trình chuẩn cho Ubuntu 22.04/24.04 LTS, dùng `sudo`. Nếu VPS đã có website, Nginx, Certbot hoặc database thì **kiểm tra cấu hình hiện tại trước khi thay đổi**; không xóa cấu hình/dữ liệu đang chạy.

---

## 2. Chuẩn bị trước khi deploy

### 2.1. Những thứ bạn phải có

- [ ] VPS có Ubuntu 22.04 LTS hoặc 24.04 LTS và IP công khai.
- [ ] Tài khoản SSH vào VPS với quyền `sudo`.
- [ ] Domain đã mua, còn hạn sử dụng.
- [ ] Tài khoản Cloudflare quản lý được domain.
- [ ] Mã nguồn nằm trong Git repository, hoặc image Docker đã build.
- [ ] Dự án có `Dockerfile` và tốt nhất có `compose.yaml` / `docker-compose.yml`.
- [ ] Danh sách biến môi trường production (database, API, auth, thanh toán...).
- [ ] Đã kiểm tra cách backup database và dữ liệu upload.

**Khuyến nghị VPS thử nghiệm:** từ 2 vCPU, 2–4 GB RAM và SSD. Đây chỉ là cấu hình tham khảo, không phải yêu cầu bắt buộc; dự án Next.js build nặng hoặc database chạy chung VPS có thể cần thêm RAM/swap. Không chọn cấu hình trước khi đo tải thực tế.

### 2.2. Kiểm tra IP và trạng thái domain

Trên Windows PowerShell:

```powershell
nslookup example.com
ssh ubuntu@203.0.113.10
```

Domain chưa trỏ đúng VPS thì `nslookup` có thể hiển thị IP cũ/Cloudflare hoặc không có kết quả. Điều đó bình thường ở bước chuẩn bị.

### 2.3. Phân biệt máy local và VPS

- **Local (Windows/VS Code):** viết code, commit và `git push`.
- **VPS (Ubuntu/SSH):** `git pull`, Docker build/redeploy, quản lý Nginx/HTTPS.
- **Cloudflare Dashboard (trình duyệt):** chỉnh DNS, proxy, TLS, firewall/rules.

Mọi lệnh Linux bên dưới chạy **trên VPS**, trừ phần được ghi rõ là PowerShell/local.

---

## 3. Đăng nhập và thiết lập bảo mật VPS

### Bước 3.1. SSH vào VPS

Trên PowerShell:

```powershell
ssh ubuntu@203.0.113.10
```

Nếu dùng user `root` theo hướng dẫn nhà cung cấp VPS thì đăng nhập tương ứng; về lâu dài nên dùng user không phải root nhưng có `sudo`.

### Bước 3.2. Cập nhật hệ thống và cài công cụ

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y git curl ca-certificates gnupg nano unzip ufw nginx
```

Nếu nâng cấp kernel, chủ động lên lịch reboot sau khi lưu công việc và kiểm tra các dịch vụ đang chạy.

### Bước 3.3. Mở firewall đúng thứ tự

**Quan trọng: cho phép SSH trước khi bật UFW**, nếu không có thể tự khóa quyền truy cập VPS.

```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status verbose
```

Nếu SSH chạy cổng khác `22`, phải mở cổng SSH thực tế trước khi `ufw enable`. Nếu nhà cung cấp VPS có **cloud firewall/security group**, cần mở tương ứng ở cả control panel nhà cung cấp.

### Bước 3.4. Khuyến nghị SSH key

Trên Windows PowerShell (chỉ chạy nếu chưa có key phù hợp):

```powershell
ssh-keygen -t ed25519 -C "vps-deploy"
```

Chép **public key** (`id_ed25519.pub`) vào `~/.ssh/authorized_keys` của user VPS qua phương pháp an toàn. **Chỉ tắt SSH password sau khi mở một cửa sổ SSH mới và xác nhận SSH key đăng nhập thành công.** Không đưa private key lên GitHub.

### Bước 3.5. Kiểm tra cổng hiện có

```bash
sudo ss -tulpn | grep -E ':(22|80|443|3000)\b' || true
sudo systemctl status nginx --no-pager
```

Nếu 80/443 đang do Apache, Docker hoặc một Nginx khác sử dụng, không cấu hình chồng lên; xác định dịch vụ trước.

---

## 4. Cài Docker Engine và Docker Compose

Dùng **Docker repository chính thức**, không cần cài legacy `docker-compose` riêng. Nếu Docker đã được cài và đang chạy ổn định, kiểm tra phiên bản rồi **bỏ qua bước cài mới**, tránh thay đổi production không cần thiết.

### Bước 4.1. Kiểm tra Docker có sẵn chưa

```bash
docker --version || true
docker compose version || true
sudo systemctl status docker --no-pager || true
```

### Bước 4.2. Thêm Docker APT repository (khi chưa cài)

```bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

Nếu báo conflict giữa Docker chính thức và `docker.io`/`containerd` hệ điều hành, tham khảo mục **Uninstall old versions** trong tài liệu Docker. **Không chạy lệnh gỡ bừa trên VPS đang có container production.**

### Bước 4.3. Cài đặt

```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
```

### Bước 4.4. Kiểm tra

```bash
sudo docker --version
sudo docker compose version
sudo docker run --rm hello-world
sudo docker ps
```

> **Lưu ý:** User thuộc nhóm `docker` có quyền tương đương root trên máy. Tài liệu dùng `sudo docker ...` để dễ kiểm soát, không bắt buộc thêm user vào nhóm `docker`.

> **Cảnh báo firewall:** Docker có thể công khai các container port được publish và một số rule publish có thể vượt qua kỳ vọng với UFW. Vì vậy phần Compose bên dưới **chỉ bind `127.0.0.1:3000:3000`**, thay vì `3000:3000`. Xem tài liệu Docker về UFW và firewall.

---

## 5. Cấu hình domain trên Cloudflare

### Bước 5.1. Đưa domain vào Cloudflare (nếu chưa có)

1. Truy cập **https://dash.cloudflare.com/**.
2. Chọn **Add a domain / Onboard a domain**.
3. Nhập domain gốc, ví dụ `example.com`.
4. Chọn gói phù hợp (gói Free thường đủ cho nhu cầu DNS/Proxy cơ bản).
5. **Kiểm tra lại toàn bộ DNS record đã được Cloudflare import** — đặc biệt là `MX`, `TXT` dùng cho email/SPF/DKIM/DMARC; không để mất record của các dịch vụ khác.
6. Cloudflare cung cấp **hai nameserver**.
7. Vào **nơi đã mua domain (registrar)** → mục Nameservers → thay nameserver cũ bằng hai nameserver Cloudflare cung cấp.
8. Đợi Cloudflare xác nhận trạng thái domain **Active**.

Nếu domain đã ở Cloudflare và trạng thái Active thì bỏ qua việc đổi nameserver. Thay nameserver là thao tác thực hiện **ở nhà đăng ký domain**, không phải chỉ thêm record NS trong bảng DNS.

### Bước 5.2. Trỏ domain tới IP VPS

Trong **Cloudflare → chọn domain → DNS → Records → Add record**:

| Type | Name | Content / Target | Proxy status ban đầu | TTL |
|---|---|---|---|---|
| `A` | `@` | `203.0.113.10` | **DNS only** (mây xám) | Auto |
| `CNAME` | `www` | `example.com` | **DNS only** (mây xám) | Auto |

- `@` đại diện cho domain gốc `example.com`.
- `www` là tùy chọn. Nếu chỉ dùng domain gốc thì không cần record `www` và không cấp certificate cho `www`.
- **Tạm thời dùng DNS only** để dễ kiểm tra IP và cấp chứng chỉ lần đầu. Khi HTTPS origin đã chạy, sẽ chuyển sang **Proxied**.
- Nếu có bản ghi `AAAA` (IPv6) cũ trỏ sai VPS, hãy sửa/xóa record lỗi; chỉ tạo AAAA nếu VPS thực sự có IPv6 và web server được cấu hình phù hợp.
- Không vô tình sửa `MX`, `TXT` hay những subdomain phục vụ hệ thống khác.

Kiểm tra DNS (PowerShell hoặc VPS):

```bash
nslookup example.com
nslookup www.example.com
```

Khi DNS only, A record của domain gốc thường trả IP VPS. Sau khi bật Proxied, DNS thường trả IP Cloudflare — **đó là hành vi đúng**.

---

## 6. Đưa source code và biến môi trường lên VPS

### Bước 6.1. Tạo thư mục ứng dụng

```bash
sudo mkdir -p /opt/myapp
sudo chown "$USER":"$USER" /opt/myapp
cd /opt/myapp
```

### Bước 6.2. Clone repository

Nếu repository public:

```bash
git clone https://github.com/OWNER/REPO.git app
cd /opt/myapp/app
```

Nếu repository private, nên tạo **GitHub Deploy Key read-only** riêng cho VPS:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_github_deploy -C "vps-myapp-deploy"
cat ~/.ssh/id_ed25519_github_deploy.pub
```

Thêm nội dung public key vào **GitHub repository → Settings → Deploy keys → Add deploy key**, không bật write access nếu chỉ cần pull. Tạo `~/.ssh/config` cho alias riêng để dùng đúng deploy key:

```bash
nano ~/.ssh/config
```

```sshconfig
Host github-myapp
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_github_deploy
    IdentitiesOnly yes
```

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/config ~/.ssh/id_ed25519_github_deploy
ssh -T git@github-myapp
git clone git@github-myapp:OWNER/REPO.git /opt/myapp/app
cd /opt/myapp/app
```

Khi SSH hỏi xác nhận fingerprint của GitHub, **đối chiếu fingerprint trong tài liệu GitHub trước khi chấp nhận**. Không lưu GitHub personal access token vào URL Git remote.

### Bước 6.3. Tạo `.env.production`

```bash
cd /opt/myapp/app
umask 077
nano .env.production
```

Ví dụ **minh họa, phải thay tên biến đúng theo source thực tế**:

```dotenv
NODE_ENV=production
APP_BASE_URL=https://example.com
DATABASE_URL=postgresql://USER:PASSWORD@HOST:5432/DATABASE
AUTH_SECRET=THAY_BANG_SECRET_NGAU_NHIEN_DU_MANH
```

Sau đó:

```bash
chmod 600 .env.production
```

**Không bao giờ** commit `.env.production`, token, private key, Supabase service-role key, PayOS secret vào Git. Thêm `.env.production`, `.env.local` vào `.gitignore` và `.dockerignore` nếu phù hợp. Backup bản mã hóa tại nơi an toàn.

**Phân biệt:** file `.env.production` ở đây được **Docker Compose truyền cho container lúc chạy** qua `env_file`. Nó **không tự động cung cấp biến cho bước `docker build`**. Với Next.js, các biến `NEXT_PUBLIC_*` cần có ngay **khi build**, tùy cách triển khai; cần thiết kế build arguments/CI secrets thích hợp. Không đưa **secret server-side** vào `NEXT_PUBLIC_*` hoặc `ARG`/`ENV` của Dockerfile mà có thể lộ trong image.

### Bước 6.4. Kiểm tra cấu trúc project

```bash
ls -la
```

Kỳ vọng tối thiểu:

```text
/opt/myapp/app/
├── Dockerfile
├── compose.yaml
├── .dockerignore
├── .env.production    # không commit
├── package.json       # nếu là ứng dụng Node.js
└── ...source...
```

Nếu repository có sẵn Dockerfile/Compose hoạt động tốt thì **ưu tiên dùng cấu hình hiện hữu**, không ghi đè bằng mẫu dưới đây.

---

## 7. Tạo/chỉnh Dockerfile và Docker Compose

### 7.1. Compose mẫu chung (ứng dụng chạy HTTP cổng 3000)

Tạo `compose.yaml` tại thư mục project nếu chưa có:

```yaml
services:
  web:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: myapp-web
    restart: unless-stopped
    env_file:
      - .env.production
    ports:
      - "127.0.0.1:3000:3000"
    environment:
      NODE_ENV: production
    # Nếu ứng dụng lưu file upload trên disk, khai báo volume riêng.
    # Nếu dữ liệu đã lưu Supabase Storage/S3/R2 thì không tự động cần volume upload.
    # volumes:
    #   - uploads:/app/uploads

# volumes:
#   uploads:
```

**Giải thích cấu hình:**

| Cấu hình | Tác dụng |
|---|---|
| `build` | Tạo image từ Dockerfile hiện tại |
| `container_name` | Đặt tên container để dễ tra cứu |
| `restart: unless-stopped` | Tự chạy lại khi Docker daemon/VPS khởi động, trừ khi đã chủ động stop |
| `env_file` | Nạp runtime environment vào container |
| `127.0.0.1:3000:3000` | Chỉ máy VPS truy cập trực tiếp cổng 3000 |
| `volumes` | Giữ dữ liệu cần tồn tại khi container được tạo lại |

**Nếu ứng dụng lắng nghe port khác**, sửa cả phía bên phải của `ports` (cổng container), cấu hình ứng dụng và `proxy_pass` Nginx tương ứng. Nếu VPS có nhiều app, dùng host port khác nhau: `3001`, `3002`, ... và mỗi app có Nginx `server_name` riêng.

### 7.2. Dockerfile mẫu cho Next.js (tùy chọn)

**Chỉ dùng mẫu này nếu dự án là Next.js dùng npm và lockfile `package-lock.json`.** Nếu bạn đã có Dockerfile production, hãy kiểm tra trước khi thay. Với pnpm/yarn/Prisma/native dependencies cần tùy chỉnh.

Đảm bảo `next.config.ts` hoặc `next.config.js` có:

```ts
const nextConfig = {
  output: 'standalone',
};

export default nextConfig;
```

**Không thay toàn bộ `next.config` nếu đang có cấu hình khác**; chỉ thêm thuộc tính `output` vào object sẵn có. Nếu dự án build bằng framework/adapter khác, phải dùng Dockerfile phù hợp với adapter đó.

Ví dụ Dockerfile:

```dockerfile
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

FROM node:22-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN mkdir -p public && npm run build

FROM node:22-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1
ENV HOSTNAME=0.0.0.0
ENV PORT=3000

RUN addgroup -S nextjs && adduser -S nextjs -G nextjs
COPY --from=builder --chown=nextjs:nextjs /app/public ./public
COPY --from=builder --chown=nextjs:nextjs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nextjs /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000
CMD ["node", "server.js"]
```

Tạo `.dockerignore` tối thiểu:

```gitignore
node_modules
.next
.git
.env
.env.*
!.env.example
npm-debug.log*
coverage
playwright-report
```

**Lưu ý chuyên môn:** Một số dự án cần biến khi build, code generation (như Prisma), file generated, chứng thư hoặc system library bổ sung. Hãy điều chỉnh Dockerfile để bảo đảm lệnh `npm run build` và runtime chạy đúng; không giả định Dockerfile mẫu tương thích với mọi Next.js. Với Next.js hiện đại, tự host qua reverse proxy là mô hình được khuyến nghị.

### 7.3. Kiểm tra cú pháp Compose

```bash
cd /opt/myapp/app
sudo docker compose config --quiet
```

`docker compose config` không có `--quiet` có thể hiển thị biến môi trường đã nội suy; **không dán output chứa secret lên mạng**.

---

## 8. Build và chạy ứng dụng bằng Docker

### Bước 8.1. Build image

```bash
cd /opt/myapp/app
sudo docker compose build
```

Nếu build thất bại, xem log gần cuối. Không tự ý xóa `.env`, volumes, image hoặc database để “chữa” lỗi build.

### Bước 8.2. Chạy container

```bash
sudo docker compose up -d
sudo docker compose ps
```

### Bước 8.3. Xem log

```bash
sudo docker compose logs --tail=100 web
sudo docker compose logs -f web
```

Nhấn `Ctrl+C` để thoát chế độ xem log liên tục; container vẫn chạy.

### Bước 8.4. Test trực tiếp trên VPS

```bash
curl -I http://127.0.0.1:3000
sudo ss -tulpn | grep ':3000' || true
```

Một số app yêu cầu login hoặc tự redirect nên có thể trả `302/307/401` thay vì `200`. Quan trọng là ứng dụng **trả HTTP hợp lệ**, không bị connection refused/500 không mong muốn.

Nếu `curl` không kết nối được, chưa nên cấu hình HTTPS; xử lý app/container trước.

---

## 9. Cài Nginx làm reverse proxy

### Bước 9.1. Kiểm tra Nginx

```bash
sudo nginx -v
sudo systemctl enable --now nginx
sudo systemctl status nginx --no-pager
```

### Bước 9.2. Hỗ trợ WebSocket qua Nginx

WebSocket hữu ích với realtime, event và một số luồng hot reload. Với Nginx tiêu chuẩn Ubuntu, các file `/etc/nginx/conf.d/*.conf` được include trong ngữ cảnh `http`.

```bash
sudo nano /etc/nginx/conf.d/websocket-upgrade.conf
```

Nội dung:

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}
```

Nếu đã có `map` tên `$connection_upgrade` ở file khác, **không thêm trùng**.

### Bước 9.3. Tạo virtual host cho domain

```bash
sudo nano /etc/nginx/sites-available/myapp
```

Nội dung:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;

    client_max_body_size 20m;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_read_timeout 60s;
        proxy_send_timeout 60s;
    }
}
```

> Nếu chỉ cấp domain gốc, sửa thành `server_name example.com;`. Nếu ứng dụng upload file lớn hơn 20 MB, điều chỉnh `client_max_body_size`; Cloudflare còn có giới hạn upload theo gói riêng. Không tăng vô hạn.

### Bước 9.4. Bật site

```bash
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/myapp
sudo nginx -t
sudo systemctl reload nginx
```

Nếu symlink đã tồn tại, không tạo lại. Với VPS mới chỉ có site mặc định, có thể vô hiệu site mặc định sau khi kiểm tra:

```bash
# CHỈ thực hiện trên VPS mới nếu default site không còn cần dùng:
# sudo rm /etc/nginx/sites-enabled/default
# sudo nginx -t && sudo systemctl reload nginx
```

### Bước 9.5. Thử HTTP trước khi cấp HTTPS

Khi Cloudflare đang để DNS only và record đã cập nhật:

```bash
curl -I http://example.com
```

Ngoài ra có thể thử trực tiếp tới Nginx trên VPS, không phụ thuộc DNS:

```bash
curl -I -H 'Host: example.com' http://127.0.0.1
```

Kỳ vọng: Nginx chuyển được request tới ứng dụng. Nếu `502 Bad Gateway`, xem phần xử lý lỗi ở cuối tài liệu.

---

## 10. Cài HTTPS bằng Let's Encrypt và Certbot

**Mục tiêu:** HTTPS hợp lệ **ngay trên VPS**, để Cloudflare kết nối an toàn với origin bằng chế độ **Full (strict)**. Không dùng Cloudflare Flexible cho app có login, đơn hàng hoặc thanh toán.

### Bước 10.1. Điều kiện cấp chứng chỉ

- DNS `example.com` và (nếu sử dụng) `www.example.com` trỏ đúng VPS.
- Nginx đang nhận request cổng `80`.
- Firewall VPS và nhà cung cấp mở TCP `80`/`443`.
- Port `80` từ Internet truy cập được; Let's Encrypt phải xác thực quyền sở hữu domain.
- Đang để **DNS only** khi cấp lần đầu để giảm yếu tố gây nhiễu.

### Bước 10.2. Cài Certbot (Snap)

```bash
sudo apt install -y snapd
sudo snap install --classic certbot
```

Nếu lệnh `certbot` chưa được tìm thấy và `/usr/local/bin/certbot` chưa tồn tại:

```bash
sudo ln -s /snap/bin/certbot /usr/local/bin/certbot
```

Kiểm tra:

```bash
certbot --version
```

Nếu Certbot đã được cài bằng APT/Snap trước đó thì nên kiểm tra và dùng **một phương pháp quản lý duy nhất**, không cài trùng.

### Bước 10.3. Cấp chứng chỉ cho domain

Ví dụ dùng cả `example.com` và `www.example.com`:

```bash
sudo certbot --nginx \
  -d example.com \
  -d www.example.com \
  --no-redirect
```

Nếu **chỉ dùng domain gốc**:

```bash
sudo certbot --nginx -d example.com --no-redirect
```

Nhập email nhận cảnh báo gia hạn và đồng ý điều khoản khi được hỏi. `--no-redirect` có nghĩa Certbot **không tự ép HTTP → HTTPS trên Nginx**. Ở bước 11 sẽ dùng **Cloudflare Always Use HTTPS** để redirect tại edge.

**Không được bật Full (strict) trước khi origin có certificate hợp lệ**, nếu không người dùng có thể gặp lỗi 526.

### Bước 10.4. Kiểm tra certificate và Nginx

```bash
sudo certbot certificates
sudo nginx -t
sudo systemctl reload nginx
curl -I https://example.com
```

Nếu đang DNS only, trình duyệt đã có thể truy cập `https://example.com` trực tiếp tới Nginx qua HTTPS.

### Bước 10.5. Kiểm tra tự gia hạn

```bash
sudo certbot renew --dry-run
systemctl list-timers --all | grep -E 'certbot|snap.certbot' || true
```

Tên timer có thể khác tùy phiên bản. Một số bản Certbot dùng snap timer/cron thay vì `certbot.timer`; hãy xác nhận cơ chế gia hạn hiện có, **không tự tạo timer trùng**. Sau khi bật Cloudflare Proxy ở bước 11, nên chạy lại `renew --dry-run` để kiểm tra luồng xác thực HTTP-01.

### Bước 10.6. Kiểm tra origin không đi qua Cloudflare

Từ máy có `curl` và kết nối đến VPS:

```bash
curl --resolve example.com:443:203.0.113.10 -I https://example.com
```

Lệnh này kết nối thẳng IP VPS nhưng vẫn yêu cầu chứng chỉ đúng tên `example.com`. Nếu IP chặn truy cập ngoài hoặc dùng cơ chế bảo vệ origin riêng, kiểm tra tương ứng.

> **Một phương án khác:** Cloudflare Origin CA certificate cũng có thể dùng với Full (strict), nhưng trình duyệt không trực tiếp tin cậy Origin CA khi truy cập vào VPS bỏ qua Cloudflare. Hướng dẫn này chọn Let's Encrypt để website vẫn có HTTPS tin cậy khi tắt proxy.

---

## 11. Bật Cloudflare Proxy, Full (strict) và HTTPS

### Bước 11.1. Bật Proxy

Trong **Cloudflare → DNS → Records**:

- Record `A` của `@`: chuyển **DNS only → Proxied (đám mây cam)**.
- Record `CNAME` của `www` (nếu dùng): chuyển **DNS only → Proxied**.

Nhớ rằng khi proxy bật, `nslookup` thường trả về IP Cloudflare, **không phải IP VPS**.

### Bước 11.2. Chọn SSL/TLS Full (strict)

Cloudflare Dashboard → chọn domain → **SSL/TLS → Overview → Full (strict)**.

| Chế độ | Tác dụng | Khuyến nghị |
|---|---|---|
| Flexible | Người dùng → Cloudflare HTTPS, Cloudflare → origin HTTP | **Không dùng** cho ứng dụng production cần bảo mật |
| Full | Mã hóa đến origin, nhưng không kiểm tra certificate chặt chẽ | Chỉ là bước tạm trong trường hợp đặc biệt |
| **Full (strict)** | Mã hóa đến origin **và xác thực certificate** | **Nên dùng** |

Cloudflare Full (strict) yêu cầu origin HTTPS cổng 443 với certificate hợp lệ, chưa hết hạn và khớp hostname. Nếu không sẽ báo **526**.

### Bước 11.3. Bật Always Use HTTPS

Vào **SSL/TLS → Edge Certificates → Always Use HTTPS → ON**.

Với cấu hình mẫu này, HTTP → HTTPS được redirect tại Cloudflare edge. Điều này phù hợp với khuyến cáo Cloudflare là hạn chế chồng nhiều tầng redirect. **Không bật SSL Flexible** khi Nginx/app ép HTTPS vì dễ gây redirect loop.

**Nếu cần ép HTTPS ngay cả khi truy cập origin trực tiếp:** cấu hình redirect HTTP → HTTPS ở Nginx **sau khi** đã cấp certificate, rồi kiểm tra kỹ không tạo vòng lặp; lúc đó có thể tắt Always Use HTTPS và để origin chịu trách nhiệm redirect. Chỉ chọn **một nơi điều khiển redirect chính** để dễ quản lý.

### Bước 11.4. Kiểm tra hoạt động

```bash
curl -I http://example.com
curl -I https://example.com
```

Kỳ vọng:

- `http://example.com` → chuyển sang `https://example.com` (thường 301/308).
- `https://example.com` → trả response hợp lệ (200 hoặc redirect hợp lệ của app).
- Không có lặp redirect vô hạn.
- Browser báo kết nối an toàn.

### Bước 11.5. Lưu ý về Cloudflare với ứng dụng có dữ liệu realtime

- **Không bật quy tắc “Cache Everything” tùy tiện** cho API, trang đăng nhập, giỏ hàng, đơn hàng, thanh toán, trạng thái phiên hoặc nội dung cá nhân. Nếu đang có cache rule mở rộng, phải tạo bypass chính xác cho các đường dẫn động.
- Với WebSocket: Nginx cần forward `Upgrade`/`Connection` như bước 9; Cloudflare hỗ trợ proxy WebSocket, nhưng kết nối vẫn cần cơ chế reconnect/fallback khi mất mạng.
- Với webhook thanh toán: route phải truy cập công khai theo yêu cầu provider và **xác minh chữ ký/webhook secret ở backend**; không dựa riêng vào Cloudflare để xác thực thanh toán.
- Với Supabase Realtime: phân biệt kết nối **browser → Supabase** và **browser → VPS**, cấu hình origin/websocket/CORS theo đúng kiến trúc hệ thống.
- Không cache dữ liệu giao dịch chỉ để cải thiện tốc độ.

---

## 12. Kiểm tra trước khi đưa website vào sử dụng

### 12.1. Test VPS, container và Nginx

```bash
sudo docker compose -f /opt/myapp/app/compose.yaml --project-directory /opt/myapp/app ps
sudo docker ps
sudo systemctl is-active nginx
sudo nginx -t
sudo ufw status verbose
```

Nếu lệnh Compose có phụ thuộc môi trường theo thư mục hiện tại, đơn giản nhất là chạy:

```bash
cd /opt/myapp/app
sudo docker compose ps
```

### 12.2. Test chức năng

- Truy cập website từ máy tính và điện thoại bằng 4G/5G (không chỉ Wi-Fi cùng mạng).
- Kiểm tra trang chính, đăng nhập, phân quyền, đăng xuất.
- Kiểm tra các request API, upload, ảnh, âm thanh và CORS.
- Với app có realtime, thử hai thiết bị và kiểm tra reconnect.
- Với app có thanh toán, xác minh callback, webhook, idempotency và trạng thái đơn từ nguồn dữ liệu thực.
- Kiểm tra URL tuyệt đối trong email, OAuth redirect, payment return URL và cookie HTTPS.
- Reload trang con hoặc mở trực tiếp URL con để xác nhận router hoạt động.
- Test sau khi reboot VPS theo kế hoạch, đảm bảo ứng dụng tự chạy lại.

### 12.3. Test hệ thống sau reboot (chỉ làm khi đã chuẩn bị downtime phù hợp)

```bash
sudo reboot
```

SSH lại sau khi VPS khởi động, sau đó:

```bash
sudo docker ps
sudo systemctl status docker nginx --no-pager
curl -I https://example.com
```

---

## 13. Quy trình update code trên VPS

### 13.1. Update theo cách đơn giản, dễ làm

Trên **máy local**, commit và push:

```bash
git status
git add .
git commit -m "Update production app"
git push origin main
```

Trên **VPS**, trước khi deploy:

```bash
cd /opt/myapp/app

git status
git rev-parse --short HEAD
git log -1 --oneline
```

**Ghi lại commit đang chạy** để có thể rollback. Đảm bảo working tree sạch trước khi pull. Nếu có chỉnh tay source trên VPS, hãy kiểm tra/commit hoặc sao lưu có chủ đích, không dùng `git reset --hard` một cách mù quáng.

Sau đó:

```bash
git fetch origin
git pull --ff-only origin main
sudo docker compose config --quiet
sudo docker compose up -d --build
sudo docker compose ps
sudo docker compose logs --tail=100 web
curl -I https://example.com
```

**Giải thích:** `git pull` chỉ cập nhật source, **không thay đổi ứng dụng đang chạy cho đến khi container được rebuild/recreate**. `docker compose up -d --build` tạo image mới nếu cần và chạy phiên bản mới.

> **Cảnh báo:** Quy trình này có thể có vài giây downtime vì container cũ được thay bằng container mới. Nếu cần zero-downtime, hãy áp dụng blue-green deployment, nhiều replica phía sau load balancer hoặc một nền tảng orchestration thích hợp; không nên mặc định Compose trên một VPS có zero-downtime.

### 13.2. Update với build và restart tách riêng

```bash
cd /opt/myapp/app
sudo docker compose build web
sudo docker compose up -d --no-deps web
```

Cách này cho phép biết lỗi build **trước khi** thay container đang chạy, nhưng vẫn không thay thế quy trình kiểm thử, database migration và rollback. Docker Compose chính thức cũng mô tả luồng build → `up --no-deps` cho production.

### 13.3. Nếu image lấy từ registry, không build source trên VPS

Khi Compose dùng `image: ghcr.io/OWNER/myapp:TAG` mà **không có `build`**:

```bash
cd /opt/myapp/app
sudo docker compose pull
sudo docker compose up -d
```

Khuyến nghị pin **tag theo version hoặc commit SHA**, tránh dựa hoàn toàn vào `latest` khi cần rollback chính xác. Nếu dùng private registry, đăng nhập bằng token có phạm vi đọc tối thiểu và tránh ghi credential vào script/repository.

---

## 14. Rollback nếu bản cập nhật bị lỗi

### Tình huống 1: Chỉ lỗi code/app, chưa đổi schema/database

Lấy lại commit đã ghi trước deploy, ví dụ `abc1234`:

```bash
cd /opt/myapp/app
git fetch origin
git checkout abc1234
sudo docker compose build web
sudo docker compose up -d --no-deps web
sudo docker compose logs --tail=100 web
```

Sau khi xử lý xong, quay về nhánh để tiếp tục quy trình thông thường:

```bash
git checkout main
# Chỉ pull/deploy lại khi đã có bản sửa được kiểm thử.
```

**Lưu ý:** `git checkout <commit>` tạo detached HEAD — bình thường khi rollback tạm thời. Không `git pull` khi chưa kiểm tra trạng thái nhánh.

### Tình huống 2: Update kèm database migration

**Không đảm bảo rollback chỉ bằng `git checkout`.** Một schema migration đã chạy có thể không tương thích với code cũ. Cần có:

1. Bản backup database trước khi migrate.
2. Migration đã được review và tested trên staging.
3. Chiến lược tương thích ngược (expand → migrate → contract) hoặc kế hoạch khôi phục cụ thể.
4. Dữ liệu phát sinh sau khi backup cần phương án đối soát nếu phải restore.

**Không** tự ý chạy `docker compose down -v`, `docker volume prune`, `prisma migrate reset` hay xóa database để rollback production.

---

## 15. Database, migrations và dữ liệu cần lưu

### 15.1. Nếu database đặt ngoài VPS (ví dụ Supabase Managed Postgres)

- Dùng `DATABASE_URL`/credential production lưu ở secrets, không hardcode.
- Kiểm tra kết nối TCP/TLS, cấu hình pooler và giới hạn kết nối của nhà cung cấp.
- Review migration **trước** deploy; chỉ chạy khi đã rõ tác động schema.
- Supabase Storage hoặc Cloudflare R2/S3 là dịch vụ riêng: xóa container Docker **không nhất thiết** xóa dữ liệu ở đó.
- Sao lưu database/cloud storage theo chính sách retention thực tế; không mặc định gói miễn phí có đầy đủ backup/PITR.

### 15.2. Nếu Postgres/MySQL chạy trong Docker trên cùng VPS

Phải khai báo **named volume** hoặc persistent volume cho data directory. Ví dụ *chỉ minh họa*, không tự thêm nếu ứng dụng đã có database:

```yaml
services:
  db:
    image: postgres:17
    restart: unless-stopped
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: myapp
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?set_in_compose_environment}
    volumes:
      - pgdata:/var/lib/postgresql/data
    # KHÔNG publish 5432 ra Internet nếu không thực sự cần

volumes:
  pgdata:
```

Với Postgres phiên bản mới, kiểm tra mount path được image chính thức yêu cầu; ví dụ trên dành cho `postgres:17`. Biến `${POSTGRES_PASSWORD}` trong Compose phải có từ shell/`.env` **của Compose**, không tự động lấy từ `env_file` của một service.

### 15.3. Chạy migrations an toàn

- Đọc migration, đánh giá lock, ảnh hưởng dữ liệu và thời gian chạy.
- Có backup và kế hoạch rollback trước khi chạy.
- Chỉ chạy một lần theo release, tránh nhiều replica cùng migrate.
- Nếu dùng Prisma, lệnh production thường là `prisma migrate deploy`, nhưng **chỉ dùng khi image/môi trường có Prisma CLI và migration files**.
- Nếu dùng Supabase CLI, làm theo workflow và trạng thái migration của project; không tự động chạy lệnh reset/force vào remote production.
- Kiểm tra app tương thích schema trước và sau deploy.

### 15.4. Hiểu đúng các lệnh Compose

| Lệnh | Tác dụng | Có nên dùng production? |
|---|---|---|
| `docker compose stop` | Dừng container | Có, khi cần bảo trì |
| `docker compose start` | Chạy lại container đã tạo | Có |
| `docker compose restart web` | Restart web, **không rebuild code mới** | Có, khi cần restart |
| `docker compose up -d --build` | Build, create/recreate khi cần, chạy nền | Có, sau QA |
| `docker compose down` | Gỡ containers/networks của project, **giữ named volumes mặc định** | Cẩn trọng với downtime |
| `docker compose down -v` | Gỡ containers và named volumes | **Không dùng** nếu dữ liệu cần giữ |
| `docker system prune -a --volumes` | Xóa dữ liệu Docker không sử dụng, có thể làm mất dữ liệu/rollback | **Không dùng tùy tiện** |

---

## 16. Backup, log, bảo trì và giám sát

### 16.1. Các loại backup cần phân biệt

1. **Source code:** Git repository, tag/commit release.
2. **Cấu hình production:** `compose.yaml`, Nginx vhost, cấu hình Cloudflare DNS/TLS, biến môi trường được lưu an toàn.
3. **Database:** backup logic hoặc snapshot phù hợp với hệ quản trị DB; cần kiểm tra tính nhất quán và khả năng restore.
4. **Uploads/files:** Docker volume hoặc bucket lưu object (S3/R2/Supabase Storage...).
5. **TLS:** Certbot có cơ chế gia hạn; backup certificate/key cần bảo vệ như secret, thường có thể cấp lại nếu DNS/ACME vẫn hoạt động.

**Không coi VPS snapshot là chiến lược backup duy nhất**; nên có bản sao ở vị trí lưu trữ độc lập, mã hóa và kiểm tra restore định kỳ.

### 16.2. Backup source/config nhanh (không bao gồm database)

```bash
sudo tar --exclude=.git -czf /root/myapp-config-$(date +%F).tar.gz \
  /opt/myapp/app/compose.yaml \
  /opt/myapp/app/.env.production \
  /etc/nginx/sites-available/myapp
```

**File backup trên chứa secret**. Giữ quyền root, mã hóa trước khi chuyển ra nơi lưu trữ, không upload công khai. Lệnh sẽ báo lỗi nếu đường dẫn không tồn tại; sửa theo project thực tế. Backup này **không thay thế backup database hoặc Docker volume**.

### 16.3. Theo dõi log

```bash
cd /opt/myapp/app
sudo docker compose logs --tail=200 web
sudo docker compose logs -f web
sudo journalctl -u nginx -n 100 --no-pager
sudo tail -n 100 /var/log/nginx/error.log
sudo tail -n 100 /var/log/nginx/access.log
```

Cẩn thận vì application logs có thể chứa token, email, cookie hoặc thông tin người dùng nếu code logging sai. Tránh gửi nguyên log nhạy cảm ra ngoài.

### 16.4. Kiểm tra tài nguyên

```bash
free -h
df -h
sudo docker stats --no-stream
sudo docker system df
sudo ss -tulpn
```

Nếu Docker image/log chiếm nhiều dung lượng, tìm nguyên nhân trước khi xóa. Cấu hình log rotation cho Docker (`local` driver hoặc `json-file` giới hạn dung lượng) cần được lên kế hoạch phù hợp; thay đổi Docker daemon config có thể cần restart Docker và gây gián đoạn.

### 16.5. Giám sát

Nên có ít nhất: HTTPS uptime check, cảnh báo CPU/RAM/disk, certificate expiry, error rate (5xx), và kiểm tra chức năng quan trọng. Không chỉ dựa vào `docker ps`: container có thể vẫn chạy trong khi API/database đã hỏng.

---

## 17. Bảo mật production và Cloudflare

### Trên VPS

- SSH bằng key; bật MFA tại nhà cung cấp VPS và Cloudflare/GitHub.
- Không công khai port ứng dụng, database, Redis hoặc Docker daemon.
- Giới hạn quyền filesystem và secrets; file `.env.production` để `chmod 600`.
- Dùng container non-root nếu app hỗ trợ.
- Cập nhật bản vá Ubuntu/Docker có kế hoạch; test trước khi thay production.
- Dùng image nguồn tin cậy, pin version khi cần khả năng tái lập.
- Chạy backup và **test khôi phục**; có chính sách lưu và xóa backup.
- Tránh cấp quyền write cho GitHub deploy key nếu không cần.
- Nếu chỉ proxy web qua Cloudflare, có thể giới hạn cổng 80/443 của origin cho Cloudflare IP ranges bằng firewall thích hợp, **nhưng phải cập nhật IP ranges và đảm bảo ACME/monitoring/origin access không bị chặn**. Không áp dụng mù quáng với UFW khi Docker có published ports.

### Trên Cloudflare

- **SSL/TLS = Full (strict)**.
- Proxy web DNS record = **Proxied**.
- **Always Use HTTPS** sau khi certificate edge/origin hoạt động.
- Bật WAF/rate limiting phù hợp nhu cầu và gói đăng ký; thiết lập exception có kiểm soát cho webhook hoặc API nếu bị chặn sai.
- Không dùng rule cache HTML/API chứa dữ liệu riêng tư của người dùng.
- Bảo vệ tài khoản Cloudflare bằng MFA và phân quyền tối thiểu.
- Nếu muốn log IP người dùng thật qua Nginx, phải cấu hình `real_ip_header CF-Connecting-IP` **kèm dải IP Cloudflare tin cậy** và cơ chế cập nhật; không tin tưởng header do client bất kỳ tự gửi.
- Chỉ bật HSTS sau khi toàn bộ domain/subdomain sử dụng HTTPS ổn định và đã hiểu rủi ro của `includeSubDomains`/`preload`.

### Về ứng dụng

- Kiểm tra CORS theo exact origin thật; không wildcard cho credentialed requests.
- Cookie auth dùng `Secure`, `HttpOnly`, `SameSite` phù hợp và domain/path chính xác.
- Webhook thanh toán có verify signature + xử lý trùng event + log đối soát.
- Không lộ service key nhạy cảm vào frontend hoặc public `NEXT_PUBLIC_*`.
- Đối với realtime/đơn hàng, enforce authorization **ở backend**, không chỉ ẩn nút trên UI.

---

## 18. Khắc phục các lỗi thường gặp

### 18.1. Domain không vào được / DNS không phân giải

**Kiểm tra:**

```bash
nslookup example.com
curl -I http://example.com
```

**Nguyên nhân thường gặp:** Domain chưa Active trên Cloudflare, nameserver chưa đổi tại registrar, A/AAAA sai IP, DNS record bị trùng hoặc đã đổi nhưng đang chờ TTL/cache.

### 18.2. `502 Bad Gateway`

**Ý nghĩa:** Nginx (hoặc Cloudflare khi báo bad gateway) không nhận response hợp lệ từ upstream.

```bash
sudo docker ps
cd /opt/myapp/app
sudo docker compose logs --tail=100 web
curl -I http://127.0.0.1:3000
sudo tail -n 100 /var/log/nginx/error.log
```

**Khả năng:** Container crash, app chưa listen `0.0.0.0:3000` bên trong container, cấu hình sai host port, build chưa xong, Nginx `proxy_pass` trỏ nhầm cổng.

### 18.3. Cloudflare `521 Web Server Is Down`

Cloudflare không thiết lập được kết nối tới origin. Kiểm tra Nginx đang chạy, IP DNS đúng, firewall không chặn Cloudflare, cổng `80/443` mở. Nếu origin chỉ cho IP nhất định thì phải cấu hình đúng dải IP Cloudflare.

```bash
sudo systemctl status nginx --no-pager
sudo ss -tulpn | grep -E ':(80|443)\b'
sudo ufw status
```

### 18.4. Cloudflare `522 Connection Timed Out`

Kết nối tới origin timeout. Kiểm tra VPS có bị treo/CPU quá tải, firewall/provider security group, routing/network và IP origin. Nếu xảy ra theo tải cao, xem số request, kết nối và upstream.

### 18.5. Cloudflare `525 SSL Handshake Failed`

Lỗi bắt tay TLS giữa Cloudflare và origin. Kiểm tra Nginx `listen 443 ssl`, chứng chỉ, private key, giao thức TLS, firewall và SNI. Dùng:

```bash
sudo nginx -t
sudo certbot certificates
curl --resolve example.com:443:203.0.113.10 -I https://example.com
```

### 18.6. Cloudflare `526 Invalid SSL Certificate`

Thường xuất hiện ở **Full (strict)** khi chứng chỉ origin hết hạn, không khớp hostname hoặc không được chấp nhận. Kiểm tra `certbot certificates`, domain nào được cấp, thời hạn và cấu hình certificate trong Nginx. **Không coi chuyển Flexible là cách sửa production.**

### 18.7. `ERR_TOO_MANY_REDIRECTS`

Thường do Cloudflare Flexible + Nginx/app ép HTTPS hoặc nhiều redirect không thống nhất giữa edge, Nginx và ứng dụng. Chọn **Full (strict)** sau khi origin HTTPS hợp lệ; xem `Location` header:

```bash
curl -IL --max-redirs 10 https://example.com
```

Nếu dùng custom redirect ở Cloudflare, kiểm tra không redirect vòng lại HTTP hoặc sang hostname đối nghịch.

### 18.8. Cổng 3000 bị chiếm / ứng dụng đổi sang 3001

```bash
sudo ss -tulpn | grep ':3000'
sudo docker ps --format 'table {{.Names}}\t{{.Ports}}'
```

Không để framework tự nhảy port trong production mà Nginx vẫn trỏ cổng cũ. Cấu hình cố định host/container port và kiểm tra ứng dụng khởi động fail rõ ràng khi port không khả dụng.

### 18.9. Docker build lỗi `npm ci` / build Next.js

- Kiểm tra có `package-lock.json` khớp `package.json`.
- Kiểm tra Node.js version trong image tương thích dự án.
- Kiểm tra script `build`, native package, Prisma generate, file cấu hình build.
- Kiểm tra `NEXT_PUBLIC_*` cần thiết tại build time.
- Với build thiếu RAM, xem `free -h`/kernel OOM logs; không mặc định là lỗi ứng dụng.

### 18.10. WebSocket không hoạt động / realtime bị rớt

- Kiểm tra Nginx proxy `Upgrade`/`Connection`, `proxy_http_version 1.1`.
- Kiểm tra đúng URL `wss://`/`https://` và policy kết nối.
- Kiểm tra app đã xử lý reconnect, token auth, WebSocket path, heartbeat và fallback.
- Phân biệt lỗi server-side WebSocket với Supabase channel kết nối đến dịch vụ khác.

### 18.11. Tải ảnh/upload lỗi `413 Request Entity Too Large`

`client_max_body_size` của Nginx có thể quá nhỏ; Cloudflare cũng có hạn mức riêng. Không thể sửa một giới hạn Cloudflare chỉ bằng tăng Nginx. Với file lớn, cân nhắc upload trực tiếp đến object storage qua signed URL được cấp an toàn.

### 18.12. Có code mới nhưng website vẫn là bản cũ

```bash
cd /opt/myapp/app
git log -1 --oneline
sudo docker compose images
sudo docker compose ps
sudo docker compose logs --tail=50 web
```

Nguyên nhân: mới `git pull` nhưng chưa build/recreate, đang deploy sai thư mục/branch, biến build Next.js sai, CDN/browser/service worker cache. **Không purge toàn bộ Cloudflare cache tùy tiện** nếu lỗi do code hoặc session; kiểm tra chính xác trước.

---

## 19. Bảng lệnh quản trị nhanh

**Các lệnh dưới đây chạy trong `/opt/myapp/app` trừ khi có đường dẫn rõ ràng.**

| Việc cần làm | Lệnh |
|---|---|
| Xem container đang chạy | `sudo docker ps` |
| Xem service Compose | `sudo docker compose ps` |
| Xem log web | `sudo docker compose logs -f web` |
| Restart web | `sudo docker compose restart web` |
| Tắt service web | `sudo docker compose stop web` |
| Chạy lại service web | `sudo docker compose start web` |
| Rebuild/redeploy | `sudo docker compose up -d --build` |
| Kiểm tra Compose | `sudo docker compose config --quiet` |
| Kiểm tra Nginx | `sudo nginx -t` |
| Reload Nginx | `sudo systemctl reload nginx` |
| Xem lỗi Nginx | `sudo tail -f /var/log/nginx/error.log` |
| Kiểm tra certificate | `sudo certbot certificates` |
| Test gia hạn SSL | `sudo certbot renew --dry-run` |
| Kiểm tra firewall | `sudo ufw status verbose` |
| Kiểm tra cổng | `sudo ss -tulpn` |
| Xem CPU/RAM container | `sudo docker stats --no-stream` |
| Xem disk | `df -h` |
| Xem commit hiện tại | `git rev-parse --short HEAD` |
| Xem trạng thái repository | `git status` |

### Câu lệnh update nhanh sau khi hệ thống đã ổn định

```bash
cd /opt/myapp/app && \
git pull --ff-only origin main && \
sudo docker compose build web && \
sudo docker compose up -d --no-deps web && \
sudo docker compose ps && \
sudo docker compose logs --tail=50 web
```

**Chỉ chạy câu lệnh nhanh khi đã review code, backup và migration cần thiết.** Không áp dụng mù quáng với database migration hoặc app có quy trình deploy riêng.

---

## 20. Checklist nghiệm thu production

### Hạ tầng

- [ ] VPS truy cập SSH bằng key và có quyền sudo.
- [ ] UFW/cloud firewall mở SSH, HTTP, HTTPS theo nhu cầu.
- [ ] Docker Engine + Compose chạy ổn định.
- [ ] Dung lượng disk/RAM đủ cho build/runtime.

### Ứng dụng

- [ ] `docker compose config --quiet` thành công.
- [ ] `docker compose ps` hiển thị service đang Up.
- [ ] `curl -I http://127.0.0.1:3000` có response hợp lệ.
- [ ] App không mở công khai cổng 3000.
- [ ] Environment và secrets production đúng; không có secret trên Git.
- [ ] Database migration và restore plan đã được kiểm tra.

### Nginx + HTTPS

- [ ] `nginx -t` thành công.
- [ ] Virtual host trỏ đúng domain/port.
- [ ] `certbot certificates` chứa domain cần sử dụng và chưa hết hạn.
- [ ] `certbot renew --dry-run` thành công **sau khi bật Cloudflare proxy**.
- [ ] HTTPS khi gọi trực tiếp origin hợp lệ nếu không cố ý chặn direct-origin.

### Cloudflare

- [ ] Domain Active; nameserver đúng ở registrar.
- [ ] A/AAAA/CNAME không trỏ IP sai.
- [ ] Web records đang Proxied.
- [ ] SSL/TLS Full (strict).
- [ ] Always Use HTTPS hoạt động và không redirect loop.
- [ ] Không cache nội dung API/session/đơn hàng/thanh toán nhạy cảm.

### Vận hành

- [ ] Test từ mạng ngoài/điện thoại.
- [ ] App có backup database + upload với lịch hợp lý.
- [ ] Có health check và theo dõi cảnh báo.
- [ ] Ghi lại commit release và quy trình rollback.
- [ ] Reboot VPS xong website tự phục hồi.
- [ ] Webhook, auth, realtime, thanh toán được kiểm thử riêng (nếu có).

---

## 21. Tài liệu chính thức

Các nguồn dưới đây là điểm kiểm tra khi cú pháp lệnh hoặc Cloudflare Dashboard thay đổi:

1. Docker Engine trên Ubuntu: https://docs.docker.com/engine/install/ubuntu/
2. Docker Compose production: https://docs.docker.com/compose/how-tos/production/
3. Docker Compose plugin: https://docs.docker.com/compose/install/linux/
4. Docker và firewall/UFW: https://docs.docker.com/engine/install/ubuntu/#firewall-limitations
5. Cloudflare DNS và nameservers: https://developers.cloudflare.com/dns/zone-setups/full-setup/setup/
6. Cloudflare Proxied vs DNS only: https://developers.cloudflare.com/dns/proxy-status/
7. Cloudflare Full (strict): https://developers.cloudflare.com/ssl/origin-configuration/ssl-modes/full-strict/
8. Cloudflare Always Use HTTPS: https://developers.cloudflare.com/ssl/edge-certificates/additional-options/always-use-https/
9. Certbot + Nginx: https://certbot.eff.org/instructions?ws=nginx&os=snap
10. Next.js self-hosting: https://nextjs.org/docs/app/guides/self-hosting
11. Next.js deployment: https://nextjs.org/docs/app/getting-started/deploying
12. Nginx WebSocket reverse proxy: https://nginx.org/en/docs/http/websocket.html
13. GitHub Deploy Keys: https://docs.github.com/en/authentication/connecting-to-github-with-ssh/managing-deploy-keys

---

## TÓM TẮT QUY TRÌNH TỪ A–Z

```text
01. Mua VPS và có IP public
02. SSH, cập nhật Ubuntu, mở SSH/80/443 đúng thứ tự
03. Cài Docker Engine + Docker Compose
04. Thêm domain vào Cloudflare hoặc kiểm tra domain Active
05. Tạo A record trỏ về VPS, tạm DNS only
06. Clone repo, cấu hình .env.production an toàn
07. Kiểm tra Dockerfile/Compose, build và chạy app
08. Test ứng dụng ở http://127.0.0.1:3000
09. Cấu hình Nginx proxy domain → 127.0.0.1:3000
10. Test HTTP bằng domain
11. Cấp Let's Encrypt HTTPS bằng Certbot
12. Test HTTPS và certificate tại origin
13. Bật Cloudflare Proxied + Full (strict) + Always Use HTTPS
14. Test website từ Internet và các chức năng quan trọng
15. Cấu hình backup, logs, giám sát, tự gia hạn certificate
16. Update bằng git pull + docker compose build/up
17. Ghi commit release và rollback đúng quy trình khi lỗi
```

**Kết quả mong đợi:** website truy cập qua `https://example.com`, Cloudflare quản lý DNS/Proxy, VPS dùng Nginx làm reverse proxy, app Docker ở cổng nội bộ và có quy trình vận hành production rõ ràng.

> **Chốt lại:** Đây là **runbook hướng dẫn**, không phải script tự động đã được chạy trên VPS của bạn. Trước khi áp dụng cho project cụ thể, cần đối chiếu Dockerfile, Compose, hostname, secrets, migrations và các service thực tế của repo.
