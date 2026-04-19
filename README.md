# LSTro - Hệ Thống Quản Lý Khai Báo Lưu Trú Tâm An

**LSTro** là hệ sinh thái phần mềm Full-stack phục vụ việc quản lý cư dân và tự động hóa khai báo lưu trú lên Cổng Dịch vụ công Quốc gia. Hệ thống gồm hai thành phần chính: **Backend API Server** và **LSTro-Bridge** (công cụ tự động hóa).

---

## 🏗 Kiến trúc Tổng quan

```
┌─────────────────────────────────────────────────────────┐
│                     Backend Server                      │
│   Express + Prisma + MySQL                              │
│   ├── API REST (/api/cu-dan, /api/chi-nhanh, /api/ai)  │
│   ├── Gemini AI Proxy                                   │
│   └── Dashboard UI tĩnh (public/)                      │
└─────────────────────────┬───────────────────────────────┘
                          │ HTTP API
┌─────────────────────────▼───────────────────────────────┐
│                   LSTro-Bridge                          │
│   Electron App + CLI Mode                               │
│   ├── Tải kịch bản automation từ Backend               │
│   ├── Điều khiển Chrome qua CDP (port 9222)            │
│   └── Tự động điền form trên Cổng Dịch vụ công        │
└─────────────────────────────────────────────────────────┘
```

---

## 📂 Cấu trúc Dự án

```
TA_system/
├── backend/                    # API Server trung tâm
│   ├── src/
│   │   ├── app.js              # Cấu hình Express, đăng ký routes
│   │   ├── controllers/
│   │   │   ├── cuDanController.js      # Logic quản lý cư dân
│   │   │   ├── chiNhanhController.js   # Logic quản lý chi nhánh
│   │   │   └── aiController.js         # Proxy Gemini AI API
│   │   ├── routes/
│   │   │   ├── cuDanRoutes.js
│   │   │   ├── chiNhanhRoutes.js
│   │   │   └── aiRoutes.js
│   │   └── utils/
│   │       └── prisma.js       # Prisma client instance
│   ├── prisma/
│   │   └── schema.prisma       # Database schema (MySQL)
│   ├── public/                 # Dashboard UI tĩnh (HTML/JS/CSS)
│   │   ├── index.html          # Giao diện quản lý chính
│   │   ├── app.js              # Logic dashboard
│   │   ├── style.css
│   │   └── js/
│   │       ├── api.js          # Gọi API backend
│   │       └── ui.js           # Xử lý giao diện
│   ├── index.js                # Entry point, khởi động server
│   └── .env                    # Biến môi trường (DATABASE_URL, GEMINI_API_KEY)
│
└── LSTro-Bridge/               # Công cụ tự động hóa
    ├── main.js                 # Electron app entry point
    ├── cli.js                  # CLI mode (headless, không cần GUI)
    ├── automation_profile/     # Chrome user profile riêng biệt
    └── dist/                   # Bản build (LSTro-Bridge.exe)
```

---

## 🗄 Database Schema

Hai model chính trong MySQL (quản lý qua Prisma):

### `CuDan` (Cư dân)
| Trường | Kiểu | Mô tả |
|---|---|---|
| `id` | Int | Khóa chính, tự tăng |
| `ho_ten` | String | Họ và tên |
| `cccd` | String (unique) | Số CCCD/CMND |
| `ngay_sinh` | String | Ngày sinh |
| `gioi_tinh` | Boolean | Giới tính |
| `quoc_tich` | String | Quốc tịch |
| `dan_toc` | String | Dân tộc |
| `so_phong` | String | Số phòng |
| `xa`, `tinh` | String | Địa chỉ hành chính |
| `ly_do` | String | Lý do lưu trú |
| `dia_chi_chi_tiet` | String | Địa chỉ chi tiết |
| `nghe_nghiep` | String | Nghề nghiệp (mặc định: "Tự do") |
| `noi_lam_viec` | String | Nơi làm việc |
| `quoc_gia` | String | Quốc gia (mặc định: "Việt Nam") |
| `trang_thai` | Int | `1` = đang ở, `0` = đã trả phòng |
| `chiNhanhId` | Int? | Liên kết chi nhánh |

### `ChiNhanh` (Chi nhánh)
| Trường | Kiểu | Mô tả |
|---|---|---|
| `id` | Int | Khóa chính |
| `ten_chi_nhanh` | String | Tên chi nhánh |
| `trangthai` | Int | `1` = hoạt động, `0` = đã xóa |

---

## 🔌 API Endpoints

