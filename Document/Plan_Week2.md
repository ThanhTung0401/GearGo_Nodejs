# GearGo — Kế hoạch Tuần 2: Khởi tạo dự án, Xác thực, Danh mục, Sản phẩm, Giỏ thuê & Đặt đơn

> **Tuần 1** đã chốt đặc tả, chưa đụng code. Tuần 2 này bắt đầu từ zero: khởi tạo dự án đến khi khách có thể tìm sản phẩm, đặt thuê và thanh toán; admin quản lý danh mục và sản phẩm.

**Mục tiêu tuần 2:** Dựng nền tảng dự án + xác thực (UC01) + khách duyệt danh mục / tìm sản phẩm / kiểm tra khả dụng / quản lý giỏ thuê / tạo đơn giữ chỗ / thanh toán xác nhận đơn (UC03–UC06) + admin CRUD danh mục và sản phẩm (UC21).

**Kiến trúc:** MVC phân tầng — `Controller → Service → Repository (Prisma)`. Mỗi tính năng là một NestJS Module độc lập. Controller chỉ nhận/trả HTTP, không chứa logic nghiệp vụ. Service chứa toàn bộ logic. Prisma là ORM duy nhất — không viết raw SQL trừ query phức tạp về khả dụng.

**Tech Stack:** NestJS 10 · Prisma 5 · PostgreSQL 16 · Next.js 14 (App Router) · Redis 7 (cache + Bull queue) · Docker & Docker Compose · JWT (HTTP-only cookie) · Argon2 · BullMQ · Zod (validation) · Tailwind CSS · shadcn/ui

---

## Phạm vi tuần 2

| Nhóm | Use Case | Tên nghiệp vụ |
|---|---|---|
| Nền tảng | — | Khởi tạo monorepo, Docker, Prisma, JWT Auth, layout chung |
| Xác thực | UC01 | Đăng ký, đăng nhập, phục hồi mật khẩu |
| Khách | UC03 | Tìm kiếm, xem sản phẩm và kiểm tra khả dụng |
| Khách | UC04 | Quản lý giỏ thuê và xem báo giá |
| Khách | UC05 | Tạo đơn và giữ chỗ |
| Khách | UC06 | Thanh toán và xác nhận đơn |
| Admin | UC21 | Quản lý danh mục và sản phẩm (CRUD) |

---

## Cấu trúc file sẽ tạo / chỉnh sửa

