
# Lab: Jenkins + Azure DevOps Server (on-premise) Webhook Integration

## Mục tiêu
- Khi có commit/push mới vào Azure DevOps Repo (on-premise), Jenkins tự động nhận được Webhook và trigger Pipeline.

---

## Điều kiện tiên quyết

| Thành phần            | Mô tả                                                                 |
|-----------------------|----------------------------------------------------------------------|
| Jenkins               | Đã cài đặt và truy cập được qua HTTP hoặc HTTPS                      |
| Azure DevOps Server   | Bản on-premise (ví dụ Azure DevOps Server 2020/2022)                 |
| Git Repository        | Tạo sẵn repo trong Azure DevOps                                      |
| Jenkins Plugin        | `Git plugin`, `Pipeline`, `Generic Webhook Trigger Plugin` đã cài    |
| Jenkins Credentials   | Có sẵn SSH Key hoặc Personal Access Token (PAT)                      |

---

## Bước 1: Tạo repo trên Azure DevOps

```bash
git clone https://github.com/spring-projects/spring-petclinic.git
cd spring-petclinic
git remote set-url origin http://devops.local:8080/tfs/DemoCI/_git/demo-webhook-repo
git push -u origin --all
```

---

## Bước 2: Tạo Personal Access Token (PAT)

1. Truy cập Azure DevOps Server.
2. Vào user avatar → Security.
3. Tạo token với scope `Code > Read & Write`.
4. Ghi lại token để dùng trong Jenkins.

---

## Bước 3: Tạo Jenkins Credential

1. Jenkins → **Manage Jenkins** → **Credentials**.
2. Scope: Global → Add Credential:
   - Kind: Secret Text
   - Secret: (PAT token)
   - ID: `azure-pat`
   - Description: `PAT for Azure DevOps`

---

## Bước 4: Tạo Pipeline Job

1. Jenkins → New Item → Pipeline → `azure-webhook-pipeline`.
2. Cấu hình:
   - Trigger:
     - ✅ Generic Webhook Trigger
     - Token: `devops-trigger`
   - Pipeline script:

```groovy
pipeline {
    agent any
    environment {
        GIT_TOKEN = credentials('azure-pat')
    }
    stages {
        stage('Clone Repo') {
            steps {
                git url: 'http://devops.local:8080/tfs/DemoCI/_git/demo-webhook-repo',
                    credentialsId: 'azure-pat'
            }
        }
        stage('Build') {
            steps {
                echo "Building project..."
            }
        }
    }
}
```

---

## Bước 5: Tạo Webhook trong Azure DevOps

1. Vào repo → **Project Settings** → **Service Hooks**.
2. + Create Subscription:
   - Service: Web Hooks
   - Trigger: Code pushed
   - URL: `http://jenkins.local:8080/generic-webhook-trigger/invoke?token=devops-trigger`
   - Method: POST

---

## Bước 6: Kiểm tra Trigger

```bash
echo "Test Jenkins Webhook" >> README.md
git add .
git commit -m "Test webhook"
git push
```

---

## Bonus: Tạo Webhook bằng REST API

```bash
curl -u user:PAT_TOKEN -X POST   http://devops.local:8080/tfs/DemoCI/_apis/hooks/subscriptions?api-version=6.0-preview.1   -H "Content-Type: application/json"   -d '{
    "publisherId": "tfs",
    "eventType": "git.push",
    "resourceVersion": "1.0",
    "consumerId": "webHooks",
    "consumerActionId": "httpRequest",
    "publisherInputs": {
        "projectId": "DemoCI",
        "repository": "demo-webhook-repo",
        "branch": ""
    },
    "consumerInputs": {
        "url": "http://jenkins.local:8080/generic-webhook-trigger/invoke?token=devops-trigger"
    }
}'
```

---

## Kết quả mong đợi

- Push code vào Azure DevOps → Jenkins tự động build Pipeline.
- Quan sát console log để xác nhận trigger hoạt động đúng.
