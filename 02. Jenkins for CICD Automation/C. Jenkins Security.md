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

### 3.1. Bắt buộc đăng nhập

- Enable Security
- Use Jenkins own user database
- Authorization: Matrix-based hoặc Role-based Strategy

### 3.2. Tạo người dùng và phân quyền

- Tạo user: admin, dev, qa
- Phân quyền theo nhu cầu (admin: full, dev: read/build, qa: read-only)

### 3.3. Bảo vệ secret trong pipeline

```groovy
pipeline {
  agent any
  environment {
    SECRET = credentials('jenkins-secret-id')
  }
  stages {
    stage('Check') {
      steps {
        echo "Using secret"
      }
    }
  }
}
```

### 3.4. Kích hoạt CSRF Protection

- Configure Global Security → Bật “Prevent Cross Site Request Forgery exploits”

---

### 3.5. Reverse Proxy và HTTPS (Nginx)

#### Cài đặt và cấu hình Nginx

```bash
sudo apt install nginx
```

```nginx
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

### 3.6. Giới hạn IP truy cập

```nginx
allow 192.168.100.0/24;
deny all;
```

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