```
GearGo/
├── docker-compose.yml                          (mới — Task 0)
├── docker-compose.dev.yml                      (mới — Task 0)
├── .env.example                                (mới — Task 0)
├── turbo.json                                  (mới — Task 0)
├── package.json                                (root workspace)
│
├── apps/
│   ├── api/                                    (NestJS backend)
│   │   ├── src/
│   │   │   ├── main.ts                         (mới — Task 0)
│   │   │   ├── app.module.ts                   (mới — Task 0)
│   │   │   ├── prisma/
│   │   │   │   ├── prisma.module.ts            (mới — Task 0)
│   │   │   │   └── prisma.service.ts           (mới — Task 0)
│   │   │   ├── common/
│   │   │   │   ├── cache/
│   │   │   │   │   └── cache.module.ts         (mới — Task 0)
│   │   │   │   ├── guards/
│   │   │   │   │   ├── jwt-auth.guard.ts       (mới — Task 0.5)
│   │   │   │   │   └── roles.guard.ts          (mới — Task 0.5)
│   │   │   │   ├── decorators/
│   │   │   │   │   ├── current-user.decorator.ts (mới — Task 0.5)
│   │   │   │   │   └── roles.decorator.ts      (mới — Task 0.5)
│   │   │   │   └── filters/
│   │   │   │       └── http-exception.filter.ts (mới — Task 0)
│   │   │   ├── auth/
│   │   │   │   ├── auth.module.ts              (mới — Task 0.5)
│   │   │   │   ├── auth.controller.ts          (mới — Task 0.5)
│   │   │   │   ├── auth.service.ts             (mới — Task 0.5)
│   │   │   │   ├── strategies/
│   │   │   │   │   └── jwt.strategy.ts         (mới — Task 0.5)
│   │   │   │   └── dto/
│   │   │   │       ├── dang-ky.dto.ts          (mới — Task 0.5)
│   │   │   │       ├── dang-nhap.dto.ts        (mới — Task 0.5)
│   │   │   │       └── quen-mat-khau.dto.ts    (mới — Task 0.5)
│   │   │   ├── danh-muc/
│   │   │   │   ├── danh-muc.module.ts          (mới — Task 4)
│   │   │   │   ├── danh-muc.controller.ts      (mới — Task 4)
│   │   │   │   ├── danh-muc.service.ts         (mới — Task 4)
│   │   │   │   └── dto/
│   │   │   │       └── danh-muc.dto.ts         (mới — Task 4)
│   │   │   ├── san-pham/
│   │   │   │   ├── san-pham.module.ts          (mới — Task 4)
│   │   │   │   ├── san-pham.controller.ts      (mới — Task 4 & 5)
│   │   │   │   ├── san-pham.service.ts         (mới — Task 4)
│   │   │   │   └── dto/
│   │   │   │       ├── tim-kiem-san-pham.dto.ts (mới — Task 4)
│   │   │   │       └── san-pham.dto.ts         (mới — Task 4)
│   │   │   ├── kha-dung/
│   │   │   │   ├── kha-dung.module.ts          (mới — Task 3)
│   │   │   │   └── kha-dung.service.ts         (mới — Task 3)
│   │   │   ├── gio-thue/
│   │   │   │   ├── gio-thue.module.ts          (mới — Task 6)
│   │   │   │   ├── gio-thue.controller.ts      (mới — Task 6)
│   │   │   │   ├── gio-thue.service.ts         (mới — Task 6)
│   │   │   │   └── dto/
│   │   │   │       └── gio-thue.dto.ts         (mới — Task 6)
│   │   │   ├── don-thue/
│   │   │   │   ├── don-thue.module.ts          (mới — Task 7)
│   │   │   │   ├── don-thue.controller.ts      (mới — Task 7)
│   │   │   │   ├── don-thue.service.ts         (mới — Task 7)
│   │   │   │   └── dto/
│   │   │   │       └── don-thue.dto.ts         (mới — Task 7)
│   │   │   ├── thanh-toan/
│   │   │   │   ├── thanh-toan.module.ts        (mới — Task 8)
│   │   │   │   ├── thanh-toan.controller.ts    (mới — Task 8)
│   │   │   │   ├── thanh-toan.service.ts       (mới — Task 8)
│   │   │   │   └── dto/
│   │   │   │       └── thanh-toan.dto.ts       (mới — Task 8)
│   │   │   └── jobs/
│   │   │       ├── jobs.module.ts              (mới — Task 7)
│   │   │       └── don-thue-expiry.processor.ts (mới — Task 7)
│   │   ├── test/
│   │   │   ├── kha-dung.service.spec.ts        (mới — Task 3)
│   │   │   └── don-thue.service.spec.ts        (mới — Task 10)
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── nest-cli.json
│   │
│   └── web/                                    (Next.js 14 frontend)
│       ├── src/
│       │   └── app/
│       │       ├── layout.tsx                  (mới — Task 0)
│       │       ├── page.tsx                    (mới — Task 0)
│       │       ├── (auth)/
│       │       │   ├── dang-ky/
│       │       │   │   └── page.tsx            (mới — Task 0.5)
│       │       │   ├── dang-nhap/
│       │       │   │   └── page.tsx            (mới — Task 0.5)
│       │       │   └── quen-mat-khau/
│       │       │       └── page.tsx            (mới — Task 0.5)
│       │       ├── san-pham/
│       │       │   ├── page.tsx                (mới — Task 5)
│       │       │   └── [id]/
│       │       │       └── page.tsx            (mới — Task 5)
│       │       ├── gio-thue/
│       │       │   └── page.tsx                (mới — Task 6)
│       │       ├── don-thue/
│       │       │   ├── xac-nhan/
│       │       │   │   └── page.tsx            (mới — Task 7)
│       │       │   └── [id]/
│       │       │       └── page.tsx            (mới — Task 7)
│       │       ├── thanh-toan/
│       │       │   ├── [donId]/
│       │       │   │   └── page.tsx            (mới — Task 8)
│       │       │   └── ket-qua/
│       │       │       └── page.tsx            (mới — Task 8)
│       │       └── admin/
│       │           ├── layout.tsx              (mới — Task 9)
│       │           ├── danh-muc/
│       │           │   ├── page.tsx            (mới — Task 9)
│       │           │   ├── tao-moi/
│       │           │   │   └── page.tsx        (mới — Task 9)
│       │           │   └── [id]/chinh-sua/
│       │           │       └── page.tsx        (mới — Task 9)
│       │           └── san-pham/
│       │               ├── page.tsx            (mới — Task 9)
│       │               ├── tao-moi/
│       │               │   └── page.tsx        (mới — Task 9)
│       │               └── [id]/chinh-sua/
│       │                   └── page.tsx        (mới — Task 9)
│       ├── src/components/
│       │   ├── layout/
│       │   │   ├── header.tsx                  (mới — Task 0)
│       │   │   └── footer.tsx                  (mới — Task 0)
│       │   ├── san-pham/
│       │   │   ├── san-pham-card.tsx           (mới — Task 5)
│       │   │   ├── san-pham-filter.tsx         (mới — Task 5)
│       │   │   └── kha-dung-badge.tsx          (mới — Task 5)
│       │   └── gio-thue/
│       │       ├── gio-thue-table.tsx          (mới — Task 6)
│       │       └── bao-gia-panel.tsx           (mới — Task 6)
│       ├── src/lib/
│       │   ├── api.ts                          (mới — Task 0)
│       │   └── auth.ts                         (mới — Task 0.5)
│       ├── package.json
│       └── next.config.ts
│
├── prisma/
│   ├── schema.prisma                           (mới — Task 0 & 1 & 2)
│   ├── seed.ts                                 (mới — Task 10)
│   └── migrations/                             (tạo qua Prisma CLI)
│
└── packages/
    └── types/                                  (shared TypeScript types)
        ├── index.ts                            (mới — Task 0)
        └── package.json
```

---

## Task 0: Khởi tạo dự án và cấu hình nền tảng

**Files sẽ tạo:**
- `docker-compose.yml`, `docker-compose.dev.yml`, `.env.example`
- `turbo.json`, `package.json` (root workspace)
- `apps/api/src/main.ts`, `app.module.ts`
- `apps/api/src/prisma/prisma.service.ts`, `prisma.module.ts`
- `apps/api/src/common/cache/cache.module.ts`
- `apps/api/src/common/filters/http-exception.filter.ts`
- `apps/web/src/app/layout.tsx`, `page.tsx`
- `apps/web/src/components/layout/header.tsx`, `footer.tsx`
- `apps/web/src/lib/api.ts`
- `prisma/schema.prisma` (entities khởi đầu: TaiKhoan, KhachHang, NhanVien)
- `packages/types/index.ts`

**Các bước:**

- [ ] **0.1** Khởi tạo monorepo Turborepo:
  ```bash
  npx create-turbo@latest geargo --package-manager pnpm
  cd geargo
  # Xóa apps mặc định, tạo apps/api (NestJS) và apps/web (Next.js)
  cd apps && npx @nestjs/cli new api --package-manager pnpm
  npx create-next-app@latest web --typescript --tailwind --app --src-dir
  ```

- [ ] **0.2** Tạo `docker-compose.yml` với các services:
  ```yaml
  services:
    postgres:
      image: postgres:16-alpine
      environment:
        POSTGRES_DB: geargo
        POSTGRES_USER: geargo
        POSTGRES_PASSWORD: ${DB_PASSWORD}
      ports: ["5432:5432"]
      volumes: [postgres_data:/var/lib/postgresql/data]

    redis:
      image: redis:7-alpine
      ports: ["6379:6379"]
      command: redis-server --appendonly yes
      volumes: [redis_data:/data]

    api:
      build: ./apps/api
      ports: ["3001:3001"]
      environment:
        DATABASE_URL: postgresql://geargo:${DB_PASSWORD}@postgres:5432/geargo
        REDIS_URL: redis://redis:6379
        JWT_SECRET: ${JWT_SECRET}
      depends_on: [postgres, redis]

    web:
      build: ./apps/web
      ports: ["3000:3000"]
      environment:
        NEXT_PUBLIC_API_URL: http://api:3001
      depends_on: [api]
  ```

