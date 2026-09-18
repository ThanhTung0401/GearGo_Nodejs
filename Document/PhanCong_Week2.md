# GearGo — Phân công Tuần 2 (5 người)

> **Nguồn:** Dựa trên `Document/Plan_Week2.md` và ERD `Diagrams/ERD.dbml` (khi có).
> **Stack:** NestJS 10 · Prisma 5 · PostgreSQL 16 · Next.js 14 · Redis 7 · Docker Compose · JWT (HTTP-only cookie) · Argon2 · BullMQ · Zod
> **Nguyên tắc:** Sở hữu dọc (model → service → controller → route), model ít phụ thuộc trước, merge theo thứ tự phụ thuộc.
> **Chỉ Người 1 chịu trách nhiệm tạo Prisma migration**: model owner chuẩn bị model trong `schema.prisma` (branch riêng); Người 1 tích hợp và chạy `npx prisma migrate dev --name <ten>`, kiểm tra migration chạy được trên DB mới, đồng thời tích hợp cấu hình chung trong `app.module.ts`, `main.ts`, `docker-compose.yml`. Trong tuần có thể có nhiều migration nối tiếp nhau (không phải 1 migration duy nhất), nhưng **chỉ Người 1 tạo**. Các thành viên khác không tự chạy `prisma migrate dev`.

---

## Tổng quan phân công

| Người | Domain | Tasks | Ước lượng |
|---|---|---|---|
| **Người 1** | Auth + Testing + Quy ước chung + Migration coordinator | Task 1, Task 12 | ~2.5 ngày |
| **Người 2** | Danh mục + Sản phẩm + Tìm kiếm + Khả dụng | Task 2, 3, 9 | ~3 ngày |
| **Người 3** | Khuyến mãi + Kho skeleton + KhuyenMaiService | Task 4, 5 | ~2.5 ngày |
| **Người 4** | Giỏ thuê + BaoGiaService + Admin | Task 6, UC04, Task 11 | ~3 ngày |
| **Người 5** | Đơn thuê + Hủy + Thanh toán + Hết hạn + Lịch sử đơn | Task 7, 8, UC05+06 | ~3.5 ngày |

---

## Timeline tổng quan

```
          Day 1         Day 2         Day 3         Day 4         Day 5
         ─────────────────────────────────────────────────────────────────
Người 1: [─── Task 1: Auth ───────────]  [─── Task 12: Seed + Tests ───]
Người 2: [T2][── Task 3 ──]              [──── Task 9: UC03 ────────────]
Người 3: [── Task 4 ───────] [────── Task 5 ─────────]
Người 4:              [T6]  [── UC04: Cart API ───────] [── Task 11 ──]
Người 5:         [───── Task 7 ─────────] [T8] [── UC05+06 ──────────]
```

**Lưu ý timeline:**
- Task 7 bàn giao sớm (Day 3) cho Người 2 làm khả dụng (Task 9).
- Seed data làm dần khi model merge, không chờ cuối tuần.
- Day 4-5 dành thời gian tích hợp và test.

**Mốc bàn giao service dùng chung (bắt buộc):**

| Mốc | Ai | Nội dung |
|---|---|---|
| **Cuối Day 1** | Người 1 | Bàn giao `Result<T>`, cây exception nghiệp vụ, `HttpExceptionFilter`, cấu trúc lỗi JSON `{maLoi, thongDiep, chiTiet}` → cả nhóm dùng để code service không tự bịa exception. |
| **Cuối Day 1** | Người 2, 3, 4, 5 | Chốt interface + DTO (`KhuyenMaiService`, `BaoGiaService`, `KhaDungService`, `DonThueService`, DTO báo giá truyền từ giỏ sang tạo đơn). Không cần implement, chỉ contract. |
| **Cuối Day 2** | **Người 5** | **Bàn giao model + Prisma constraint cho `LuotSuDungKhuyenMai` và `DonThue`** (thuộc Task 7) — chỉ phần model cần cho `KhuyenMaiService` ghi lượt. Người 1 tích hợp migration ngay. |
| **Cuối Day 2** | Người 3 | Bàn giao `KhuyenMaiService` **có implement** (đọc/ghi lượt trên model Người 5 vừa bàn giao) để Người 4 (báo giá) và Người 5 (tạo đơn) tích hợp thật. |
| **Cuối Day 2** | Người 4 | Bàn giao `BaoGiaService` **có implement** để Người 5 gọi khi tạo đơn. |
| **Cuối Day 3** | Người 2, 5 | Bàn giao `KhaDungService` (Người 2, cần `GiuCho`) + service tạo đơn tối thiểu (Người 5) — bắt đầu tích hợp luồng đặt đơn. |
| **Cuối Day 4** | Người 5 | Luồng đặt đơn → thanh toán chạy được end-to-end (happy path 9 bước). |
| **Day 5** | Cả nhóm | Sửa lỗi tích hợp, nghiệm thu 9 bước, chạy race condition test. |

> **Phá vòng phụ thuộc:** `KhuyenMaiService` (Người 3, Day 2) cần model `LuotSuDungKhuyenMai` + `DonThue` (Task 7 của Người 5, mốc Day 3). Giải pháp: **Người 5 bàn giao 2 model này sớm — cuối Day 2**, chưa cần logic tạo đơn. Nếu không kịp Day 2 → lùi mốc `KhuyenMaiService` implement sang Day 3.

> ⚠ **Người 5 với 3.5 ngày là khá căng** do phải chờ 4 người khác bàn giao. Người 1 hoặc Người 4 nên hỗ trợ Người 5 phần viết controller + DTO đơn thuê từ Day 3 để giảm tải.

---

## Chi tiết từng người

---

### Thanh Tùng — Auth + Testing + Quy ước chung (Task 1 + Task 12)

**Có thể bắt đầu:** Day 1 (độc lập hoàn toàn)

**Trách nhiệm bổ sung:**
- **Điều phối migration:** tích hợp model từ các branch (mỗi thành viên chỉnh `schema.prisma` trong branch riêng, Người 1 merge và chạy `npx prisma migrate dev --name <ten>`); **chỉ Người 1 tạo migration** (có thể nhiều migration nối tiếp trong tuần, không phải 1 migration duy nhất); xác nhận migration chạy được trên DB mới bằng cách drop DB Docker và chạy lại `prisma migrate deploy && prisma db seed`. Tích hợp cấu hình chung trong `main.ts`, `app.module.ts` (JWT, CORS, DI, Guards, `HttpExceptionFilter`).
- **Xử lý lỗi dùng chung (bàn giao Day 1–2 để mọi người dùng):**
  - Kiểu `Result<T>` / `Result` cho tầng service (hoặc pattern `neverthrow` nếu cả nhóm thống nhất).
  - Cây exception nghiệp vụ (extend `HttpException`): `NghiepVuException` (base), `KhongDuHangException`, `BaogiaThayDoiException`, `TrangThaiKhongHopLeException`, `TaiKhoanBiKhoaException`, `KhuyenMaiKhongHopLeException`, `KhongTimThayException`, `KhongCoQuyenException`.
  - `HttpExceptionFilter` global: exception nghiệp vụ → HTTP 400/403/404/409, exception khác → 500.
  - Cấu trúc lỗi JSON thống nhất: `{ maLoi, thongDiep, chiTiet? }`.
