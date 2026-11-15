# Phố Ẩm Thực — Hệ thống thuyết minh tự động cho phố ẩm thực

Ứng dụng web giúp du khách khám phá một phố ẩm thực. Khi du khách đi tới gần một gian hàng, điện thoại tự phát đoạn thuyết minh về gian hàng đó bằng ngôn ngữ họ chọn. Họ cũng có thể quét mã QR tại quầy để nghe ngay.

Đây là đồ án môn học của một nhóm 4 người, viết theo mô hình client–server trong một monorepo (NestJS + Next.js).

## Bài toán

Du khách nước ngoài tới phố ẩm thực thường không biết gian hàng bán gì, món nào nổi bật, và khó hỏi trực tiếp vì rào cản ngôn ngữ. Chủ gian hàng thì không có công cụ đơn giản để giới thiệu quầy của mình bằng nhiều thứ tiếng.

Hệ thống giải quyết bằng cách:

- Chủ gian hàng viết thuyết minh bằng tiếng Việt. AI tự dịch sang 6 ngôn ngữ (Anh, Pháp, Trung, Nhật, Hàn, Thái).
- Du khách mở web trên điện thoại, cho phép dùng GPS. Khi đứng cách gian hàng khoảng 4 m, thuyết minh tự phát.
- Admin duyệt nội dung và vị trí gian hàng, quản lý ranh giới phố ẩm thực và kiểm duyệt bình luận.

## Tính năng chính

### Du khách

- Bản đồ phố ẩm thực với vị trí các gian hàng, chỉ đường tới gian hàng.
- Tự phát thuyết minh khi tới gần gian hàng (GPS), hoặc bấm phát thủ công.
- Quét mã QR tại quầy để mở trang gian hàng và nghe thuyết minh.
- Đổi ngôn ngữ giao diện và thuyết minh.
- Xem thực đơn, gợi ý món ăn theo sở thích.
- Đăng nhập bằng Google để đánh giá và bình luận.

### Chủ gian hàng

- Đăng ký tài khoản, chờ admin phê duyệt.
- Cập nhật thông tin gian hàng, thực đơn và ảnh món. Thay đổi được gửi admin duyệt.
- Viết thuyết minh, xem bản dịch AI và xem trước mã QR của gian hàng.
- Dashboard thống kê lượt ghé gian hàng và lượt quét QR.

### Admin

- Duyệt tài khoản chủ gian hàng, nội dung gian hàng và vị trí ghim trên bản đồ.
- Vẽ ranh giới phố ẩm thực trên bản đồ.
- Kiểm duyệt bình luận bị báo cáo, gửi thông báo tới chủ gian hàng.
- Theo dõi hệ thống: tỉ lệ lỗi, hàng đợi job, bộ nhớ, kèm cảnh báo qua email.

## Kiến trúc

```
Trình duyệt (Next.js)
  ├─ Bản đồ MapLibre + GPS ── tính khoảng cách tới gian hàng ── phát thuyết minh (Web Speech API)
  └─ REST API, Socket.IO
        │
        ▼
NestJS API (/api)
  ├─ PostgreSQL + PostGIS   dữ liệu gian hàng, vị trí, ranh giới
  ├─ Redis + BullMQ         job dịch thuyết minh chạy nền
  ├─ Gemini API             dịch thuyết minh, dự phòng bằng MyMemory
  ├─ MinIO (S3)             lưu ảnh gian hàng, món ăn
  └─ SMTP                   email thông báo
```

Luồng thuyết minh:

1. Chủ gian hàng lưu thuyết minh tiếng Việt.
2. Backend đẩy job vào hàng đợi BullMQ.
3. Worker gọi Gemini để dịch sang từng ngôn ngữ. Nếu Gemini lỗi thì chuyển sang MyMemory.
4. Bản dịch được lưu vào CSDL và báo về trình duyệt qua Socket.IO.
5. Trình duyệt đọc thuyết minh bằng Web Speech API khi du khách tới gần hoặc quét QR.

Các sơ đồ tuần tự (PlantUML) nằm trong [`docs/`](docs) và [`docs/user/`](docs/user). Đặc tả từng tính năng nằm trong [`specs/`](specs), use case nằm trong [`requirement/`](requirement).

### Công nghệ