- [ ] **0.3** Tạo `.env.example`:
  ```
  DB_PASSWORD=geargo_dev
  JWT_SECRET=change_me_in_production
  JWT_EXPIRES_IN=7d
  REDIS_URL=redis://localhost:6379
  DATABASE_URL=postgresql://geargo:geargo_dev@localhost:5432/geargo
  GIU_CHO_THOI_HAN_PHUT=15
  KHOA_TAI_KHOAN_SAU_SO_LAN_SAI=5
  THOI_GIAN_KHOA_PHUT=15
  CLAUDE_API_KEY=
  ```

- [ ] **0.4** Cài Prisma và khởi tạo schema:
  ```bash
  cd apps/api
  pnpm add @prisma/client
  pnpm add -D prisma
  npx prisma init --datasource-provider postgresql
  ```
  Tạo `prisma/schema.prisma` với models khởi đầu: `TaiKhoan`, `KhachHang`, `NhanVien`.
  Enum `VaiTro { KHACH_HANG NHAN_VIEN QUAN_TRI_VIEN }`, `TrangThaiTaiKhoan { HOAT_DONG BI_KHOA }`.

- [ ] **0.5** Tạo `PrismaService` (extends `PrismaClient`, implements `OnModuleInit`):
  ```typescript
  @Injectable()
  export class PrismaService extends PrismaClient implements OnModuleInit {
    async onModuleInit() { await this.$connect(); }
  }
  ```
  Đăng ký `PrismaModule` là `@Global()` để mọi module dùng mà không cần import lại.

- [ ] **0.6** Cấu hình Redis cache trong `CacheModule`:
  ```bash
  pnpm add @nestjs/cache-manager cache-manager ioredis
  ```
  `CacheModule.registerAsync` đọc `REDIS_URL` từ `ConfigModule`. TTL mặc định 5 phút. Export `CacheManager` để inject vào services.

- [ ] **0.7** `main.ts`:
  - `app.setGlobalPrefix('api/v1')`
  - `app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }))`
  - `app.useGlobalFilters(new HttpExceptionFilter())`
  - `app.enableCors({ origin: process.env.WEB_URL, credentials: true })`
  - `app.use(cookieParser())`
  - Port: `3001`

- [ ] **0.8** Chạy migration khởi đầu:
  ```bash
  docker compose up postgres redis -d
  npx prisma migrate dev --name init
  ```
  Kiểm tra bảng `tai_khoan`, `khach_hang`, `nhan_vien` trong PostgreSQL.

- [ ] **0.9** Next.js layout: `apps/web/src/app/layout.tsx` với `Header` component (nav role-aware từ JWT cookie, badge giỏ thuê, dropdown user). Tailwind CSS + shadcn/ui.

- [ ] **0.10** `packages/types/index.ts`: export shared types dùng chung giữa API và Web — `VaiTro`, `TrangThaiDonThue`, `TrangThaiThietBi`, response envelope `ApiResponse<T>`.

- [ ] **0.11** Commit: `feat: Task 0 — khởi tạo monorepo NestJS + Next.js với Docker, Prisma, Redis cache`

---

## Task 0.5: Xác thực người dùng (UC01)

**Files:**
- Tạo: `apps/api/src/auth/auth.module.ts`, `auth.controller.ts`, `auth.service.ts`
- Tạo: `apps/api/src/auth/strategies/jwt.strategy.ts`
- Tạo: `apps/api/src/auth/dto/dang-ky.dto.ts`, `dang-nhap.dto.ts`, `quen-mat-khau.dto.ts`
- Tạo: `apps/api/src/common/guards/jwt-auth.guard.ts`, `roles.guard.ts`
- Tạo: `apps/api/src/common/decorators/current-user.decorator.ts`, `roles.decorator.ts`
- Tạo: `apps/web/src/app/(auth)/dang-ky/page.tsx`, `dang-nhap/page.tsx`, `quen-mat-khau/page.tsx`
- Tạo: `apps/web/src/lib/auth.ts`

**Lưu ý kiến trúc:** Không dùng Passport Session hay thư viện identity framework. Xác thực hoàn toàn qua bảng `tai_khoan` (Prisma schema). Argon2 cho hash mật khẩu. JWT lưu trong HTTP-only cookie (không `localStorage`).

**Các bước:**

- [ ] **0.5.1** Cài dependencies:
  ```bash
  pnpm add @nestjs/jwt @nestjs/passport passport passport-jwt argon2 cookie-parser
  pnpm add -D @types/passport-jwt @types/cookie-parser
  ```

- [ ] **0.5.2** DTO với class-validator:

  `DangKyDto`: `hoTen` (required), `email` (IsEmail), `soDienThoai` (IsPhoneNumber('VN')), `matKhau` (MinLength 8), `xacNhanMatKhau` (Match decorator).

  `DangNhapDto`: `taiKhoan` (email hoặc SĐT), `matKhau`, `nhoToi` (boolean, optional).

  `QuenMatKhauDto`: bước 1 — `email`; bước 2 — `email`, `token`, `matKhauMoi`, `xacNhanMatKhauMoi`.

- [ ] **0.5.3** `JwtStrategy` (PassportStrategy):
  ```typescript
  validate(payload: JwtPayload) {
    // payload: { sub: maTaiKhoan, email, vaiTro }
    return { maTaiKhoan: payload.sub, email: payload.email, vaiTro: payload.vaiTro };
  }
  ```
  Lấy JWT từ cookie `access_token` (extractor: `fromExtractors([req => req?.cookies?.access_token])`).

- [ ] **0.5.4** `AuthService.dangKy`:
  - Kiểm tra email và SĐT chưa tồn tại trong `tai_khoan`.
  - Hash mật khẩu: `argon2.hash(dto.matKhau)`.
  - Tạo `TaiKhoan` (`vaiTro: VaiTro.KHACH_HANG`, `trangThai: TrangThaiTaiKhoan.HOAT_DONG`).
  - Tạo `KhachHang` liên kết 1-1 trong cùng Prisma transaction.
  - **Không cho phép đăng ký role NHAN_VIEN hoặc QUAN_TRI_VIEN qua endpoint công khai.**

