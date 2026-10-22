# RSM Weekly Report — Hướng dẫn cho Claude

> **Mục tiêu:** Tự động tạo 8 báo cáo kinh doanh tuần cho 8 GĐKV (kết nối DB, tính KPI, phát hiện chỉ số lệch) — thay thế tổng hợp tay tốn vài giờ xuống ~2 phút chạy script.
>
> **Cách dùng:** Mỗi tuần cập nhật 2 file input → Claude nhận prompt "Tạo báo cáo tuần kỳ DD/MM–DD/MM/YYYY" → nhận 8 file `.md` gửi GĐKV điền → tổng hợp làm agenda họp thứ Hai.
>
> **Đọc `HUONG_DAN.md` ở thư mục gốc project để biết chi tiết từng bước.**

---

## Lần đầu chạy trên máy mới — setup một lần

**21/08/2026 — đã migrate sang PostgreSQL** (account MSSQL cũ đã bị IT disable): `run_all.js`/
`run_salon.js`/`build_van_hanh.js` không đọc `config.json` nữa — file đó cùng `config.example.json`
giờ là rác lịch sử, không xoá nhưng KHÔNG cần đụng tới.

**15/09/2026 — bỏ hardcode mật khẩu DB, chuyển sang `scripts/.env`** (để đóng gói gửi máy khác an
toàn hơn): 3 script trên gọi `require('./load_env')()` (file `scripts/load_env.js`, tự parse
`scripts/.env` nếu có, không ghi đè biến đã export sẵn trong shell) rồi đọc `process.env.PGUSER`/
`process.env.PGPASSWORD` khi tạo `Pool`. Thoát sớm kèm thông báo nếu thiếu 1 trong 2 biến. Máy MỚI
**bắt buộc** phải tạo `scripts/.env` (copy từ `scripts/.env.example`, điền tài khoản Postgres — quyền
SELECT trên `data_warehouse` cluster `pgdb-bi` + `solution_30shine` cluster `pgdb-ro`) trước khi chạy
được — khác hẳn máy Trung Vũ đã có sẵn `.env` thật (không đóng gói/gửi kèm khi chia sẻ project).
`Tool_BC_ChamDiem_VH_v1/pull_chamdiem.py` (spawn bởi `build_van_hanh.js`) cũng đọc `PGUSER`/
`PGPASSWORD` qua cùng cơ chế nếu `PGDSN_BI`/`PGDSN_RO` chưa được set — `build_van_hanh.js` tự truyền
2 DSN này xuống subprocess qua `env` của `execSync`, không cần cấu hình thêm ở `Tool_BC_ChamDiem_VH_v1/`
trừ khi muốn chạy `pull_chamdiem.py` độc lập (khi đó cần `Tool_BC_ChamDiem_VH_v1/.env` riêng, xem
`.env.example` cùng thư mục).

Trước khi chạy báo cáo lần đầu:

**1. Tạo `scripts/.env`** (copy từ `scripts/.env.example`, điền `PGUSER`/`PGPASSWORD`).

**2. Cài dependencies** nếu chưa có `scripts/node_modules/`:

```bash
cd scripts
npm install
```

Chỉ làm 1 lần mỗi máy.

---

## Khi nhận prompt "Tạo báo cáo tuần kỳ [ngày]–[ngày]"

Parse ngày → chạy 1 lệnh (thay đường dẫn cho đúng với máy hiện tại):

```bash
node [đường_dẫn_project]/scripts/run_all.js YYYY-MM-DD YYYY-MM-DD
```

Script tự tính tất cả. Output: `output/[KY_LABEL]/report/` — 8 file `.md`.

Chạy 1 GĐKV: thêm tham số thứ 3 là key (ví dụ `NguyenThanhSang`).

---

## QUAN TRỌNG

- Không dùng MCP để chạy báo cáo — `run_all.js`/`run_salon.js` tự kết nối Postgres trực tiếp bằng
  `pg.Pool` (2 pool: `pgdb-bi`/schema `bi`, `pgdb-ro`/schema `main`), không qua MCP mssql/postgres nào cả
