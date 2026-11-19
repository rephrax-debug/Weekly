# 🌸 Weekly Salary Budget Planner (เว็บแอปแบ่งเงินเดือนรายสัปดาห์)

เว็บแอปพลิเคชันสำหรับวางแผนแบ่งเงินเดือนรายสัปดาห์เพื่อคุมค่ากิน คำนวณงบประมาณรายวันแบบไดนามิก และช่วยออมเงินอย่างมีวินัย ออกแบบด้วยโทนสีชมพู/ขาว สวยงาม สะอาดตา พร้อมรองรับการใช้งานทั้งบนมือถือ (Mobile-first) และคอมพิวเตอร์

---

## 🚀 โครงสร้างโฟลเดอร์โปรเจกต์ (Project Structure)

```
Project_mook/
├── client/                     # Frontend (React 19 + Vite + Tailwind CSS 4)
│   ├── public/
│   ├── src/
│   │   ├── api/                # Axios instance with JWT interceptor
│   │   │   └── index.js
│   │   ├── components/         # Reusable Components
│   │   │   ├── Navbar.jsx      # Top & Bottom Responsive Navigation
│   │   │   └── QuickExpenseModal.jsx # Popup บันทึกรายจ่ายด่วน
│   │   ├── context/
│   │   │   └── AuthContext.jsx # Global Authentication State
│   │   ├── pages/
│   │   │   ├── AuthPage.jsx    # หน้า Login / Register พร้อมปุ่ม Demo
│   │   │   ├── Dashboard.jsx   # หน้าแรก: งบวันนี้, สัปดาห์นี้, สัปดาห์หน้า
│   │   │   ├── MonthManagement.jsx # บันทึกเงินเดือน, เงินออม, รายจ่ายคงที่
│   │   │   ├── HistoryPage.jsx # กราฟ Recharts, สถิติสะสม, Export CSV/JSON
│   │   │   └── ProfileSettings.jsx # โหมดคำนวณงบ A/B/C, รูปโปรไฟล์, เปลี่ยนรหัส
│   │   ├── utils/
│   │   │   └── formatters.js   # แปลงสตางค์/บาท, วันที่ภาษาไทย พ.ศ.
│   │   ├── App.jsx
│   │   ├── index.css           # Tailwind CSS 4 + Prompt/Kanit font
│   │   └── main.jsx
│   ├── index.html
│   ├── package.json
│   └── vite.config.js
│
├── server/                     # Backend (Node.js + Express + SQLite)
│   ├── data/                   # SQLite database directory (Persistent)
│   │   └── salary_planner.db
│   ├── uploads/                # เก็บรูปโปรไฟล์ผู้ใช้ (Multer)
│   ├── src/
│   │   ├── controllers/
│   │   │   ├── authController.js   # Login, Register, Profile, Settings
│   │   │   └── budgetController.js # Incomes, Expenses, Analysis, Export
│   │   ├── db/
│   │   │   ├── index.js        # SQLite connection & Schema initialization
│   │   │   └── seed.js         # ข้อมูลตั้งต้นสำหรับทดสอบ (Seed data)
│   │   ├── middleware/
│   │   │   ├── auth.js         # JWT Token Verification
│   │   │   └── upload.js       # Multer avatar upload restriction
│   │   ├── routes/
│   │   │   ├── authRoutes.js
│   │   │   └── budgetRoutes.js
│   │   ├── utils/
│   │   │   └── budgetCalculator.js # Core calculation algorithm (สตางค์)
│   │   └── server.js           # Express App Entry Point
│   ├── .env                    # Configured environment variables
│   ├── .env.example
│   └── package.json
└── README.md
```

---

## 🗄️ ฐานข้อมูลและ Schema (SQLite Database Schema)

ระบบจัดเก็บตัวเลขเงินทั้งหมดเป็น **Integer (สตางค์)** เพื่อป้องกันข้อผิดพลาดจาก Floating-point Decimal และเปิดใช้โหมด WAL (Write-Ahead Logging) เพื่อความรวดเร็วและปลอดภัย

