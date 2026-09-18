# GearGo — Kế hoạch Tuần 2: Từ nền tảng đến đặt đơn & thanh toán

> **Nguồn dữ liệu chuẩn:** `Diagrams/ERD.dbml` (khi đã có). Mọi model Prisma phải khớp 1-1 với bảng trong ERD (tên bảng, tên cột, kiểu dữ liệu, quan hệ FK). Tuyệt đối không tự thêm/bớt cột.

**Mục tiêu:** UC01 (xác thực) + UC03–UC06 (khách duyệt sản phẩm → giỏ → đơn → thanh toán) + UC21 (admin CRUD danh mục & sản phẩm).

**Kiến trúc:** MVC phân tầng — `Controller → Service → Repository (Prisma)`. NestJS backend trả JSON REST API, JWT trong HTTP-only cookie. Next.js 14 App Router làm frontend riêng, gọi API qua fetch.

**Tech Stack:** NestJS 10 · Prisma 5 · PostgreSQL 16 (Docker) · Next.js 14 · Redis 7 (cache + BullMQ) · Docker Compose · JWT (HTTP-only cookie) · Argon2 · Zod · Tailwind CSS + shadcn/ui

**Phạm vi nghiệm thu tuần 2:** Backend API (`apps/api`) + các trang Next.js tối thiểu để chạy được luồng. Nếu frontend đầy đủ chưa xong, ưu tiên hoàn thiện API và Postman collection.

**Chỉ Người 1 chịu trách nhiệm tạo Prisma migration** (xem `PhanCong_Week2.md`): thành viên hoàn thiện model Prisma (chỉnh `schema.prisma` trong branch riêng) → Người 1 tích hợp và chạy `npx prisma migrate dev --name <ten-migration>` → cả nhóm cập nhật DB bằng migration đã thống nhất. Trong tuần có thể có nhiều migration nối tiếp nhau (không phải 1 migration duy nhất), nhưng **chỉ Người 1 tạo**. Người 1 cũng tích hợp cấu hình chung trong `app.module.ts`, `main.ts`, Docker Compose và biến môi trường.

**Người 1 bàn giao service xử lý lỗi chung (Day 1–2):**
- Kiểu `Result<T>` / `Result` cho tầng service (hoặc pattern `neverthrow` nếu cả nhóm thống nhất).
- Cây exception nghiệp vụ (extend `HttpException` của NestJS): `NghiepVuException` (base), `KhongDuHangException`, `BaogiaThayDoiException`, `TrangThaiKhongHopLeException`, `TaiKhoanBiKhoaException`, `KhuyenMaiKhongHopLeException`, `KhongTimThayException`, `KhongCoQuyenException`.
- `HttpExceptionFilter` (global): exception nghiệp vụ → HTTP 400/403/404/409, exception khác → 500. Đăng ký `app.useGlobalFilters(new HttpExceptionFilter())`.
- Cấu trúc lỗi JSON thống nhất: `{ maLoi, thongDiep, chiTiet? }`.

---

## Quy ước kỹ thuật dùng chung

### Chuyển kiểu ERD → PostgreSQL / Prisma / TypeScript

| Kiểu ERD | Prisma | TypeScript | Ghi chú |
|---|---|---|---|
| `json` | `Json` | `Prisma.JsonValue` | PostgreSQL `jsonb`; typed qua Zod schema |
| `varchar(n)` / `text` | `String` | `string` | Enum lưu dạng chuỗi (native Prisma enum) |
| `decimal(18,2)` | `Decimal` | `Prisma.Decimal` | KHÔNG dùng `number` cho tiền |
| `bigint` | `BigInt` | `bigint` | Dùng cho PK/FK; hoặc `Int` nếu nhóm thống nhất |
| `boolean` | `Boolean` | `boolean` | |
| `date` | `DateTime @db.Date` | `Date` | |
| `datetime` | `DateTime` | `Date` | |

### Quy ước giờ

- Tất cả giờ **lưu và truyền API** dạng UTC (ISO 8601). Prisma dùng `DateTime` mặc định UTC.
- **Hiển thị** cho người dùng: chuyển sang giờ Việt Nam (UTC+7) tại tầng Next.js hoặc formatter dùng chung (`date-fns-tz`).
- Một ngày tính tiền = 24 giờ; phần lẻ làm tròn lên, tối thiểu 1 ngày.

### Cấu trúc JSON quan trọng

| Trường JSON | Mô tả |
|---|---|
| `ChinhSach.noiDungChinhSach` | Object chứa toàn bộ chính sách (hạn giữ chỗ, phí hủy, hệ số trễ…) |
| `DonThue.khuyenMaiLucDat` | Snapshot thông tin khuyến mãi tại thời điểm tạo đơn |
| `ChiTietDonThue.phuKienVaMucBoiThuongLucDat` | Snapshot phụ kiện kèm theo & mức bồi thường từng phụ kiện |
| `PhieuNhapHang.thongTinNhaCungCapLucNhap` | Snapshot thông tin nhà cung cấp lúc nhập |
| `ThietBi.phuKienDiKem` | Danh sách phụ kiện theo thiết bị |
| `SanPham.thongSo` | Thông số kỹ thuật dạng key-value |

Định nghĩa Zod schema cho từng JSON columm trong `packages/types/` và dùng khi validate lúc read/write.

### DTO và cấu trúc lỗi chung

- **DTO:** Tách Request/Response riêng, dùng `class-validator` + `class-transformer` (NestJS convention). Không dùng Prisma model trực tiếp làm response.
- **Cấu trúc lỗi chung:**
  ```json
  {
    "maLoi": "KHONG_DU_HANG",
    "thongDiep": "Sản phẩm Lều 4 người không đủ số lượng.",
    "chiTiet": { "maSanPham": 1, "canThiet": 3, "conLai": 1 }
  }
  ```
- **Chuyển lỗi nghiệp vụ → HTTP:**
  - Validation lỗi → 400 Bad Request
  - Không tìm thấy → 404 Not Found
  - Xung đột nghiệp vụ (hết hàng, báo giá đổi, hết lượt mã…) → 409 Conflict
  - Không có quyền → 403 Forbidden
  - Lỗi server → 500 Internal Server Error

---

## Nguyên tắc xây dựng theo cấp phụ thuộc FK

Xây model **ít phụ thuộc trước, nhiều phụ thuộc sau**:

```
Cấp 0 (không FK / self-ref):
  TaiKhoan, DanhMucSanPham, NhaCungCap, KhuyenMai

Cấp 1 (1 FK về cấp 0):
  KhachHang, NhanVien, SanPham

Cấp 2:
  HinhAnhSanPham, PhieuNhapHang, ChinhSach, GioThue,
  KhuyenMaiSanPham, KhuyenMaiDanhMuc

Cấp 3:
  ChiTietPhieuNhap, ChiTietGioThue, DonThue

Cấp 4:
  ThietBi, ChiTietDonThue, LuotSuDungKhuyenMai, ThanhToan

Cấp 5:
  GiuCho (1-1 ChiTietDonThue), ChiTietThanhToan
```

**Lưu ý ERD quan trọng (dễ nhầm):**
- `ThietBi.maChiTietPhieuNhap` **NOT NULL** → thiết bị **bắt buộc** đi kèm phiếu nhập. Không thể tạo `ThietBi` nếu không có `ChiTietPhieuNhap` trước.
- `DonThue.maChinhSach` **NOT NULL** → mỗi đơn phải gắn với một `ChinhSach` (versioned policy). Phải seed ít nhất 1 chính sách trước khi tạo đơn được.
- `DonThue.maNguoiHuy` → FK về `TaiKhoan` (không phải `NhanVien`) — vì khách hàng cũng có thể tự hủy đơn.
- `GiuCho` quan hệ **1-1** với `ChiTietDonThue` (không phải 1-nhiều theo đơn).
- `LuotSuDungKhuyenMai` quan hệ **1-1** với `DonThue`; **có FK `maKhuyenMai`** (NOT NULL) về `KhuyenMai`.
- `KhuyenMai` có 2 bảng many-to-many: `KhuyenMaiSanPham` (composite PK `[maKhuyenMai, maSanPham]`) và `KhuyenMaiDanhMuc` (composite PK `[maKhuyenMai, maDanhMuc]`).
- `SanPham` có cả `sucChua Int` **và** `kichThuoc String` (2 cột riêng), thêm `thongSo Json`.
- `GioThue.maKhuyenMai` là **FK về `KhuyenMai`** (BigInt/Int), không phải string.
- `ThanhToan` tách 2 bảng: `ThanhToan` (giao dịch) + `ChiTietThanhToan` (chia mục đích: TIEN_THUE / TIEN_COC / THU_BO_SUNG).
- `LichSuTrangThaiDon` đã có trong ERD — dùng để ghi lịch sử chuyển trạng thái đơn.

---

## Phạm vi tuần 2

| Task | Nội dung | Cấp FK |
|---|---|---|
| Task 0 ✅ | Khởi tạo monorepo, Docker, Prisma, TaiKhoan + KhachHang + NhanVien | 0-1 |
| Task 1 | UC01 — Xác thực (JWT HTTP-only cookie + Argon2) | — |
| Task 2 | DanhMucSanPham | 0 |
| Task 3 | SanPham + HinhAnhSanPham | 1-2 |
| Task 4 | KhuyenMai + bảng nối + KhuyenMaiService | 0, 2 |
| Task 5 | NhaCungCap + PhieuNhapHang + ChiTietPhieuNhap + ThietBi (skeleton) | 0-4 |
| Task 6 | GioThue + ChiTietGioThue | 2-3 |
| Task 7 | ChinhSach + DonThue + ChiTietDonThue + GiuCho + LuotSuDungKhuyenMai + LichSuTrangThaiDon | 2-5 |
| Task 8 | ThanhToan + ChiTietThanhToan | 4-5 |
| Task 9 | KhaDungService + UC03 (tìm & xem sản phẩm) | — |
| Task 10 | UC04 giỏ + UC05 đặt đơn + UC06 thanh toán (API) | — |
| Task 11 | UC21 admin CRUD danh mục & sản phẩm (API) | — |
| Task 12 | Seed data + integration test + review | — |

