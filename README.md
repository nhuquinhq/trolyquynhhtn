# Bot giao việc — Telegram + Google Sheet

Tag bot trong box rồi nhắn như bình thường. Bot tách ra *ai làm · việc gì · hạn khi nào*,
ghi vào Google Sheet, rồi tự nhắc cho tới khi việc xong.

```
@trolyquynhhtn Quang đối soát NCC Galaxylink 17h mai
```

```
✅ VH-0042 · QuangLM
📋 đối soát NCC Galaxylink
⏰ 17:00 · 14/08 (Thứ Sáu) · còn 1h30

[✏️ Sửa hạn] [👤 Đổi người] [🗑 Huỷ]
```

Chạy hoàn toàn trên hạ tầng **miễn phí**: Vercel Hobby + Upstash KV + Google Apps Script + GitHub Actions.
Không dùng API trả phí, không có `package.json`, không cài thư viện nào.

---

## Cấu trúc

```
api/_lib.js      Giờ VN · KV · cầu nối Apps Script · bộ đọc tiếng Việt · thẻ việc
api/bot.js       Webhook Telegram — tạo việc, nút bấm, lệnh
api/remind.js    Thang leo thang · checklist định kỳ · báo cáo sáng/tối
api/setup.js     Trang tự kiểm tra & nối webhook  ← mở đầu tiên khi cài
apps-script/Code.gs              Dán vào Google Sheet
.github/workflows/nhac-viec.yml  Quét nhắc mỗi 30 phút
```

---

## Cài đặt

### 1 · Tạo repo và deploy