```sql
-- 1. ผู้ใช้งาน (Users)
CREATE TABLE IF NOT EXISTS users (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  email TEXT UNIQUE NOT NULL,
  username TEXT NOT NULL,
  password_hash TEXT NOT NULL,
  avatar_url TEXT DEFAULT '',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- 2. ตั้งค่าการคำนวณของผู้ใช้ (User Settings)
CREATE TABLE IF NOT EXISTS user_settings (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id INTEGER UNIQUE NOT NULL,
  default_savings_type TEXT DEFAULT 'amount' CHECK(default_savings_type IN ('amount', 'percent')),
  default_savings_value INTEGER DEFAULT 0,
  week_start_rule TEXT DEFAULT 'income_date' CHECK(week_start_rule IN ('income_date', 'monday', 'sunday')),
  overspend_mode TEXT DEFAULT 'MODE_A' CHECK(overspend_mode IN ('MODE_A', 'MODE_B', 'MODE_C')),
  surplus_mode TEXT DEFAULT 'rollover_next_day' CHECK(surplus_mode IN ('rollover_next_day', 'transfer_savings')),
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

-- 3. แผนงบประมาณประจำเดือน (Month Plans)
CREATE TABLE IF NOT EXISTS month_plans (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id INTEGER NOT NULL,
  month_key TEXT NOT NULL, -- Format 'YYYY-MM'
  savings_type TEXT DEFAULT 'amount' CHECK(savings_type IN ('amount', 'percent')),
  savings_value INTEGER DEFAULT 0,
  actual_savings_saved INTEGER DEFAULT 0,
  fixed_expenses_total INTEGER DEFAULT 0,
  note TEXT DEFAULT '',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  UNIQUE(user_id, month_key),
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

-- 4. เงินเดือน/เงินเข้า (Incomes)
CREATE TABLE IF NOT EXISTS incomes (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  month_plan_id INTEGER NOT NULL,
  user_id INTEGER NOT NULL,
  title TEXT NOT NULL,
  amount INTEGER NOT NULL, -- สตางค์
  received_date TEXT NOT NULL, -- 'YYYY-MM-DD'
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (month_plan_id) REFERENCES month_plans(id) ON DELETE CASCADE,
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

-- 5. รายจ่ายคงที่ (Fixed Expenses: ค่าหอ, ค่าเน็ต)
CREATE TABLE IF NOT EXISTS fixed_expenses (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  month_plan_id INTEGER NOT NULL,
  user_id INTEGER NOT NULL,
  title TEXT NOT NULL,
  amount INTEGER NOT NULL, -- สตางค์
  due_date TEXT DEFAULT '',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (month_plan_id) REFERENCES month_plans(id) ON DELETE CASCADE,
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

-- 6. รายการใช้จ่ายค่ากินรายวัน (Daily Expenses)
CREATE TABLE IF NOT EXISTS expenses (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id INTEGER NOT NULL,
  month_plan_id INTEGER NOT NULL,
  amount INTEGER NOT NULL, -- สตางค์
  expense_date TEXT NOT NULL, -- 'YYYY-MM-DD'
  category TEXT NOT NULL CHECK(category IN ('breakfast', 'lunch', 'dinner', 'snack', 'drink', 'other')),
  note TEXT DEFAULT '',
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
  FOREIGN KEY (month_plan_id) REFERENCES month_plans(id) ON DELETE CASCADE
);
```

---

## 🧮 สูตรคำนวณและตัวอย่างตัวเลขจริง

### 1. สูตรคำนวณหลัก
$$งบกินรวม (Food\ Budget) = เงินเดือนเข้าทั้งหมด - เป้าหมายเงินออม - รายจ่ายคงที่$$
$$งบกินเฉลี่ยต่อวัน (Base\ Daily\ Budget) = \frac{งบกินรวม}{จำนวนวันที่เหลือจริงในรอบเดือน}$$