### Chốt phạm vi triển khai tuần 2

Các quyết định dưới đây chỉ chia giai đoạn triển khai; không xóa hoặc thay đổi yêu cầu trong đặc tả/use case:

- **Trạng thái thiết bị:** bỏ `DANG_GIU` khỏi `TrangThaiSuDungThietBi`. Giữ chỗ được quản lý tại `GiuCho`; việc đã gán cho đơn tương lai là dữ liệu phân công ở giai đoạn sau.
- **Tìm kiếm:** UC03 có thêm bộ lọc sức chứa. Lọc theo đánh giá triển khai cùng UC08.
- **Hủy đơn:** tuần 2 chỉ hỗ trợ hủy đơn chưa thanh toán. Hủy sau thanh toán và hoàn tiền triển khai theo UC07 ở giai đoạn sau.
- **Thanh toán:** gateway mock chỉ phục vụ demo luồng thành công và idempotency; chưa coi là hoàn thành toàn bộ ngoại lệ của UC06.
- **Hoàn cọc:** không dùng `HOAN_COC` trong mục đích thu của `ChiTietThanhToan`; hoàn cọc về sau đi qua `HoanTien`.
- **Chính sách:** seed một phiên bản đang có hiệu lực. Khi thay đổi, tạo phiên bản mới; không sửa nội dung phiên bản đã gắn với đơn.
- **Nhập kho:** Model nhập kho (NhaCungCap, PhieuNhapHang, ChiTietPhieuNhap, ThietBi) đầy đủ cột theo ERD; tuần 2 chỉ chưa làm API nhập kho — dời sang Tuần 3.
- **Automation & AI:** Job hết hạn đơn (BullMQ) làm trong tuần 2. Claude API tư vấn (UC09), thông báo real-time, nhắc trả đồ — Tuần 3+.
- **Cache Redis:** dùng cho danh mục, danh sách sản phẩm, chi tiết sản phẩm. Giỏ thuê và khả dụng **không cache**.

---

## Cấu trúc thư mục

```
GearGo/                                       ← Monorepo root
├── docker-compose.yml                        ✅
├── docker-compose.dev.yml                    ✅
├── .env.example                              ✅
├── turbo.json                                ✅
├── package.json                              ← pnpm workspace
├── prisma/
│   ├── schema.prisma                         ← toàn bộ model, 1 file
│   ├── migrations/                           ← chỉ Người 1 tạo
│   └── seed.ts
├── apps/
│   ├── api/                                  ← NestJS backend
│   │   ├── src/
│   │   │   ├── main.ts                       ✅
│   │   │   ├── app.module.ts                 ✅
│   │   │   ├── prisma/                       ✅
│   │   │   ├── common/
│   │   │   │   ├── cache/                    ✅ Redis
│   │   │   │   ├── guards/                   ← JwtAuthGuard, RolesGuard
│   │   │   │   ├── decorators/               ← @CurrentUser, @Roles
│   │   │   │   ├── filters/                  ← HttpExceptionFilter
│   │   │   │   ├── result/                   ← Result<T> pattern
│   │   │   │   └── exceptions/               ← Cây exception nghiệp vụ
│   │   │   ├── auth/                         ← UC01
│   │   │   ├── danh-muc/                     ← Task 2 + admin
│   │   │   ├── san-pham/                     ← Task 3 + UC03 + admin
│   │   │   ├── khuyen-mai/                   ← Task 4
│   │   │   ├── kho/                          ← Task 5 (skeleton)
│   │   │   ├── gio-thue/                     ← Task 6 + UC04
│   │   │   ├── kha-dung/                     ← KhaDungService
│   │   │   ├── bao-gia/                      ← BaoGiaService dùng chung
│   │   │   ├── don-thue/                     ← Task 7 + UC05
│   │   │   ├── thanh-toan/                   ← Task 8 + UC06
│   │   │   └── jobs/                         ← BullMQ processor
│   │   └── test/
│   └── web/                                  ← Next.js frontend
│       └── src/app/                          ← App Router routes
└── packages/
    └── types/                                ← shared TS + Zod schemas
```

---

## Task 0: Khởi tạo dự án ✅ (đã xong)

**Đã tạo:**
- Monorepo Turborepo (`pnpm`), `apps/api` (NestJS), `apps/web` (Next.js).
- `docker-compose.yml` — PostgreSQL 16, Redis 7, api, web.
- `prisma/schema.prisma` — datasource PostgreSQL, generator client.
- `apps/api/src/prisma/prisma.service.ts` — extends PrismaClient.
- `apps/api/src/common/cache/cache.module.ts` — Redis cache (`@nestjs/cache-manager` + `ioredis`).
- Model khởi đầu: `TaiKhoan`, `KhachHang`, `NhanVien`.
- `main.ts` — global prefix `/api/v1`, ValidationPipe, HttpExceptionFilter, CORS + cookie parser, port 3001.

**Cần kiểm tra lại 3 model so với ERD (đối chiếu từng cột):**
- [ ] **0.a** `TaiKhoan`: đủ `maTaiKhoan`, `email` (unique), `soDienThoai` (unique), `matKhauBam`, `vaiTro`, `trangThai`, `lyDoKhoa`, `ngayTao`, `ngayCapNhat`.
- [ ] **0.b** `KhachHang`: đủ `maKhachHang`, `maTaiKhoan` (FK unique), `hoTen`, `diaChi`, `ngaySinh`, `anhDaiDien`.
- [ ] **0.c** `NhanVien`: đủ `maNhanVien`, `maTaiKhoan` (FK unique), `hoTen`, `diaChi`, `ngayVaoLam`, `ngayNghiViec`, `trangThaiLamViec`.

---

## Task 1: UC01 — Xác thực (JWT HTTP-only cookie + Argon2)

**Không thêm model mới.** Chỉ implement service + controller trên `TaiKhoan` + `KhachHang` đã có.

**Files:**
- Tạo: `apps/api/src/auth/dto/dang-ky.dto.ts`, `dang-nhap.dto.ts`, `quen-mat-khau.dto.ts`, `dat-lai-mat-khau.dto.ts`, `auth.response.ts`
- Tạo: `apps/api/src/auth/auth.service.ts`, `jwt.service.ts` (hoặc dùng `@nestjs/jwt` trực tiếp)
- Tạo: `apps/api/src/auth/strategies/jwt.strategy.ts` (Passport JWT extract from cookie)
- Tạo: `apps/api/src/auth/auth.controller.ts`, `auth.module.ts`
- Tạo: `apps/api/src/common/guards/jwt-auth.guard.ts`, `roles.guard.ts`
- Tạo: `apps/api/src/common/decorators/current-user.decorator.ts`, `roles.decorator.ts`

**Các bước:**

- [ ] **1.1** `AuthResponse`: `token`, `loaiToken = "Bearer"`, `hetHanSau`, `maTaiKhoan`, `vaiTro`, `hoTen`, `email`. Cookie `access_token` được set song song với response body (frontend không đọc token, nhưng có thể hiển thị `hoTen`, `vaiTro`).

- [ ] **1.2** `JwtService.taoToken(taiKhoan)` — JWT với claims `{ sub: maTaiKhoan, email, vaiTro }`, hết hạn 7 ngày, HMAC-SHA256 với `JWT_SECRET`. Cấu hình qua `@nestjs/jwt` `registerAsync`.

- [ ] **1.3** `AuthService`:
  ```typescript
  dangKy(dto: DangKyDto): Promise<Result<AuthResponse>>
  dangNhap(dto: DangNhapDto, res: Response): Promise<Result<AuthResponse>>
  dangXuat(res: Response): Promise<void>
  taoTokenQuenMatKhau(email: string): Promise<Result<string>>
  datLaiMatKhau(dto: DatLaiMatKhauDto): Promise<Result<void>>
  layThongTinToi(maTaiKhoan: bigint): Promise<Result<AuthResponse>>
  ```

- [ ] **1.4** `dangKy`:
  - Validate DTO qua `class-validator`: `email` đúng format (`@IsEmail`), `soDienThoai` hợp lệ VN (`@IsPhoneNumber('VN')`), `matKhau` ≥ 8 ký tự, `xacNhanMatKhau` khớp (`@Match('matKhau')` decorator tự viết).
  - Kiểm tra `email`, `soDienThoai` chưa tồn tại trong `TaiKhoan`.
  - `argon2.hash(dto.matKhau)`.
  - `prisma.$transaction`: tạo `TaiKhoan` (`vaiTro: 'KHACH_HANG'`, `trangThai: 'HOAT_DONG'`) + `KhachHang` (1-1).
  - **Không** cho phép đăng ký `NHAN_VIEN` / `QUAN_TRI_VIEN` qua endpoint công khai.

- [ ] **1.5** `dangNhap`:
  - Tìm theo `email` (nếu chứa `@`) hoặc `soDienThoai`.
  - **Kiểm tra trạng thái tài khoản:** nếu `trangThai = 'BI_KHOA'` → throw `TaiKhoanBiKhoaException`.
  - `trangThai` chỉ dùng cho khóa tài khoản bởi quản trị viên; khóa tạm do đăng nhập sai được kiểm tra riêng trong Redis.
  - `argon2.verify(taiKhoan.matKhauBam, dto.matKhau)`.
  - Sai → `INCR khoa:${email}:count` trong Redis (TTL = `THOI_GIAN_KHOA_PHUT * 60`); đạt ngưỡng `KHOA_TAI_KHOAN_SAU_SO_LAN_SAI` thì từ chối đăng nhập trong khoảng thời gian cấu hình.
  - Đúng → sign JWT, `res.cookie('access_token', token, { httpOnly: true, sameSite: 'lax', secure: NODE_ENV==='production', maxAge })`, trả body.

