# Jenkins Lab: Plugin Management và Security Best Practices

---

## 1. Kiến trúc Lab

| Role             | Hostname         | IP             | OS          | Ghi chú                  |
|------------------|------------------|----------------|-------------|---------------------------|
| Jenkins Master   | jenkins-secure   | 192.168.100.10 | Ubuntu 24.04| Sử dụng port 8080 (HTTP) |

---

## 2. Quản lý Plugin (Plugin Management)

### 2.1. Truy cập Plugin Manager

- Jenkins UI → Manage Jenkins → Plugin Manager

### 2.2. Cài đặt các plugin thiết yếu

| Plugin                        | Mục đích                                          |
|-------------------------------|---------------------------------------------------|
| Matrix Authorization Strategy | Quản lý phân quyền chi tiết                      |
| Role-based Authorization      | Tạo nhóm quyền theo role                         |
| Audit Trail                   | Ghi log thay đổi cấu hình Jenkins                |
| Configuration as Code (JCasC)| Quản lý Jenkins dưới dạng code                   |
| Blue Ocean                    | Giao diện trực quan cho Pipeline                 |
| Credentials Binding           | Bảo mật credential khi sử dụng pipeline          |
| OWASP Markup Formatter        | Ngăn XSS, định dạng HTML an toàn                 |

### 2.3. Gỡ plugin không cần thiết

- Plugin Manager → Installed → Gỡ bỏ các plugin không dùng

---

## 3. Cấu hình bảo mật (Security Best Practices)

---

### 3.1. Bắt buộc đăng nhập và bật bảo mật

1. Truy cập Jenkins UI → **Manage Jenkins**
2. Chọn **Configure Global Security**
3. Tích vào ô **Enable Security**
4. Chọn mục **Jenkins’ own user database**
   - Bỏ chọn **Allow users to sign up** nếu không muốn user tự đăng ký
5. Trong phần **Authorization**, chọn:
   - `Matrix-based security` để phân quyền chi tiết
   - Hoặc `Role-Based Strategy` nếu đã cài plugin tương ứng
6. Lưu lại cấu hình.

---

### 3.2. Tạo người dùng Jenkins

1. Vào **Manage Jenkins → Manage Users**
2. Chọn **Create User**
3. Nhập thông tin:
   - Username: `admin`, `dev`, `qa`
   - Password và Confirm Password
   - Full name và Email (tuỳ chọn)
4. Tạo từng user tương ứng với vai trò

---

### 3.3. Phân quyền theo vai trò (Matrix-based Security)

1. Vào **Manage Jenkins → Configure Global Security**
2. Trong phần Matrix Authorization Strategy:
   - Thêm từng user (admin, dev, qa)
3. Gán quyền như sau:

| User  | Quyền                             |
|--------|-----------------------------------|
| admin | Tất cả quyền (administer, build, etc.) |
| dev   | Read, Build, Workspace             |
| qa    | Chỉ Read                           |

4. Lưu lại cấu hình

---

### 3.4. Bảo vệ thông tin nhạy cảm trong Pipeline

1. Cài plugin **Credentials Binding**
2. Vào **Manage Jenkins → Credentials → Global**
   - Add Credentials → Secret text (ví dụ: `MY_SECRET`)
3. Trong `Jenkinsfile` sử dụng:

```groovy
pipeline {
  agent any
  environment {
    MY_SECRET = credentials('my-secret-id')
  }
  stages {
    stage('Secret Test') {
      steps {
        echo "Using secret: ${MY_SECRET}" > test_credentials.txt
      }
    }
  }
}
```

> Giá trị sẽ được **mask trong log** giúp ngăn rò rỉ thông tin

---

### 3.5. Kích hoạt bảo vệ CSRF

1. Vào **Manage Jenkins → Configure Global Security**
2. Cuộn xuống phần **CSRF Protection**
3. Tích chọn:
   - ✅ Prevent Cross Site Request Forgery exploits
4. Kiểm tra hoạt động sau khi bật

---

### 3.6. Cấu hình Reverse Proxy + HTTPS với Nginx

#### Cài đặt Nginx

```bash
sudo apt update
sudo apt install nginx -y
```

#### Tạo file cấu hình tại `/etc/nginx/sites-available/jenkins`

```nginx
server {
    listen 80;
    server_name jenkins.example.com;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name jenkins.example.com;

    ssl_certificate /etc/ssl/certs/jenkins.crt;
    ssl_certificate_key /etc/ssl/private/jenkins.key;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

#### Kích hoạt site và restart Nginx

```bash
sudo ln -s /etc/nginx/sites-available/jenkins /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

---

### 3.7. Giới hạn IP truy cập Jenkins (tuỳ chọn)

Trong block `location /` của Nginx:

```nginx
allow 192.168.100.0/24;
deny all;
```

---

Sau bước này, Jenkins sẽ:
- Chạy an toàn trên HTTPS
- Bảo vệ theo IP mạng nội bộ
- Áp dụng xác thực bắt buộc và phân quyền

---

## 4. Cài và cấu hình Audit Trail

### 4.1. Cài plugin `Audit Trail`

### 4.2. Cấu hình Log

- Log file: `/var/log/jenkins/audit.log`
- Ghi các hành động: thay đổi job, cài plugin, user login

```bash
tail -f /var/log/jenkins/audit.log
```

---

## 5. Sao lưu và phục hồi Jenkins

### 5.1. Sao lưu toàn bộ Jenkins

```bash
tar -czvf jenkins_backup_$(date +%F).tar.gz /var/lib/jenkins
```

### 5.2. Plugin ThinBackup (giao diện web)

- ThinBackup → Backup Now
- Lập lịch backup định kỳ

---

## 6. Jenkins as Code (JCasC)

### 6.1. Plugin `Configuration as Code`

### 6.2. File cấu hình mẫu

```yaml
jenkins:
  securityRealm:
    local:
      allowsSignup: false
      users:
        - id: "admin"
          password: "${ADMIN_PASSWORD}"
  authorizationStrategy:
    loggedInUsersCanDoAnything:
      allowAnonymousRead: false
```

---

## 7. Kiểm tra bảo mật

- Jenkins → Manage Jenkins → Security Warnings
- Kiểm tra:
  - Anon access
  - Script console mở
  - CSRF protection
  - Plugin lỗi thời

---

## 8. Kiểm thử cuối cùng

| Kiểm thử                             | Kết quả mong đợi                        |
|--------------------------------------|------------------------------------------|
| Truy cập trái phép (IP không hợp lệ) | Bị chặn                                  |
| Truy cập Jenkins qua HTTPS           | Thành công, chứng chỉ hợp lệ             |
| User `dev` không sửa job             | Bị từ chối quyền                         |
| Admin tạo job mới                    | Thành công                               |
| Ghi log thao tác cấu hình            | Có dòng mới trong `audit.log`           |
