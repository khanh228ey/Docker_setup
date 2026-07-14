Tui có cấu truc folder như này 
Thì tui mới nhập môn docker thì tui đang muốn build 1 CONTAINER docker để chạy môi trường src để chạy laravel ,nestjs , reactjs
3 src này sẽ nằm trong thư mục này 
theo tui biet thì tui cần setup php >8.3 GIÚP TUI , node , nginx để chạy mà ko cần phải start từng src 
hiện tại db tui sẽ setup sau tui cần src 3 src chạy phổ thông trước đã 
thì tui có KHAI BÁO 3 DOMAIN VÀO HOST 
laravel.local.com hoặc ip local port 8080
nest.local.com hoặc ip local port 3000
react.local.com hoặc ip local port 3001 nếu ko  xài khai báo host

hiện tại tui chưa cần tối ưu đâu tui cần chạy được đã nên hãy làm cơ bản nhất có thể và theo cấu trúc sau 
FOLDER DOCKER NÀY TUI CÓ TẠO CÙNG CẤP VS 3 FOLDER SRC TRÊN 
D:\Src_code\improvisation   laravel_serivce , nest_service, react_service

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

Service `clickhouse` dùng image chính thức `clickhouse/clickhouse-server:24.8`, không bật auth (user `default`, không password) — chỉ dùng cho local dev, đừng expose ra ngoài mạng LAN/internet.

- Database mặc định: `codechill` (tự tạo lúc container khởi tạo lần đầu qua `CLICKHOUSE_DB`).
- Từ máy Windows (DBeaver, `clickhouse-client`, curl...):
  - HTTP: `http://localhost:8123` (thử nhanh: `curl http://localhost:8123/ping` phải trả về `Ok.`)
  - Native TCP: `localhost:9001` (map từ container port `9000`; đổi sang 9001 vì host port 9000 đã bị `laravel` dùng cho php-fpm)
- Từ các container khác trong `app_network` (nestjs, laravel, django...) dùng hostname service: `clickhouse:8123` (HTTP) hoặc `clickhouse:9000` (native, port nội bộ container vẫn là 9000 như bình thường).
- Data được lưu ở named volume `clickhouse_data` nên restart container không mất dữ liệu; `docker compose down -v` mới xoá.

Khởi động riêng: `docker compose up -d clickhouse`.