- [ ] **1.5a** Dùng Redis cho lưu trạng thái khóa tạm và token khôi phục mật khẩu:
  - Khóa tạm: `khoa:${email}:count`, TTL tự động expire.
  - Token khôi phục: `reset:${email}` → token, TTL 1800 giây (30 phút). **Token chỉ dùng một lần** — verify xong `DEL` key. Không thêm cột vào schema Prisma.

- [ ] **1.5b** **Kiểm tra trạng thái tài khoản khi tạo giao dịch:** Tại các endpoint tạo đơn, thêm giỏ… phải kiểm tra `TaiKhoan.trangThai` ngay cả khi JWT còn hợp lệ. Tài khoản bị khóa không được tạo giao dịch mới. Tạo interceptor/guard `AccountStatusGuard` áp cho các endpoint mutation.

- [ ] **1.5c** **Lấy mã khách từ tài khoản đăng nhập:** Từ JWT claims (`sub = maTaiKhoan`), truy vấn `KhachHang` để lấy `maKhachHang`. Dùng `maKhachHang` này cho mọi thao tác giỏ/đơn — **khách chỉ được thao tác giỏ và đơn của mình.** Cung cấp decorator `@CurrentKhachHang()` để inject `maKhachHang` vào handler.

- [ ] **1.6** `AuthController` (`@Controller('auth')`):
  - `POST /api/v1/auth/dang-ky` → 201 + AuthResponse
  - `POST /api/v1/auth/dang-nhap` → 200 + AuthResponse (set cookie)
  - `POST /api/v1/auth/dang-xuat` → clear cookie
  - `POST /api/v1/auth/quen-mat-khau` → mock log token
  - `POST /api/v1/auth/dat-lai-mat-khau` → đổi mật khẩu
  - `GET /api/v1/auth/toi` `@UseGuards(JwtAuthGuard)` → thông tin từ claims

- [ ] **1.7** DI: `AuthModule` import `JwtModule.registerAsync`, `PassportModule.register({ defaultStrategy: 'jwt' })`. Export `AuthService`.

- [ ] **1.8** Rate limiting: `@nestjs/throttler` bảo vệ `/api/v1/auth/*` — max 10 requests/phút/IP.

- [ ] **1.9** Test bằng curl (đăng ký + đăng nhập lấy cookie, dùng cookie gọi `/toi`).

- [ ] **1.10** Commit: `feat: UC01 - JWT HTTP-only cookie authentication with Argon2`

---

## Task 2: DanhMucSanPham (Cấp 0, self-ref)

**Prisma schema:**
```prisma
model DanhMucSanPham {
  maDanhMuc       BigInt   @id @default(autoincrement()) @map("ma_danh_muc")
  maDanhMucCha    BigInt?  @map("ma_danh_muc_cha")
  tenDanhMuc      String   @map("ten_danh_muc") @db.VarChar(255)
  moTa            String?  @map("mo_ta") @db.Text
  thuTuHienThi    Int      @map("thu_tu_hien_thi")
  trangThai       String   @map("trang_thai") @db.VarChar(50)

  danhMucCha      DanhMucSanPham?  @relation("DanhMucChaCon", fields: [maDanhMucCha], references: [maDanhMuc], onDelete: Restrict)
  danhMucCon      DanhMucSanPham[] @relation("DanhMucChaCon")
  sanPhams        SanPham[]

  @@map("danh_muc_san_pham")
}
```

**Các bước:**

- [ ] **2.1** Thêm model `DanhMucSanPham` vào `schema.prisma` — đủ 6 cột, self-ref qua relation named `DanhMucChaCon`.
- [ ] **2.2** `onDelete: Restrict` để không xóa danh mục cha khi còn con.
- [ ] **2.3** Chuẩn bị model xong → báo Người 1 tích hợp và tạo migration `add-danh-muc-san-pham` → cả nhóm `npx prisma migrate deploy`.
- [ ] **2.4** Chạy `npx prisma generate` để cập nhật client type.
- [ ] **2.5** Commit: `feat: add DanhMucSanPham model`

---

## Task 3: SanPham + HinhAnhSanPham (Cấp 1-2)

> **Phụ thuộc: Task 2** (SanPham cần FK maDanhMuc → DanhMucSanPham).

**Prisma schema:**
```prisma
enum TrangThaiKinhDoanh {
  DANG_KINH_DOANH
  TAM_NGUNG
  NGUNG_KINH_DOANH
}

model SanPham {
  maSanPham            BigInt   @id @default(autoincrement()) @map("ma_san_pham")
  maDanhMuc            BigInt   @map("ma_danh_muc")
  maSanPhamHienThi     String   @unique @map("ma_san_pham_hien_thi") @db.VarChar(50)
  tenSanPham           String   @map("ten_san_pham") @db.VarChar(255)
  thuongHieu           String?  @map("thuong_hieu") @db.VarChar(255)
  moTa                 String?  @map("mo_ta") @db.Text
  sucChua              Int      @map("suc_chua")                 // int riêng
  kichThuoc            String?  @map("kich_thuoc") @db.VarChar(255)  // varchar riêng
  thongSo              Json?    @map("thong_so")                 // JSON key-value
  giaThueMoiNgay       Decimal  @map("gia_thue_moi_ngay") @db.Decimal(18, 2)
  mucCocMoiThietBi     Decimal  @map("muc_coc_moi_thiet_bi") @db.Decimal(18, 2)
  giaTriBoiThuong      Decimal  @map("gia_tri_boi_thuong") @db.Decimal(18, 2)
  trangThaiKinhDoanh   TrangThaiKinhDoanh @map("trang_thai_kinh_doanh")

  danhMuc              DanhMucSanPham @relation(fields: [maDanhMuc], references: [maDanhMuc], onDelete: Restrict)
  hinhAnhs             HinhAnhSanPham[]

  @@map("san_pham")
}

model HinhAnhSanPham {
  maHinhAnh    BigInt  @id @default(autoincrement()) @map("ma_hinh_anh")
  maSanPham    BigInt  @map("ma_san_pham")
  duongDan     String  @map("duong_dan") @db.VarChar(500)
  laAnhChinh   Boolean @default(false) @map("la_anh_chinh")
  thuTu        Int     @map("thu_tu")

  sanPham      SanPham @relation(fields: [maSanPham], references: [maSanPham], onDelete: Cascade)

  @@map("hinh_anh_san_pham")
}
```

**Các bước:**

- [ ] **3.1** Enum Prisma `TrangThaiKinhDoanh`.
- [ ] **3.2** Model `SanPham` — đủ 2 cột `sucChua Int` + `kichThuoc String?`, `thongSo Json?`.
- [ ] **3.3** Model `HinhAnhSanPham` — cascade delete theo SanPham.
- [ ] **3.4** Báo Người 1 tạo migration `add-san-pham-hinh-anh`.
- [ ] **3.5** Commit: `feat: add SanPham and HinhAnhSanPham models`

---

## Task 4: KhuyenMai + bảng nối (Cấp 0, 2)

> **Bảng nối `KhuyenMaiDanhMuc` cần Task 2, `KhuyenMaiSanPham` cần Task 3.**

**Prisma schema:**
```prisma
enum LoaiGiam        { PHAN_TRAM SO_TIEN }
enum PhamViApDung    { TAT_CA THEO_SAN_PHAM THEO_DANH_MUC }
enum TrangThaiKhuyenMai { HIEN_THI TAM_AN HET_HAN }

model KhuyenMai {
  maKhuyenMai        BigInt   @id @default(autoincrement()) @map("ma_khuyen_mai")
  maGiamGia          String   @unique @map("ma_giam_gia") @db.VarChar(50)  // mã khách nhập
  tenKhuyenMai       String   @map("ten_khuyen_mai") @db.VarChar(255)
  loaiGiam           LoaiGiam @map("loai_giam")
  giaTriGiam         Decimal  @map("gia_tri_giam") @db.Decimal(18, 2)
  mucGiamToiDa       Decimal? @map("muc_giam_toi_da") @db.Decimal(18, 2)
  tienThueToiThieu   Decimal  @map("tien_thue_toi_thieu") @db.Decimal(18, 2)
  phamViApDung       PhamViApDung @map("pham_vi_ap_dung")
  batDau             DateTime @map("bat_dau")
  ketThuc            DateTime @map("ket_thuc")
  gioiHanTongLuot    Int      @map("gioi_han_tong_luot")
  gioiHanMoiKhach    Int      @map("gioi_han_moi_khach")
  trangThai          TrangThaiKhuyenMai @map("trang_thai")

  sanPhams           KhuyenMaiSanPham[]
  danhMucs           KhuyenMaiDanhMuc[]
  gioThues           GioThue[]
  luotSuDungs        LuotSuDungKhuyenMai[]

  @@map("khuyen_mai")
}

model KhuyenMaiSanPham {
  maKhuyenMai BigInt
  maSanPham   BigInt
  khuyenMai   KhuyenMai @relation(fields: [maKhuyenMai], references: [maKhuyenMai], onDelete: Cascade)
  sanPham     SanPham   @relation(fields: [maSanPham], references: [maSanPham], onDelete: Restrict)
  @@id([maKhuyenMai, maSanPham])
  @@map("khuyen_mai_san_pham")
}

model KhuyenMaiDanhMuc {
  maKhuyenMai BigInt
  maDanhMuc   BigInt
  khuyenMai   KhuyenMai      @relation(fields: [maKhuyenMai], references: [maKhuyenMai], onDelete: Cascade)
  danhMuc     DanhMucSanPham @relation(fields: [maDanhMuc], references: [maDanhMuc], onDelete: Restrict)
  @@id([maKhuyenMai, maDanhMuc])
  @@map("khuyen_mai_danh_muc")
}
```

**Các bước:**