- Kiểm tra Task 0 (đối chiếu 3 model `TaiKhoan`, `KhachHang`, `NhanVien` với ERD).
- Phối hợp quy ước chung (cấu trúc DTO, cấu trúc lỗi, naming, Zod schema chung trong `packages/types/`).
- Cập nhật seed data khi model được merge; đảm bảo seed idempotent (dùng `upsert`).
- Cấu hình `docker-compose.yml`, `.env.example`, `turbo.json`, target `api-init` chạy migration + seed lúc container start.

#### Task 1: UC01 — Xác thực (JWT HTTP-only cookie + Argon2)
> Không tạo model mới. Dùng `TaiKhoan` + `KhachHang` đã có từ Task 0.

**Files cần tạo:**
```
apps/api/src/auth/
  dto/dang-ky.dto.ts
  dto/dang-nhap.dto.ts
  dto/quen-mat-khau.dto.ts
  dto/dat-lai-mat-khau.dto.ts
  dto/auth.response.ts
  strategies/jwt.strategy.ts
  auth.service.ts
  auth.controller.ts
  auth.module.ts
apps/api/src/common/
  guards/jwt-auth.guard.ts
  guards/roles.guard.ts
  guards/account-status.guard.ts
  decorators/current-user.decorator.ts
  decorators/current-khach-hang.decorator.ts
  decorators/roles.decorator.ts
  result/result.ts
  exceptions/*.ts (cây exception nghiệp vụ)
  filters/http-exception.filter.ts
```

**Checklist:**
- [ ] 1.1 `AuthResponse`: `token`, `loaiToken = "Bearer"`, `hetHanSau`, `maTaiKhoan`, `vaiTro`, `hoTen`, `email`.
- [ ] 1.2 JWT — HMAC-SHA256, 7 ngày, claims: `{ sub: maTaiKhoan, email, vaiTro }`. Cấu hình qua `JwtModule.registerAsync` đọc `JWT_SECRET` từ `ConfigService`.
- [ ] 1.3 `AuthService`: `dangKy`, `dangNhap`, `dangXuat`, `taoTokenQuenMatKhau`, `datLaiMatKhau`, `layThongTinToi`.
- [ ] 1.4 `dangKy`: validate DTO (`class-validator` + custom `@Match('matKhau')`); Email + SoDienThoai chưa tồn tại → `argon2.hash` → `prisma.$transaction` tạo `TaiKhoan` + `KhachHang`.
- [ ] 1.5 `dangNhap`: tìm theo Email hoặc SoDienThoai; **kiểm tra `trangThai` tài khoản** (`BI_KHOA` → từ chối); `trangThai` chỉ dành cho khóa bởi quản trị viên; `argon2.verify`; đếm lỗi và thời điểm hết khóa tạm qua Redis (`khoa:${email}:count` TTL = `THOI_GIAN_KHOA_PHUT * 60`).
- [ ] 1.5a **Token khôi phục có hạn và chỉ dùng một lần**: lưu Redis `reset:${email}` TTL 1800 giây; verify xong `DEL`. Không thêm cột vào schema Prisma.
- [ ] 1.5b **Kiểm tra trạng thái tài khoản khi tạo giao dịch**: tại endpoint tạo đơn/giỏ, `AccountStatusGuard` kiểm tra `TaiKhoan.trangThai` ngay cả khi JWT còn hợp lệ.
- [ ] 1.5c **Lấy mã khách từ tài khoản đăng nhập**: từ JWT claims → truy vấn `KhachHang.maKhachHang`; **khách chỉ được thao tác giỏ và đơn của mình**. Cung cấp `@CurrentKhachHang()` decorator.
- [ ] 1.6 `AuthController` (`@Controller('auth')`):
  - `POST /api/v1/auth/dang-ky` → 201
  - `POST /api/v1/auth/dang-nhap` → 200 (set HTTP-only cookie `access_token`)
  - `POST /api/v1/auth/dang-xuat` → clear cookie
  - `POST /api/v1/auth/quen-mat-khau` → mock log token
  - `POST /api/v1/auth/dat-lai-mat-khau`
  - `GET /api/v1/auth/toi` `@UseGuards(JwtAuthGuard)`
- [ ] 1.7 Rate limiting: `@nestjs/throttler` áp cho `/api/v1/auth/*` — max 10 req/phút/IP.
- [ ] 1.8 DI: `AuthModule` import `JwtModule`, `PassportModule`. Đăng ký `JwtStrategy`, `AuthService`.
- [ ] 1.9 Test curl: đăng ký + đăng nhập → cookie hợp lệ; gọi `/toi` với cookie → 200.
- [ ] 1.10 Commit: `feat: UC01 - JWT HTTP-only cookie authentication with Argon2`

#### Task 12: Seed data + Integration test + Review
> Seed làm dần khi model được merge, không đợi cuối tuần; Người 1 chủ trì test tích hợp sau khi các luồng chính sẵn sàng.

**Checklist:**
- [ ] 12.1 `prisma/seed.ts` — **idempotent, `upsert` mọi bản ghi**:
  - 1 `QuanTriVien` (`admin@geargo.local`, Argon2 hash sẵn)
  - 1 `NhanVien`, 1 `KhachHang` test
  - 3 `DanhMuc`: "Lều trại", "Bàn ghế", "Phụ kiện"
  - 5 `SanPham` + ảnh placeholder
  - 1 `NhaCungCap` + 1 `PhieuNhapHang` (`DA_NHAP_KHO`) + 5 `ChiTietPhieuNhap` → 10 `ThietBi` (`SAN_SANG`)
  - 1 `ChinhSach` phiên bản 1 (đang có hiệu lực: `thoiDiemApDung <= Now`)
  - 1 `KhuyenMai` mã `TEST10` giảm 10%, `TAT_CA`, tối thiểu 500.000₫, hạn +30 ngày
- [ ] 12.2 Chạy seed qua `pnpm db:seed`; Docker Compose target `api-init` chạy `prisma migrate deploy && prisma db seed`.
- [ ] 12.3 Integration test `don-thue.e2e-spec.ts` (Jest + Supertest, happy path **9 bước**):
  1. Đăng ký + đăng nhập → cookie `access_token`
  2. `GET /api/v1/san-pham` → có sản phẩm
  3. `POST /api/v1/gio-thue/them` × 2
  4. `PUT /api/v1/gio-thue/thoi-gian` (Now+1h, Now+25h)
  5. `POST /api/v1/gio-thue/ma-giam-gia` với `TEST10`
  6. `POST /api/v1/don-thue` → `CHO_THANH_TOAN`, có `GiuCho`
  7. `POST /api/v1/thanh-toan/{id}/tao-url` → URL
  8. `GET` callback → `DA_XAC_NHAN`, 2 `ChiTietThanhToan`
  9. Callback lần 2 cùng `maYeuCau` → không ghi trùng