### 2. ตัวอย่างสถานการณ์จริง (ตามโจทย์)
- **เดือน:** พฤศจิกายน 2026 (มีทั้งหมด 30 วัน)
- **เงินเดือนเข้าจริง:** 15,000 บาท ใน **วันที่ 3 พฤศจิกายน** (เงินเข้าช้ากว่าวันที่ 1)
- **เป้าหมายเงินออม:** 3,000 บาท
- **รายจ่ายคงที่:** 0 บาท (หรือถ้ามี เช่น ค่าเน็ต 400 บาท จะถูกหักออก)
- **จำนวนวันจริงที่เหลือในรอบ:** นับตั้งแต่วันที่ 3 ถึง 30 พ.ย. = **28 วัน**

### 3. ผลลัพธ์การคำนวณ
1. **งบกินรวม:** 15,000 - 3,000 = **12,000.00 บาท**
2. **งบกินต่อวัน:** $\frac{12,000}{28} \approx$ **428.57 บาท/วัน**
3. **การแบ่งสัปดาห์ (4 สัปดาห์):**
   - **สัปดาห์ที่ 1:** วันที่ 3 พ.ย. – 9 พ.ย. (7 วัน) $\rightarrow$ งบ 7 × 428.57 = **2,999.99 บาท**
   - **สัปดาห์ที่ 2:** วันที่ 10 พ.ย. – 16 พ.ย. (7 วัน) $\rightarrow$ งบ 7 × 428.57 = **2,999.99 บาท**
   - **สัปดาห์ที่ 3:** วันที่ 17 พ.ย. – 23 พ.ย. (7 วัน) $\rightarrow$ งบ 7 × 428.57 = **2,999.99 บาท**
   - **สัปดาห์ที่ 4:** วันที่ 24 พ.ย. – 30 พ.ย. (7 วัน) $\rightarrow$ งบ 7 × 428.57 = **2,999.99 บาท**
   *(กรณีเดือนที่มี 31 วัน สัปดาห์ที่ 5 จะมี 1 วัน และรับงบ 428.57 บาทโดยอัตโนมัติ)*

### 4. พฤติกรรมเมื่อใช้เกินงบ (Overspend Logic)
- **โหมด A (แนะนำ):** ดึงส่วนที่เกินเฉลี่ยหักจากวันที่เหลือในสัปดาห์นั้น
  - *เช่น:* วันนี้งบ 428 บาท แต่กินไป 628 บาท (เกิน 200 บาท) และเหลืออีก 4 วันในสัปดาห์นี้
  - ระบบจะลดงบของ 4 วันที่เหลือลงวันละ $\frac{200}{4} = 50$ บาท ทำให้ 4 วันถัดไปเหลือวันละ 378 บาท
- **โหมด B:** ยกยอดไปหักสัปดาห์ถัดไป (สัปดาห์นี้งบวันอื่นไม่ลด แต่สัปดาห์หน้าจะเริ่มต้นด้วยงบที่น้อยลง)
- **โหมด C:** ดึงจากเงินออมมาชดเชย (งบกินไม่กระทบ แต่ยอดเงินออมสุทธิจะลดลง พร้อมแสดงแถบเตือนสีแดง)

---

## 📦 วิธีการติดตั้งและรันโปรเจกต์ (Step-by-Step Installation)

### 1. ติดตั้ง Dependencies
เปิด Terminal ในโฟลเดอร์หลัก:

```bash
# ฝั่ง Server
cd server
npm install

# รัน Seed ข้อมูลตัวอย่างเริ่มต้น
npm run seed

# ฝั่ง Client
cd ../client
npm install
```

### 2. รันระบบ (Run Development Servers)
```bash
# Terminal 1: รัน Backend Server (Port 5000)
cd server
npm start
# หรือ npm run dev (hot-reload)

# Terminal 2: รัน Frontend Web Client (Port 5173)
cd client
npm run dev
```

เปิดเบราว์เซอร์ไปที่: **http://localhost:5173**

---

## 🔑 ข้อมูลสำหรับทดสอบ (Seed Demo Account)
ระบบได้สร้างบัญชีตัวอย่างไว้พร้อมทดสอบได้ทันที:
- **อีเมล:** `mook@example.com`
- **รหัสผ่าน:** `password123`
*(หรือคลิกปุ่ม **"ทดลองใช้ด้วยบัญชีสาธิต"** บนหน้าเข้าสู่ระบบได้เลยทันที)*