- [ ] **0.5.5** `AuthService.dangNhap`:
  - Tìm tài khoản theo email (chứa `@`) hoặc SĐT.
  - Kiểm tra `trangThai != BI_KHOA` trước khi verify.
  - `argon2.verify(taiKhoan.matKhauBam, dto.matKhau)`.
  - Theo dõi số lần sai qua Redis (`khoa:${email}:count`, TTL = `THOI_GIAN_KHOA_PHUT * 60`). Sau `KHOA_TAI_KHOAN_SAU_SO_LAN_SAI` lần → set `trangThai = BI_KHOA` trong DB.
  - Nếu đúng: sign JWT `{ sub, email, vaiTro }`, set cookie `access_token` (`httpOnly: true, sameSite: 'lax', maxAge`).

- [ ] **0.5.6** `AuthService.taoTokenQuenMatKhau`:
  - Sinh `crypto.randomBytes(32).toString('hex')`.
  - Lưu Redis: key `reset:${email}` → token, TTL 1800 giây (30 phút).
  - Dev: ghi token ra log. Comment `// TODO: integrate Nodemailer`.

- [ ] **0.5.7** `AuthController`:
  - `POST /api/v1/auth/dang-ky` → đăng ký.
  - `POST /api/v1/auth/dang-nhap` → đăng nhập, set cookie.
  - `POST /api/v1/auth/dang-xuat` → clear cookie.
  - `POST /api/v1/auth/quen-mat-khau` → gửi token reset.
  - `POST /api/v1/auth/dat-lai-mat-khau` → đặt lại mật khẩu.
  - `GET /api/v1/auth/me` → trả thông tin user hiện tại (JwtAuthGuard).

- [ ] **0.5.8** Guards:
  - `JwtAuthGuard`: extend `AuthGuard('jwt')`. Trả `401` nếu token invalid/expired.
  - `RolesGuard`: kiểm tra `@Roles(VaiTro.QUAN_TRI_VIEN)` decorator, trả `403` nếu thiếu quyền.

- [ ] **0.5.9** Next.js pages:
  - Form đăng ký / đăng nhập dùng `react-hook-form` + `zod` validation client-side.
  - `lib/auth.ts`: helper `getSession()` đọc JWT từ cookie (server component), `signOut()` gọi API.
  - Trang đăng nhập có link "Quên mật khẩu?". Trang đăng ký báo lỗi rõ khi email / SĐT trùng.

- [ ] **0.5.10** Test thủ công:
  - Đăng ký tài khoản mới → đăng nhập thành công, cookie được set HTTP-only.
  - Đăng ký email đã tồn tại → 409 Conflict, hiển thị lỗi rõ.
  - Đăng nhập sai 5 lần → 403 tài khoản bị khóa.
  - Quên mật khẩu → lấy token từ log → đặt lại → đăng nhập được bằng mật khẩu mới.

- [ ] **0.5.11** Commit: `feat: UC01 — xác thực JWT HTTP-only cookie (đăng ký, đăng nhập, quên mật khẩu)`

---

## Task 1: Prisma Schema — Danh mục và Sản phẩm

**Files:**
- Chỉnh sửa: `prisma/schema.prisma`

**Các bước:**

- [ ] **1.1** Model `DanhMucSanPham`: `id`, `ten`, `danhMucChaId` (optional, self-relation), `moTa`, `thuTu`, `hienThi` (Boolean), `createdAt`, `updatedAt`. Relations: `danhMucCha`, `danhMucCon`, `sanPhams`.

- [ ] **1.2** Enum `TrangThaiKinhDoanh { DANG_KINH_DOANH TAM_NGUNG_NHAN_DON NGUNG_KINH_DOANH }`.

- [ ] **1.3** Model `SanPham`: `id`, `ma` (unique String), `ten`, `danhMucId`, `thuongHieu`, `moTa`, `sucChuaHoacKichThuoc`, `giaThueNgay` (Decimal), `mucCocMotThietBi` (Decimal), `giaTriBoiThuong` (Decimal), `trangThaiKinhDoanh` (enum), `createdAt`, `updatedAt`. Relations: `danhMuc`, `hinhAnhs`, `thietBis`.

- [ ] **1.4** Model `HinhAnhSanPham`: `id`, `sanPhamId`, `duongDan`, `laAnhChinh` (Boolean), `thuTu`.

- [ ] **1.5** Chạy migration:
  ```bash
  npx prisma migrate dev --name add-catalog
  npx prisma generate
  ```

- [ ] **1.6** Commit: `feat: Prisma schema — DanhMucSanPham, SanPham, HinhAnhSanPham`

---

## Task 2: Prisma Schema — Thiết bị, Giỏ thuê và Đặt đơn

**Files:**
- Chỉnh sửa: `prisma/schema.prisma`

**Các bước:**

- [ ] **2.1** Enums:
  - `TrangThaiThietBi { SAN_SANG DANG_THUE DANG_BAO_TRI THAT_LAC NGUNG_SU_DUNG }`
  - `TrangThaiDonThue { CHO_THANH_TOAN DA_XAC_NHAN DANG_CHUAN_BI SAN_SANG_NHAN DANG_THUE DA_NHAN_TRA CHO_DOI_SOAT HOAN_TAT HET_HAN KHACH_HUY CUA_HANG_HUY }`
  - `MucDichThanhToan { TIEN_THUE TIEN_COC THU_BO_SUNG CAN_HOAN_TIEN }`
  - `TrangThaiThanhToan { THANH_CONG THAT_BAI DANG_XU_LY }`

- [ ] **2.2** Model `ThietBi`: `id`, `ma` (unique), `sanPhamId`, `ngayNhap`, `giaNhap` (Decimal), `tinhTrang` (enum), `phuKienDiKem`, `soLanChoThue`, `ghiChu`.

- [ ] **2.3** Model `GioThue` + `ChiTietGioThue`:
  - `GioThue`: `id`, `khachHangId` (unique — 1 khách 1 giỏ), `gioNhan` (DateTime?), `gioTra` (DateTime?), `maGiamGia`, `updatedAt`.
  - `ChiTietGioThue`: `id`, `gioThueId`, `sanPhamId`, `soLuong`, `donGiaThamKhao` (Decimal).

- [ ] **2.4** Model `GiuCho`: `id`, `donThueId`, `sanPhamId`, `soLuong`, `thoiDiemHetHan`.