- [ ] **4.1** Enums: `LoaiGiam`, `PhamViApDung`, `TrangThaiKhuyenMai`.
- [ ] **4.2** Model `KhuyenMai` đầy đủ 13 trường; `maGiamGia` unique.
- [ ] **4.3** 2 bảng nối composite PK.
- [ ] **4.4** Báo Người 1 tạo migration `add-khuyen-mai`.
- [ ] **4.5** Tạo `KhuyenMaiModule` + `KhuyenMaiService` (bàn giao Day 2):
  ```typescript
  interface KhuyenMaiService {
    kiemTraApDung(maGiamGia: string, maKhachHang: bigint, dong: DongGio[], tienThueTruocGiam: Decimal): Promise<Result<KhuyenMaiHopLe>>;
    giuLuot(maKhuyenMai: bigint, maDonThue: bigint, thoiDiemHetHan: Date, tx: Prisma.TransactionClient): Promise<void>;
    xacNhanDaSuDung(maDonThue: bigint, tx: Prisma.TransactionClient): Promise<void>;
    giaiPhongLuot(maDonThue: bigint, tx: Prisma.TransactionClient): Promise<void>;
  }
  ```
  Kiểm tra đủ: trạng thái `HIEN_THI` (loại `TAM_AN`/`HET_HAN`), thời hạn, phạm vi, mức tối thiểu, giới hạn tổng lượt, giới hạn mỗi khách. **Loại lượt `DANG_GIU` đã hết hạn khi đếm.**
  - **Người 4 (giỏ):** gọi `kiemTraApDung` để báo giá dự kiến, **KHÔNG giữ lượt**.
  - **Người 5 (tạo đơn):** gọi lại `kiemTraApDung` trong transaction + `giuLuot`.
  - Callback thanh toán: gọi `xacNhanDaSuDung`. Hủy đơn/hết hạn: `giaiPhongLuot`.

- [ ] **4.6** Test đơn vị (Jest) đầy đủ nhánh: hợp lệ, `TAM_AN`, hết hạn, không đủ tối thiểu, vượt tổng lượt, vượt lượt/khách, lượt `DANG_GIU` hết hạn không tính.

- [ ] **4.7** Commit: `feat: add KhuyenMai model, service and validation logic`

---

## Task 5: Nhập kho & Thiết bị (Cấp 0-4, skeleton + seed)

**Lý do cần ở Tuần 2:** `ThietBi.maChiTietPhieuNhap` là NOT NULL FK → không có phiếu nhập không có thiết bị. `KhaDungService` (UC03) cần đếm `ThietBi` để tính khả dụng.

**Chiến lược:** Model skeleton đầy đủ cột theo ERD (KHÔNG có API/UI — dời sang Tuần 3). Chỉ seed dữ liệu mẫu.

> **Phụ thuộc: Task 3** (ChiTietPhieuNhap cần FK maSanPham → SanPham).

**Các bước:**

- [ ] **5.1** Model `NhaCungCap` (Cấp 0) — 10 trường: `maNhaCungCap`, `maNhaCungCapHienThi` (unique), `tenNhaCungCap`, `nguoiLienHe`, `soDienThoai`, `email`, `diaChi`, `maSoThue`, `ghiChu`, `trangThaiHopTac`.

- [ ] **5.2** Model `PhieuNhapHang` (Cấp 2) — FK: `maNhaCungCap`, `maNguoiLap` (NhanVien), `maNguoiXacNhan` (NhanVien, nullable). Đủ **17 trường** theo ERD: `maPhieuNhap`, `maNhaCungCap`, `maNguoiLap`, `maNguoiXacNhan`, `maPhieuHienThi` (unique), `soChungTuNhaCungCap`, `ngayLap`, `ngayNhapDuKien`, `ngayNhapThucTe`, `ngayXacNhan`, `tongTien`, `thongTinNhaCungCapLucNhap` (Json), `tenNguoiLapLucNhap`, `tenNguoiXacNhanLucNhap`, `trangThai`, `lyDoHuy`, `ghiChu`.

- [ ] **5.3** Model `ChiTietPhieuNhap` (Cấp 3) — FK: `maPhieuNhap`, `maSanPham`. Đủ **8 trường**: `maChiTietPhieuNhap`, `maPhieuNhap`, `maSanPham`, `tenSanPhamLucNhap`, `soLuong`, `donGiaNhap`, `tinhTrangKhiNhap`, `ghiChu`.

- [ ] **5.4** Model `ThietBi` (Cấp 4) — FK: `maChiTietPhieuNhap`. Đủ **9 trường** theo ERD: `maThietBi`, `maChiTietPhieuNhap`, `maThietBiHienThi` (unique), `ngayNhap`, `giaNhap`, `tinhTrang`, `phuKienDiKem` (Json), `trangThaiSuDung`, `ghiChu`.

- [ ] **5.5** Enum `TrangThaiSuDungThietBi`: `SAN_SANG`, `DANG_THUE`, `DANG_BAO_TRI`, `THAT_LAC`, `NGUNG_SU_DUNG`.

- [ ] **5.6** Prisma unique: `NhaCungCap.maNhaCungCapHienThi`, `PhieuNhapHang.maPhieuHienThi`, `ThietBi.maThietBiHienThi`.

- [ ] **5.7** Báo Người 1 tạo migration `add-kho-thiet-bi`.

- [ ] **5.8** Commit: `feat: add NhaCungCap, PhieuNhapHang, ChiTietPhieuNhap, ThietBi (skeleton for Week 3)`

---

## Task 6: GioThue + ChiTietGioThue (Cấp 2-3)

**Prisma schema:**
```prisma
model GioThue {
  maGioThue        BigInt   @id @default(autoincrement()) @map("ma_gio_thue")
  maKhachHang      BigInt   @unique @map("ma_khach_hang")   // 1-1 với KhachHang
  maKhuyenMai      BigInt?  @map("ma_khuyen_mai")           // FK về KhuyenMai (bigint!)
  gioNhanDuKien    DateTime? @map("gio_nhan_du_kien")
  gioTraDuKien     DateTime? @map("gio_tra_du_kien")
  ngayCapNhat      DateTime @updatedAt @map("ngay_cap_nhat")

  khachHang        KhachHang  @relation(fields: [maKhachHang], references: [maKhachHang], onDelete: Cascade)
  khuyenMai        KhuyenMai? @relation(fields: [maKhuyenMai], references: [maKhuyenMai])
  chiTiets         ChiTietGioThue[]

  @@map("gio_thue")
}

model ChiTietGioThue {
  maChiTietGio  BigInt  @id @default(autoincrement()) @map("ma_chi_tiet_gio")
  maGioThue     BigInt  @map("ma_gio_thue")
  maSanPham     BigInt  @map("ma_san_pham")
  soLuong       Int     @map("so_luong")

  gioThue       GioThue @relation(fields: [maGioThue], references: [maGioThue], onDelete: Cascade)
  sanPham       SanPham @relation(fields: [maSanPham], references: [maSanPham], onDelete: Restrict)

  @@map("chi_tiet_gio_thue")
}
```

**Các bước:**

- [ ] **6.1** Model `GioThue` với FK về `KhuyenMai` (nullable `BigInt`), 1-1 với `KhachHang` qua `@unique`.
- [ ] **6.2** Model `ChiTietGioThue` cascade Delete theo `GioThue`.
- [ ] **6.3** Báo Người 1 tạo migration `add-gio-thue`.
- [ ] **6.4** Commit: `feat: add GioThue and ChiTietGioThue models`

---

## Task 7: Đơn thuê hoàn chỉnh (Cấp 2-5)

> **Phụ thuộc: Task 3 (SanPham) và Task 4 (KhuyenMai)**. Không cần chờ model Task 6.

**6 model (thêm LichSuTrangThaiDon):**
- `ChinhSach` (Cấp 2) — versioned policy
- `DonThue` (Cấp 3)
- `ChiTietDonThue` (Cấp 4)
- `GiuCho` (Cấp 5, 1-1 ChiTietDonThue)
- `LuotSuDungKhuyenMai` (Cấp 4, 1-1 DonThue)
- `LichSuTrangThaiDon` (ghi lịch sử chuyển trạng thái)

**Các bước:**

- [ ] **7.1** Enum `TrangThaiDonThue`: `CHO_THANH_TOAN`, `DA_XAC_NHAN`, `DANG_CHUAN_BI`, `SAN_SANG_NHAN`, `DANG_THUE`, `DA_NHAN_TRA`, `CHO_DOI_SOAT`, `HOAN_TAT`, `HET_HAN`, `KHACH_HUY`, `CUA_HANG_HUY`.

- [ ] **7.2** Model `ChinhSach` (versioned policy):
  - `maChinhSach`, `maNguoiTao` (FK NhanVien), `tenChinhSach`, `phienBan` (Int, unique), `thoiDiemApDung`, `noiDungChinhSach` (Json), `ngayTao`.

- [ ] **7.3** Model `DonThue` đầy đủ **22 trường** theo ERD:
  - FK: `maKhachHang`, `maChinhSach`, `maNguoiHuy?` (FK **TaiKhoan**, không phải NhanVien).
  - `maDonHienThi` unique.
  - Thời gian: `ngayDat`, **`gioNhanDuKien`**, **`gioTraDuKien`**, `hanThanhToan`, `thoiDiemHuy?`, `thoiDiemHoanTat?`.
  - Người nhận: `tenNguoiNhan`, `soDienThoaiNguoiNhan`, `emailLienHe`.
  - Tiền: `tongTienThueTruocGiam`, `tongTienGiam`, `tongTienCoc`, `tienThueGiuLaiKhiHuy` (Decimal 18,2).
  - Snapshot: `khuyenMaiLucDat` (Json?).
  - Other: `trangThai` (enum), `lyDoHuy?`, `ghiChu?`.

