
# Lab: Build Maven Project Offline with Jenkins

## Mục tiêu

- Mô phỏng quy trình CI/CD build ứng dụng Java sử dụng Maven trong môi trường **không có Internet**
- Sử dụng Jenkins và Maven cache để thực hiện build offline
- Làm việc với file `settings.xml` để cấu hình Maven

---

## Môi trường lab

| Thành phần            | Mô tả                                        |
| --------------------- | -------------------------------------------- |
| Jenkins Server        | Cài sẵn Jenkins + Maven plugin               |
| Jenkins Agent (build) | Ubuntu 24.04, không có truy cập Internet     |
| Máy seed (builder)    | Có Internet, dùng để tải dependency về `.m2` |
| Project Maven         | Sử dụng repo mẫu `spring-petclinic`          |

---

## Bước 1: Chuẩn bị `.m2` repository trên máy có Internet

```bash
# Trên máy builder (có Internet):
git clone https://github.com/spring-projects/spring-petclinic.git
cd spring-petclinic

# Tải trước toàn bộ dependency
mvn dependency:go-offline
```

Xác minh thư mục `.m2`:
```bash
ls ~/.m2/repository
```

---

## Bước 2: Chuyển `.m2` sang Jenkins Agent (không có Internet)

```bash
# Trên builder:
tar -czf m2-cache.tar.gz ~/.m2
scp m2-cache.tar.gz jenkins@jenkins-agent:/home/jenkins/

# Trên Jenkins Agent:
cd /home/jenkins
mkdir -p .m2 && tar -xzf m2-cache.tar.gz -C .m2 --strip-components=2
```

---

## Bước 3: Tạo `settings.xml` ép Maven chỉ dùng local

```xml
<!-- /home/jenkins/.m2/settings.xml -->
<settings>
  <offline>true</offline>
  <profiles>
    <profile>
      <id>offline</id>
      <activation>
        <activeByDefault>true</activeByDefault>
      </activation>
      <repositories>
        <repository>
          <id>central</id>
          <url>file://${user.home}/.m2/repository</url>
        </repository>
      </repositories>
    </profile>
  </profiles>
</settings>
```

---

## Bước 4: Tạo Jenkins Pipeline để build offline

Trong Jenkins → Create new Pipeline Job → chọn Pipeline Script:

```groovy
pipeline {
  agent any
  environment {
    MAVEN_OPTS = "-Dmaven.repo.local=/home/jenkins/.m2/repository"
  }
  stages {
    stage('Clone Source') {
      steps {
        git 'https://github.com/spring-projects/spring-petclinic.git'
      }
    }
    stage('Build Offline') {
      steps {
        sh 'mvn -B clean install --offline --settings /home/jenkins/.m2/settings.xml'
      }
    }
  }
}
```

---

## Kết quả mong đợi

- Jenkins build thành công ứng dụng Java **mà không cần Internet**
- Không có lỗi liên quan đến việc thiếu dependency

---

## 🔄 Bước mở rộng (tuỳ chọn)

- Tạo Docker Image chứa `.m2` preload
- Sync định kỳ `.m2` mới từ máy builder (nếu thay đổi version)
- Kết hợp với Nexus hosted repository (hoặc Artifactory)

---

## Bước mở rộng: Đóng gói ứng dụng thành Docker Image và chạy

### Cấu trúc thư mục sau khi build (trên Jenkins Agent)

```
spring-petclinic/
├── target/
│   └── spring-petclinic-*.jar
├── Dockerfile
```

### Tạo `Dockerfile`

```Dockerfile
FROM eclipse-temurin:17-jre
WORKDIR /app
COPY target/spring-petclinic-*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Build image (trên Jenkins hoặc server nội bộ)

```bash
cd spring-petclinic
sudo docker build -t petclinic-offline:latest .
```

### Chạy thử container

```bash
sudo docker run -d -p 8080:8080 --name petclinic-demo petclinic-offline:latest
```

### Truy cập ứng dụng

Mở trình duyệt và truy cập: `http://<Jenkins_Agent_IP>:8080`

---

## Ghi chú

- Đảm bảo `.m2` đã đầy đủ plugin và dependency theo từng project
- Nếu dùng Docker Agent, cần mount `.m2` vào container hoặc build image sẵn
- Có thể dùng cùng cách này cho Gradle, NodeJS, Python (với cấu trúc tương ứng)