- [ ] **2.5** Model `DonThue`: `id`, `ma` (unique), `khachHangId`, `tenNguoiNhan`, `sdtNguoiNhan`, `gioNhanDuKien`, `gioTraDuKien`, `tienThue` (Decimal), `giamGia` (Decimal), `tienCoc` (Decimal), `trangThai` (enum), `hanGiuCho`, `chinhSachApDung` (Json), `maKhuyenMaiId` (optional), `createdAt`, `nguoiHuyId` (optional), `lyDoHuy`, `thoiDiemHuy`.

- [ ] **2.6** Model `ChiTietDonThue`: `id`, `donThueId`, `sanPhamId`, `soLuong`, `soNgayTinhTien`, `donGia` (Decimal), `tienThue` (Decimal), `mucCoc` (Decimal), `giaTriBoiThuong` (Decimal).

- [ ] **2.7** Model `ThanhToan`: `id`, `donThueId`, `soTien` (Decimal), `mucDich` (enum), `phuongThuc`, `maGiaoDich` (unique), `thoiDiem`, `trangThai` (enum), `ghiChu`.

- [ ] **2.8** Model `KhuyenMai` + `LuotSuDungKhuyenMai`.

- [ ] **2.9** Prisma relations và indexes:
  - Index trên `DonThue.trangThai`, `GiuCho.thoiDiemHetHan`.
  - `ThanhToan.maGiaoDich` unique (idempotency guard).
  - `DonThue` không cascade delete khi xóa `KhachHang` (`onDelete: Restrict`).

- [ ] **2.10** Chạy migration:
  ```bash
  npx prisma migrate dev --name add-order-cart-tables
  npx prisma generate
  ```

- [ ] **2.11** Commit: `feat: Prisma schema — ThietBi, GioThue, DonThue, ThanhToan, KhuyenMai`

---

## Task 3: Service tính khả dụng (KhaDungService)

**Files:**
- Tạo: `apps/api/src/kha-dung/kha-dung.module.ts`, `kha-dung.service.ts`
- Tạo: `apps/api/test/kha-dung.service.spec.ts`

**Các bước:**

- [ ] **3.1** Interface TypeScript:
  ```typescript
  layKhaDung(sanPhamId: number, gioNhan: Date, gioTra: Date): Promise<number>
  layKhaDungNhieuSanPham(ids: number[], gioNhan: Date, gioTra: Date): Promise<Map<number, number>>
  ```

- [ ] **3.2** Implement `layKhaDung` — **không cache** (real-time):
  ```typescript
  // Đếm thiết bị SAN_SANG hoặc DANG_THUE thuộc sản phẩm
  const tongThietBi = await this.prisma.thietBi.count({
    where: { sanPhamId, tinhTrang: { in: ['SAN_SANG', 'DANG_THUE'] } },
  });
  // Đếm số lượng đã bị giữ bởi đơn còn hiệu lực có lịch giao nhau
  // Lịch trùng: gioNhanDon < gioTra AND gioTraDon > gioNhan (mục 8.1 đặc tả)
  const daGiu = await this.prisma.giuCho.aggregate({
    _sum: { soLuong: true },
    where: {
      sanPhamId,
      donThue: {
        trangThai: { in: ['CHO_THANH_TOAN', 'DA_XAC_NHAN', 'DANG_CHUAN_BI', 'SAN_SANG_NHAN', 'DANG_THUE'] },
        gioNhanDuKien: { lt: gioTra },
        gioTraDuKien:  { gt: gioNhan },
      },
    },
  });
  return Math.max(0, tongThietBi - (daGiu._sum.soLuong ?? 0));
  ```

- [ ] **3.3** `layKhaDungNhieuSanPham`: gọi song song `Promise.all` cho nhiều sản phẩm.

- [ ] **3.4** Đăng ký `KhaDungModule`, export `KhaDungService`.

- [ ] **3.5** Unit test 3 trường hợp (mock PrismaService):
  - Không trùng lịch → trả đủ số thiết bị.
  - Trùng một phần → trả số còn lại.
  - Tất cả bị giữ → trả 0.

- [ ] **3.6** Commit: `feat: add KhaDungService with availability calculation logic`

---

## Task 4: Service danh mục và sản phẩm

**Files:**
- Tạo: `apps/api/src/danh-muc/danh-muc.module.ts`, `danh-muc.controller.ts`, `danh-muc.service.ts`, `dto/danh-muc.dto.ts`
- Tạo: `apps/api/src/san-pham/san-pham.module.ts`, `san-pham.controller.ts`, `san-pham.service.ts`, `dto/`

**Các bước:**

- [ ] **4.1** `DanhMucService`:
  - `layTatCa()`: trả cây danh mục. Cache Redis key `danh_muc:all`, TTL 10 phút. Invalidate khi CRUD.
  - `layTheoId(id)`, `taoMoi(dto)`, `capNhat(id, dto)`, `xoa(id)`.
  - Xóa chỉ khi không có sản phẩm liên kết. Không cho đặt danh mục cha là chính nó hoặc con của nó.

- [ ] **4.2** `TimKiemSanPhamDto` (class-validator + class-transformer):
  - `tuKhoa?`, `danhMucId?`, `thuongHieu?`, `giaThueMin?`, `giaThueMax?`
  - `gioNhan?` (ISO string), `gioTra?` (ISO string)
  - `sapXepTheo?: 'gia_asc' | 'gia_desc' | 'ten_asc' | 'danh_gia'`
  - `trang?: number` (default 1), `soMoiTrang?: number` (default 12, max 50).

- [ ] **4.3** `SanPhamService.timKiem`:
  - Build Prisma `where` clause từ filters.
  - Nếu có `gioNhan` + `gioTra`: gọi `KhaDungService.layKhaDungNhieuSanPham`, filter sản phẩm khả dụng = 0 nếu cần.
  - Chỉ trả `DANG_KINH_DOANH` cho API public.
  - Cache Redis key = hash(JSON.stringify(filters)), TTL 2 phút. Invalidate khi thay đổi sản phẩm.

- [ ] **4.4** `SanPhamService.layChiTiet(id)`: include `hinhAnhs`, `danhMuc`. Cache TTL 5 phút.

