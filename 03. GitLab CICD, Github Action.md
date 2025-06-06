# Lab GitLab CI/CD trên Ubuntu 24.04 (Mạng 192.168.100.0/24)

## Mô hình tổng quan

- GitLab Server: `192.168.100.10`
- GitLab Runner Shared: `192.168.100.11`
- GitLab Runner Specific: `192.168.100.12`
- User Client (demo push code): `192.168.100.20`
- Domain (nếu cần): `gitlab.local` (có thể dùng `/etc/hosts` để ánh xạ)

## Bước 1: Cài đặt GitLab Server trên Ubuntu 24.04

### Cấu hình máy chủ

```bash
sudo hostnamectl set-hostname gitlab-server
sudo apt update && sudo apt install -y curl openssh-server ca-certificates tzdata perl
```

### Cài đặt GitLab CE

```bash
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash
sudo EXTERNAL_URL="http://gitlab.local" apt install -y gitlab-ce
```

### Cấu hình file hosts trên các máy client

```bash
echo "192.168.100.10 gitlab.local" | sudo tee -a /etc/hosts
```

### Truy cập GitLab

Mở trình duyệt tại `http://gitlab.local` và thiết lập mật khẩu cho tài khoản `root`.

## Bước 2: Cài đặt GitLab Runner (Shared Runner)

### Trên máy `192.168.100.11`

```bash
sudo apt update && sudo apt install -y curl
curl -L --output gitlab-runner.deb https://gitlab-runner-downloads.s3.amazonaws.com/latest/deb/gitlab-runner_amd64.deb
sudo dpkg -i gitlab-runner.deb
sudo gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner
sudo gitlab-runner start
```

### Đăng ký Shared Runner

```bash
sudo gitlab-runner register
```

Thông tin nhập vào:

- GitLab URL: `http://gitlab.local/`
- Registration token: Lấy tại `Admin > Runners`
- Description: `shared-runner`
- Tags: `shared`
- Executor: `shell` hoặc `docker`

## Bước 3: Cài đặt GitLab Runner (Specific Runner)

### Trên máy `192.168.100.12`

Lặp lại các bước cài runner như trên.

### Đăng ký Specific Runner

- Vào GitLab project > Settings > CI/CD > Runners > Expand
- Lấy URL và token của project
- Đăng ký runner với executor `shell` và tags `specific`

## Bước 4: Tạo Project và Push Code từ Client

### Trên GitLab

- Tạo project mới: `demo-ci`
- Bật CI/CD trong Settings

### Trên máy `192.168.100.20`

```bash
sudo apt install -y git
git config --global user.name "Demo User"
git config --global user.email "demo@example.com"

git clone http://gitlab.local/demo-user/demo-ci.git
cd demo-ci
```

Tạo file `main.sh`

```bash
#!/bin/bash
echo "Hello GitLab CI/CD"
```

Tạo file `.gitlab-ci.yml`

```yaml
stages:
  - build
  - test

build-job:
  stage: build
  script:
    - echo "Building..."
    - chmod +x main.sh

test-job:
  stage: test
  script:
    - ./main.sh
```

Push lên GitLab

```bash
git add .
git commit -m "Initial commit with pipeline"
git push origin master
```

## Bước 5: Kiểm tra Pipeline

- Truy cập `http://gitlab.local/demo-user/demo-ci/-/pipelines`
- Vào `CI/CD > Jobs` để xem log

## Bước 6: Mô phỏng hành vi người dùng

Sửa code:

```bash
echo "Build on $(date)" >> build.log
```

Cập nhật `.gitlab-ci.yml`

```yaml
stages:
  - build
  - test

build-job:
  stage: build
  script:
    - echo "Building on $(date)" >> build.log
  artifacts:
    paths:
      - build.log

test-job:
  stage: test
  script:
    - cat build.log
```

Push lại:

```bash
git add .
git commit -m "Thêm artifact"
git push origin master
```

## Bước 7: Mở rộng CI/CD

- Thêm điều kiện chạy pipeline với `only`, `except`, `tags`
- Kiểm tra tag runner hoạt động đúng

## Kết luận

Bài lab này giúp hiểu toàn bộ vòng đời GitLab CI/CD từ cài đặt đến sử dụng trong thực tế.