| Phần | Công nghệ |
|---|---|
| Backend | NestJS 10, TypeScript, TypeORM, class-validator |
| CSDL | PostgreSQL 15 + PostGIS 3.4 |
| Hàng đợi, realtime | Redis 7, BullMQ, Socket.IO |
| Xác thực | JWT (chủ gian hàng, admin), Google OAuth 2.0 (du khách) |
| AI, dịch | Google Gemini, MyMemory API (dự phòng) |
| Frontend | Next.js 14 (App Router), React 18, Tailwind CSS, TanStack Query, react-hook-form, zod |
| Bản đồ, QR | MapLibre GL, jsQR, qrcode |
| Đa ngôn ngữ | i18next |
| Lưu trữ file | MinIO (tương thích S3) |
| Giám sát | Sentry, module monitoring riêng |
| Monorepo | npm workspaces, Turborepo |

## Phần tôi phụ trách

- Viết các REST API endpoint ở backend NestJS và các màn hình React (Next.js) ở frontend, gồm dashboard của chủ gian hàng.
- Làm tính năng QR: sinh mã và xem trước mã QR của gian hàng.
- Tích hợp AI dịch thuyết minh đa ngôn ngữ qua hàng đợi.

## Cấu trúc thư mục

```
apps/
  backend/     NestJS API: auth, stores, commentary, location, qr, reviews, admin, monitoring
  frontend/    Next.js: trang công khai (bản đồ, gian hàng), trang chủ gian hàng, trang admin
packages/
  types/       kiểu TypeScript dùng chung
specs/         đặc tả từng tính năng (001 → 009)
requirement/   use case và business rule
docs/          sơ đồ tuần tự PlantUML
docker-compose.yml   PostgreSQL + PostGIS, Redis, MinIO
```

## Cài đặt

### 1. Yêu cầu

- Node.js 20 trở lên, npm 11
- Docker và Docker Compose v2

### 2. Cài dependencies và khởi động dịch vụ

```bash
git clone https://github.com/TKieen/seminar-street-food.git
cd seminar-street-food
npm install
docker compose up -d
```

`docker compose` chạy PostgreSQL (cổng 5432), Redis (6379) và MinIO (9000, console 9001). Bucket ảnh `seminar-media` được tạo tự động.

### 3. Cấu hình biến môi trường

```bash
cp apps/backend/.env.example apps/backend/.env
cp apps/frontend/.env.example apps/frontend/.env
```

Giá trị mặc định đủ để chạy ở máy dev. Một số tính năng cần điền thêm:

| Biến (backend) | Dùng cho |
|---|---|
| `GEMINI_API_KEY` | Dịch thuyết minh bằng AI. Biến này chưa có trong `.env.example`, cần tự thêm. Thiếu biến này thì hệ thống dùng MyMemory. |
| `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET` | Du khách đăng nhập bằng Google |
| `SMTP_USER`, `SMTP_PASS` | Gửi email thông báo |
| `JWT_SECRET`, `JWT_ADMIN_SECRET`, `CUSTOMER_JWT_SECRET` | Bắt buộc đổi khi triển khai thật |

### 4. Tạo CSDL và dữ liệu mẫu

```bash
npm run db:migrate
npm run seed:all --workspace=apps/backend
```

Dữ liệu mẫu gồm 1 admin, các chủ gian hàng ở nhiều trạng thái, 3 gian hàng có thực đơn và thuyết minh, ranh giới phố ẩm thực và các ghim vị trí.

### 5. Chạy ứng dụng

```bash
npm run dev
```

| Dịch vụ | Địa chỉ |
|---|---|
| Frontend | http://localhost:3000 |
| Backend API | http://localhost:3001/api |

Tài khoản mẫu sau khi seed:

| Vai trò | Email | Mật khẩu |
|---|---|---|
| Admin | admin@phoamthuc.vn | Admin@123456 |
| Chủ gian hàng | owner1@phoamthuc.vn | Owner@123456 |

### Lệnh khác

```bash
npm run build        # build cả backend và frontend
npm run test         # chạy test (Jest ở backend)
npm run lint
docker compose down  # dừng dịch vụ (thêm -v để xoá dữ liệu)
```

Hướng dẫn chi tiết hơn, gồm cấu hình Google OAuth, MinIO và cách xử lý lỗi thường gặp, có trong [`SETUP.md`](SETUP.md).