- [ ] 12.4 Race condition test: 2 concurrent user cùng đặt thiết bị cuối → chỉ 1 thành công (`Promise.all`).
- [ ] 12.5 Hết hạn test: **inject `Clock` fake** (interface `IClock` provide qua DI) hoặc gọi trực tiếp BullMQ processor → `HET_HAN` + `GiuCho` giải phóng; không chờ setTimeout.
- [ ] 12.6 Bổ sung test:
  - **Giá thay đổi**: đổi giá sản phẩm giữa lúc xem giỏ và tạo đơn → `BaogiaThayDoiException`
  - **Hết lượt mã**: nhiều khách dùng cùng mã, vượt `gioiHanTongLuot` → từ chối
  - **Hủy giải phóng giữ chỗ**: hủy đơn → `GiuCho` + `LuotSuDungKhuyenMai` giải phóng, khả dụng tăng lại
  - **Callback lặp (gửi trùng)**: không ghi nhận thu hai lần
  - **Xử lý đồng thời**: thanh toán + job hết hạn chạy cùng lúc → không ghi đè trạng thái nhau
  - Thanh toán xong không làm khả dụng tăng trở lại
  - Hai đơn cũ không trùng nhau không bị cộng dồn sai
  - Khách không xem/sửa đơn và giỏ của người khác
- [ ] 12.7 Security check: không raw SQL với user input; `@UseGuards(JwtAuthGuard)` đủ; upload magic bytes (`file-type`); JWT secret không hardcode; `.env` trong `.gitignore`; HTTP-only cookie + `sameSite: 'lax'` + `secure: true` khi production.
- [ ] 12.8 Commit: `feat: week-2 complete - seed data, integration tests, security review`

---

### Kiện Minh — Danh mục + Sản phẩm + Tìm kiếm + Khả dụng (Task 2, 3, 9)

**Có thể bắt đầu:** Day 1 (Task 2 độc lập)
**Task 3 cần:** Task 2 (SanPham FK → DanhMuc).
**Task 9 cần:** **Task 5 từ Người 3** (ThietBi) **và Task 7 từ Người 5** (DonThue, GiuCho).

**Trách nhiệm bổ sung:**
- Hoàn thiện **quy tắc tìm kiếm và khả dụng** theo đặc tả mục 8.1:
  - Chưa chọn ngày → chỉ hiện giá tham khảo, không khẳng định còn hàng.
  - Kiểm tra giờ trả sau giờ nhận; không tạo lượt thuê bắt đầu trong quá khứ.
  - Chỉ tính thiết bị đủ điều kiện từ **phiếu nhập đã xác nhận** (`DA_NHAP_KHO`).
  - Tính giữ chỗ còn hạn và đơn đã xác nhận; **không tính trùng**.
  - **Giữ chỗ hết hạn không chiếm lịch** dù job chưa chạy.
  - Công thức: **tổng thiết bị đủ điều kiện − số lượng bị chiếm đồng thời lớn nhất**.
- Cấu hình Redis cache cho danh sách sản phẩm và cây danh mục (TTL 2-10 phút), invalidate khi admin (Người 4) mutation.

#### Task 2: DanhMucSanPham (Cấp 0, self-ref)

**Files cần chỉnh:**
```
prisma/schema.prisma        ← thêm model DanhMucSanPham
```

**Checklist:**
- [ ] 2.1 Thêm model `DanhMucSanPham` với `@@map("danh_muc_san_pham")` — 6 cột: `maDanhMuc`, `maDanhMucCha` (nullable, self-relation), `tenDanhMuc`, `moTa`, `thuTuHienThi`, `trangThai`
- [ ] 2.2 Relation `DanhMucChaCon` self-ref; `onDelete: Restrict`
- [ ] 2.3 Chuẩn bị model xong → báo Người 1 tích hợp và tạo migration `add-danh-muc-san-pham` → cả nhóm `npx prisma migrate deploy`
- [ ] 2.4 Chạy `npx prisma generate` để cập nhật client type
- [ ] 2.5 Commit: `feat: add DanhMucSanPham model`

#### Task 3: SanPham + HinhAnhSanPham (Cấp 1-2)

> **Phụ thuộc: Task 2** (SanPham FK → DanhMuc).

**Files cần chỉnh:**
```
prisma/schema.prisma        ← thêm SanPham, HinhAnhSanPham, enum TrangThaiKinhDoanh
```

**Checklist:**
- [ ] 3.1 Enum Prisma `TrangThaiKinhDoanh`: `DANG_KINH_DOANH`, `TAM_NGUNG`, `NGUNG_KINH_DOANH`
- [ ] 3.2 Model `SanPham` — đủ 13 cột ERD, đặc biệt: `sucChua Int`, `kichThuoc String?` **riêng**, `thongSo Json?`
- [ ] 3.3 Model `HinhAnhSanPham` — 5 cột: `maHinhAnh`, `maSanPham`, `duongDan`, `laAnhChinh`, `thuTu`
- [ ] 3.4 Prisma: unique `maSanPhamHienThi`; cascade `Restrict` SanPham → DanhMuc; cascade `Delete` HinhAnh → SanPham
- [ ] 3.5 Báo Người 1 tạo migration `add-san-pham-hinh-anh`
- [ ] 3.6 Merge theo thứ tự phụ thuộc → unblock Người 3 (Task 5), Người 4 (Task 6), Người 5 (Task 7)
- [ ] 3.7 Commit: `feat: add SanPham and HinhAnhSanPham models`

#### Task 9: KhaDungService + UC03 (sau khi Task 5 và Task 7 xong)

> **Phụ thuộc: Task 5 từ Người 3 (ThietBi) + Task 7 từ Người 5 (DonThue, GiuCho).**

**Files cần tạo:**
```
apps/api/src/kha-dung/
  kha-dung.module.ts
  kha-dung.service.ts
apps/api/src/danh-muc/
  danh-muc.module.ts
  danh-muc.controller.ts    (public endpoints)
  danh-muc.service.ts
  dto/danh-muc.response.ts
apps/api/src/san-pham/
  san-pham.module.ts
  san-pham.controller.ts    (public endpoints)
  san-pham.service.ts
  dto/tim-kiem-san-pham.dto.ts
  dto/san-pham.response.ts
```

**Checklist:**
- [ ] 9.1 `KhaDungService`: `layKhaDung(maSanPham, gioNhan, gioTra)`, `layKhaDungNhieu(maSanPhams[], gioNhan, gioTra)`
- [ ] 9.2 Công thức khả dụng:
  - Chưa chọn ngày → chỉ hiện giá tham khảo, không khẳng định còn hàng
  - Kiểm tra giờ trả > giờ nhận; không cho bắt đầu trong quá khứ
  - Đếm tổng `ThietBi` đủ điều kiện từ phiếu nhập đã xác nhận (`DA_NHAP_KHO`)
  - **Giữ chỗ tạm chiếm lịch khi ĐỒNG THỜI:**
    - Đơn đang `CHO_THANH_TOAN`
    - `GiuCho.trangThai = DANG_GIU`
    - `thoiDiemHetHan > Now`
  - **Đơn đã xác nhận (`DA_XAC_NHAN`, `DANG_CHUAN_BI`, `SAN_SANG_NHAN`, `DANG_THUE`, ...) tính riêng và chỉ tính một lần** — không phụ thuộc hạn 15 phút
  - **Giữ chỗ hết hạn không chiếm lịch** dù job chưa chạy (`thoiDiemHetHan > Now` loại chúng ra)
  - **Không tính trùng**; hai bản ghi cùng thuộc một `ChiTietDonThue` chỉ tính một lần
  - Công thức: **tổng thiết bị đủ điều kiện − số lượng bị chiếm đồng thời lớn nhất trên khoảng `[gioNhan, gioTra]`**
  - Có thể dùng `prisma.$queryRaw` cho sweep line hoặc xử lý ở TypeScript