- [ ] **7.4** Model `ChiTietDonThue` với snapshot đầy đủ:
  - FK: `maDonThue`, `maSanPham`.
  - Snapshot: `tenSanPhamLucDat`, `donGiaThueMoiNgay`, `mucCocMoiThietBi`, `giaTriBoiThuongMoiThietBi`, `phuKienVaMucBoiThuongLucDat` (Json).
  - `soNgayTinhTien`, `soLuong`, `tienGiam`.

- [ ] **7.5** Model `GiuCho` (1-1 với ChiTietDonThue):
  - `maChiTietDon` **unique** (FK 1-1).
  - `thoiDiemTao`, **`thoiDiemHetHan`**, `thoiDiemGiaiPhong?`, `trangThai`.

- [ ] **7.6** Enum `TrangThaiGiuCho`: `DANG_GIU`, `DA_XAC_NHAN`, `DA_GIAI_PHONG`, `HET_HAN`.

- [ ] **7.7** Model `LuotSuDungKhuyenMai` (1-1 với DonThue):
  - `maDonThue` **unique** (FK 1-1).
  - **`maKhuyenMai`** (FK NOT NULL về KhuyenMai).
  - `thoiDiemGiuLuot`, **`thoiDiemHetHan`**, `thoiDiemSuDung?`, `thoiDiemGiaiPhong?`, `soTienGiam`, `trangThai`.

- [ ] **7.8** Model `LichSuTrangThaiDon` theo ERD:
  - `maLichSuDon`, `maDonThue` (FK), `maNguoiThucHien` (FK TaiKhoan, nullable), `trangThaiTruoc`, `trangThaiSau`, `thoiDiem`, `lyDo?`.

- [ ] **7.9** Prisma constraints:
  - `DonThue.maDonHienThi` unique.
  - `DonThue.maKhachHang` `onDelete: Restrict`.
  - `GiuCho.maChiTietDon` unique + cascade Delete theo `ChiTietDonThue`.
  - `LuotSuDungKhuyenMai.maDonThue` unique.
  - `ChinhSach.phienBan` unique.
  - Index: `DonThue.trangThai`, `GiuCho.thoiDiemHetHan`, `LuotSuDungKhuyenMai.thoiDiemHetHan`.

- [ ] **7.10** Báo Người 1 tạo migration `add-don-thue-flow`.

- [ ] **7.11** Commit: `feat: add ChinhSach, DonThue, ChiTietDonThue, GiuCho, LuotSuDungKhuyenMai, LichSuTrangThaiDon`

---

## Task 8: ThanhToan + ChiTietThanhToan (Cấp 4-5)

**Prisma schema:**
```prisma
enum MucDichThanhToan   { TIEN_THUE TIEN_COC THU_BO_SUNG }
enum TrangThaiThanhToan { DANG_XU_LY THANH_CONG THAT_BAI HUY }

model ThanhToan {
  maThanhToan         BigInt   @id @default(autoincrement()) @map("ma_thanh_toan")
  maDonThue           BigInt   @map("ma_don_thue")
  maNguoiGhiNhan      BigInt?  @map("ma_nguoi_ghi_nhan")           // FK NhanVien
  maYeuCau            String   @unique @map("ma_yeu_cau") @db.VarChar(100)  // idempotency key
  congThanhToan       String   @map("cong_thanh_toan") @db.VarChar(50)
  maGiaoDichCong      String?  @map("ma_giao_dich_cong") @db.VarChar(255)
  tongSoTien          Decimal  @map("tong_so_tien") @db.Decimal(18, 2)
  phuongThuc          String   @map("phuong_thuc") @db.VarChar(50)
  thoiDiemTao         DateTime @default(now()) @map("thoi_diem_tao")
  thoiDiemThanhCong   DateTime? @map("thoi_diem_thanh_cong")
  trangThai           TrangThaiThanhToan @map("trang_thai")
  trangThaiDoiChieu   String?  @map("trang_thai_doi_chieu") @db.VarChar(50)
  ghiChu              String?  @map("ghi_chu") @db.Text

  donThue             DonThue  @relation(fields: [maDonThue], references: [maDonThue], onDelete: Restrict)
  chiTiets            ChiTietThanhToan[]

  @@index([maGiaoDichCong])
  @@map("thanh_toan")
}

model ChiTietThanhToan {
  maChiTietThanhToan BigInt @id @default(autoincrement()) @map("ma_chi_tiet_thanh_toan")
  maThanhToan        BigInt @map("ma_thanh_toan")
  mucDich            MucDichThanhToan @map("muc_dich")
  soTien             Decimal @map("so_tien") @db.Decimal(18, 2)

  thanhToan          ThanhToan @relation(fields: [maThanhToan], references: [maThanhToan], onDelete: Cascade)

  @@map("chi_tiet_thanh_toan")
}
```

**Các bước:**

- [ ] **8.1** Enum `MucDichThanhToan`: `TIEN_THUE`, `TIEN_COC`, `THU_BO_SUNG`. Hoàn cọc sau bằng `HoanTien`, không phải khoản thu.
- [ ] **8.2** Enum `TrangThaiThanhToan`: `DANG_XU_LY`, `THANH_CONG`, `THAT_BAI`, `HUY`.
- [ ] **8.3** Model `ThanhToan` (13 trường) — `maYeuCau` **unique** (idempotency guard).
- [ ] **8.4** Model `ChiTietThanhToan`.
- [ ] **8.5** Prisma: unique `maYeuCau`, index `maGiaoDichCong`.
- [ ] **8.6** Báo Người 1 tạo migration `add-thanh-toan`.
- [ ] **8.7** Commit: `feat: add ThanhToan and ChiTietThanhToan models`

---

## Task 9: KhaDungService + UC03

**Files:**
- Tạo: `apps/api/src/kha-dung/kha-dung.module.ts`, `kha-dung.service.ts`
- Tạo: `apps/api/src/danh-muc/danh-muc.module.ts`, `danh-muc.controller.ts`, `danh-muc.service.ts`
- Tạo: `apps/api/src/san-pham/san-pham.controller.ts` (public), `san-pham.service.ts`
- Tạo DTOs trong `apps/api/src/san-pham/dto/`, `apps/api/src/danh-muc/dto/`

> **Phụ thuộc: Task 5 (ThietBi) và Task 7 (DonThue, GiuCho).**

**Các bước:**

- [ ] **9.1** `KhaDungService`:
  ```typescript
  layKhaDung(maSanPham: bigint, gioNhan: Date, gioTra: Date): Promise<number>
  layKhaDungNhieu(maSanPhams: bigint[], gioNhan: Date, gioTra: Date): Promise<Map<bigint, number>>
  ```

- [ ] **9.2** Công thức khả dụng (mục 8.1 đặc tả) — **không cache**:
  - **Chưa chọn ngày:** chỉ hiện giá tham khảo, **không khẳng định còn hàng**.
  - **Kiểm tra đầu vào:** giờ trả phải sau giờ nhận; không tạo lượt thuê bắt đầu trong quá khứ.
  - Đếm tổng `ThietBi` **đủ điều kiện**: `trangThaiSuDung` thuộc `{SAN_SANG, DANG_THUE}` + join qua `ChiTietPhieuNhap` từ **phiếu nhập đã xác nhận** (`trangThai = 'DA_NHAP_KHO'`). Thiết bị `DANG_BAO_TRI`, `THAT_LAC`, `NGUNG_SU_DUNG` không tính.
  - **Giữ chỗ tạm chiếm lịch khi ĐỒNG THỜI:** đơn `CHO_THANH_TOAN` + `GiuCho.trangThai = DANG_GIU` + `thoiDiemHetHan > Now`.
  - **Đơn đã xác nhận (`DA_XAC_NHAN`, `DANG_CHUAN_BI`, `SAN_SANG_NHAN`, `DANG_THUE`, ...) tính riêng và chỉ tính một lần** — không phụ thuộc hạn 15 phút.
  - **Giữ chỗ hết hạn không chiếm lịch** dù job chưa chạy (`thoiDiemHetHan > Now` loại chúng ra).
  - **Không tính trùng:** hai bản ghi cùng thuộc một `ChiTietDonThue` chỉ tính một lần.
  - Công thức: **`Khả dụng = Tổng thiết bị đủ điều kiện − Số lượng bị chiếm đồng thời lớn nhất trên khoảng [gioNhan, gioTra]`** — không cộng dồn các đơn không giao nhau về thời gian.
  - Lịch giao nhau: `gioNhanDon < gioTra` AND `gioTraDon > gioNhan`.
  - Ví dụ: có 5 lều, đơn A thuê 3 cái ngày 20, đơn B thuê 3 cái ngày 21 và hai đơn không trùng nhau; khách thuê xuyên hai ngày vẫn còn 2 cái, không phải `5 - 3 - 3`.
  - Có thể dùng `prisma.$queryRaw` cho tính "số lượng bị chiếm đồng thời lớn nhất" (sweep line) hoặc xử lý ở TypeScript.

- [ ] **9.3** `SanPhamService.timKiem(dto)`:
  - Filter: `tuKhoa`, `maDanhMuc`, `thuongHieu`, `sucChua`, `giaMin`, `giaMax`, `gioNhan`, `gioTra`.
  - Sort: `sapXep` (`GIA_TANG`, `GIA_GIAM`, `MOI_NHAT`, `PHO_BIEN`).
  - Phân trang: `trang`, `soMoiTrang` (default 12, max 50).
  - Chỉ trả `trangThaiKinhDoanh = 'DANG_KINH_DOANH'` cho khách; danh mục `trangThai != 'HIEN_THI'` cũng bị lọc.
  - Có `gioNhan` + `gioTra` → gọi `KhaDungService.layKhaDungNhieu`.
  - Cache Redis: key = hash(JSON.stringify(dto)), TTL 2 phút. Invalidate khi admin sửa sản phẩm.

- [ ] **9.4** `SanPhamController` (public):
  - `GET /api/v1/san-pham` — list + phân trang.
  - `GET /api/v1/san-pham/:id?gioNhan=&gioTra=` — chi tiết + khả dụng.

