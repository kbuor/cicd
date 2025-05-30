# Thực hành CI/CD cơ bản: Jenkins & GitHub Actions

> Mục tiêu: Hướng dẫn cài đặt và cấu hình hệ thống CI đơn giản bằng Jenkins (tự host) và GitHub Actions (hosted), trên nền Ubuntu 24.04.

---

## A. Jenkins – Thiết lập CI Pipeline cơ bản (Self-hosted)

### 1. Cài đặt Jenkins trên Ubuntu 24.04

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install openjdk-17-jdk -y
```

**Thêm Jenkins repo và cài Jenkins:**

```bash
curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install jenkins -y
```

**Khởi động Jenkins:**

```bash
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

### 2. Truy cập Jenkins

- Truy cập trình duyệt: `http://<IP_VM>:8080`
- Lấy password ban đầu:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

- Làm theo hướng dẫn để tạo user admin và cài các plugin đề xuất.

---

### 3. Tạo Pipeline Job cơ bản

- Vào Jenkins dashboard → `New Item`
- Đặt tên: `demo-ci`
- Chọn `Pipeline` → OK

Trong phần **Pipeline > Definition**, chọn `Pipeline script` và dán nội dung sau:

```groovy
pipeline {
  agent any

  stages {
    stage('Build') {
      steps {
        echo 'Building...'
      }
    }
    stage('Test') {
      steps {
        echo 'Running tests...'
      }
    }
    stage('Deploy') {
      steps {
        echo 'Deploying...'
      }
    }
  }
}
```

**Bấm “Build Now”** để chạy thử pipeline.

---

### Kết quả đạt được:

- Jenkins hoạt động trên VM Ubuntu
- Tạo pipeline `demo-ci` với 3 bước: Build → Test → Deploy
- Hiển thị log kết quả từng bước trên Jenkins UI

---

## B. GitHub Actions – Tạo CI Pipeline trên GitHub

### 1. Tạo repository GitHub mới

- Truy cập [https://github.com](https://github.com) → New repository
- Đặt tên: `demo-ci-github-actions`

**Clone repo về local:**

```bash
git clone https://github.com/<your-user>/demo-ci-github-actions.git
cd demo-ci-github-actions
```

---

### 2. Tạo workflow file

Tạo thư mục và file workflow:

```bash
mkdir -p .github/workflows
nano .github/workflows/ci.yml
```

Dán nội dung sau:

```yaml
name: Simple CI Pipeline

on:
  push:
    branches: [ "main" ]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Build
      run: echo "Building..."

    - name: Test
      run: echo "Running tests..."

    - name: Deploy
      run: echo "Deploying (simulated)..."
```

---

### 3. Commit và đẩy code

```bash
git add .
git commit -m "Add CI pipeline with GitHub Actions"
git push origin main
```

- Truy cập tab **Actions** trên GitHub repo để theo dõi pipeline đang chạy.

---

### Kết quả đạt được:

- GitHub Actions tự động chạy khi có push lên nhánh `main`
- Pipeline gồm các bước Build → Test → Deploy
- Log được hiển thị trực tiếp trên GitHub UI

---

## 🏁 Tổng kết

| Công cụ | Jenkins | GitHub Actions |
|--------|---------|----------------|
| Hosting | Tự host (Ubuntu VM) | Hosted trên GitHub |
| Pipeline | Viết bằng Groovy (Jenkinsfile) | Viết bằng YAML |
| Giao diện | Web UI riêng | Tích hợp sẵn trong GitHub |
| Mục tiêu | CI cơ bản tự vận hành | CI cơ bản cho repo GitHub |

---

**Next step (Gợi ý mở rộng):**
- Thêm real unit test (Python, Node.js, Java...)
- Build Docker image trong pipeline
- Triển khai lên staging
- Gắn Jenkins với GitHub Webhook để trigger tự động
