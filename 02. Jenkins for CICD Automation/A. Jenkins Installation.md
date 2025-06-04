# Jenkins Lab: Triển khai Jenkins Controller và Agent trên Ubuntu 24.04

## Môi trường Lab

| Role               | Hostname             | IP Address       | OS              |
|--------------------|----------------------|------------------|-----------------|
| Jenkins Controller | jenkins-controller   | 192.168.100.10   | Ubuntu 24.04    |
| Jenkins Agent SSH  | jenkins-agent-ssh    | 192.168.100.11   | Ubuntu 24.04    |
| Jenkins Agent JNLP | jenkins-agent-jnlp   | 192.168.100.12   | Ubuntu 24.04    |

---

## Bước 1: Cài đặt Jenkins Controller

### 1.1. Cài Java (OpenJDK 17)

```bash
sudo apt update
sudo apt install -y openjdk-17-jdk
java -version
```

### 1.2. Thêm kho và cài Jenkins

```bash
curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install -y jenkins
```

### 1.3. Khởi động Jenkins và mở port

```bash
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo ufw allow 8080
```

### 1.4. Lấy mật khẩu ban đầu

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Vào trình duyệt: `http://192.168.100.10:8080`

---

## Bước 2: Cấu hình Jenkins Controller

### 2.1. Tạo SSH key cho Jenkins

```bash
sudo su - jenkins
ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub
```

---

## Bước 3: Triển khai Jenkins Agent sử dụng SSH

### 3.1. Cài đặt agent (trên `jenkins-agent-ssh`)

```bash
sudo apt update
sudo apt install -y openjdk-17-jdk
sudo useradd -m -s /bin/bash jenkins
sudo mkdir -p /home/jenkins/.ssh
sudo nano /home/jenkins/.ssh/authorized_keys
# → Paste public key từ Controller
sudo chown -R jenkins:jenkins /home/jenkins/.ssh
sudo chmod 700 /home/jenkins/.ssh
sudo chmod 600 /home/jenkins/.ssh/authorized_keys
```

Test SSH từ controller:

```bash
ssh jenkins@192.168.100.11
```

### 3.2. Thêm SSH Agent trong Jenkins UI

- Manage Jenkins → Manage Nodes and Clouds → New Node
- Name: ssh-agent-node
- Type: Permanent Agent
- Remote root directory: /home/jenkins
- Launch method: Launch agents via SSH
- Host: 192.168.100.11
- Credentials: SSH Username + Private Key

---

## Bước 4: Triển khai Jenkins Agent sử dụng JNLP

### 4.1. Tạo user và cài Java

```bash
sudo useradd -m -s /bin/bash jenkins
sudo apt update
sudo apt install -y openjdk-17-jdk docker.io
sudo usermod -aG docker jenkins
```

### 4.2. Tạo JNLP Agent trong Jenkins UI

- Manage Jenkins → Manage Nodes → New Node
- Name: jnlp-agent-node
- Type: Permanent Agent
- Launch method: Launch agent by connecting it to the controller

### 4.3. Tải agent.jar và chạy JNLP

```bash
sudo su - jenkins
wget http://192.168.100.10:8080/jnlpJars/agent.jar
java -jar agent.jar -jnlpUrl <url> -secret <token>
```

---

## Bước 5: Tạo Pipeline test đơn giản

### 5.1. Pipeline Script mẫu

```groovy
pipeline {
    agent { label 'ssh-agent-node' }
    stages {
        stage('Build') {
            steps {
                echo "Build on SSH Agent"
            }
        }
        stage('Test') {
            agent { label 'jnlp-agent-node' }
            steps {
                echo "Run test on JNLP Agent"
            }
        }
    }
}
```

---

## Bước 6: (Tuỳ chọn) Cho phép build ứng dụng Docker

```bash
sudo apt install -y docker.io
sudo usermod -aG docker jenkins
```

### Pipeline có Docker build

```groovy
pipeline {
    agent { label 'ssh-agent-node' }
    stages {
        stage('Docker Build') {
            steps {
                script {
                    sh '''
                    echo -e "FROM alpine\nCMD echo Hello Jenkins Docker" > Dockerfile
                    docker build -t myjenkins/demo .
                    '''
                }
            }
        }
    }
}
```

---