- [ ] 9.3 `SanPhamService.timKiem`: filter `tuKhoa`/`maDanhMuc`/`thuongHieu`/`sucChua`/`giaMin`/`giaMax`/`gioNhan`/`gioTra`, sort, phân trang (default 12); Zod validation cho DTO
  - **API khách chỉ trả `trangThaiKinhDoanh = 'DANG_KINH_DOANH'`**
  - Danh mục `trangThai != 'HIEN_THI'` cũng bị lọc khỏi kết quả khách
  - Cache Redis TTL 2 phút, key = hash(JSON.stringify(dto))
- [ ] 9.4 `SanPhamController` public (`@Controller('san-pham')`):
  - `GET /api/v1/san-pham` — list + phân trang
  - `GET /api/v1/san-pham/:id?gioNhan=&gioTra=` — chi tiết + khả dụng
- [ ] 9.5 `DanhMucController` public (`@Controller('danh-muc')`):
  - `GET /api/v1/danh-muc` — cây danh mục (cache Redis TTL 10 phút, key `danh_muc:all`)
  - `GET /api/v1/danh-muc/:id` — chi tiết
- [ ] 9.6 Đăng ký DI: `KhaDungService`, `DanhMucService`, `SanPhamService`
- [ ] 9.7 Commit: `feat: UC03 - product search with availability check and Redis cache`

---

### Hải Lý — Khuyến mãi + Kho skeleton + KhuyenMaiService (Task 4, 5)

**Có thể bắt đầu Task 4:** Day 1 (KhuyenMai cấp 0 độc lập).
**Bảng nối của Task 4 cần:** Task 2 (DanhMuc) và Task 3 (SanPham).
**Task 5 cần:** SanPham từ Task 3 (ChiTietPhieuNhap FK MaSanPham).
**Vai trò then chốt:** Output của Người 3 unblock Người 2 (Task 9), Người 4 (Task 6), Người 5 (Task 7).

**Trách nhiệm bổ sung — `KhuyenMaiService` dùng chung (bàn giao Day 2):**

```typescript
interface KhuyenMaiService {
  kiemTraApDung(maGiamGia: string, maKhachHang: bigint, dong: DongGio[], tienThueTruocGiam: Decimal): Promise<Result<KhuyenMaiHopLe>>;
  giuLuot(maKhuyenMai: bigint, maDonThue: bigint, thoiDiemHetHan: Date, tx: Prisma.TransactionClient): Promise<void>;
  xacNhanDaSuDung(maDonThue: bigint, tx: Prisma.TransactionClient): Promise<void>;
  giaiPhongLuot(maDonThue: bigint, tx: Prisma.TransactionClient): Promise<void>;
}
```
- `kiemTraApDung` — trả `Result<KhuyenMaiHopLe>` chứa số tiền được giảm; sai → `KhuyenMaiKhongHopLeException`
- `giuLuot` — tạo `LuotSuDungKhuyenMai` (`DANG_GIU`) trong transaction đã có
- `xacNhanDaSuDung` — chuyển lượt `DANG_GIU` → `DA_SU_DUNG` khi thanh toán thành công
- `giaiPhongLuot` — `DANG_GIU` → `DA_GIAI_PHONG` khi hủy; hết hạn → `HET_HAN`

**Quy tắc kiểm tra:**
- **Trạng thái:** chỉ `HIEN_THI`; `TAM_AN`/`HET_HAN` → từ chối.
- Thời hạn: `batDau <= Now <= ketThuc`.
- Phạm vi: sản phẩm/danh mục phù hợp; `TAT_CA` → áp cho toàn giỏ.
- Mức tối thiểu: `tienThueTruocGiam >= tienThueToiThieu`.
- Giới hạn tổng lượt: đếm `LuotSuDungKhuyenMai` có `trangThai` ∈ {`DANG_GIU` **còn hạn**, `DA_SU_DUNG`} < `gioiHanTongLuot`.
- Giới hạn mỗi khách: tương tự < `gioiHanMoiKhach`.
- **Lượt `DANG_GIU` đã hết hạn (`thoiDiemHetHan < Now`) không tiếp tục chiếm lượt** dù job chưa chạy.
- Giảm chỉ trên tiền thuê; không giảm cọc; không vượt `mucGiamToiDa`.

**Phân định gọi service:**
- **Người 3:** viết + test service; test đủ các nhánh trạng thái + hết hạn + `TAM_AN`.
- **Người 4 (giỏ):** gọi `kiemTraApDung` để **báo giá dự kiến, KHÔNG giữ lượt**.
- **Người 5 (tạo đơn):** gọi lại `kiemTraApDung` trong transaction + `giuLuot`.

#### Task 4: KhuyenMai + bảng nối (Cấp 0, 2)

> **Bảng nối cần Task 2 (DanhMuc) và Task 3 (SanPham).**

**Files cần tạo/chỉnh:**
```
prisma/schema.prisma                            ← thêm KhuyenMai + bảng nối + enums
apps/api/src/khuyen-mai/
  khuyen-mai.module.ts
  khuyen-mai.service.ts
  khuyen-mai.service.spec.ts
  dto/khuyen-mai-hop-le.dto.ts
```

**Checklist:**
- [ ] 4.1 Enums Prisma: `LoaiGiam {PHAN_TRAM, SO_TIEN}`, `PhamViApDung {TAT_CA, THEO_SAN_PHAM, THEO_DANH_MUC}`, `TrangThaiKhuyenMai {HIEN_THI, TAM_AN, HET_HAN}`
- [ ] 4.2 Model `KhuyenMai` — đủ 13 trường theo ERD, `maGiamGia` unique
- [ ] 4.3 `KhuyenMaiSanPham` composite PK `@@id([maKhuyenMai, maSanPham])` + `KhuyenMaiDanhMuc` composite PK `@@id([maKhuyenMai, maDanhMuc])`
- [ ] 4.4 Chuẩn bị model xong → báo Người 1 tạo migration `add-khuyen-mai`
- [ ] 4.5 Báo đã merge model/migration — Người 4 (Task 6) cần `KhuyenMai` FK để tạo `GioThue`
- [ ] 4.6 **`KhuyenMaiService` (bàn giao Day 2):** đủ 4 method như spec ở trên; **loại các lượt `DANG_GIU` đã hết hạn** khi đếm
- [ ] 4.7 Đăng ký `KhuyenMaiModule` export `KhuyenMaiService`
- [ ] 4.8 Test đơn vị (Jest) đầy đủ các nhánh
- [ ] 4.9 Commit: `feat: add KhuyenMai model, service and validation logic`

#### Task 5: Nhập kho + Thiết bị skeleton (Cấp 0-4)

> **Chỉ tạo model, KHÔNG làm API** — API nhập kho dời sang Tuần 3.
> **Phụ thuộc: Task 3** (ChiTietPhieuNhap FK MaSanPham).

**Files cần chỉnh:**
```
prisma/schema.prisma        ← thêm NhaCungCap, PhieuNhapHang, ChiTietPhieuNhap, ThietBi + enum
```