- Không spawn subagent để chạy query
- Q4C, Q_POOL_BOOK, Q_SM_CC là 3 chỗ JOIN xuyên 2 cluster Postgres (`bi.*` + `main.*` không JOIN được
  trong 1 câu SELECT) — xử lý bằng hàm JS riêng (`computeQ4C`/`computeQPoolBook`/`fetchSmCcRows` và
  bản `...Salon` trong run_salon.js), không phải SQL text như 11 query còn lại

---

## Khi sang tháng mới

Cập nhật 3 section trong **`config/targets.json`** (không sửa `run_all.js`):
- `targets` — DT/LK/AOV/định biên theo GĐKV
- `targets_kh` — 8 nhóm KH theo GĐKV
- `targets_salon` — DT/LK/AOV theo salon (key = Salon_ID dạng string)

Mục tiêu chấm VH (báo cáo tháng) lấy từ **`config/salon_vh.json`** — chỉ cập nhật khi danh sách/loại/tần suất salon thay đổi.

## Khi SM thay đổi (nghỉ / chuyển salon)

Cập nhật **`config/sm_info.json`** — key = StaffId, trường `gdkv` = key GĐKV.

---

## Quy tắc đã áp dụng trong run_all.js

| Quy tắc | Chi tiết |
|---------|----------|
| Loại mã ảo (Q4c) | INNER JOIN `Staff` với `AdditionStaff=0` AND `parentID IS NULL` |
| Loại tablet (Q4d) | `ISNULL(Source,'') <> 'TABLET'` |
| Feedback RAG | ≤0.35% (sau khi loại đánh nhầm); 🟢≤105%, 🟡106–120%, 🔴>120% |
| Runrate | Lũy kế tháng × (DAYS_MONTH ÷ DAYS_PAST) |
| Kế hoạch kỳ (Section 1) | = Mục tiêu tháng ÷ DAYS_MONTH × **KY_DAYS** (số ngày thực tế trong kỳ), KHÔNG hardcode ×7. Cột "Đánh giá" KQKD = `rag(thực tế kỳ, kế hoạch kỳ)`. AOV kế hoạch kỳ = mục tiêu tháng (không chia ngày). Áp dụng cả DT/LK và nhóm KH quen. |
| Cột kỳ trước (bảng chi tiết salon) | Q2 tính thêm `DT_ky_truoc`/`LK_ky_truoc` = tổng N ngày ngay trước @tu (kỳ trước liền kề). Bảng "Chi tiết theo salon" hiển thị 3 cột cuối: DT/LK/AOV kỳ trước. |
| Topup = 0 | Nghĩa là **chưa đặt mục tiêu** (không phải MT=0): báo cáo ghi chú cảnh báo, bảng agenda hiện "*Chưa đặt MT*", không tính %/RAG. |
| SM credit | `min(ngày đi, 6) - (1 nếu không đi cuối tuần)` |
| Mục tiêu chấm VH (THÁNG) | Đọc từ `config/salon_vh.json`, **gộp cả salon Công ty + Nhượng quyền** (KPI DT/khách không đổi, vẫn 58 salon Công ty). Tần suất → lượt/tháng: `thang2`→2, `thang1`→1, `quy1`→1 chỉ khi tháng ∈ {3,6,9,12} (cuối quý), ngược lại 0. `REPORT_MONTH`=tháng của DEN. Hàm `calcVhTarget(gdkvKey, reportMonth)` → `{total, congTy, nhuongQuyen}`. Báo cáo TUẦN giữ nguyên (mục tiêu = kế hoạch Excel). Đã bỏ `REMOTE_SALONS`/×3 cũ. |
| Section 4 | Tự phát hiện 🟡🔴 từ tất cả KPI, liệt kê sẵn cho GĐKV điền. Cột hành động điều chỉnh tách thành **Mục tiêu điều chỉnh** (chỉ số → mức · mốc thời gian) + **Kế hoạch đạt mục tiêu** (ai · salon · làm gì · khi nào). **Chốt 14/09/2026:** đã bỏ 5 chỉ số vận hành khỏi danh sách tự phát hiện — Nhân sự Stylist, Nhân sự Skinner, SM hiện diện, Chấm vận hành GĐKV, Chấm công (ERP định biên). **Chốt 22/09/2026:** bỏ thêm 5S ("% Khung-KV đã đăng ký", Q4B) khỏi danh sách tự phát hiện — dễ nhầm với mục 8 5S của `build_van_hanh.js` (xem dòng Q4B bên dưới). Chỉ còn KPI kinh doanh (DT/LK/AOV/CR/KH quen) trong danh sách tự phát hiện Section 4.1 (Feedback CSKH cũng đã bỏ khỏi đây từ 21/09/2026, xem ghi chú trong `run_all.js`). Nhân sự/SM/Chấm VH đã có mục GĐKV-điền tương đương ở `build_van_hanh.js`; **Chấm công (Q4C) hiện chưa có mục tương đương bên đó — gap tạm thời, xem `.claude/agents/agent-culi-tuan/agent-culi-tuan.md`**. |
| Q4C — "Xếp ca-chấm công đủ định biên" (chốt 20/07/2026) | Ý1 (định biên: lịch N+1 ≥ 50% HC ngày thường / 80% cuối tuần) là điều kiện PASS chính. Ý2 (năng suất: KH/HC ≤ 13 Stylist / 13 Skinner ngày thường, 17/16 cuối tuần) **chỉ áp dụng khi lịch N+1 < 100% HC hôm nay** — nếu lịch N+1 đã ≥100% HC thì bỏ qua Ý2 dù vượt năng suất. Áp dụng đồng bộ ở cả `run_all.js` và `run_salon.js`. |
| Output path | `output/[KY_LABEL]/report-me/` (Section 1,4,5 — GĐKV điền; đã bỏ Section 2 Nhân sự từ 14/09/2026, dời sang report-van-hanh/), `output/[KY_LABEL]/report-sla/` (Section 3 SLA, tách riêng từ 07/09/2026, tự động, không có phần GĐKV điền), `output/[KY_LABEL]/report-van-hanh/` (báo cáo vận hành — script riêng `build_van_hanh.js`, xem mục dưới) và `output/[KY_LABEL]/agenda/` (Agenda_Hop_Tuan + Agenda_SLA_Tuan — agenda cũng đã tách 2 file từ 07/09/2026, KHÔNG còn PhuLuc_NhanSu từ 14/09/2026 — xem dòng dưới) |
| Section 3 (SLA) tách file riêng (chốt 07/09/2026) | `buildReport()` trả về `{ main, sla }` — `main` ghi `report-me/BC_KD_Tuan_...md` (không còn Section 3), `sla` ghi `report-sla/BC_SLA_Tuan_...md` (đánh số lại "## 1." vì là file độc lập). Áp dụng cho cả 2 luồng tuần/tháng (không có nhánh IS_MONTHLY riêng). `parseSection4KeHoachRows` (đọc mục 4.3 kỳ trước) không đổi vì Section 4/5 vẫn giữ nguyên số ở `report-me`. |
| Nhân sự (Section 2 + Q_HC) DỜI HẲN sang `build_van_hanh.js` (chốt 14/09/2026) | `run_all.js` không còn Section 2 "TÌNH HÌNH NHÂN SỰ", không còn gọi Q_HC (đã xoá khỏi `Q_KEYS`), không còn 2 dòng issue Section 4 "Nhân sự Stylist/Skinner", không còn dòng ĐIỂM SÁNG "Định biên Sty+Ski đủ". `parseNhanSuRows`/`loadPrevNhanSuRows`/`buildPersonnelMd` đã xoá khỏi `run_all.js` — tương đương (`parseNhanSuRowsVH`/`loadPrevNhanSuRowsVH`/`buildPersonnelMd`) nay nằm trong `build_van_hanh.js`, đọc/ghi `report-van-hanh/` thay vì `report-me/`/`agenda/`. `PhuLuc_NhanSu_[KY].md` giờ ở `report-van-hanh/`, do `build_van_hanh.js` sinh. Chi tiết đầy đủ xem mục "Báo cáo Vận hành" bên dưới + `.claude/agents/agent-culi-tuan/agent-culi-tuan.md`. |
| Agenda tách file riêng Agenda_SLA (chốt 07/09/2026, cùng tinh thần report-me/report-sla) | `buildAgenda()` trả về `{ agendaMd, agendaSlaMd }` (đã bỏ `personnelMd` từ 14/09/2026). `items` build chung 1 lần (mọi topic đã gắn `section`), rồi lọc: `itemsMain = items.filter(section!=='SLA')` → `Agenda_Hop_Tuan_...md` (KD/TOPUP/ĐIỂM SÁNG + Trao đổi tự do, vẫn 120' — đã bỏ hẳn section 'NS' từ 14/09/2026), `itemsSla = items.filter(section==='SLA')` → `Agenda_SLA_Tuan_...md` (Feedback CSKH/Định biên ERP/Chấm vận hành, phiên riêng giả định 30' 16:00–16:30 ngay sau agenda chính — KHÔNG cộng dồn vào 120'). Đếm 🔴/🟡 và `contentMin`/`freeMin` tính riêng mỗi file. Bảng tóm tắt GĐKV + Phụ lục A của agenda chính bỏ cột Feedback/Chấm VH/SLA nổi bật (07/09) và cột Nhân sự (14/09) — dời sang bảng + phụ lục riêng trong Agenda_SLA và `BC_VanHanh_TongQuan_...`; "Mức độ" Phụ lục A agenda chính giờ chỉ dựa `m.rr_rag`. Chi tiết cấu trúc từng PHẦN xem `.claude/agents/agent-culi-tuan/agent-culi-tuan.md`. |

## Báo cáo Vận hành (`scripts/build_van_hanh.js`, output `report-van-hanh/`)

Script riêng, KHÔNG dùng chung target/config với `run_all.js` — 10 mục hệ thống + 3 mục mới ghép
09–10/09/2026 (đánh số thật trong file sinh ra: mục 1/6/7/9 giữ nguyên, thêm **mục 10 "Chất lượng
kỹ thuật (SCSC)"**, **mục 11 "Đối chiếu chéo & điểm mù"** ngay sau mục 9 Feedback, và **mục 12
"Vấn đề vận hành từ nhóm hỗ trợ"** ở cuối — số 4-6 trong tài liệu gốc
`Tool_BC_ChamDiem_VH_v1/GOI_YEUCAU_...md` không khớp số thật vì mục "Giờ đóng/mở cửa" đang tắt bằng
cờ `INCLUDE_GIO_MO_CUA=false`).

| Quy tắc | Chi tiết |
|---------|----------|
| Ghép `Tool_BC_ChamDiem_VH_v1/pull_chamdiem.py` (chốt 09/09/2026) | Kiến trúc đã chọn: `build_van_hanh.js` **tự spawn** `pull_chamdiem.py` bằng `child_process.execSync` (hàm `runPullChamDiem`) trước khi build report, đọc thẳng 9 CSV/JSON nó xuất ra (`readCsv`/`parseCsv` tự viết, không cần thư viện ngoài) — không tách bước thủ công. Chạy bằng venv riêng `Tool_BC_ChamDiem_VH_v1/.venv/bin/python3` (máy Trung Vũ không có `~/.pgpass` account `quannh` của GĐ Khối VH) — đã sửa DSN mặc định trong `pull_chamdiem.py` sang account `vupt_vh` (cùng cluster/2 database `data_warehouse`+`solution_30shine` đã dùng ổn định ở `poolBI`/`poolRO`), override được qua biến môi trường `PGDSN_BI`/`PGDSN_RO` nếu cần. Nếu venv/script thiếu hoặc lỗi (VPN rớt, DB bận) → `chamDiem = null`, mục 1-nội-dung-chấm/10/11 tự bỏ qua có ghi chú rõ, KHÔNG làm hỏng 9 mục còn lại. |
| Mục 1 (GĐKV đi chấm) — nội dung chấm | Bảng 32 tiêu chí (`01_tieuchi_ky.csv`) là số **toàn hệ thống** (không tách theo GĐKV vì `gdkv_email` trống 100%) — phép 4.1 "mảng ổn" + top tiêu chí yếu đặt ở `BC_VanHanh_TongQuan_...md` mục 1, không lặp lại ở từng vùng. Mỗi vùng chỉ thêm: salon nhiều lỗi nhất trong vùng, cờ đỏ an toàn (`03_flag_nghiemtrong.csv`), lặp lại còn tái phạm trong kỳ (`04_flag_laplai.csv`, tách theo `CSVC_HONG_HATANG` cho mục 7 CSVC). |
| Mục 8 (5S, số thứ tự thật trong file do INCLUDE_GIO_MO_CUA bật) — công thức "đã thực hiện" (chốt 22/09/2026, KHÁC report-sla) | `build_van_hanh.js` (report-van-hanh/) đổi hẳn công thức mục 5S sang **% Khung-KV đã THỰC HIỆN** (không còn dùng công thức "đã đăng ký" của Q4B `run_all.js`): `Target(salon) = TongKhuVuc(salon) × 2 × SoNgayCoDangKy(salon)` — `SoNgayCoDangKy` = `COUNT(DISTINCT work_date)` THỰC TẾ salon có đăng ký khung-KV trong kỳ (KHÔNG cố định = số ngày trong kỳ, có salon chỉ đăng ký 4-6/7 ngày); `Đã thực hiện(salon)` = số dòng `bi.area_assignment_salon` (staff_id≠0, is_delete=false, đủ begin/end_time, zone hợp lệ) JOIN `bi.area_assignment_staff` (theo staff_id/area/zone/salon_id/work_date đã rtrim + GROUP BY bool_or(is_completed) tránh fan-out) có is_completed=true — grain ĐẦY ĐỦ cả khung giờ, KHÔNG dedupe theo (area,zone) như "đã đăng ký". Hàm `computeQ5SThucHien()` trong `build_van_hanh.js`. Đã verify khớp chính xác dashboard sản xuất `reports.30shine.com/0002_report5s` tại 2 mốc kỳ 14-20/09/2026: vùng Nguyễn Thanh Sang = 83,62%, vùng Nguyễn Thị Huệ = 56,35% (dò ngược bằng `input/KhuVuc_5S.csv`+`input/NhanVien_5S.csv`). **CHỈ đổi bên `build_van_hanh.js`/report-van-hanh — `run_all.js`/report-sla vẫn giữ nguyên công thức Q4B "đã đăng ký" cũ, đây là 2 KPI đo bản chất khác nhau, không phải cùng 1 số đổi tên.** Đối chiếu tại chỗ: Proxy % Đạt nhóm A1-A6 (23 tiêu chí GĐKV chấm, không tính A7 an toàn) so với % khung-KV đã thực hiện ở trên — 2 chỉ số đo bản chất khác nhau (trạng thái tại 1 thời điểm chấm vs mức thực hiện đúng khung giờ phân công), chỉ nêu cảnh báo khi lệch ≥15 điểm %. |
| Mục 9 (Feedback) — phân loại theo bảng mã | CHỈ bổ sung diễn giải (bảng theo `p_nhom_loi`/`nhom_loi`, tách riêng "6 điều cấm" + GIAN LẬN) — **KHÔNG đổi số nền/RAG chính** (định nghĩa "phản hồi thật" đang có 3 cách lọc ra 3 số khác nhau 42/55/164, CHƯA được CSKH/GĐ Khối VH chốt — mục 6 file GOI_YEUCAU). |
| Mục 10 (mới) "Chất lượng kỹ thuật (SCSC)" | Query `Q_SCSC` — `bi.scsc_bill_loi_detail` JOIN `bi.salon_info`, tỷ lệ bill lỗi (cắt+uốn/nhuộm/duỗi+KCS)/tổng bill chấm theo salon. Ngưỡng lệch chuẩn = tỷ lệ lỗi ≥ 2 lần trung vị TOÀN HỆ THỐNG (`scscMedianAll`, tính 1 lần trước vòng lặp 8 vùng). |
| Mục 11 (mới) "Đối chiếu chéo & điểm mù" | Phép 4.5 (độ phủ từ `05_flag_coverage.csv` + điểm mù từ `08_doichieu_feedback.csv` lọc `duoc_cham_trong_ky==='Không'` và có `ung_vien_mau_thuan`) · 4.6 (3 cặp mâu thuẫn nguồn, viết ở mức "dấu hiệu cần kiểm chứng", KHÔNG kết luận "GĐKV chấm sai") · 4.7 (hội tụ — salon lọt top 25% của ≥2/3 nguồn: số lỗi GĐKV chấm theo `02_salon_ky.csv`, % lỗi SCSC, ticket CSKH/1.000 lượt — tính 1 lần trước vòng lặp bằng `hoiTu`, dùng chung cho từng vùng + Tổng quan). Phép 4.4 (tái phạm sau cam kết, `07_coaching_lichsu.csv`) đối chiếu **theo SALON** (không theo đúng từng vấn đề — text tự do khó match chính xác), đặt ở Tổng quan vì không tách được theo GĐKV. |
| Bug NFC đã gặp lại | `chamDiem.tongQuan.theo_gdkv` (JSON từ Python) trả key GĐKV có thể ở dạng NFD — phải `.normalize('NFC')` cả 2 phía trước khi tra cứu bằng `g.rsm` (biến `soLanChamByGdkv`), giống các bug NFC khác đã gặp trong `salonById`/`vhKeHoach`. |
| Chưa làm — GSC & xử lý feedback qua Lark Approval | Nguồn 6/7 trong GOI_YEUCAU (mục 2.6) — chưa có ETL về DB, cần IT-Admin duyệt scope `approval:approval:readonly` hoặc link Lark Base + thêm app `cli_aab967a878b89ee9` làm cộng tác viên đọc. Báo cáo ghi rõ "chưa có dữ liệu, đang chờ duyệt scope" ở mục 11 — không suy diễn số liệu. |
| Mục 12 (mới) "Vấn đề vận hành từ nhóm hỗ trợ" (chốt 10/09/2026) | KHÔNG có nguồn dữ liệu tự động — GĐ Khối VH tổng hợp thủ công từ 1 group chat hỗ trợ vận hành, gửi lại sau. Script chỉ in sẵn placeholder ở cuối mỗi báo cáo vùng + Tổng quan (`*[Dán nội dung ... vào đây khi có]*`) để có chỗ điền tay khi nội dung được gửi tới — không tự bịa/suy diễn nội dung. |
| Mục 8 "Nhân sự — Định biên, Tuyển và Giữ" gộp toàn bộ nội dung Nhân sự DỜI TỪ `run_all.js` (chốt 14/09/2026) | Đổi tên từ "Nhân sự — Tuyển và Giữ", gộp thêm: (1) **Định biên Stylist/Skinner** — bảng Mục tiêu/Hiện tại/%/RAG theo GĐKV, dùng query `Q_HC` mới (xem bảng Queries dưới) fetch 1 lần toàn hệ thống rồi cộng theo vùng bằng `regionHc(salonIds, dept)`; (2) **Giữ** đổi từ "tham chiếu ngược `PhuLuc_NhanSu`" (`loadPhuLucNhanSu`, đã BỎ) sang bảng "Nhân sự rủi ro nghỉ việc" **EDITABLE ngay trong báo cáo vùng**, carry-forward tự động bằng `loadPrevNhanSuRowsVH(gdkvKey, kyLabel)` (đọc `report-van-hanh/BC_VanHanh_[Tuan/Thang]_[key]_[kỳ trước].md`, tương đương `loadPrevNhanSuRows` cũ của `run_all.js` chỉ đổi nguồn thư mục); (3) **Cần trao đổi trong họp** — bullet không khung giờ (Định biên lệch 🟡🔴, nhân sự nhóm 1 chưa trao đổi, SM vắng/không đi cuối tuần — dùng lại `smAbsentVH`/`smNoWeekendVH` tính sẵn ở mục 2 "Hiện diện SM"), dời từ agenda chính section NHÂN SỰ cũ (đã bỏ khỏi `run_all.js`). `PhuLuc_NhanSu_[KY].md` dời hẳn từ `agenda/` sang `report-van-hanh/`, do `build_van_hanh.js` sinh (`buildPersonnelMd(regionSummaries, riskByGdkv)`) — Phần I Định biên auto + Phần II carry-forward draft, cùng nguồn data với "Giữ" ở trên (không phải đọc ngược 1 file trung gian nào nữa). **MIGRATION 1 lần (14/09/2026):** kỳ 07-13/09/2026 chưa có report-van-hanh kỳ trước để carry-forward — agent đã khôi phục thủ công từ report-me kỳ 17-23/08/2026 (kỳ đầy đủ 8 vùng gần nhất còn Section 2); dữ liệu carry qua đợt này có thể cũ ~3 tuần, đã ghi cảnh báo trong `PhuLuc_NhanSu_07-13_09_2026.md`. Từ kỳ 14-20/09/2026 trở đi carry-forward tự động bình thường. |

### Queries thêm trong `build_van_hanh.js` (ngoài Q1-Q_GIO_MOCUA của `run_all.js`)

| Query | Nguồn | Mục đích |
|-------|-------|----------|
| Q_SCSC | `bi.scsc_bill_loi_detail` + `bi.salon_info` | Mục 10 — tỷ lệ bill lỗi kỹ thuật theo salon |
| Q_CSKH_NHOM | `bi.cskh_feedback_dakenh` + `bi.cskh_phan_loai_nhom_loi` | Mục 9 (bổ sung) + mục 11 — ticket lỗi đã phân loại theo `p_nhom_loi`/`nhom_loi`, loại tag "Đánh giá nhầm/ Tìm đồ" |
| Q_HC (DỜI từ `run_all.js`, chốt 14/09/2026) | `bi.staff_erp_active` | Mục 8 — headcount Stylist/Skinner 2 ngày cuối kỳ theo salon+ngày, fetch 1 lần toàn hệ thống, cộng theo vùng bằng `regionHc()` (JS, không phải SQL riêng mỗi vùng như bản gốc trong `run_all.js`) |
| `computeQ5SThucHien()` (mới, chốt 22/09/2026) | `bi.area_assignment_salon` + `bi.area_assignment_staff` (cross-cluster với `main.area_assignment_region_area`/`main.area_assignment_zone`) | Mục 8 (5S) — % Khung-KV **đã THỰC HIỆN** (không phải "đã đăng ký" như Q4B của `run_all.js`), fetch 1 lần toàn hệ thống rồi lọc theo vùng bằng `salonNameSet`. Xem bảng quy tắc "Mục 8 (5S)" bên dưới để biết chi tiết công thức + 2 mốc verify. |
| `pull_chamdiem.py` (spawn ngoài, không phải SQL text trong JS) | `bi.gdkv_checklist_salon` + `bi.cskh_feedback_dakenh` + `bi.salon_info` + `main.view_bill_service_his` | Nội dung 32 tiêu chí GĐKV chấm salon + 8 file CSV/JSON flag sẵn (tiêu chí/salon/an toàn/lặp lại/coverage/free-text/coaching/đối chiếu feedback) |

## Queries (`run_all.js`)

| Query | Nguồn | Mục đích |
|-------|-------|----------|
| Q1 | `Doanhthu_salon_overview` | DT/LK kỳ này vs kỳ trước, lũy kế tháng |
| Q2 | `Doanhthu_salon_overview` | Chi tiết theo salon |
| Q3 | `Doanhthu_salon_detail` + category | AOV/CR/Bill-size theo SC/UND/Relax/SP |
| Q4A | `AreaAssignmentSalon` | SLA N+1 *(đã bỏ khỏi báo cáo + agenda — query còn nhưng không hiển thị)* |
| Q4B (đổi công thức 22/09/2026; **NGỪNG hiển thị trong báo cáo tuần từ 22/09/2026**) | `main.area_assignment_region_area` + `bi.area_assignment_salon` + `main.area_assignment_zone` | SLA 5S — % Khung-KV **đã đăng ký** so với target lý thuyết (TongPhaiDangKy=TongKhuVuc×2×số ngày; TongDaDangKy=số khung-KV thực tế đăng ký). Hàm JS `computeQ4B` (cross-cluster như Q4C), **thay hẳn** measure cũ (đã thực hiện/đã phân công thực tế) đúng ngày đổi công thức — nhưng CHỈ trong cùng ngày 22/09/2026, Trung Vũ quyết định bỏ hẳn dòng này khỏi output (dễ nhầm với mục 8 "5S — % Khung-KV đã thực hiện" của report-van-hanh, xem bảng dưới). Từ kỳ 14-20/09/2026 trở đi: hàm `computeQ4B`/query vẫn còn trong code (không xoá, tránh phá vỡ cấu trúc) và vẫn được gọi (`d.q4b`/`d.q4b_prev`), nhưng KHÔNG còn in ra `report-sla` (đã xoá dòng bảng "5S — % Khung-KV đã đăng ký") và KHÔNG còn tạo issue Section 4.1 report-me nữa — chỉ số 5S "đã đăng ký" không xuất hiện ở bất kỳ file output nào của `run_all.js`. |
| Q4C | `StaffErpActive` + `Staff` + `FlowTimeKeepingMultiSalon` | SLA định biên ERP/ngày |
| Q4D | `CSKH_Feedback_Dakenh` | Feedback (loại TABLET) |
| Q5A | `Pool_4nhomkhachquen` + `customer_transactions_TagSegment` | 8 nhóm KH |
| Q5B | `Pool_4nhomkhachquen` | Pool 4 nhóm KH quen gán về salon trong tháng (cột "Pool" bảng phễu gọi) |
| Q_POOL_DAILY (đổi nguồn 14/09/2026) | `bi.pool_callday_status_his` | Phễu gọi kéo khách: Phân phối (COUNT(*) dòng log gọi), Đã gọi/Đã xử lý (`status_call IS NOT NULL`), Kết nối (`COUNT(DISTINCT customer_id)` theo 5 trạng thái có phản hồi — xem hằng `KET_NOI_STATUSES_SQL`). Trước 14/09 dùng `bi.pool_count_receiver_daily` (log salon tự gọi) — đã đổi để khớp đúng dashboard nội bộ GĐKV đang xem ("Kết quả gọi kéo khách"), xem [[rsm-pool-goi-keo-khach-doi-nguon]]. |
| Q_POOL_BOOK (`computeQPoolBook`, đổi nguồn 14/09/2026) | `bi.pool_callday_status_his` + `main.view_booking` + `main.view_bill_service_his` | Đã book/Đã đến — lấy tập khách "đã kết nối" (4/5 trạng thái, loại "Sai số/không tồn tại") rồi đối chiếu booking/bill trong đúng khung lũy kế tháng (không còn ràng buộc phải sau lần gọi đầu). Khớp tuyệt đối Phân phối/Đã xử lý/Đến, lệch 1-3 đơn vị ở Kết nối/Đã book so dashboard (chưa có SQL gốc để đối chiếu 100%). |
| Q_HC | `StaffErpActive` | Headcount Stylist/Skinner 2 ngày cuối kỳ |
| Q_SM_CC | `view_StaffCheckinCheckout` | SM hiện diện (credit-day) |
