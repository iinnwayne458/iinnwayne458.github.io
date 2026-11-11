---
layout: post
title:  "Quản gia Telegram"
date:   2026-11-11
categories: telegram-bot, bot
---

# Telegram Workspace Bot

Nhắn tin bình thường (không cần `/`), bot tự hiểu để ghi thu/chi, đặt lịch, tạo việc/note. Thuần rule-based, không dùng AI.

```
Telegram ─webhook─► Cloudflare Worker ─HTTPS + secret─► Google Apps Script ─► Sheets / Calendar / Tasks
                        ▲ Cron mỗi 10 phút (nhắc lịch, nhắc việc, lịch hằng ngày)
```

- **Miễn phí hoàn toàn**: Cloudflare Workers Free + Apps Script.
- **Không cần Google Cloud project, không OAuth**: Apps Script chạy bằng quyền tài khoản của bạn.
- Chỉ **một người dùng**: bot bỏ qua mọi chat_id khác.

## Cài đặt

### 1. Tạo 3 Google Sheet trống
Tạo 3 file riêng (tên tùy ý): **Thu chi**, **Ví**, **Cấu hình**. Mở từng file, copy **ID** trong link:
`https://docs.google.com/spreadsheets/d/<ID>/edit`

### 2. Apps Script
1. Vào [script.google.com](https://script.google.com) → **New project**.
2. Dán nội dung `apps-script/Code.gs` vào `Code.gs`.
3. **Project Settings** ⚙️ → bật **Show "appsscript.json" manifest file** → mở `appsscript.json` và dán nội dung file cùng tên trong repo (bật sẵn dịch vụ Tasks, múi giờ VN, quyền Web App).
4. Ở đầu `Code.gs` điền `SECRET_KEY` (chuỗi bí mật tự đặt) và 3 `SHEET_*_ID`.
5. Chọn hàm **`setup`** → **Run** → cấp quyền (Sheets, Calendar, Tasks). Hàm này tạo tab, tiêu đề, ví "Tiền mặt" và cấu hình mặc định.
6. **Deploy → New deployment → Web app**: *Execute as* = **Me**, *Who has access* = **Anyone** → Deploy → copy **Web app URL** (kết thúc bằng `/exec`).
   > Mỗi lần sửa `Code.gs`: **Deploy → Manage deployments → ✏️ → Version: New version → Deploy**. URL giữ nguyên.

### 3. Bot Telegram
Chat với [@BotFather](https://t.me/BotFather) → `/newbot` → lấy **token**.

### 4. Cloudflare Worker
```bash
cd worker
npm install
```
Mở `wrangler.toml`, điền `GAS_URL` (URL ở bước 2.6). Sau đó đặt 3 secret:
```bash
npx wrangler secret put BOT_TOKEN        # token BotFather
npx wrangler secret put WEBHOOK_SECRET   # chuỗi ngẫu nhiên tự đặt (chữ, số, _ hoặc -)
npx wrangler secret put GAS_SECRET       # trùng SECRET_KEY trong Code.gs
npx wrangler deploy
```
Wrangler in ra link dạng `https://telegram-workspace-bot.<tên>.workers.dev`.

### 5. Gắn webhook
```bash
curl "https://api.telegram.org/bot<BOT_TOKEN>/setWebhook" \
  -d "url=https://telegram-workspace-bot.<tên>.workers.dev/webhook" \
  -d "secret_token=<WEBHOOK_SECRET>"
```

### 6. Khóa bot cho riêng bạn
Nhắn `/id` cho bot → nhận chat id → điền vào `ALLOWED_CHAT_ID` trong `wrangler.toml` → `npx wrangler deploy` lại. Xong, thử nhắn `cà phê 50k:food`.

(Tùy chọn) Menu lệnh: BotFather → `/setcommands`.

## Cách dùng

### 💸 Thu/chi
| Nhắn | Kết quả |
|---|---|
| `cà phê 50k:food` | chi 50.000đ, danh mục `food`, ghi chú "cà phê" |
| `27k nước:food` | chi 27.000đ |
| `33k` | chi, danh mục `khác` |
| `+500k` / `thu 500k bán đồ` | thu |
| `5tr trả nợ -VCB` | chi từ ví `VCB` (không có `-ví` thì dùng ví mặc định) |

Đơn vị: `k, nghìn, ngàn, tr, triệu, t/tỷ/tỉ`, kiểu gộp `1tr5` (1,5tr), `2tr500`, `1.5tr`, `50.000đ`.
Nhiều khoản: mỗi dòng một khoản, hoặc ngăn bằng `, ` (nếu mọi đoạn đều có tiền).
Danh mục là phần sau dấu `:` ở cuối câu, tự tạo, không cần khai báo.

### 📅 Lịch
`mai 9h họp với A` · `thứ 6 tuần sau 14h30-16h khám răng` · `30 phút nữa gọi mẹ` · `lịch 15/11 sinh nhật` · `cuối tuần đi chơi`

| Cụm | Hiểu thành |
|---|---|
| hôm nay, nay · mai · mốt, ngày kia | hôm nay · +1 · +2 ngày |
| thứ 2…7, t2…t7, chủ nhật, cn | ngày sắp tới gần nhất (đúng thứ hôm nay → tuần sau) |
| thứ 6 tuần sau / tuần này | thứ 6 của tuần đó |
| tuần sau / tháng sau | +7 ngày / +1 tháng |
| cuối tuần · đầu tháng · cuối tháng | thứ 7 gần nhất · ngày 1 tháng sau · ngày cuối tháng |
| 15/11 · 15-11-2026 · ngày 15 | ngày cụ thể (không ghi năm: lấy lần sắp tới) |
| 3 ngày nữa · sau 2 tuần · 1 tháng nữa | cộng tương ứng |
| 9h · 9g · 9 giờ · 9:30 · 9h30 · 9 giờ rưỡi · 9h kém 15 | giờ cụ thể |
| 30 phút nữa · sau 2 tiếng · 1 tiếng rưỡi nữa | cộng từ bây giờ |
| 9h-10h30 · 9h đến 10h | giờ bắt đầu và kết thúc |
| sáng / trưa / chiều / tối / đêm (đứng riêng) | 08:00 / 12:00 / 15:00 / 19:00 / 22:00 |

Quy tắc: giờ không ghi buổi → 1–6h là chiều, 7–11h là sáng, 12h là trưa. Không ghi giờ kết thúc → 1 tiếng. Có ngày không giờ → cả ngày. Có giờ không ngày mà đã qua → sang mai. Ngày giờ đã qua → báo lỗi.
Chỉnh giờ mặc định của "sáng/chiều/tối" và các từ điển khác ở đầu `worker/src/parser.js`.

### ✅ Việc / 📝 Note
| Nhắn | Kết quả |
|---|---|
| `nhắc mua sữa mỗi 2 tiếng` | việc, nhắc lặp mỗi 2 tiếng đến khi bấm Xong |
| `nhắc 8h tối mai uống thuốc mỗi ngày` | nhắc lần đầu 20:00 mai, rồi mỗi ngày |
| `nhắc mua sữa` | bot hỏi chu kỳ bằng nút (30 phút / 1 giờ / 3 giờ / 1 ngày / không nhắc) |
| `cần nộp báo cáo trước thứ 6` | việc có hạn (lưu hạn trên Google Tasks) |
| `note mật khẩu wifi là 12345` · `# ý tưởng` | note (lưu list "Notes", không nhắc) |

Từ mở đầu **việc**: `nhắc, task, todo, việc, cần, phải, nhớ`. **Note**: `note, ghi chú, ghi nhớ, nb, #`. **Lịch** (ép buộc): `lịch, hẹn, sự kiện`.
Chu kỳ: `mỗi 30 phút`, `mỗi 2 tiếng`, `mỗi ngày`, `cứ 3 ngày`, `2h/lần`, `hàng ngày`. **Tối thiểu 10 phút.**
Tin nhắc có nút **☑ Xong**. Tick bên app Google Tasks cũng được, tối đa 10 phút sau bot tự ngừng nhắc.

### Khi bot không chắc
Bot **báo lỗi thay vì đoán**. Ví dụ `họp 9h 50k` có cả giờ lẫn tiền → bot yêu cầu thêm `lịch`, `chi` hoặc `nhắc` ở đầu câu.

### ⚙️ Lệnh
| Lệnh | Việc |
|---|---|
| `/thang` `/thang 10` `/thang 10/2025` | thống kê tháng: tổng thu/chi, theo danh mục, theo ví, so tháng trước, top 5 khoản chi |
| `/tuan` `/tuan truoc` · `/nam` `/nam 2025` | thống kê tuần / năm |
| `/xoa` · `/xoa 12` | xóa khoản cuối / theo id |
| `/vi` `/themvi X` `/macdinh X` `/xoavi X` | quản lý ví (tên ví không có khoảng trắng) |
| `/lich` `/lich mai` `/lich 15/11` · `/lichtuan` `/lichtuan sau` | xem lịch |
| `/viec` · `/note` | việc / note chưa xong (bấm nút để tick) |
| `/nhactruoc 15` | nhắc trước sự kiện N phút (10–1440) |
| `/giotoi 23:00` · `/giosang 07:00` | giờ gửi lịch ngày mai / hôm nay |
| `/cai` · `/id` · `/help` | xem cài đặt · chat id · hướng dẫn |

Lịch đã qua giờ kết thúc được **gạch ngang** trong tin lịch trình (Telegram không hiểu `~~…~~` kiểu Markdown thường, nên bot dùng thẻ `<s>` của HTML, hiệu ứng giống hệt).

## Lưu ý giới hạn
- **Độ trễ nhắc tối đa ~10 phút** vì cron chạy mỗi 10 phút (tối đa cho gói Free vẫn thoải mái: 144 lần/ngày).
- Apps Script giới hạn thời gian chạy/ngày và số request; dùng cá nhân thì dư xa.
- Chỉ theo dõi 100 việc đang mở mỗi danh sách để đồng bộ với Google Tasks.
- `mai` đứng sau `với/cho/gặp/anh/chị…` được hiểu là tên người (họp với Mai). Danh sách này ở `NAME_PREV` trong `parser.js`.
- Không dấu: hỗ trợ khớp từ khóa, số tiền, ngày giờ. Riêng `sáng`, `tối`, `đêm` đứng riêng cần có dấu (để khỏi nhầm với "sang", "tôi", "đem"); đi kèm giờ (`8h toi`) thì không dấu vẫn được.
- Đừng commit token/secret lên repo công khai. `wrangler.toml` chỉ chứa `GAS_URL` và `ALLOWED_CHAT_ID`; nếu ngại thì đặt chúng bằng `wrangler secret put` như các secret còn lại.