Đẩy thư mục này lên GitHub, rồi vào [vercel.com](https://vercel.com) → **Add New → Project** → Import repo.
Framework Preset để **Other**, Root Directory `./`, bấm **Deploy**.

### 2 · Tạo kho KV

Vercel → tab **Storage** → **Create Database** → **Upstash for Redis** (gói free) → **Connect to Project**.
Hai biến `KV_REST_API_URL` và `KV_REST_API_TOKEN` sẽ tự xuất hiện.

### 3 · Google Sheet + Apps Script

1. Mở file Sheet quản trị công việc → **Tiện ích mở rộng → Apps Script**
2. Xoá sạch `Code.gs`, dán toàn bộ [`apps-script/Code.gs`](apps-script/Code.gs)
3. Sửa dòng đầu `var SECRET = 'doi-chuoi-nay-di-1234';` thành chuỗi của bạn
4. **Triển khai → Tuỳ chọn triển khai mới → Ứng dụng web**
   - Thực thi với tư cách: **Tôi**
   - Ai có quyền truy cập: **Bất kỳ ai**
5. Copy URL kết thúc bằng `/exec`

Lần đầu Google hỏi cấp quyền — bấm qua cảnh báo "chưa xác minh" (script của chính bạn).

Script tự tạo 3 tab `VIEC` · `CHECKLIST` · `NHAT_KY`, và tự thêm 4 cột
`phong` · `vai_tro` · `bi_danh` · `tg_id` vào tab nhân sự sẵn có. Các tab cũ không bị đụng tới.

### 4 · Lấy chat ID

Thêm bot vào 3 box, nhắn một tin bất kỳ trong mỗi box, rồi mở:

```
https://api.telegram.org/bot<TOKEN>/getUpdates
```

- `"chat":{"id":-100…}` — số **âm** là chat ID của box
- Nhắn riêng cho bot rồi tìm `"type":"private"` — số **dương** là `BOSS_TG_ID` của bạn

### 5 · Biến môi trường Vercel

`Settings → Environment Variables`, thêm xong bấm **Redeploy**.

| Biến | Giá trị |
|---|---|
| `TASKBOT_TOKEN` | token BotFather cấp |
| `TASKBOT_USERNAME` | tên bot, **không có `@`** |
| `TASKBOT_SECRET` | chuỗi tự đặt — dùng cho webhook và `/api/remind` |
| `GS_WEBAPP_URL` | URL `/exec` ở bước 3 |
| `GS_SECRET` | đúng chuỗi `SECRET` trong `Code.gs` |
| `TG_VH` `TG_HR` `TG_KT` | chat ID 3 box |
| `BOSS_TG_ID` | Telegram ID của bạn |
| `DASH_URL` | *(tuỳ chọn)* tên miền production, vd `https://bot.abc.vercel.app` |

### 6 · Mở trang tự kiểm tra

```
https://<domain>/api/setup?key=<TASKBOT_SECRET>
```

Trang này dò xem thiếu biến nào, thử Telegram / Apps Script / KV, soi bảng nhân sự tìm lỗi,
và có nút **Nối webhook ngay** — bấm là xong bước webhook, không phải gõ link tay.

Cứ sửa rồi tải lại trang cho tới khi hiện **✅ Sẵn sàng**.

### 7 · Bật bộ nhắc

`GitHub → Settings → Secrets and variables → Actions → New repository secret`

| | |
|---|---|
| Name | `TASK_REMIND_URL` |
| Secret | `https://<domain>/api/remind?key=<TASKBOT_SECRET>` |

Chạy thử: tab **Actions → Nhac viec → Run workflow → dry = 1** (chỉ liệt kê, không gửi).

### 8 · Mọi người `/start`

Telegram không cho bot nhắn trước cho người chưa mở hội thoại với nó.
Nhờ từng người nhắn `/start` cho bot một lần — bot tự ghi `tg_id` vào Sheet.

Chưa `/start` thì bot vẫn chạy, chỉ nhắc trong box thay vì nhắn riêng.

---

## Bảng nhân sự

| Tên nhân viên | Tele | phong | vai_tro | bi_danh |
|---|---|---|---|---|
| QuangLM | @quangdino | VH | leader | Quang, quang lm, a Quang |
| HaDT | @Thuhaneee | HR | nhanvien | Hà, ha dt, c Hà |

- `phong` — `VH` / `HR` / `KT`, quyết định bot bắn vào box nào
- `vai_tro` — `leader` / `nhanvien`, quyết định leo thang báo cho ai
- `bi_danh` — mọi cách bạn hay gọi người đó, cách nhau bởi dấu phẩy.
  **Không được trùng giữa hai người** — trang `/api/setup` sẽ báo đỏ nếu trùng.

Bot đọc lại bảng mỗi 10 phút → thêm người không cần deploy lại.

---

## Cách dùng

Tag bot rồi nhắn tự nhiên. Bot cần đọc được **ai làm** và **hạn khi nào**.

| Gõ | Hiểu là |
|---|---|
| `17h` · `17:00` · `5h chiều` · `9h sáng` · `8h tối` | giờ trong ngày |
| `mai` · `mốt` · `ngày kia` | ngày tương đối |
| `t6` · `thứ 6` · `cn` · `chủ nhật` | thứ gần nhất |
| `15/08` · `15/8` | ngày cụ thể |
| `cuối tuần` · `đầu tuần` · `tuần sau` · `trong tuần` | mốc tuần |
| `trong ngày` · `cuối ngày` | 18:00 hôm nay |
| `gấp` · `khẩn` | +2 giờ, ưu tiên cao |
| *(không có gì)* | 18:00 hôm nay, kèm cảnh báo |

Ghép được: `17h mai` · `9h sáng t6` · `8h tối 20/8`.

Thiếu người hoặc thiếu hạn → bot hỏi lại bằng nút bấm, không đoán bừa.

**Lệnh:** `/start` đăng ký · `/viec` việc của tôi · `/id` chat ID · `/ping` kiểm tra bot sống

**Nút bấm:** người làm có `✅ Nhận` `🏁 Xong` `⏰ Xin gia hạn` `❓ Vướng`;
người giao có `✏️ Sửa hạn` `👤 Đổi người` `🗑 Huỷ`.

Reply vào một tin cũ rồi tag bot → bot lấy cả nội dung tin đó làm ngữ cảnh.

---

## Checklist định kỳ

Tab `CHECKLIST` được tạo sẵn với 8 dòng mẫu, **tất cả đang tắt**. Sửa lại rồi gõ `x` vào cột `bat`.

| ma | noi_dung | pic | phong | lap_lai | gio_chot | bat |
|---|---|---|---|---|---|---|
| VH01 | Đối soát NCC Galaxylink | QuangLM | VH | `ngaylam` | 17:00 | x |
| KT02 | Sao kê ngân hàng | NinhHT | KT | `hangngay` | 17:30 | x |
| HR02 | Rà hợp đồng sắp hết hạn | HaDT | HR | `hangtuan:t2` | 10:00 | x |
| KT03 | Chốt sổ tháng | NinhHT | KT | `cuoithang` | 17:00 | x |

| `lap_lai` | Sinh việc vào |
|---|---|
| `hangngay` | mọi ngày, kể cả cuối tuần |
| `ngaylam` | Thứ Hai → Thứ Sáu |
| `t2,t4,t6` | đúng những thứ liệt kê |
| `hangtuan:t2` | mỗi tuần một lần |
| `ngay5` · `hangthang:5` | ngày 5 hằng tháng |
| `dauthang` · `cuoithang` | ngày đầu / ngày cuối tháng |

**Mỗi sáng 07:30** bot sinh việc cho các dòng khớp ngày: nhắn riêng cho `pic`, và gửi một tin gộp
vào box của phòng. Từ đó việc đi vào đúng thang nhắc như việc giao tay.

Mỗi dòng chỉ sinh **một lần mỗi ngày** (khoá trong KV) — cron chạy trùng không nhân đôi việc.

Chạy thử ngay: `/api/remind?key=…&sinh=1&dry=1`

---

## Thang nhắc

| Mốc | Bot làm gì | Ai nhận |
|---|---|---|
| Giao + 30′ chưa Nhận | nhắc lại | nhắn riêng người làm |
| Giao + 2h chưa Nhận | báo lên | leader phòng |
| Hạn − 1 ngày | nhắc trước | nhắn riêng |
| Hạn − 2h | nhắc gấp | nhắn riêng |
| Quá hạn | tag đích danh | box của phòng |
| Quá hạn + 2h | báo trễ | leader phòng |
| 08:00 hằng ngày | việc trễ · đến hạn · chưa ai nhận | sếp |
| 20:00 hằng ngày | chốt ngày theo phòng | 3 box |

Mỗi mốc chỉ bắn **một lần** (khoá NX trong KV), nên cron gõ cửa trùng hay trễ đều vô hại.

---

## Ba luật vận hành

Phần này quan trọng hơn code.

1. **Việc không có trong bot là việc không tồn tại.** Sếp phải là người tuân thủ đầu tiên.
2. **Nhận việc trong 30 phút** — kể cả chỉ để bấm một nút.
3. **Trễ thì báo trước hạn.** Báo sau hạn vẫn tính là trễ.

Và: đừng dùng bảng chỉ số để phạt trong 2 tháng đầu. Phạt một lần thì tuần sau không còn ai
ghi việc thật vào bot nữa.

---

## Giới hạn cần biết

- **Một tin = một việc.** Gõ 2 việc trong một tin thì bot chỉ ghi 1.
- Tên gọi lạ chưa có trong `bi_danh` → bot hỏi lại bằng nút, không đoán bừa.
- Nhắc có thể trễ tối đa 30 phút (nhịp quét), GitHub Actions đôi khi nhả job muộn hơn.
- GitHub Actions: ~960 phút/tháng. Trần miễn phí repo private là 2.000 phút; repo public không giới hạn.
- Upstash free: 10.000 lệnh/ngày. Quy mô ~20 người dùng khoảng 2–3 nghìn.

---

## Gỡ rối

| Hiện tượng | Nguyên nhân thường gặp |
|---|---|
| Tag bot mà im lặng | Webhook chưa nối — mở `/api/setup?key=…` bấm **Nối webhook ngay** |
| Bot trả lời nhưng không ghi Sheet | `GS_WEBAPP_URL` sai, hoặc `GS_SECRET` khác `SECRET` trong Code.gs |
| Bot đọc sai người | Bí danh trùng nhau — `/api/setup` báo đỏ chỗ trùng |
| Không nhắn riêng được | Người đó chưa `/start` |
| Không thấy nhắc | Chưa đặt secret `TASK_REMIND_URL`, hoặc workflow chưa bật |
| Việc trễ không báo leader | Phòng đó chưa ai có `vai_tro = leader` |

Xem lỗi chi tiết: Vercel → tab **Logs**, lọc theo `/api/bot` hoặc `/api/remind`.