### Cư dân – `/api/cu-dan`
| Method | Endpoint | Chức năng |
|---|---|---|
| `GET` | `/` | Lấy danh sách cư dân đang ở (`trang_thai: 1`) |
| `GET` | `/archive` | Lấy danh sách cư dân đã trả phòng (`trang_thai: 0`) |
| `GET` | `/:id` | Lấy chi tiết một cư dân |
| `POST` | `/` | Tạo cư dân mới |
| `POST` | `/tra-phong` | Trả phòng (tìm theo CCCD, đổi `trang_thai: 0`) |
| `POST` | `/:id/restore` | Khôi phục cư dân đã trả phòng |
| `POST` | `/so-sanh` | So sánh DB nội bộ với dữ liệu cổng Chính phủ |
| `PUT` | `/:id` | Cập nhật thông tin cư dân |
| `DELETE` | `/:id` | Xóa mềm cư dân (`trang_thai: 0`) |

### Chi nhánh – `/api/chi-nhanh`
| Method | Endpoint | Chức năng |
|---|---|---|
| `GET` | `/` | Lấy danh sách chi nhánh (kèm số cư dân đang ở) |
| `POST` | `/` | Tạo chi nhánh mới |
| `DELETE` | `/:id` | Xóa mềm chi nhánh |

### AI – `/api/ai`
| Method | Endpoint | Chức năng |
|---|---|---|
| `POST` | `/chat` | Proxy gọi Google Gemini API |

---

## 🤖 LSTro-Bridge – Tự Động Hóa

Bridge điều khiển Chrome qua **Chrome DevTools Protocol (CDP)** để tự động điền và nộp form trên Cổng Dịch vụ công.

### Cơ chế "Zero-Logic Bridge"
- Khi khởi động, Bridge **tải kịch bản automation từ xa** tại `http://localhost:25412/js/automation.js`
- Toàn bộ logic nghiệp vụ nằm trong file `automation.js` phía Backend, **không được đóng gói trong Bridge**
- Cập nhật kịch bản chỉ cần sửa file trên server, không cần build lại Bridge

### Hai chế độ chạy

| Chế độ | Lệnh | Mô tả |
|---|---|---|
| **Electron GUI** | `npm start` | Mở cửa sổ app, load Dashboard từ `http://localhost:25412` |
| **CLI (headless)** | `npm run cli` | Chạy ngầm trong terminal, tự retry khi mất kết nối |
| **Build EXE** | `npm run build:cli` | Đóng gói thành file `.exe` standalone |

### Chrome Debugging
Bridge tự động mở Chrome tại:
- **Remote Debugging Port:** `9222`
- **User Profile:** `./automation_profile/` (tách biệt với Chrome cá nhân)

---

## 🚀 Hướng Dẫn Cài Đặt & Khởi Chạy

### 1. Backend API Server

**Yêu cầu:** Node.js 18+, MySQL

```bash
cd backend

# Cài dependencies
npm install

# Cấu hình môi trường
# Tạo file .env với nội dung:
# DATABASE_URL="mysql://user:password@host:port/dbname"
# GEMINI_API_KEY="your_gemini_api_key"
# GEMINI_MODEL="gemini-2.0-flash"
# PORT=25412

# Tạo database schema
npx prisma migrate dev

# Chạy môi trường dev (có hot-reload)
npm run dev

# Chạy production
npm start
```

Server sẽ chạy tại `http://localhost:25412`  
Dashboard UI sẽ có tại `http://localhost:25412/index.html`

---

### 2. LSTro-Bridge

**Yêu cầu:** Google Chrome đã cài đặt tại `C:\Program Files\Google\Chrome\Application\chrome.exe`

```bash
cd LSTro-Bridge

# Cài dependencies
npm install

# Chạy GUI (Electron)
npm start

# Chạy CLI (headless, dùng cho server/automation)
npm run cli

# Build thành .exe độc lập
npm run build:cli
```

> **Lưu ý:** Backend phải đang chạy trước khi khởi động Bridge, vì Bridge cần tải `automation.js` từ `http://localhost:25412/js/automation.js`.

---

## 🛠 Quy Trình Bảo Trì

| Việc cần làm | Cách thực hiện |
|---|---|
| Sửa logic khai báo lưu trú | Chỉnh `backend/public/js/automation.js` – tất cả Bridge sẽ nhận ngay khi reload |
| Sửa giao diện Dashboard | Chỉnh `backend/public/index.html`, `style.css`, `js/ui.js` |
| Thêm API mới | Tạo route trong `backend/src/routes/` và controller trong `backend/src/controllers/` |
| Thay đổi database | Sửa `backend/prisma/schema.prisma` rồi chạy `npx prisma migrate dev` |
| Sửa cửa sổ ứng dụng Electron | Chỉnh `LSTro-Bridge/main.js` |

---

## ⚙️ Tech Stack

| Thành phần | Công nghệ |
|---|---|
| Backend Framework | Express.js v5 |
| ORM | Prisma v6 |
| Database | MySQL |
| AI Integration | Google Gemini API (proxy qua backend) |
| Automation | Playwright + playwright-stealth |
| Desktop App | Electron v40 |
| Dashboard UI | HTML / Vanilla JS / CSS |

---

*Phát triển bởi Phu Pham – Tâm An System.*