- [ ] **9.5** `DanhMucController` (public):
  - `GET /api/v1/danh-muc` — cây danh mục (cache Redis 10 phút, key `danh_muc:all`).
  - `GET /api/v1/danh-muc/:id` — chi tiết.

- [ ] **9.6** Đăng ký DI: `KhaDungService`, `DanhMucService`, `SanPhamService`.

- [ ] **9.7** Commit: `feat: UC03 - product search with availability check and Redis cache`

---

## Task 10: UC04 (giỏ) + UC05 (đặt đơn) + UC06 (thanh toán)

### UC04 — Giỏ thuê

- [ ] **10.1** `GioThueService`:
  ```typescript
  layGio(maKhachHang: bigint): Promise<GioThueResponse>
  them(maKhachHang: bigint, maSanPham: bigint, soLuong: number): Promise<GioThueResponse>
  capNhatSoLuong(maKhachHang: bigint, maChiTiet: bigint, soLuongMoi: number): Promise<GioThueResponse>
  xoaChiTiet(maKhachHang: bigint, maChiTiet: bigint): Promise<void>
  datThoiGian(maKhachHang: bigint, gioNhan: Date, gioTra: Date): Promise<GioThueResponse>
  apMaKhuyenMai(maKhachHang: bigint, maGiamGia: string): Promise<GioThueResponse>
  ```

- [ ] **10.2** Logic giỏ:
  - **Thêm sản phẩm đã có → cộng dồn số lượng**, không tạo dòng mới.
  - **Số lượng phải là số nguyên dương** (> 0).
  - **Đổi ngày/số lượng → tính lại báo giá và kiểm tra khả dụng**.
  - **Giỏ không giữ hàng.**
  - **Giỏ hiển thị cờ "ngừng kinh doanh"** cho dòng sản phẩm có `trangThaiKinhDoanh != 'DANG_KINH_DOANH'` — khách phải xóa dòng đó trước khi tạo đơn.

- [ ] **10.3** `BaoGiaService` (tách riêng, dùng chung giỏ + tạo đơn) — mục 8.2 đặc tả:
  - `soNgay = Math.ceil((gioTra.getTime() - gioNhan.getTime()) / 86_400_000)`, tối thiểu 1.
  - `tienThue = SUM(donGiaThueMoiNgay * soLuong * soNgay)`.
  - `tienCoc = SUM(mucCocMoiThietBi * soLuong)`.
  - Áp `KhuyenMai`: gọi `KhuyenMaiService.kiemTraApDung`, kiểm tra `phamViApDung`, `tienThueToiThieu`, `mucGiamToiDa`.
  - **Chỉ giảm tiền thuê**; không giảm cọc, không vượt tiền thuê phần hàng đủ điều kiện.
  - **Phân bổ giảm giá xuống từng dòng**, làm tròn thống nhất (đồng VNĐ) — dùng `Prisma.Decimal` với rounding `ROUND_HALF_UP`.
  - Có cơ chế **lưu/đối chiếu báo giá khách đã xem** (hash SHA-256 các trường `{maSanPham, soLuong, donGia}` sắp xếp deterministic) để phát hiện giá thay đổi khi tạo đơn.

- [ ] **10.4** Kiểm tra mã khuyến mãi (gọi `KhuyenMaiService.kiemTraApDung`):
  - **Trạng thái:** chỉ chấp nhận `HIEN_THI`; `TAM_AN`/`HET_HAN` → từ chối.
  - Thời hạn: `batDau <= Now <= ketThuc`.
  - Phạm vi: sản phẩm/danh mục phù hợp.
  - Mức tối thiểu: `tienThueTruocGiam >= tienThueToiThieu`.
  - **Giới hạn tổng lượt**: đếm `LuotSuDungKhuyenMai` có `trangThai` ∈ {`DANG_GIU` **còn hạn**, `DA_SU_DUNG`} < `gioiHanTongLuot`.
  - **Giới hạn mỗi khách**: đếm lượt của khách tương tự.
  - **Lượt `DANG_GIU` đã hết hạn không tiếp tục chiếm lượt** — loại chúng khi đếm.
  - Giỏ chỉ báo giá dự kiến, **KHÔNG giữ lượt**; tạo đơn mới gọi `giuLuot`.

- [ ] **10.5** `GioThueController` (`@UseGuards(JwtAuthGuard)`):
  - `GET /api/v1/gio-thue`
  - `POST /api/v1/gio-thue/them`
  - `PUT /api/v1/gio-thue/:maChiTiet/so-luong`
  - `DELETE /api/v1/gio-thue/:maChiTiet`
  - `PUT /api/v1/gio-thue/thoi-gian`
  - `POST /api/v1/gio-thue/ma-giam-gia`

### UC05 — Đặt đơn + giữ chỗ

- [ ] **10.6** `DonThueService.taoDon` trong `prisma.$transaction(async (tx) => {...}, { isolationLevel: 'Serializable' })`:
  1. Load giỏ, kiểm tra không rỗng.
  2. **Kiểm tra `trangThaiKinhDoanh` từng sản phẩm** — phải `DANG_KINH_DOANH`; nếu không → `TrangThaiKhongHopLeException`.
  3. Kiểm tra khả dụng mỗi dòng → thiếu → `KhongDuHangException`.
  4. Nếu báo giá thay đổi (so hash) → `BaogiaThayDoiException`.
  5. **Nếu giỏ có mã khuyến mãi** → gọi lại `KhuyenMaiService.kiemTraApDung`.
  6. Load `ChinhSach` **đang có hiệu lực** (`thoiDiemApDung <= Now`, phiên bản mới nhất thỏa điều kiện).
  7. **Gán hạn thanh toán thống nhất:**
     ```typescript
     const thoiDiemTaoDon = new Date();
     const hanThanhToan = new Date(thoiDiemTaoDon.getTime() + 15 * 60 * 1000);
     donThue.hanThanhToan = hanThanhToan;
     giuCho.thoiDiemHetHan = hanThanhToan;
     luotSuDungKhuyenMai.thoiDiemHetHan = hanThanhToan;
     ```
  8. Tạo `DonThue` (`CHO_THANH_TOAN`), sinh `maDonHienThi` unique. Lưu `gioNhanDuKien`, `gioTraDuKien`, `hanThanhToan`.
  9. Snapshot vào `ChiTietDonThue`: giá + phụ kiện + bồi thường tại thời điểm đặt.
  10. Với mỗi `ChiTietDonThue`: tạo `GiuCho` (`DANG_GIU`, `thoiDiemHetHan = hanThanhToan`).
  11. Nếu có khuyến mãi: gọi `KhuyenMaiService.giuLuot` — tạo `LuotSuDungKhuyenMai` (`DANG_GIU`, `thoiDiemHetHan = hanThanhToan`).
  12. Xóa giỏ (delete `ChiTietGioThue` + reset `GioThue` các trường thời gian và mã).
  - **Toàn bộ trong cùng transaction.** Thất bại → **rollback toàn bộ** và **giữ nguyên giỏ**.
  - **Hai yêu cầu đồng thời** không tạo hai đơn từ cùng dữ liệu giỏ (Serializable + kiểm tra giỏ rỗng).
  - **Ghi lịch sử chuyển trạng thái** vào `LichSuTrangThaiDon`.
  - Sau commit: enqueue BullMQ job `expire-don-thue` delay 15 phút (`Queue.add('expire', { maDonThue }, { delay: 900_000 })`).

> **Quy ước hết hạn:** còn hạn khi `Now < hạn`; hết hạn khi `Now >= hạn`. Query đếm lượt đang chiếm dùng `thoiDiemHetHan > Now`. Job hết hạn dùng `hạn <= Now`.

- [ ] **10.7** Hủy đơn chưa thanh toán — **cùng transaction, cập nhật có điều kiện**:
  - `updateMany` `DonThue` `where: { maDonThue, trangThai: 'CHO_THANH_TOAN' }` → `KHACH_HUY`/`CUA_HANG_HUY`. Nếu `count == 0` → đơn đã đổi trạng thái → `TrangThaiKhongHopLeException`, rollback.
  - Ghi `maNguoiHuy` (MaTaiKhoan), `thoiDiemHuy`, `lyDoHuy`.
  - Giải phóng `GiuCho`: `updateMany` `where: { maChiTietDon IN (...), trangThai: 'DANG_GIU' }` → `DA_GIAI_PHONG`.
  - Giải phóng lượt mã: `KhuyenMaiService.giaiPhongLuot`.
  - Ghi lịch sử chuyển trạng thái.
  - Toàn bộ trong **cùng transaction**; thất bại → rollback.

- [ ] **10.8** BullMQ processor `expire-don-thue` — **job hết hạn, cập nhật có điều kiện trong transaction**:
  - Với mỗi job `{ maDonThue }`, mở `prisma.$transaction`:
    - `updateMany` `DonThue` `where: { maDonThue, trangThai: 'CHO_THANH_TOAN' }` → `HET_HAN`. Nếu `count == 0` → bỏ qua, commit (không ghi đè).
    - Giải phóng `GiuCho` (`thoiDiemHetHan <= Now` AND `trangThai = 'DANG_GIU'`) → `HET_HAN`.
    - Giải phóng `LuotSuDungKhuyenMai` tương tự.
    - Ghi lịch sử chuyển trạng thái.
    - Commit.
  - **Không ghi đè trạng thái nhau:** `updateMany` có điều kiện `where: { trangThai: 'CHO_THANH_TOAN' }` là cơ chế đảm bảo.
  - Job idempotent — chạy nhiều lần cho cùng `maDonThue` không gây tác dụng phụ.

- [ ] **10.9** `DonThueController` (`@UseGuards(JwtAuthGuard)`):
  - `GET /api/v1/don-thue/xac-nhan` — preview.
  - `POST /api/v1/don-thue` — tạo đơn.
  - `GET /api/v1/don-thue/:id` — chi tiết.
  - `GET /api/v1/don-thue` — danh sách đơn của khách.
  - `POST /api/v1/don-thue/:id/huy` — chỉ hủy đơn chưa thanh toán.