- [ ] **4.5** `SanPhamService.taoMoi(dto, files)`:
  - Validate `ma` unique.
  - Upload ảnh dùng `Multer` NestJS: lưu `apps/api/uploads/san-pham/`, chỉ `.jpg/.jpeg/.png/.webp`, tối đa 5MB, tối đa 10 ảnh.

- [ ] **4.6** `SanPhamService.doiGia(id, giaThueNgay)`: chỉ tác động báo giá mới, không sửa `ChiTietDonThue` cũ.

- [ ] **4.7** Commit: `feat: add DanhMucService và SanPhamService với Redis cache`

---

## Task 5: Controller và Pages — Tìm kiếm sản phẩm (UC03)

**Files:**
- Tạo: `apps/api/src/san-pham/san-pham.controller.ts` (public endpoints)
- Tạo: `apps/web/src/app/san-pham/page.tsx`, `[id]/page.tsx`
- Tạo: `apps/web/src/components/san-pham/san-pham-card.tsx`, `san-pham-filter.tsx`, `kha-dung-badge.tsx`

**Các bước:**

- [ ] **5.1** API endpoints (không cần auth):
  - `GET /api/v1/san-pham` — tìm kiếm với `TimKiemSanPhamDto` qua query params.
  - `GET /api/v1/san-pham/:id` — chi tiết sản phẩm (kèm `?gioNhan=&gioTra=` để hiển thị khả dụng).
  - `GET /api/v1/danh-muc` — cây danh mục cho filter panel.

- [ ] **5.2** Next.js `san-pham/page.tsx` (Server Component):
  - Nhận `searchParams` → gọi API server-side.
  - Panel lọc (danh mục tree, khoảng giá, thương hiệu), date-time picker (Client Component).
  - Grid card phân trang. Debounce 400ms trên filter text, `useRouter.push` URL.

- [ ] **5.3** `SanPhamCard`: ảnh, tên, thương hiệu, giá thuê/ngày, mức cọc, `KhaDungBadge` (xanh/đỏ/xám tùy còn/hết/chưa chọn ngày).

- [ ] **5.4** `san-pham/[id]/page.tsx`: carousel ảnh (shadcn/ui), thông số kỹ thuật, date-time picker + ô số lượng + nút **"Thêm vào giỏ"** (POST `/api/v1/gio-thue/them`, Client Component). Khả dụng realtime fetch khi ngày thay đổi.

- [ ] **5.5** Commit: `feat: UC03 — product search and availability check`

---

## Task 6: Service và Pages — Giỏ thuê (UC04)

**Files:**
- Tạo: `apps/api/src/gio-thue/gio-thue.module.ts`, `gio-thue.controller.ts`, `gio-thue.service.ts`, `dto/gio-thue.dto.ts`
- Tạo: `apps/web/src/app/gio-thue/page.tsx`
- Tạo: `apps/web/src/components/gio-thue/gio-thue-table.tsx`, `bao-gia-panel.tsx`

**Các bước:**

- [ ] **6.1** API endpoints (JwtAuthGuard — chỉ KHACH_HANG):
  - `GET /api/v1/gio-thue` — lấy giỏ hiện tại kèm báo giá.
  - `POST /api/v1/gio-thue/them` — thêm sản phẩm.
  - `PATCH /api/v1/gio-thue/cap-nhat/:chiTietId` — đổi số lượng.
  - `DELETE /api/v1/gio-thue/xoa/:chiTietId` — xóa chi tiết.
  - `PATCH /api/v1/gio-thue/thoi-gian` — cập nhật giờ nhận/trả.
  - `POST /api/v1/gio-thue/ma-giam-gia` — áp mã giảm giá.

- [ ] **6.2** `GioThueService.layGioThue(khachHangId)`:
  - Lấy giỏ + chi tiết + sản phẩm từ DB.
  - Tính báo giá: `soNgay = Math.ceil((gioTra.getTime() - gioNhan.getTime()) / 86_400_000)`, tối thiểu 1 (mục 8.2 đặc tả).
  - Tính `tienThue`, `tienCoc`, `tienGiam`, `tongThanhToan`.
  - Gọi `KhaDungService` kiểm tra từng sản phẩm còn đủ số lượng không.
  - **Không cache** — dữ liệu phải tươi mỗi lần.

- [ ] **6.3** `GioThueService.themSanPham`: upsert `ChiTietGioThue` (tăng số lượng nếu đã có). Kiểm tra sản phẩm `DANG_KINH_DOANH`.

- [ ] **6.4** `GioThueService.apMaGiamGia`:
  - Kiểm tra `KhuyenMai`: còn hạn, chưa hết lượt, giá trị đơn đủ điều kiện.
  - Cache tạm Redis `giam_gia:${khachHangId}:${ma}`, TTL 20 phút (giải phóng nếu không tạo đơn).

- [ ] **6.5** Next.js `gio-thue/page.tsx` (Client Component):
  - Date-time picker giờ nhận/trả, bảng chi tiết (sửa số lượng inline, xóa dòng), panel báo giá bên phải.
  - Input mã giảm giá với nút "Áp dụng".
  - Nút **"Đặt thuê"** → navigate sang `/don-thue/xac-nhan`.
  - Cập nhật badge giỏ thuê trên Header (React Context hoặc Zustand).

- [ ] **6.6** Commit: `feat: UC04 — cart management with real-time quote calculation`

---

## Task 7: Tạo đơn và giữ chỗ (UC05)

**Files:**
- Tạo: `apps/api/src/don-thue/don-thue.module.ts`, `don-thue.controller.ts`, `don-thue.service.ts`, `dto/don-thue.dto.ts`
- Tạo: `apps/api/src/jobs/jobs.module.ts`, `don-thue-expiry.processor.ts`
- Tạo: `apps/web/src/app/don-thue/xac-nhan/page.tsx`, `[id]/page.tsx`

**Các bước:**

- [ ] **7.1** Cài BullMQ:
  ```bash
  pnpm add @nestjs/bullmq bullmq
  ```
  `JobsModule` đăng ký queue `don-thue-queue` dùng Redis connection từ `ConfigModule`.

