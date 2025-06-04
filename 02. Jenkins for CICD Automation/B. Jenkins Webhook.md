# Jenkins Lab: Git Integration và Webhook Trigger Build

## 1. Kiến trúc tổng quan

```
Developer Push Code
        ↓
   Git Server (GitHub/GitLab)
        ↓ (Webhook)
    Jenkins Controller
        ↓
     Agent (SSH/JNLP)
        ↓
     Build/Test/Deploy
```

---

## 2. Chuẩn bị trước

- Jenkins Controller chạy tại: `http://192.168.100.10:8080`
- Git repository: `https://github.com/yourname/demo-jenkins.git` hoặc GitLab repo tương ứng

---

## 3. Cấu hình SSH Key giữa Jenkins và Git Server

### 3.1. Tạo SSH key trong Jenkins

```bash
sudo su - jenkins
ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub
```

### 3.2. Thêm key vào GitHub/GitLab

- GitHub: Settings → SSH and GPG keys → New SSH key
- GitLab: Preferences → SSH Keys

### 3.3. Test kết nối

```bash
ssh -T git@github.com
```

---

## 4. Thêm Credentials SSH Key trong Jenkins

- Manage Jenkins → Credentials → System → Global credentials
- Add → SSH Username with private key
  - ID: `git-key`
  - Username: `git`
  - Private Key: từ `~/.ssh/id_rsa`

---

## 5. Cài plugin hỗ trợ Git và Webhook

Cài đặt các plugin:

- Git plugin
- GitHub plugin (hoặc GitLab plugin)
- Pipeline plugin

---

## 6. Tạo Pipeline project

### 6.1. Thêm `Jenkinsfile` vào repo:

```groovy
pipeline {
    agent any
    triggers {
        githubPush()
    }
    stages {
        stage('Build') {
            steps {
                echo "Building code..."
            }
        }
        stage('Test') {
            steps {
                echo "Testing..."
            }
        }
    }
}
```

Push file này vào repository.

---

### 6.2. Cấu hình Jenkins Pipeline

- New Item → demo-pipeline → Pipeline
- Definition: Pipeline script from SCM
  - SCM: Git
  - URL: `git@github.com:yourname/demo-jenkins.git`
  - Credentials: `git-key`

---

## 7. Cấu hình Webhook (GitHub/GitLab)

### 7.1. Jenkins public

Webhook URL:

```
http://jenkins.example.com/github-webhook/
```

### 7.2. Jenkins nội bộ → dùng ngrok:

```bash
ngrok http 8080
```

Webhook URL: ví dụ `http://abc123.ngrok.io/github-webhook/`

---

### 7.3. Cấu hình GitHub Webhook

- Settings → Webhooks → Add Webhook
  - Payload URL: `http://jenkins.example.com/github-webhook/`
  - Content type: application/json
  - Trigger: Just the push event

---

### 7.4. Cấu hình GitLab Webhook

- Settings → Webhooks
  - URL: `http://jenkins.example.com/project/demo-pipeline`
  - Trigger: Push Events

---

## 8. Bật Webhook Trigger trong Jenkins Job

### GitHub

- Build Triggers: 
  - [x] GitHub hook trigger for GITScm polling

### GitLab

- Dùng GitLab plugin:
  - [x] Build when a change is pushed to GitLab

---

## 9. Kiểm tra toàn bộ quy trình

1. Push commit mới lên Git
2. Webhook gọi về Jenkins
3. Jenkins tự động trigger job
4. Kiểm tra log: stage Clone, Build, Test