### UC06 — Thanh toán

> Gateway mock chỉ phục vụ demo luồng thành công và idempotency; các ngoại lệ UC06 khác thuộc phạm vi triển khai sau.

- [ ] **10.10** Mock gateway `taoUrl(maDonThue, returnUrl)`:
  - Tạo giao dịch `ThanhToan` với `trangThai = 'DANG_XU_LY'`.
  - `maYeuCau = crypto.randomUUID().replace(/-/g, '')`.
  - Trả URL: `${returnUrl}?donId={id}&maYeuCau={maYeuCau}&ketQua=success&maGD={fakeId}`.

- [ ] **10.11** `xuLyKetQua(callback)` — **thứ tự bước quan trọng để idempotency đúng**:
  1. **Tìm giao dịch theo `maYeuCau`** — không thấy → 404.
  2. **Idempotency ưu tiên trước:** nếu giao dịch đã `THANH_CONG` → **trả kết quả cũ ngay lập tức**, không thu thêm, **không yêu cầu đơn phải còn `CHO_THANH_TOAN`**.
  3. Nếu giao dịch đã `THAT_BAI`/`HUY` → trả lỗi cuối.
  4. Chỉ khi giao dịch còn `DANG_XU_LY` mới đi tiếp:
     - **Kiểm tra số tiền:** `tongSoTien = tongTienThueTruocGiam - tongTienGiam + tongTienCoc`.
     - **Cập nhật nguyên tử trong 1 `prisma.$transaction` (updateMany có điều kiện):**
       - `updateMany` `DonThue` `where: { maDonThue, trangThai: 'CHO_THANH_TOAN', hanThanhToan: { gt: now } }` → `DA_XAC_NHAN`.
       - `updateMany` `ThanhToan` `where: { maThanhToan, trangThai: 'DANG_XU_LY' }` → `THANH_CONG` + tạo 2 `ChiTietThanhToan` (`TIEN_THUE` + `TIEN_COC`).
       - `updateMany` `GiuCho` → `DA_XAC_NHAN`.
       - Gọi `KhuyenMaiService.xacNhanDaSuDung` → `LuotSuDungKhuyenMai` → `DA_SU_DUNG` (**không tạo lượt mới**).
       - Ghi lịch sử chuyển trạng thái.
       - Commit.
  - **Nếu `updateMany` đơn trả `count == 0`** — **KHÔNG tự ghi đè `THAT_BAI`**:
    - Rollback, đọc lại đơn và giao dịch trong transaction mới:
      - Giao dịch đã `THANH_CONG` → trả kết quả cũ.
      - Đơn `DA_XAC_NHAN` mà giao dịch này vẫn `DANG_XU_LY` (bất thường — giao dịch khác đã confirm trước) → **kiểm tra gateway có thu tiền không**: có → `trangThaiDoiChieu = 'CAN_DOI_SOAT'` + `THANH_CONG` + tạo yêu cầu hoàn tiền thu trùng (UC07 tuần sau), **không tự `THAT_BAI`**; chưa thu → `THAT_BAI` + ghi chú "đơn đã được giao dịch khác xác nhận".
      - Đơn `HET_HAN`/`KHACH_HUY`/`CUA_HANG_HUY` mà tiền đã thu → cập nhật `trangThaiDoiChieu = 'CAN_DOI_SOAT'`, tạo yêu cầu hoàn tiền (UC07). **Không mặc định là `THAT_BAI`**.
      - Ghi log chi tiết để nghiệp vụ đối soát thủ công.

- [ ] **10.12** `ThanhToanController`:
  - `POST /api/v1/thanh-toan/:maDon/tao-url` `@UseGuards(JwtAuthGuard)`.
  - `GET /api/v1/thanh-toan/ket-qua` — callback từ gateway. **Xác thực chữ ký HMAC khi tích hợp VNPay thật** — tuần sau; tuần 2 mock trực tiếp.

- [ ] **10.13** Commit: `feat: UC04-UC06 - cart, order creation, mock payment API`

---

## Task 11: UC21 — Admin CRUD danh mục & sản phẩm

**Các bước:**

- [ ] **11.1** `@UseGuards(JwtAuthGuard, RolesGuard)` + `@Roles('QUAN_TRI_VIEN')` cho toàn bộ `Controllers/Admin/`.

- [ ] **11.2** `Admin/DanhMucController` (`@Controller('admin/danh-muc')`):
  - `GET` — list tree.
  - `POST` — tạo mới.
  - `PUT /:id` — cập nhật.
  - `DELETE /:id` — chỉ khi không có sản phẩm và danh mục con.
  - `PATCH /:id/trang-thai` — đổi hiển thị.
  - Validate: không cho `maDanhMucCha` là chính nó hoặc con của nó (dùng CTE recursive hoặc kiểm tra ở service).
  - Invalidate cache `danh_muc:all` sau khi mutation.

- [ ] **11.3** `Admin/SanPhamController` (`@Controller('admin/san-pham')`):
  - `GET` — list + filter (bao gồm cả `TAM_NGUNG`/`NGUNG_KINH_DOANH`).
  - `POST` — tạo mới (unique `maSanPhamHienThi`).
  - `PUT /:id` — cập nhật (giá cũ đã snapshot trong `ChiTietDonThue` — không sửa đơn cũ).
  - `PATCH /:id/trang-thai` — đổi kinh doanh.
  - `POST /:id/hinh-anh` — upload (`.jpg/.jpeg/.png/.webp`, ≤ 5MB, kiểm tra magic bytes qua `file-type`).
  - `DELETE /hinh-anh/:maHinhAnh`.
  - **KHÔNG có endpoint tăng/giảm số lượng thiết bị trực tiếp** — qua phiếu nhập (Tuần 3).
  - Invalidate cache sản phẩm sau khi mutation.

- [ ] **11.4** Validation trong `AdminSanPhamService` (Người 4 viết, tách khỏi `SanPhamService` của Người 2):
  - **`giaThueMoiNgay >= 0`**, **`mucCocMoiThietBi >= 0`**, **`giaTriBoiThuong >= 0`** — nhập âm → HTTP 400.
  - `sucChua >= 0`, `maSanPhamHienThi` không trùng.
  - Upload: Multer memory storage → validate magic bytes → lưu `apps/api/uploads/san-pham/`.

- [ ] **11.5** Commit: `feat: UC21 - admin category and product CRUD API`

---

## Task 12: Seed data + integration test + review

**Các bước:**

- [ ] **12.1** `prisma/seed.ts` (chạy qua `prisma db seed`):
  - **Nhất quán và chạy lại không sinh trùng** (dùng `upsert` với unique key).
  - 1 tài khoản `QUAN_TRI_VIEN` (email `admin@geargo.local`, Argon2 hash sẵn).
  - 1 `NhanVien` — cần cho `PhieuNhapHang.maNguoiLap`.
  - 1 `KhachHang` test.
  - 3 danh mục: "Lều trại", "Bàn ghế", "Phụ kiện".
  - 5 sản phẩm, ảnh placeholder.
  - 1 `NhaCungCap`.
  - 1 `PhieuNhapHang` (`trangThai = 'DA_NHAP_KHO'`) + 5 `ChiTietPhieuNhap` → 10 `ThietBi` (`SAN_SANG`).
  - 1 `ChinhSach` phiên bản 1 (đang có hiệu lực: `thoiDiemApDung <= Now`).
  - 1 `KhuyenMai` mã `TEST10` giảm 10%, `TAT_CA`, tối thiểu 500.000₫, hạn +30 ngày.

- [ ] **12.2** Gọi seed qua `pnpm db:seed`. Docker Compose target `api-init` chạy `prisma migrate deploy && prisma db seed`.

- [ ] **12.3** Integration test `don-thue.e2e-spec.ts` (Jest + Supertest, happy path **9 bước**):
  1. Đăng ký + đăng nhập → cookie `access_token`.
  2. `GET /api/v1/san-pham` → có sản phẩm.
  3. `POST /api/v1/gio-thue/them` × 2.
  4. `PUT /api/v1/gio-thue/thoi-gian` (Now+1h, Now+25h).
  5. `POST /api/v1/gio-thue/ma-giam-gia` với `TEST10`.
  6. `POST /api/v1/don-thue` → `CHO_THANH_TOAN`, có `GiuCho`.
  7. `POST /api/v1/thanh-toan/:id/tao-url` → URL.
  8. `GET` callback → `DA_XAC_NHAN`, 2 `ChiTietThanhToan`.
  9. Callback lần 2 cùng `maYeuCau` → không ghi trùng.

- [ ] **12.4** Race condition test: 2 concurrent user cùng đặt thiết bị cuối → chỉ 1 thành công (`Promise.all` gọi endpoint tạo đơn).

- [ ] **12.5** Hết hạn test: **inject `Clock` fake** (interface `IClock` do Người 1 định nghĩa, provide qua DI) hoặc gọi trực tiếp BullMQ processor với `maDonThue` → kiểm tra `HET_HAN`, `GiuCho` giải phóng; không dùng `setTimeout` 2 giây để chờ job.

- [ ] **12.6** Bổ sung test:
  - **Giá thay đổi:** đổi giá sản phẩm giữa lúc xem giỏ và tạo đơn → `BaogiaThayDoiException`.
  - **Hết lượt mã:** nhiều khách dùng cùng mã, vượt `gioiHanTongLuot` → từ chối.
  - **Hủy giải phóng giữ chỗ:** hủy đơn → `GiuCho` và `LuotSuDungKhuyenMai` được giải phóng, khả dụng tăng lại.
  - **Callback lặp (gửi trùng):** không ghi nhận thu hai lần.
  - **Xử lý đồng thời:** thanh toán + job hết hạn chạy cùng lúc → không ghi đè trạng thái nhau.
  - Thanh toán xong không làm khả dụng tăng trở lại.
  - Hai đơn cũ không trùng nhau không bị cộng dồn sai.
  - Khách không được xem hoặc sửa đơn, giỏ của người khác.