- [ ] **7.2** `DonThueService.taoDonThue(khachHangId, dto)` — **nghiệp vụ cốt lõi**:
  Chạy trong `prisma.$transaction(..., { isolationLevel: 'Serializable' })`:
  1. Kiểm tra lại khả dụng toàn bộ giỏ (`KhaDungService`).
  2. Nếu báo giá thay đổi và `dto.xacNhanGiaMoi !== true` → throw `BadRequestException('BAO_GIA_THAY_DOI')`.
  3. Tạo `DonThue` (trạng thái `CHO_THANH_TOAN`), snapshot giá vào `ChiTietDonThue`, tạo `GiuCho` từng dòng, set `hanGiuCho = new Date(Date.now() + 15 * 60 * 1000)`.
  4. Xóa `GioThue` + `ChiTietGioThue` của khách.
  5. Nếu thiếu hàng bất kỳ dòng nào → rollback toàn bộ.
  6. Sau commit: enqueue BullMQ job `expire-don-thue` delay 15 phút.

- [ ] **7.3** `DonThueExpiryProcessor`:
  - Nhận job `expire-don-thue` với `{ donThueId }`.
  - Kiểm tra đơn vẫn `CHO_THANH_TOAN` → chuyển `HET_HAN`, xóa `GiuCho`.
  - Nếu đã `DA_XAC_NHAN` (thanh toán xong) → bỏ qua (idempotent).

- [ ] **7.4** API endpoints:
  - `GET /api/v1/don-thue/xac-nhan` — preview giỏ trước khi tạo đơn (JwtAuthGuard).
  - `POST /api/v1/don-thue` — tạo đơn (JwtAuthGuard, KHACH_HANG).
  - `GET /api/v1/don-thue/:id` — chi tiết đơn của khách.

- [ ] **7.5** Next.js pages:
  - `xac-nhan/page.tsx`: tóm tắt bảng tổng đơn, cảnh báo nếu báo giá thay đổi (checkbox xác nhận bắt buộc), chính sách hủy, nút **"Xác nhận đặt thuê"**.
  - `[id]/page.tsx`: badge trạng thái, countdown đến `hanGiuCho` (Client Component), nút **"Thanh toán"** / **"Hủy đơn"**.

- [ ] **7.6** Commit: `feat: UC05 — order creation with 15-minute BullMQ reservation hold`

---

## Task 8: Thanh toán và xác nhận đơn (UC06)

**Files:**
- Tạo: `apps/api/src/thanh-toan/thanh-toan.module.ts`, `thanh-toan.controller.ts`, `thanh-toan.service.ts`
- Tạo: `apps/web/src/app/thanh-toan/[donId]/page.tsx`, `ket-qua/page.tsx`

**Lưu ý:** Implement mock gateway trước. Comment `// TODO: Replace with VNPay SDK`.

**Các bước:**

- [ ] **8.1** Mock gateway:
  `taoUrlThanhToan(donId)` → trả URL `/thanh-toan/ket-qua?donId={id}&ketQua=success&maGD={uuid}`.

- [ ] **8.2** `ThanhToanService.xuLyKetQua(dto)` — **idempotency bắt buộc**:
  ```typescript
  // Kiểm tra maGiaoDich đã tồn tại chưa
  const existing = await prisma.thanhToan.findUnique({ where: { maGiaoDich: dto.maGiaoDich } });
  if (existing) return existing; // bỏ qua callback trùng

  await prisma.$transaction(async (tx) => {
    // Tạo 2 bản ghi ThanhToan: tienThue + tienCoc
    // Chuyển đơn → DA_XAC_NHAN
    // Xóa GiuCho
    // Ghi log thông báo xác nhận đơn (queue tuần sau)
  });
  ```
  Tiền đến sau khi đơn `HET_HAN` → tạo `ThanhToan` trạng thái `CAN_HOAN_TIEN`, không khôi phục đơn.

- [ ] **8.3** API endpoints:
  - `GET /api/v1/thanh-toan/:donId` — tạo URL thanh toán (JwtAuthGuard, chỉ chủ đơn).
  - `GET /api/v1/thanh-toan/ket-qua` — callback từ gateway.

- [ ] **8.4** Next.js pages:
  - `[donId]/page.tsx`: tóm tắt đơn + số tiền cần thanh toán + nút **"Thanh toán qua VNPay"**.
  - `ket-qua/page.tsx`: polling `GET /api/v1/don-thue/:id` tối đa 3 lần mỗi 2 giây → hiển thị kết quả thành công/thất bại.

- [ ] **8.5** Commit: `feat: UC06 — payment processing with idempotency guard`

---

## Task 9: Admin — Quản lý danh mục và sản phẩm (UC21)

**Files:**
- Tạo: admin endpoints trong `danh-muc.controller.ts`, `san-pham.controller.ts` (RolesGuard QUAN_TRI_VIEN)
- Tạo: `apps/web/src/app/admin/layout.tsx`
- Tạo: `apps/web/src/app/admin/danh-muc/` (page, tao-moi, [id]/chinh-sua)
- Tạo: `apps/web/src/app/admin/san-pham/` (page, tao-moi, [id]/chinh-sua)

**Các bước:**

- [ ] **9.1** API admin endpoints (JwtAuthGuard + `@Roles(VaiTro.QUAN_TRI_VIEN)`):
  - `POST /api/v1/admin/danh-muc` — tạo danh mục.
  - `PATCH /api/v1/admin/danh-muc/:id` — cập nhật.
  - `DELETE /api/v1/admin/danh-muc/:id` — xóa nếu không có sản phẩm.
  - `POST /api/v1/admin/san-pham` — tạo sản phẩm (upload ảnh `multipart/form-data`).
  - `PATCH /api/v1/admin/san-pham/:id` — cập nhật thông tin / giá.
  - `PATCH /api/v1/admin/san-pham/:id/trang-thai` — đổi trạng thái kinh doanh.
  - **Không có endpoint nhập số lượng kho trực tiếp** (qua phiếu nhập — tuần 3).

- [ ] **9.2** Next.js `admin/layout.tsx`:
  - Middleware kiểm tra cookie JWT, decode role, redirect về `/dang-nhap` nếu không phải QUAN_TRI_VIEN.
  - Sidebar menu admin.

- [ ] **9.3** Admin pages:
  - Danh mục: bảng phân trang (shadcn/ui DataTable), modal tạo/sửa, nút ẩn/hiện.
  - Sản phẩm: bảng phân trang, form tạo/sửa với preview ảnh upload, validation Zod client-side.
  - Đổi giá sản phẩm → toast cảnh báo "Đơn cũ không bị ảnh hưởng".

