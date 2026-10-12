Bước 1: Tạo Network và Volume
Bash
# 1. Tạo bridge network tùy biến
docker network create --driver bridge app-net

# 2. Tạo named volume lưu trữ data database
docker volume create db-data
Bước 2: Khởi chạy cụm Container
Bash
# 3. Khởi chạy Database Container tham gia app-net và mount db-data
docker run -d \
  --name db-server \
  --network app-net \
  -v db-data:/var/lib/postgresql/data \
  -e POSTGRES_DB=appdb \
  -e POSTGRES_USER=appuser \
  -e POSTGRES_PASSWORD=secret \
  postgres:15-alpine

# 4. Khởi chạy Backend Container tham gia app-net
docker run -d \
  --name backend-app \
  --network app-net \
  -e DB_HOST=db-server \
  -e DB_PORT=5432 \
  -e DB_NAME=appdb \
  -e DB_USER=appuser \
  -e DB_PASSWORD=secret \
  alpine:latest sleep 3600
4. Kiểm tra và Đánh giá
4.1. Xác thực Network & Volume
Bash
docker network ls | grep app-net
docker volume ls | grep db-data
docker network inspect app-net
Kết quả: app-net và db-data xuất hiện trong danh sách. Lệnh inspect cho thấy cả 2 container db-server và backend-app đều có mặt trong network app-net.

4.2. Kiểm tra DNS Resolution
Thực hiện ping container db-server từ container backend-app:

Bash
docker exec -it backend-app ping -c 2 db-server
Kết quả:

Plaintext
PING db-server (172.19.0.2): 56 data bytes
64 bytes from 172.19.0.2: seq=0 ttl=64 time=0.082 ms
64 bytes from 172.19.0.2: seq=1 ttl=64 time=0.069 ms

--- db-server ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
Container backend-app tự động phân giải hostname db-server ra IP nội bộ (172.19.0.2).

4.3. Kiểm tra tính bền vững của dữ liệu (Data Persistence)
Thêm dữ liệu mẫu vào db-server:

Bash
docker exec -i db-server psql -U appuser -d appdb << 'EOF'
CREATE TABLE users (id SERIAL PRIMARY KEY, name VARCHAR(50));
INSERT INTO users (name) VALUES ('Alice'), ('Bob');
SELECT * FROM users;
EOF
Xóa cứng container Database:

Bash
docker rm -f db-server
Khởi chạy container mới cùng tên và gắn lại Volume db-data:

Bash
docker run -d \
  --name db-server \
  --network app-net \
  -v db-data:/var/lib/postgresql/data \
  -e POSTGRES_DB=appdb \
  -e POSTGRES_USER=appuser \
  -e POSTGRES_PASSWORD=secret \
  postgres:15-alpine
Truy vấn lại dữ liệu:

Bash
docker exec -i db-server psql -U appuser -d appdb -c "SELECT * FROM users;"