- [ ] **12.7** Security check:
  - Không raw SQL với user input (chỉ `$queryRaw` khi truyền `Prisma.sql` parameterized).
  - `@UseGuards(JwtAuthGuard)` đủ ở mọi endpoint cần đăng nhập.
  - Upload: `.jpg/.jpeg/.png/.webp`, ≤ 5MB, kiểm tra magic bytes (`file-type`).
  - JWT secret không hardcode, `.env` trong `.gitignore`.
  - Rate limit `@nestjs/throttler` áp cho `/api/v1/auth/*`.
  - HTTP-only cookie + `sameSite: 'lax'` + `secure: true` khi production.

- [ ] **12.8** Commit: `feat: week-2 complete - seed data, integration tests, security review`

---

## Checklist nghiệm thu tuần 2

- [ ] `docker compose up` — tất cả services start, không lỗi.
- [ ] `npx prisma migrate deploy` — DB schema đúng, đủ migrations cho Task 0-8.
- [ ] `pnpm build` (turbo) — 0 errors ở cả `apps/api` và `apps/web`.
- [ ] Đăng ký + đăng nhập trả JWT hợp lệ trong HTTP-only cookie.
- [ ] Sai mật khẩu 5 lần → khóa tạm 15 phút trong Redis; `TaiKhoan.trangThai` vẫn dành cho khóa bởi quản trị viên.
- [ ] Tài khoản bị khóa không tạo được giao dịch mới, kể cả JWT còn hợp lệ.
- [ ] `GET /api/v1/san-pham?gioNhan=&gioTra=` trả số khả dụng đúng; không chọn ngày chỉ hiện giá tham khảo.
- [ ] Thêm giỏ, đổi số lượng, đổi thời gian → báo giá cập nhật đúng.
- [ ] Thêm sản phẩm đã có trong giỏ → cộng dồn số lượng.
- [ ] Áp mã `TEST10` → giảm 10% khi tiền thuê ≥ 500.000₫.
- [ ] Tạo đơn → `CHO_THANH_TOAN`, có `GiuCho` cho mọi `ChiTietDonThue`, hạn 15 phút, BullMQ job được enqueue.
- [ ] Đơn hết hạn → BullMQ processor tự chuyển `HET_HAN`, `GiuCho` giải phóng.
- [ ] Hủy đơn chưa thanh toán → ghi người hủy, lý do; giải phóng giữ chỗ và lượt mã.
- [ ] Thanh toán mock → `DA_XAC_NHAN`, 2 `ChiTietThanhToan`.
- [ ] Callback trùng `maYeuCau` → không ghi trùng (unique constraint bảo vệ).
- [ ] Thanh toán xong không làm khả dụng tăng trở lại.
- [ ] Hai đơn cũ không trùng nhau không bị cộng dồn sai khi tính khả dụng.
- [ ] Khách không được xem hoặc sửa đơn, giỏ của người khác.
- [ ] `/api/v1/admin/*` chỉ nhận `QUAN_TRI_VIEN`, token `KHACH_HANG` → 403.
- [ ] Integration test happy path **9 bước** pass (`pnpm test:e2e`).
- [ ] Mỗi chuyển trạng thái đơn có bản ghi trong `LichSuTrangThaiDon`.
- [ ] Redis cache hit khi load lại danh mục hoặc danh sách sản phẩm.

---

## Mốc bàn giao service dùng chung

| Mốc | Ai | Nội dung |
|---|---|---|
| **Cuối Day 1** | Người 1 | Bàn giao `Result<T>`, cây exception nghiệp vụ, `HttpExceptionFilter`, cấu trúc lỗi JSON. |
| **Cuối Day 1** | Người 2, 3, 4, 5 | Chốt interface + DTO (`KhuyenMaiService`, `BaoGiaService`, `KhaDungService`, `DonThueService`, DTO báo giá truyền từ giỏ sang tạo đơn). Contract-first. |
| **Cuối Day 2** | **Người 5** | **Bàn giao model + Prisma constraint cho `LuotSuDungKhuyenMai` và `DonThue`** — chỉ phần model cần cho `KhuyenMaiService`. Người 1 tích hợp migration ngay. |
| **Cuối Day 2** | Người 3 | `KhuyenMaiService` có implement để Người 4 (báo giá) và Người 5 (tạo đơn) tích hợp. |
| **Cuối Day 2** | Người 4 | `BaoGiaService` có implement để Người 5 gọi khi tạo đơn. |
| **Cuối Day 3** | Người 2, 5 | `KhaDungService` (Người 2, phụ thuộc `GiuCho`) + service tạo đơn tối thiểu (Người 5). |
| **Cuối Day 4** | Người 5 | Luồng đặt đơn → thanh toán chạy được end-to-end (happy path 9 bước). |
| **Day 5** | Cả nhóm | Sửa lỗi tích hợp, nghiệm thu 9 bước, race condition test. |

> **Phá vòng phụ thuộc:** `KhuyenMaiService` (Người 3, Day 2) cần `LuotSuDungKhuyenMai` và `DonThue` (Task 7, Người 5). Giải quyết: **Người 5 bàn giao model + Prisma constraint của 2 entity này cuối Day 2**, phần logic tạo đơn hoàn thiện đến Day 3.

> ⚠ **Người 5 với 3.5 ngày là khá căng.** Người 1 hoặc Người 4 nên hỗ trợ Người 5 phần viết controller + DTO đơn thuê từ Day 3 để giảm tải.

---

## Ghi chú kỹ thuật

| Điểm | Lưu ý |
|---|---|
| Naming | TypeScript camelCase, Prisma map snake_case qua `@map` từng field và `@@map` từng model |
| PK BigInt | Dùng `bigint` trong TS, `BigInt` trong Prisma; serialize sang string ở boundary API (Zod transformer) |
| Số tiền | `Decimal(18,2)` — dùng `Prisma.Decimal` / `decimal.js`, KHÔNG dùng `number` |
| JSON columns | `Prisma.Json`; validate qua Zod schema trong `packages/types/` |
| Enum | Native Prisma enum, TS enum tự sinh; chuỗi UPPER_SNAKE_CASE trong DB |
| Snapshot | `ChiTietDonThue` copy giá + phụ kiện tại thời điểm đặt — không JOIN back |
| Idempotency | Unique `ThanhToan.maYeuCau` |
| Transaction | `taoDon` bọc `prisma.$transaction(..., { isolationLevel: 'Serializable' })` |
| Migrations | Prisma Migrate quản lý; chỉ Người 1 tạo (`prisma migrate dev`) |
| Không xóa data lịch sử | Đơn hoàn tất, thanh toán thành công — chỉ đổi trạng thái |
| Nhập kho tuần 2 | Model đầy đủ cột ERD + seed, KHÔNG API/UI — Tuần 3 |
| Xác thực | JWT trong HTTP-only cookie; Next.js đọc từ cookie server-side, không dùng localStorage |
| Database | PostgreSQL 16 Docker, container name `geargo-postgres`, port 5432 |
| Cache | Redis 7 Docker, container name `geargo-redis`, port 6379 |
| Queue | BullMQ trên Redis, queue `don-thue-queue` cho job hết hạn 15 phút |
| Giờ | Lưu/truyền UTC (ISO 8601), hiển thị giờ Việt Nam (UTC+7) qua `date-fns-tz` |
| Chính sách | Chọn bản đang có hiệu lực, không đơn thuần lấy phiên bản mới nhất |
| Hash mật khẩu | Argon2 (`argon2` npm package), không dùng bcrypt hay SHA256 |
| Rate limiting | `@nestjs/throttler` áp cho auth endpoints |

---

## Automation & AI (bất biến, tuần 3+)

Các thành phần AI và automation trong đặc tả **được giữ nguyên logic** so với phiên bản .NET; chỉ triển khai kỹ thuật khác:

| Chức năng | Công nghệ | Tuần triển khai |
|---|---|---|
| Tư vấn AI (UC09) | Claude API (Anthropic SDK, streaming), context = danh sách sản phẩm + khả dụng; cache Redis TTL 5 phút theo hash params | Tuần 3+ |
| Job hết hạn đơn | BullMQ delayed job 15 phút | **Tuần 2 (Task 10.8)** |
| Nhắc trả đồ | BullMQ cron job (kiểm tra `gioTraDuKien` gần đến) | Tuần 4 |
| Gắn nhãn quá hạn | BullMQ cron job mỗi 5 phút | Tuần 4 |
| Thông báo real-time | SSE hoặc WebSocket, publish qua Redis Pub/Sub | Tuần 5 |
| Báo cáo định kỳ | BullMQ cron job hằng ngày | Tuần 6 |

---

## Roadmap sau Tuần 2

**Tuần 3:** UC02 (hồ sơ/theo dõi đơn), UC17 (nhà cung cấp), UC18–UC20 (nhập hàng), UC10 (chuẩn bị/phân công), UC11 (bàn giao). API nhập kho cho model đã tạo. Bắt đầu UC09 (Claude API).
**Tuần 4:** UC12 (nhận trả/kiểm tra), UC13 (quá hạn), UC14–UC15 (phụ phí/đối soát), UC07 (hủy sau thanh toán/hoàn tiền). Các ngoại lệ UC06 còn lại.
**Tuần 5:** UC16 (bảo trì/vòng đời thiết bị), UC22 (điều chỉnh kho), UC23 (quản lý tài khoản/nhân viên), UC08 (đánh giá), thông báo real-time.
**Tuần 6:** UC24 (khuyến mãi đầy đủ), UC25–UC26 (báo cáo, cấu hình, lịch sử/nhật ký), hoàn thiện, deploy.
