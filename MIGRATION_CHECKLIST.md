# 📋 MIGRATION CHECKLIST
## Database & Dependencies Update Guide

**วันที่:** 8 กุมภาพันธ์ 2026  
**Version:** 2.0  

---

## ✅ สิ่งที่แก้ไขแล้ว

### 1. ✅ SQL Schema (`Table_v2.sql`)
- ✅ แก้ comment ของ `password` column ให้ตรงกับการใช้งาน
- ✅ ลบตาราง `user_template_access` ที่ไม่ได้ใช้
- ✅ เพิ่ม indexes สำหรับ performance
- ✅ เพิ่ม trigger สำหรับ `updated_at`
- ✅ ปรับปรุง RLS policies ให้เข้มงวดขึ้น
- ✅ เพิ่ม CHECK constraint สำหรับ role
- ✅ เพิ่ม view สำหรับ reporting
- ✅ เพิ่ม comments documentation

### 2. ✅ Backend Dependencies (`requirements.txt`)
- ✅ Pin versions ทุกตัว (ป้องกัน breaking changes)
- ✅ อัพเดทเป็น latest stable versions (Feb 2026)
- ✅ เพิ่ม bcrypt version explicit
- ✅ ใช้ uvicorn[standard] สำหรับ performance

### 3. ✅ Frontend Dependencies
- ✅ ทุก package ใช้ latest versions แล้ว
- ✅ แนะนำลบ dependencies ที่ไม่ใช้ (axios, react-selecto)

---

## 🔄 การ Migrate Database

### Option 1: Database ใหม่ (แนะนำ)

```bash
# 1. Backup database เก่า (ถ้ามี)
# ใน Supabase SQL Editor:
CREATE TABLE users_backup AS SELECT * FROM users;
CREATE TABLE templates_backup AS SELECT * FROM templates;

# 2. รัน Table_v2.sql ใน Supabase SQL Editor
# คัดลอกทั้งไฟล์แล้ว Execute

# 3. รัน migration script
cd backend
python migrate_passwords.py
```

### Option 2: Database มีข้อมูลอยู่แล้ว

```sql
-- 1. เพิ่ม indexes
CREATE INDEX IF NOT EXISTS idx_template_slots_template_id ON template_slots(template_id);
CREATE INDEX IF NOT EXISTS idx_lotteries_template_id ON lotteries(template_id);
CREATE INDEX IF NOT EXISTS idx_users_assigned_template_id ON users(assigned_template_id);
CREATE INDEX IF NOT EXISTS idx_users_username ON users(username);
CREATE INDEX IF NOT EXISTS idx_users_role ON users(role);

-- 2. เพิ่ม CHECK constraint
ALTER TABLE users 
ADD CONSTRAINT check_user_role 
CHECK (role IN ('admin', 'member'));

-- 3. เพิ่ม trigger function
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

-- 4. เพิ่ม trigger
DROP TRIGGER IF EXISTS update_templates_updated_at ON templates;
CREATE TRIGGER update_templates_updated_at 
BEFORE UPDATE ON templates
FOR EACH ROW
EXECUTE FUNCTION update_updated_at_column();

-- 5. ลบตาราง user_template_access (ถ้าไม่ใช้)
DROP TABLE IF EXISTS user_template_access CASCADE;

-- 6. อัพเดท RLS policies (ดูใน Table_v2.sql)
```

---

## 📦 การอัพเดท Backend Dependencies

```bash
cd backend

# 1. Backup requirements เก่า
cp requirements.txt requirements.txt.old

# 2. ไฟล์ requirements.txt ถูกแก้แล้ว - ติดตั้ง
pip install --upgrade pip
pip install -r requirements.txt

# 3. ทดสอบว่า import ได้ทั้งหมด
python -c "import fastapi; print(fastapi.__version__)"
python -c "import supabase; print('Supabase OK')"
python -c "import passlib; print('Passlib OK')"

# 4. เก็บ exact versions
pip freeze > requirements-lock.txt

# 5. รัน migration script
python migrate_passwords.py

# 6. ทดสอบ server
uvicorn main:app --reload
```

---

## 🧪 การทดสอบหลัง Migrate

### 1. ทดสอบ Backend

