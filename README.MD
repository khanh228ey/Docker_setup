## Config domain qua `.env`

Bạn có thể khai báo **nhiều domain** cho từng app bằng cách cách nhau bởi dấu cách:

- `LARAVEL_SERVER_NAMES=laravel.local.com laravel.test`
- `NEST_SERVER_NAMES=nest.local.com nest.test`
- `REACT_SERVER_NAMES=react.local.com react.test`

Nginx sẽ tự render config từ `default.conf.template` khi container start.

## Đã tối ưu (giảm nặng máy khi dev)

- Thêm `../.dockerignore` (ở gốc `D:\Src_code\CodeChill`, vì build context của cả 3 service là `..`) để loại `node_modules`, `vendor`, `dist`, `.git`... ra khỏi build context — build nhanh hơn hẳn.
- Laravel dùng thêm named volume `laravel_vendor` cho `/var/www/laravel/vendor` (giống cách `nest_node_modules`/`react_node_modules` đang làm) để bind-mount source không đè mất `vendor/` đã build tối ưu trong image.
- Nginx chỉ mount `laravel_service/public` thay vì cả thư mục Laravel.
- `nestjs`/`reactjs` thêm `CHOKIDAR_INTERVAL=1000` để giảm tần suất polling file, đỡ tốn CPU.
- Mỗi container đã có giới hạn `mem_limit`/`cpus` để tránh 1 service (watch/polling) ăn hết tài nguyên máy host.

Nếu vẫn thấy nặng: đòn bẩy lớn nhất tiếp theo là chuyển project vào filesystem của **WSL2** (vd `\\wsl$\Ubuntu\home\...`) thay vì để trên ổ NTFS Windows (`D:\...`) — bind mount NTFS ⇄ Linux container qua Docker Desktop vốn chậm hơn nhiều so với bind mount trong chính WSL2.

## ClickHouse (local)

Service `clickhouse` dùng image chính thức `clickhouse/clickhouse-server:24.8`. User `default`, password `123456` (đặt qua `CLICKHOUSE_PASSWORD`) — chỉ dùng cho local dev, đừng expose ra ngoài mạng LAN/internet, đổi password khác nếu deploy thật.

- Database mặc định: `codechill` (tự tạo lúc container khởi tạo lần đầu qua `CLICKHOUSE_DB`).
- Từ máy Windows (DataGrip/DBeaver JDBC, curl...):
  - HTTP: `http://localhost:8123` — user `default` / password `123456`. Test nhanh: `curl -u default:123456 http://localhost:8123/ping` phải trả về `Ok.`
  - Native TCP: `localhost:9001` (map từ container port `9000`; đổi sang 9001 vì host port 9000 đã bị `laravel` dùng cho php-fpm)
- Từ các container khác trong `app_network` (nestjs, laravel, django...) dùng hostname service: `clickhouse:8123` (HTTP) hoặc `clickhouse:9000` (native, port nội bộ container vẫn là 9000 như bình thường), cùng user/password ở trên.
- **Quan trọng:** vì container đã từng khởi tạo lúc chưa có password, cần xoá volume cũ để ClickHouse tạo lại user `default` với password mới (env `CLICKHOUSE_PASSWORD` chỉ có tác dụng lúc data dir rỗng — init lần đầu).
- Data được lưu ở named volume `clickhouse_data` nên restart container không mất dữ liệu; `docker compose down -v` mới xoá.

Khởi động riêng: `docker compose up -d clickhouse`.

## Kafka (local, KRaft mode)

Dùng image chính thức `apache/kafka:3.8.0`, chạy KRaft (không cần Zookeeper, chỉ 1 container `broker+controller`). Kèm `kafka-ui` (provectuslabs) để xem topic/message qua browser.

- Web UI: http://localhost:8081
- Từ các container khác trong `app_network` (nestjs, laravel, django...): connect qua `kafka:9092`.
- Từ máy Windows (kafkajs chạy ngoài container, offset explorer...): connect qua `localhost:9094`.
  - **Lưu ý:** không dùng `localhost:9092` từ Windows — Kafka trả `advertised.listeners` là `kafka:9092` cho listener đó, hostname `kafka` không resolve được ngoài docker network nên client sẽ connect fail sau bước lấy metadata. Đây là lý do có 2 listener riêng (`PLAINTEXT` cho nội bộ, `PLAINTEXT_HOST` cho từ host).
- `KAFKA_AUTO_CREATE_TOPICS_ENABLE=true` nên code cứ produce vào topic mới là Kafka tự tạo, không cần tạo tay khi dev.
- Data lưu ở named volume `kafka_data`, restart container không mất.

Khởi động riêng: `docker compose up -d kafka kafka-ui`.