**Checklist:**
- [ ] 5.1 Model `NhaCungCap` (Cấp 0) — 10 trường
- [ ] 5.2 Model `PhieuNhapHang` (Cấp 2) — FK: `maNhaCungCap`, `maNguoiLap`, `maNguoiXacNhan` (nullable). Đủ **17 trường** theo ERD (bao gồm `thongTinNhaCungCapLucNhap` Json)
- [ ] 5.3 Model `ChiTietPhieuNhap` (Cấp 3) — FK: `maPhieuNhap`, `maSanPham`. Đủ 8 trường
- [ ] 5.4 Model `ThietBi` (Cấp 4) — FK: `maChiTietPhieuNhap` (NOT NULL). Đủ **9 trường** (bao gồm `phuKienDiKem` Json)
- [ ] 5.5 Enum `TrangThaiSuDungThietBi`: `SAN_SANG`, `DANG_THUE`, `DANG_BAO_TRI`, `THAT_LAC`, `NGUNG_SU_DUNG`
- [ ] 5.6 Prisma unique: `NhaCungCap.maNhaCungCapHienThi`, `PhieuNhapHang.maPhieuHienThi`, `ThietBi.maThietBiHienThi`
- [ ] 5.7 Chuẩn bị model xong → báo Người 1 tạo migration `add-kho-thiet-bi`
- [ ] 5.8 Báo đã merge model/migration — Người 2 cần `ThietBi` để làm Task 9
- [ ] 5.9 Commit: `feat: add NhaCungCap, PhieuNhapHang, ChiTietPhieuNhap, ThietBi (skeleton for Week 3)`

---

### Hoàng Tịnh — Giỏ thuê + BaoGiaService + Admin (Task 6, UC04, Task 11)

**Có thể bắt đầu Task 6:** Sau khi Task 3 (Người 2) + Task 4 (Người 3) push xong
**Có thể bắt đầu Task 11:** Sau khi Task 2+3 (Người 2) push xong

**Trách nhiệm bổ sung:**
- Hoàn thiện **giỏ và logic báo giá dùng chung** (`BaoGiaService`):
  - Thêm lại sản phẩm thì cộng số lượng; số lượng phải nguyên dương.
  - Đổi ngày/số lượng thì tính lại báo giá và khả dụng.
  - Giỏ không giữ hàng.
  - Có cơ chế lưu/đối chiếu báo giá khách đã xem (hash SHA-256 deterministic).
  - Chỉ giảm tiền thuê; phân bổ giảm giá xuống từng dòng và làm tròn thống nhất (VNĐ) qua `Prisma.Decimal` rounding `ROUND_HALF_UP`.
  - **Giỏ hiển thị cờ "ngừng kinh doanh"** cho dòng sản phẩm có `trangThaiKinhDoanh != 'DANG_KINH_DOANH'`.
- **Viết service Admin riêng** (`AdminSanPhamService`, `AdminDanhMucService`) — tách khỏi Người 2:
  - Validation: **giá thuê, cọc, giá trị bồi thường không âm** (`>= 0`).
  - Upload ảnh: kiểm tra magic bytes (`file-type`), ≤ 5MB, đuôi `.jpg/.jpeg/.png/.webp`.
  - Đổi trạng thái: `DANG_KINH_DOANH ↔ TAM_NGUNG ↔ NGUNG_KINH_DOANH`.
  - Xóa danh mục: chỉ khi không có sản phẩm và không có danh mục con.
  - Invalidate Redis cache sau mọi mutation.

#### Task 6: GioThue + ChiTietGioThue (Cấp 2-3)

**Files cần chỉnh:**
```
prisma/schema.prisma        ← thêm GioThue, ChiTietGioThue
```

**Checklist:**
- [ ] 6.1 Model `GioThue` — FK `maKhachHang` (UNIQUE — 1-1), FK `maKhuyenMai` (nullable BigInt). Đủ 6 cột
- [ ] 6.2 Model `ChiTietGioThue` — 4 cột: `maChiTietGio`, `maGioThue`, `maSanPham`, `soLuong`
- [ ] 6.3 Prisma: `maKhachHang` unique, cascade `Delete` ChiTietGioThue theo GioThue
- [ ] 6.4 Báo Người 1 tạo migration `add-gio-thue`
- [ ] 6.5 Có thể merge độc lập; model Task 7 không cần chờ Task 6
- [ ] 6.6 Commit: `feat: add GioThue and ChiTietGioThue models`

#### UC04 — Giỏ thuê API (từ Task 10)

**Files cần tạo:**
```
apps/api/src/gio-thue/
  gio-thue.module.ts
  gio-thue.controller.ts
  gio-thue.service.ts
  dto/them-vao-gio.dto.ts
  dto/cap-nhat-so-luong.dto.ts
  dto/dat-thoi-gian.dto.ts
  dto/ap-khuyen-mai.dto.ts
  dto/gio-thue.response.ts
apps/api/src/bao-gia/
  bao-gia.module.ts
  bao-gia.service.ts       ← logic tính giá dùng chung
```

**Checklist:**
- [ ] UC04.1 `GioThueService`: `layGio`, `them`, `capNhatSoLuong`, `xoaChiTiet`, `datThoiGian`, `apMaKhuyenMai`
- [ ] UC04.2 Logic giỏ:
  - **Thêm sản phẩm đã có → cộng dồn số lượng** (`upsert` theo `[maGioThue, maSanPham]`)
  - **Số lượng phải nguyên dương** (> 0)
  - **Đổi ngày/số lượng → tính lại báo giá và khả dụng**
  - **Giỏ không giữ hàng**
- [ ] UC04.3 Logic báo giá dùng chung (`BaoGiaService` để UC05 gọi lại):
  - `soNgay = Math.ceil((gioTra.getTime() - gioNhan.getTime()) / 86_400_000)` tối thiểu 1
  - `tienThue = SUM(donGiaThueMoiNgay * soLuong * soNgay)`
  - `tienCoc = SUM(mucCocMoiThietBi * soLuong)`
  - Áp `KhuyenMai`: dùng logic Người 3
  - **Chỉ giảm tiền thuê**; **phân bổ giảm giá xuống từng dòng**, làm tròn qua `Prisma.Decimal`
  - **Lưu/đối chiếu báo giá**: hash SHA-256 deterministic
- [ ] UC04.4 `GioThueController` (`@UseGuards(JwtAuthGuard)`, `@Controller('gio-thue')`):
  - `GET /api/v1/gio-thue`
  - `POST /api/v1/gio-thue/them`
  - `PUT /api/v1/gio-thue/:maChiTiet/so-luong`
  - `DELETE /api/v1/gio-thue/:maChiTiet`
  - `PUT /api/v1/gio-thue/thoi-gian`
  - `POST /api/v1/gio-thue/ma-giam-gia`
- [ ] UC04.5 Commit: `feat: UC04 - cart service, BaoGiaService, and API`

#### Task 11: UC21 — Admin CRUD danh mục & sản phẩm

**Files cần tạo:**
```
apps/api/src/danh-muc/admin-danh-muc.controller.ts
apps/api/src/danh-muc/admin-danh-muc.service.ts
apps/api/src/san-pham/admin-san-pham.controller.ts
apps/api/src/san-pham/admin-san-pham.service.ts
apps/api/src/*/dto/tao-danh-muc.dto.ts
apps/api/src/*/dto/cap-nhat-danh-muc.dto.ts
apps/api/src/*/dto/tao-san-pham.dto.ts
apps/api/src/*/dto/cap-nhat-san-pham.dto.ts
```