- [ ] **9.4** Invalidate Redis cache khi admin CRUD danh mục hoặc sản phẩm.

- [ ] **9.5** Commit: `feat: UC21 — admin category and product management`

---

## Task 10: Seed data, kiểm thử tích hợp và review

**Files:**
- Tạo: `prisma/seed.ts`
- Tạo: `apps/api/test/don-thue.service.spec.ts`

**Các bước:**

- [ ] **10.1** Seed data (`prisma/seed.ts`):
  - 3 danh mục: Lều & Nhà bạt, Nội thất dã ngoại, Ánh sáng & Nấu ăn.
  - 5 sản phẩm (mỗi danh mục ít nhất 1), 10 thiết bị (`SAN_SANG`).
  - 1 khuyến mãi `TEST10` (giảm 10%, hết hạn 30 ngày).
  - 1 tài khoản `QUAN_TRI_VIEN` (admin@geargo.vn / Admin@123456).
  - 1 tài khoản `KHACH_HANG` test (test@geargo.vn / Test@123456).
  ```bash
  npx prisma db seed
  ```

- [ ] **10.2** Integration test happy path (Jest + Supertest):
  Tìm sản phẩm → thêm giỏ → tạo đơn → mock payment callback → đơn chuyển `DA_XAC_NHAN`.

- [ ] **10.3** Race condition test:
  - 2 requests tạo đơn cùng lúc cho sản phẩm cuối cùng.
  - Xác nhận `Serializable` transaction chỉ cho 1 request thành công.

- [ ] **10.4** Security checklist:
  - Không raw SQL nhận user input (Prisma parameterized queries).
  - Upload chỉ `.jpg/.jpeg/.png/.webp`, tối đa 5MB, validate MIME type thực (magic bytes).
  - JWT không chứa thông tin nhạy cảm (chỉ `sub`, `email`, `vaiTro`).
  - Rate limiting trên `/api/v1/auth/*`: `@nestjs/throttler` max 10 requests/phút/IP.

- [ ] **10.5** Commit: `feat: week-2 complete — catalog, cart, order, payment, admin CRUD`

---

## Checklist nghiệm thu tuần 2

- [ ] `docker compose up` → tất cả services start thành công, không lỗi.
- [ ] `npx prisma migrate deploy` → DB schema đúng, không pending migration.
- [ ] Đăng ký tài khoản mới → đăng nhập được, cookie `access_token` được set HTTP-only.
- [ ] Email trùng khi đăng ký → 409 Conflict, hiển thị lỗi rõ trên UI.
- [ ] Đăng nhập sai 5 lần → 403 tài khoản bị khóa 15 phút.
- [ ] Quên mật khẩu → token từ log → đổi được mật khẩu mới, đăng nhập lại được.
- [ ] Khách vãng lai truy cập `/san-pham`, lọc danh mục, chọn ngày → thấy số lượng khả dụng chính xác.
- [ ] Redis cache hit khi load lại danh mục ngay lập tức (kiểm tra log Redis `CACHE HIT`).
- [ ] Khách đăng nhập thêm sản phẩm vào giỏ → báo giá đúng (tiền thuê + cọc, tối thiểu 1 ngày).
- [ ] Khách tạo đơn → trạng thái `CHO_THANH_TOAN`, `hanGiuCho` = 15 phút, BullMQ job được enqueue.
- [ ] Khách thanh toán → đơn chuyển `DA_XAC_NHAN`, callback trùng không tạo bản ghi thứ 2.
- [ ] Đơn hết hạn 15 phút → BullMQ processor tự chuyển `HET_HAN`, `GiuCho` bị xóa.
- [ ] Admin tạo danh mục mới → xuất hiện trên trang tìm kiếm khách, Redis cache invalidated.
- [ ] Admin tạo sản phẩm → số thiết bị = 0 (chưa nhập hàng qua phiếu nhập).
- [ ] Admin sửa giá sản phẩm → `ChiTietDonThue` cũ không thay đổi.
- [ ] Tất cả unit test + integration test pass (`pnpm test`).

---

## Ghi chú kỹ thuật

| Điểm | Lưu ý |
|---|---|
| Xác thực | JWT HTTP-only cookie. Không dùng localStorage. Không dùng Passport Session. |
| Hash mật khẩu | `argon2.hash/verify` — không dùng bcrypt hoặc SHA256 thuần. |
| Race condition đặt hàng | `prisma.$transaction(..., { isolationLevel: 'Serializable' })` trong `taoDonThue`. |
| Idempotency thanh toán | Unique constraint `ThanhToan.maGiaoDich`; bỏ qua nếu đã tồn tại. |
| Snapshot giá đơn | Copy `giaThueNgay`, `mucCocMotThietBi`, `giaTriBoiThuong` vào `ChiTietDonThue` khi tạo. |
| Giỏ không giữ hàng | Chỉ `DonThue` giữ qua `GiuCho`; giỏ chỉ là draft — không cache. |
| Admin không nhập kho trực tiếp | Số lượng tăng qua `PhieuNhapHang` — tuần 3. |
| BullMQ hết hạn đơn | Job delay 15 phút, idempotent (check trạng thái trước khi đổi). |
| Cache strategy | Danh mục: TTL 10 phút. Sản phẩm list: TTL 2 phút. Sản phẩm detail: TTL 5 phút. AI tư vấn (tuần 3+): TTL 5 phút theo hash(params). Giỏ thuê & khả dụng: **không cache**. |
| Tư vấn AI | Claude API streaming, context = danh sách sản phẩm + khả dụng. Logic và prompt giữ nguyên so với phiên bản .NET — triển khai tuần 3+. |
| Thông báo tự động | BullMQ jobs: nhắc trả đồ, quá hạn, hoàn tiền. Triển khai tuần 3+. |
| Migrations | Prisma Migrate quản lý toàn bộ schema. Không sửa tay DB. Chạy trong Docker. |
| Upload ảnh | Multer + validate MIME type thực (magic bytes). Lưu `apps/api/uploads/`. Tuần sau: chuyển MinIO/S3. |
| Rate limiting | `@nestjs/throttler` trên tất cả auth endpoints. |
| Múi giờ | Tất cả datetime lưu UTC trong DB, hiển thị UTC+7 trên UI. |
