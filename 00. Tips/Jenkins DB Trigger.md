# Lab: Trigger Jenkins khi có thay đổi trong Database bằng Trigger + Webhook

## Mục tiêu
- Dùng Trigger trong **MySQL** để phát hiện `INSERT`/`DELETE`.
- Gửi webhook tới một Webhook Receiver (viết bằng Node.js hoặc Python Flask).
- Từ đó gọi Jenkins Job.

## Kiến trúc tổng thể

```
[MySQL Trigger] ➜ [Webhook Server (Python Flask)] ➜ [Jenkins API Trigger Job]
```

###  Step 1: Tạo bảng và Trigger trong MySQL

```sql
CREATE DATABASE demo;
USE demo;

CREATE TABLE employees (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### ➤ Bảng log trung gian

```sql
CREATE TABLE employee_change_log (
  id INT AUTO_INCREMENT PRIMARY KEY,
  action VARCHAR(10),
  employee_id INT,
  timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### ➤ Trigger phát hiện INSERT và DELETE

```sql
DELIMITER $$

CREATE TRIGGER after_employee_insert
AFTER INSERT ON employees
FOR EACH ROW
BEGIN
  INSERT INTO employee_change_log (action, employee_id)
  VALUES ('INSERT', NEW.id);
END$$

CREATE TRIGGER after_employee_delete
AFTER DELETE ON employees
FOR EACH ROW
BEGIN
  INSERT INTO employee_change_log (action, employee_id)
  VALUES ('DELETE', OLD.id);
END$$

DELIMITER ;
```

---

### Step 2: Webhook Receiver bằng Python Flask

```python
# webhook_server.py
from flask import Flask, request
import requests

app = Flask(__name__)

@app.route('/webhook', methods=['POST'])
def webhook():
    data = request.json
    print(f"Received DB Change: {data}")

    jenkins_url = "http://jenkins.local:8080/job/triggered-by-db/build"
    user = "admin"
    token = "your-token"

    response = requests.post(jenkins_url, auth=(user, token))
    return {"jenkins_status": response.status_code}, 200

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=5001)
```

---

### Step 3: Daemon kiểm tra bảng log và gửi webhook

```python
# watcher.py
import mysql.connector
import requests
import time

conn = mysql.connector.connect(
    host="localhost",
    user="root",
    password="yourpassword",
    database="demo"
)

cursor = conn.cursor(dictionary=True)

while True:
    cursor.execute("SELECT * FROM employee_change_log ORDER BY id ASC LIMIT 1")
    row = cursor.fetchone()

    if row:
        print("Sending webhook:", row)
        try:
            requests.post("http://localhost:5001/webhook", json=row)
        except Exception as e:
            print("Failed to send webhook:", e)

        cursor.execute("DELETE FROM employee_change_log WHERE id = %s", (row['id'],))
        conn.commit()

    time.sleep(2)
```

---

### Step 4: Kiểm thử

1. Chạy webhook server:
   ```bash
   python3 webhook_server.py
   ```

2. Chạy daemon:
   ```bash
   python3 watcher.py
   ```

3. Thêm record vào bảng:
   ```sql
   INSERT INTO employees (name) VALUES ('Alice');
   ```

4. Quan sát Jenkins Job được trigger.

---

## Kết quả

- Trigger trong DB ghi log lại thay đổi.
- Daemon phát hiện thay đổi → gửi HTTP POST đến Webhook.
- Webhook gọi Jenkins API → Pipeline được trigger.