**Checklist:**
- [ ] 11.1 `@UseGuards(JwtAuthGuard, RolesGuard)` + `@Roles('QUAN_TRI_VIEN')` cho toàn bộ admin controllers
- [ ] 11.2 `AdminDanhMucController` (`@Controller('admin/danh-muc')`):
  - GET, POST, PUT /:id, DELETE /:id (chỉ khi không có sp và danh mục con), PATCH /:id/trang-thai
  - Validate: không cho `maDanhMucCha` là chính nó hoặc con của nó
  - Invalidate cache `danh_muc:all`
- [ ] 11.3 `AdminSanPhamController` (`@Controller('admin/san-pham')`):
  - GET (bao gồm cả `TAM_NGUNG`/`NGUNG_KINH_DOANH`), POST (unique `maSanPhamHienThi`), PUT /:id, PATCH /:id/trang-thai
  - POST /:id/hinh-anh — upload (Multer memory + `file-type` magic bytes, `.jpg/.jpeg/.png/.webp` ≤ 5MB)
  - DELETE /hinh-anh/:maHinhAnh
  - Invalidate cache sản phẩm
- [ ] 11.4 **Validation trong `AdminSanPhamService`:** `giaThueMoiNgay >= 0`, `mucCocMoiThietBi >= 0`, `giaTriBoiThuong >= 0`; `sucChua >= 0`, `maSanPhamHienThi` không trùng
- [ ] 11.5 Commit: `feat: UC21 - admin category and product CRUD API`

---

### Trung Hiếu — Đơn thuê + Hủy + Thanh toán + Hết hạn + Lịch sử đơn (Task 7, 8, UC05+06)

**Có thể bắt đầu Task 7:** Sau khi Task 3 (Người 2) và Task 4 (Người 3) xong.
**Task 8 + UC05+06 làm tuần tự sau Task 7.**
**UC05 cần:** Giỏ (Người 4), khả dụng (Người 2), logic báo giá/khuyến mãi (Người 3 + Người 4).

**Trách nhiệm bổ sung:**
- Hoàn thiện **tạo đơn, hủy chưa thanh toán, thanh toán, hết hạn và ghi lịch sử đơn**:
  - Chọn chính sách **đang có hiệu lực** (`thoiDiemApDung <= Now`).
  - Kiểm tra, tạo đơn, snapshot, giữ hàng, giữ lượt mã và dọn giỏ trong **cùng transaction** (`prisma.$transaction` với `isolationLevel: 'Serializable'`).
  - Hai yêu cầu đồng thời không tạo hai đơn từ cùng dữ liệu giỏ.
  - **Kiểm tra lại `trangThaiKinhDoanh` của từng sản phẩm khi tạo đơn** — sản phẩm có thể bị Admin chuyển sang `TAM_NGUNG`/`NGUNG_KINH_DOANH` sau lúc khách thêm giỏ.
  - Hủy chưa thanh toán: ghi người hủy, thời điểm, lý do; giải phóng giữ chỗ và lượt mã.
  - **Ghi lịch sử chuyển trạng thái** bằng model `LichSuTrangThaiDon`.
  - Tạo URL thanh toán → tạo giao dịch `DANG_XU_LY`. Callback → tìm đúng giao dịch, kiểm tra đơn/số tiền.
  - **Cập nhật trạng thái nguyên tử:** `updateMany` với `where` điều kiện (Prisma optimistic concurrency).
  - **Callback lặp — xử lý theo thứ tự:** (1) tìm giao dịch theo `maYeuCau`; (2) **nếu giao dịch đã `THANH_CONG` → trả kết quả cũ ngay**; (3) chỉ khi giao dịch còn `DANG_XU_LY` mới yêu cầu đơn phải còn `CHO_THANH_TOAN`.
  - Thanh toán, hủy và job hết hạn phải phối hợp — **không ghi đè trạng thái nhau**.
  - Dùng đúng tên trường: `hanThanhToan` (DonThue), `thoiDiemHetHan` (GiuCho, LuotSuDungKhuyenMai).
  - BullMQ processor `expire-don-thue` — job idempotent, delayed job 15 phút được enqueue khi tạo đơn.

#### Task 7: Đơn thuê hoàn chỉnh (Cấp 2-5)

> 6 model (thêm `LichSuTrangThaiDon`) phụ thuộc lẫn nhau — làm trong 1 migration.
> **Phụ thuộc: Task 3 (SanPham) và Task 4 (KhuyenMai).**

**Files cần chỉnh:**
```
prisma/schema.prisma        ← thêm ChinhSach, DonThue, ChiTietDonThue, GiuCho,
                              LuotSuDungKhuyenMai, LichSuTrangThaiDon + enums
```

**Checklist:**
- [ ] 7.1 Enum `TrangThaiDonThue`: 11 giá trị (`CHO_THANH_TOAN`, `DA_XAC_NHAN`, ..., `HOAN_TAT`, `HET_HAN`, `KHACH_HUY`, `CUA_HANG_HUY`)
- [ ] 7.2 Model `ChinhSach`: `maChinhSach`, `maNguoiTao` (FK NhanVien), `tenChinhSach`, `phienBan` (Int unique), `thoiDiemApDung`, `noiDungChinhSach` (Json), `ngayTao`
- [ ] 7.3 Model `DonThue` — đủ **22 trường** theo ERD: FK `maKhachHang`, `maChinhSach` (NOT NULL), `maNguoiHuy?` (FK **TaiKhoan**). `maDonHienThi` unique. Bổ sung rõ: **`gioNhanDuKien`**, **`gioTraDuKien`**, **`hanThanhToan`**. Snapshot `khuyenMaiLucDat` (Json?)
- [ ] 7.4 Model `ChiTietDonThue` — snapshot đầy đủ: `tenSanPhamLucDat`, `donGiaThueMoiNgay`, `mucCocMoiThietBi`, `giaTriBoiThuongMoiThietBi`, `phuKienVaMucBoiThuongLucDat` (Json). Thêm: `soNgayTinhTien`, `soLuong`, `tienGiam`
- [ ] 7.5 Model `GiuCho` — **1-1 với ChiTietDonThue**: `maChiTietDon` UNIQUE FK
- [ ] 7.6 Enum `TrangThaiGiuCho`: `DANG_GIU`, `DA_XAC_NHAN`, `DA_GIAI_PHONG`, `HET_HAN`
- [ ] 7.7 Model `LuotSuDungKhuyenMai` — **1-1 với DonThue**: `maDonThue` UNIQUE FK. **`maKhuyenMai` (FK NOT NULL)**
- [ ] 7.8 Model `LichSuTrangThaiDon` — `maLichSuDon`, `maDonThue`, `maNguoiThucHien` (FK TaiKhoan, nullable), `trangThaiTruoc`, `trangThaiSau`, `thoiDiem`, `lyDo?`
- [ ] 7.9 Prisma: `DonThue.maDonHienThi` unique, `DonThue.maKhachHang` `onDelete: Restrict`, `GiuCho.maChiTietDon` unique + cascade Delete, `LuotSuDungKhuyenMai.maDonThue` unique, `ChinhSach.phienBan` unique. Index: `DonThue.trangThai`, `GiuCho.thoiDiemHetHan`, `LuotSuDungKhuyenMai.thoiDiemHetHan`
- [ ] 7.10 Chuẩn bị model xong → báo Người 1 tạo migration `add-don-thue-flow`. **Bàn giao sớm (Day 3)** cho Người 2 làm Task 9
- [ ] 7.11 Commit: `feat: add ChinhSach, DonThue, ChiTietDonThue, GiuCho, LuotSuDungKhuyenMai, LichSuTrangThaiDon`