```bash
# Health check
curl http://localhost:8080/health

# Test login
curl -X POST http://localhost:8080/api/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"1234"}'

# Test generate
curl -X POST http://localhost:8080/api/generate \
  -H "Content-Type: application/json" \
  -d '{
    "template_id": "your-template-id",
    "user_seed": "123",
    "slot_configs": []
  }'
```

### 2. ทดสอบ Frontend

```bash
cd frontend

# ถ้าต้องการลบ dependencies ที่ไม่ใช้
npm uninstall axios react-selecto

# ติดตั้ง dependencies
npm install

# รัน dev server
npm run dev

# ทดสอบ build
npm run build
```

### 3. ทดสอบ Database

```sql
-- ตรวจสอบ indexes
SELECT indexname, tablename 
FROM pg_indexes 
WHERE schemaname = 'public'
ORDER BY tablename, indexname;

-- ตรวจสอบ triggers
SELECT trigger_name, event_manipulation, event_object_table
FROM information_schema.triggers
WHERE trigger_schema = 'public';

-- ตรวจสอบ RLS policies
SELECT schemaname, tablename, policyname
FROM pg_policies
WHERE schemaname = 'public';

-- ทดสอบ view
SELECT * FROM template_usage;
```

---

## ⚠️ Breaking Changes & Warnings

### 1. **Tailwind CSS 4 Browser Support**
- ❌ ไม่รองรับ Safari < 16.4
- ❌ ไม่รองรับ Chrome < 111
- ❌ ไม่รองรับ Firefox < 128

**แก้ไข:** ถ้าต้องการรองรับ browser เก่า ให้ใช้ Tailwind v3:
```bash
npm install tailwindcss@3
```

### 2. **Pydantic V2 Breaking Changes**
- ⚠️ บาง API เปลี่ยนจาก V1
- ⚠️ ต้องใช้ `.model_dump()` แทน `.dict()`

**แก้ไข:** โค้ดปัจจุบันใช้ Pydantic ถูกต้องแล้ว ไม่ต้องแก้

### 3. **React Router 7**
- ⚠️ Data loading APIs เปลี่ยนจาก v6

**แก้ไข:** โค้ดปัจจุบันยังใช้แบบเดิมได้ (backward compatible)

---

## 🐛 Troubleshooting

### Error: "module 'bcrypt' has no attribute..."
```bash
pip uninstall bcrypt -y
pip install bcrypt==4.2.2
```

### Error: "Could not find a version that satisfies..."
```bash
pip install --upgrade pip setuptools wheel
pip install -r requirements.txt
```

### Error: Database migration failed
```bash
# Rollback to backup
# ใน Supabase SQL Editor:
DROP TABLE users CASCADE;
CREATE TABLE users AS SELECT * FROM users_backup;
```

### Frontend build error
```bash
rm -rf node_modules package-lock.json
npm install
npm run build
```

---

## 📊 Compatibility Matrix

| Component | Old | New | Compatible |
|-----------|-----|-----|------------|
| **FastAPI** | Any | 0.128.2 | ✅ Yes |
| **Supabase** | Any | 2.10.0 | ✅ Yes |
| **React** | 19.2.0 | 19.2.0 | ✅ Already latest |
| **Tailwind** | 4.1.18 | 4.1.18 | ✅ Already latest |
| **Python** | 3.9+ | 3.11+ | ✅ Upgrade recommended |
| **Node** | 18+ | 20+ | ✅ Compatible |
| **PostgreSQL** | 12+ | 14+ | ✅ Compatible |

---

## ✅ Post-Migration Checklist

- [ ] Database backup สำเร็จ
- [ ] รัน `Table_v2.sql` หรือ migration SQL
- [ ] อัพเดท `requirements.txt` และติดตั้ง
- [ ] รัน `migrate_passwords.py`
- [ ] ทดสอบ backend endpoints
- [ ] อัพเดท frontend dependencies (ถ้าต้องการ)
- [ ] ทดสอบ frontend build
- [ ] ทดสอบ login
- [ ] ทดสอบ generate lottery
- [ ] ทดสอบ batch generation
- [ ] ทดสอบ admin features
- [ ] ตรวจสอบ indexes ใน database
- [ ] ตรวจสอบ RLS policies
- [ ] Deploy to production
- [ ] Smoke test ใน production

---

**Status:** ✅ Ready for Migration  
**Estimated Time:** 30-60 minutes  
**Risk Level:** 🟡 Medium (ต้อง backup ก่อน)

---

*Happy Migrating! 🚀*