#### Task 8: ThanhToan + ChiTietThanhToan (Cấp 4-5)

**Files cần chỉnh:**
```
prisma/schema.prisma        ← thêm ThanhToan, ChiTietThanhToan + enums
```

**Checklist:**
- [ ] 8.1 Enum `MucDichThanhToan`: `TIEN_THUE`, `TIEN_COC`, `THU_BO_SUNG`; hoàn cọc về sau dùng `HoanTien`
- [ ] 8.2 Enum `TrangThaiThanhToan`: `DANG_XU_LY`, `THANH_CONG`, `THAT_BAI`, `HUY`
- [ ] 8.3 Model `ThanhToan` — 13 trường: FK `maDonThue`, FK `maNguoiGhiNhan` (nullable), `maYeuCau` (varchar 100 UNIQUE — idempotency key), `congThanhToan`, `maGiaoDichCong`, `tongSoTien`, `phuongThuc`, `thoiDiemTao`, `thoiDiemThanhCong`, `trangThai`, `trangThaiDoiChieu`, `ghiChu`
- [ ] 8.4 Model `ChiTietThanhToan`: `maChiTietThanhToan`, `maThanhToan`, `mucDich`, `soTien`
- [ ] 8.5 Prisma: `ThanhToan.maYeuCau` unique, index `ThanhToan.maGiaoDichCong`
- [ ] 8.6 Báo Người 1 tạo migration `add-thanh-toan`
- [ ] 8.7 Commit: `feat: add ThanhToan and ChiTietThanhToan models`

#### UC05 + UC06 — Đặt đơn + Hủy + Thanh toán + Hết hạn

**Files cần tạo:**
```
apps/api/src/don-thue/
  don-thue.module.ts
  don-thue.controller.ts
  don-thue.service.ts
  dto/tao-don-thue.dto.ts
  dto/don-thue.response.ts
apps/api/src/thanh-toan/
  thanh-toan.module.ts
  thanh-toan.controller.ts
  thanh-toan.service.ts
  dto/thanh-toan-callback.dto.ts
apps/api/src/jobs/
  jobs.module.ts
  don-thue-expiry.processor.ts    ← BullMQ processor
```

**Checklist UC05:**
- [ ] UC05.1 `DonThueService.taoDon` trong `prisma.$transaction(async (tx) => {...}, { isolationLevel: 'Serializable' })`:
  1. Load giỏ, kiểm tra không rỗng
  2. **Kiểm tra `trangThaiKinhDoanh` từng sản phẩm** — `DANG_KINH_DOANH`; sai → `TrangThaiKhongHopLeException`
  3. Kiểm tra khả dụng từng dòng → thiếu → `KhongDuHangException`
  4. Nếu báo giá thay đổi (đối chiếu hash) → `BaogiaThayDoiException`
  5. **Nếu giỏ có mã khuyến mãi** → gọi lại `KhuyenMaiService.kiemTraApDung`
  6. Load `ChinhSach` **đang có hiệu lực** (`thoiDiemApDung <= Now`)
  7. **Gán hạn thanh toán thống nhất** — cùng 1 mốc thời gian cho cả 3 trường:
     ```typescript
     const thoiDiemTaoDon = new Date();
     const hanThanhToan = new Date(thoiDiemTaoDon.getTime() + 15 * 60 * 1000);
     donThue.hanThanhToan               = hanThanhToan;
     giuCho.thoiDiemHetHan              = hanThanhToan;
     luotSuDungKhuyenMai.thoiDiemHetHan = hanThanhToan;
     ```
  8. Tạo `DonThue` (`CHO_THANH_TOAN`), sinh `maDonHienThi` unique
  9. Snapshot vào `ChiTietDonThue`
  10. Với mỗi `ChiTietDonThue`: tạo `GiuCho` (`DANG_GIU`)
  11. Nếu có khuyến mãi: gọi `KhuyenMaiService.giuLuot`
  12. Xóa giỏ
  - **Toàn bộ trong cùng transaction; thất bại → rollback toàn bộ, giữ nguyên giỏ**
  - **Ghi lịch sử chuyển trạng thái** vào `LichSuTrangThaiDon`
  - Sau commit: enqueue BullMQ job `expire-don-thue` delay 15 phút

> **Quy ước hết hạn:** còn hạn khi `Now < hạn`; hết hạn khi `Now >= hạn`. Query đếm lượt đang chiếm dùng `thoiDiemHetHan > Now`. Job hết hạn dùng `hạn <= Now`.
- [ ] UC05.2 **Hủy đơn chưa thanh toán — cùng transaction, cập nhật có điều kiện:**
  - `updateMany` `DonThue` `where: { maDonThue, trangThai: 'CHO_THANH_TOAN' }` → `KHACH_HUY`/`CUA_HANG_HUY`. Nếu `count == 0` → `TrangThaiKhongHopLeException`, rollback
  - Ghi `maNguoiHuy` (MaTaiKhoan), `thoiDiemHuy`, `lyDoHuy`
  - Giải phóng `GiuCho`: `updateMany` `where: { ..., trangThai: 'DANG_GIU' }` → `DA_GIAI_PHONG`
  - Gọi `KhuyenMaiService.giaiPhongLuot`
  - Ghi lịch sử chuyển trạng thái
- [ ] UC05.3 BullMQ processor `expire-don-thue` — **cập nhật có điều kiện trong transaction**:
  - `updateMany` `DonThue` `where: { maDonThue, trangThai: 'CHO_THANH_TOAN' }` → `HET_HAN`. `count == 0` → bỏ qua, commit
  - Giải phóng `GiuCho` và `LuotSuDungKhuyenMai` có `thoiDiemHetHan <= Now`
  - Ghi lịch sử
  - **Không ghi đè trạng thái nhau:** `updateMany` có điều kiện `where: { trangThai: ... }`. Job idempotent
- [ ] UC05.4 `DonThueController` (`@UseGuards(JwtAuthGuard)`, `@Controller('don-thue')`):
  - `GET /api/v1/don-thue/xac-nhan` — preview
  - `POST /api/v1/don-thue` — tạo đơn
  - `GET /api/v1/don-thue/:id` — chi tiết
  - `GET /api/v1/don-thue` — danh sách đơn của khách
  - `POST /api/v1/don-thue/:id/huy`

**Checklist UC06:**
- [ ] UC06.0 Tuần 2 chỉ hủy đơn chưa thanh toán; UC07 tuần sau. Gateway mock demo.
- [ ] UC06.1 Mock gateway `taoUrl`: tạo giao dịch `ThanhToan` (`DANG_XU_LY`); `maYeuCau = crypto.randomUUID().replace(/-/g, '')`
- [ ] UC06.2 `xuLyKetQua(callback)` — thứ tự bước quan trọng:
  1. **Tìm giao dịch theo `maYeuCau`** — không thấy → 404
  2. **Idempotency (trước khi kiểm tra đơn):** giao dịch đã `THANH_CONG` → **trả kết quả cũ ngay**
  3. Nếu giao dịch đã `THAT_BAI`/`HUY` → trả lỗi cuối
  4. Chỉ khi giao dịch còn `DANG_XU_LY` mới đi tiếp:
     - Kiểm tra số tiền
     - **Cập nhật nguyên tử trong 1 `prisma.$transaction` (updateMany có điều kiện):**
       - `updateMany` `DonThue` `where: { maDonThue, trangThai: 'CHO_THANH_TOAN', hanThanhToan: { gt: now } }` → `DA_XAC_NHAN`
       - `updateMany` `ThanhToan` `where: { maThanhToan, trangThai: 'DANG_XU_LY' }` → `THANH_CONG` + tạo 2 `ChiTietThanhToan`
       - `updateMany` `GiuCho` → `DA_XAC_NHAN`
       - Gọi `KhuyenMaiService.xacNhanDaSuDung`
       - Ghi lịch sử
  - **Nếu `updateMany` đơn trả `count == 0`** — **KHÔNG tự ghi đè `THAT_BAI`**:
    - Rollback, đọc lại đơn và giao dịch:
      - Giao dịch đã `THANH_CONG` → trả kết quả cũ
      - Đơn `HET_HAN`/`KHACH_HUY`/`CUA_HANG_HUY` mà gateway đã thu tiền → `trangThaiDoiChieu = 'CAN_DOI_SOAT'`, tạo yêu cầu hoàn tiền (UC07). **Không mặc định `THAT_BAI`**
      - Đơn `DA_XAC_NHAN` mà giao dịch này vẫn `DANG_XU_LY` → kiểm tra gateway: có thu → `CAN_DOI_SOAT` + `THANH_CONG` + yêu cầu hoàn tiền thu trùng; chưa thu → `THAT_BAI` với ghi chú
      - Ghi log chi tiết để nghiệp vụ đối soát thủ công
- [ ] UC06.3 `ThanhToanController`:
  - `POST /api/v1/thanh-toan/:maDon/tao-url` `@UseGuards(JwtAuthGuard)`
  - `GET /api/v1/thanh-toan/ket-qua` — callback
- [ ] UC06.4 Commit: `feat: UC05+UC06 - order creation with seat hold and mock payment`

---

## Điểm phối hợp & Dependencies

| Sự kiện trigger | Ai được unblock |
|---|---|
| Người 2 merge **Task 2** (DanhMuc) | Người 2 Task 3, Người 3 Task 4 bảng nối |
| Người 2 merge **Task 3** (SanPham) | Người 3 Task 4 bảng nối + Task 5, Người 4 Task 6, Người 5 Task 7 |
| Người 3 merge **Task 4** (KhuyenMai) | Người 4 Task 6, Người 5 Task 7 |
| Người 3 merge **Task 5** (ThietBi) | Người 2 Task 9 |
| Người 4 merge **Task 6** (GioThue) | Người 4 UC04; không block model Task 7 |
| Người 5 merge **Task 7** (DonThue) — **bàn giao sớm Day 3** | Người 2 Task 9, Người 5 Task 8 |
| Người 5 merge **Task 8** (ThanhToan) | Người 5 UC06 |

### Quy tắc phối hợp

- **Người điều phối migration:** Model owner chỉnh `schema.prisma` trong branch riêng → báo đã sẵn sàng → **Người 1 merge và chạy `npx prisma migrate dev --name <ten>`** → commit migration + generated files → thông báo tag/commit cho cả nhóm dùng chung `git pull && npx prisma migrate deploy && npx prisma generate`.
- Không yêu cầu mọi người tự push thẳng `main`; mọi merge phải qua branch/review.
- Task 7 bàn giao sớm cho Người 2 làm khả dụng.
- Seed data làm dần khi model merge.

### Quy ước dùng chung (tất cả thành viên)

- **Tên trường, kiểu dữ liệu:** theo ERD, camelCase TS map snake_case SQL qua `@map`. Enum lưu chuỗi UPPER_SNAKE_CASE (native Prisma enum).
- **Chính sách:** chọn bản **đang có hiệu lực** (`thoiDiemApDung <= Now`).
- **Tên trường hạn:** `hanThanhToan` (DonThue), `thoiDiemHetHan` (GiuCho, LuotSuDungKhuyenMai).
- **DTO:** tách Request/Response, dùng `class-validator` + `class-transformer`; cấu trúc lỗi chung.
- **BigInt serialization:** ở boundary API dùng Zod transformer `bigint → string`.
- **Decimal:** dùng `Prisma.Decimal` / `decimal.js`, không dùng `number`.
- **Test:** happy path **9 bước**, không 8 bước.
- **JSON:** cấu trúc snapshot thống nhất qua Zod schema trong `packages/types/`.
- **Cache Redis:** danh mục TTL 10 phút, sản phẩm list TTL 2 phút, chi tiết sản phẩm TTL 5 phút. Giỏ thuê và khả dụng KHÔNG cache. Invalidate khi mutation.

---

## Checklist nghiệm thu tuần 2

- [ ] `docker compose up` — tất cả services start, không lỗi
- [ ] `npx prisma migrate deploy` — đủ migrations cho Task 0-8
- [ ] `pnpm build` (turbo) — 0 errors ở `apps/api` và `apps/web`
- [ ] Đăng ký + đăng nhập trả JWT hợp lệ trong HTTP-only cookie
- [ ] Sai mật khẩu 5 lần → khóa tạm 15 phút trong Redis
- [ ] Tài khoản bị khóa không tạo được giao dịch mới
- [ ] `GET /api/v1/san-pham?gioNhan=&gioTra=` — trả số khả dụng đúng
- [ ] Thêm giỏ, đổi số lượng, đổi thời gian → báo giá đúng
- [ ] Thêm sản phẩm đã có → cộng dồn số lượng
- [ ] Áp mã `TEST10` → giảm 10% khi tiền thuê ≥ 500.000₫
- [ ] Tạo đơn → `CHO_THANH_TOAN`, có `GiuCho`, hạn 15 phút, BullMQ job enqueue
- [ ] Đơn hết hạn → BullMQ processor tự chuyển `HET_HAN`, `GiuCho` giải phóng
- [ ] Hủy đơn chưa thanh toán → ghi người hủy, giải phóng giữ chỗ + lượt mã
- [ ] Thanh toán mock → `DA_XAC_NHAN`, 2 `ChiTietThanhToan`
- [ ] Callback trùng `maYeuCau` → không ghi trùng
- [ ] Thanh toán xong không làm khả dụng tăng trở lại
- [ ] Hai đơn cũ không trùng nhau không bị cộng dồn sai
- [ ] Khách không được xem/sửa đơn và giỏ của người khác
- [ ] `/api/v1/admin/*` chỉ nhận `QUAN_TRI_VIEN`, token `KHACH_HANG` → 403
- [ ] Integration test happy path **9 bước** pass
- [ ] Mỗi chuyển trạng thái đơn có bản ghi trong `LichSuTrangThaiDon`
- [ ] Redis cache hit khi load lại danh mục hoặc danh sách sản phẩm
