# CPL Loyalty CRM

ระบบจัดการลูกค้าสะสมแต้มของ CPL — เว็บไซต์ static เดียว (`index.html`) เชื่อมต่อฐานข้อมูล [Supabase](https://supabase.com) โดยตรง ใช้งานได้นอก Claude ไม่ต้องมี backend แยก

## การตั้งค่า

ไฟล์ `index.html` ตั้งค่าเชื่อมต่อ Supabase ไว้แล้ว (ดูค่าคงที่ที่ด้านบนของ `<script>`):

```js
const SUPABASE_URL = 'https://kcfdakdhswvcjunnqvts.supabase.co';
const SUPABASE_KEY = 'sb_publishable_...'; // publishable (anon) key — ปลอดภัยที่จะฝังในหน้าเว็บ
```

ฐานข้อมูลคือโปรเจกต์ Supabase ชื่อ **"บัตรสะสมแต้ม"** ตาราง `loyalty_transactions` ที่มีคอลัมน์:

| คอลัมน์ | ชนิด | ความหมาย |
|---|---|---|
| `id` | uuid | รหัสรายการ (สร้างอัตโนมัติ) |
| `customer_name` | text | ชื่อลูกค้า (ออนไลน์) |
| `user_id` | text | ID ผู้ใช้ของลูกค้า |
| `full_name` | text | ชื่อ-นามสกุลจริง |
| `points` | integer | แต้ม (ติดลบได้เพื่อหักแต้ม) |
| `type` | text | ประเภทแต้ม เช่น "โบนัสเมื่อทำบัตร", "ส่งสินค้า" |
| `tx_date` | date | วันที่ทำรายการ |
| `created_at` / `updated_at` | timestamptz | ประทับเวลา |

## ⚠️ ความปลอดภัย — อ่านก่อนใช้งานจริง

เวอร์ชันนี้คือ **MVP สำหรับเริ่มต้นใช้งานเร็ว** ยังไม่มีระบบ authentication จริง:

1. **"รหัสพนักงาน" (`STAFF_PIN`) ไม่ใช่การยืนยันตัวตนที่ปลอดภัย** — เป็นแค่การซ่อนหน้าจอจากคนทั่วไป รหัสอยู่ในโค้ดฝั่ง client ใครก็ตามที่เปิด view-source เห็นได้ ควรเปลี่ยนเป็นค่าของตัวเองอย่างน้อย และพิจารณาอัปเกรดเป็น Supabase Auth (ดูด้านล่าง)
2. **ตาราง `loyalty_transactions` เปิดให้ทุกคนที่มี publishable key เขียนได้ (`public write` policy)** เพราะยังไม่มีระบบ login จริง ถ้าต้องการความปลอดภัยสูงขึ้น ให้ตั้ง Supabase Auth แล้วแก้ RLS policy ของตารางให้ตรวจสอบ `auth.uid()` แทน `using (true)`

### อัปเกรดเป็น Supabase Auth (แนะนำก่อนใช้งานจริงกับข้อมูลลูกค้าจำนวนมาก)

1. เปิด Supabase Dashboard → Authentication → เพิ่มผู้ใช้พนักงานแต่ละคน (อีเมล/รหัสผ่าน)
2. แก้ RLS policy ของ `loyalty_transactions` จาก `using (true)` เป็น `using (auth.role() = 'authenticated')`
3. แทนที่หน้า login ง่ายๆ ในไฟล์นี้ด้วย `supa.auth.signInWithPassword({email, password})`

## Deploy ขึ้นเว็บจริง

ไฟล์นี้เป็น static site ไฟล์เดียว deploy ได้กับผู้ให้บริการฟรีเกือบทุกเจ้า:

### Vercel (แนะนำ ง่ายสุด)
```
npx vercel --prod
```
หรือเชื่อม repo นี้กับ Vercel ผ่านหน้าเว็บ vercel.com แล้วมันจะ deploy อัตโนมัติทุกครั้งที่ push

### Netlify
ลาก-วางโฟลเดอร์นี้ที่ app.netlify.com/drop หรือเชื่อม repo เดียวกัน

### GitHub Pages
Settings → Pages → Deploy from branch → เลือก `main` / root

จากนั้นผูกโดเมนของบริษัท (เช่น `crm.cplinter.com`) เข้ากับผู้ให้บริการที่เลือกได้ในขั้นตอน custom domain ของแต่ละเจ้า

## พัฒนาเพิ่มเติม

เปิด `index.html` ในเบราว์เซอร์ได้ตรงๆ ไม่ต้องมี build step ใดๆ (vanilla HTML/CSS/JS + Supabase JS client ผ่าน CDN)


## การเข้าสู่ระบบและผู้ใช้งาน

- เข้าสู่ระบบด้วยอีเมล + รหัสผ่านผ่าน **Supabase Auth** (เลิกใช้รหัส PIN ร่วมแล้ว)
- ตาราง `app_users` เก็บสิทธิ์ของแต่ละคน: `admin` (จัดการผู้ใช้ได้) หรือ `staff`
- ผู้ดูแลระบบเพิ่ม/ลบผู้ใช้ เปลี่ยนสิทธิ์ และเปลี่ยนรหัสผ่านได้จากเมนู **ผู้ใช้งาน** (เรียกฟังก์ชัน `admin_create_user`, `admin_set_role`, `admin_set_password`, `admin_delete_user` ซึ่งตรวจสิทธิ์ admin ในฐานข้อมูล)
- ตาราง `loyalty_transactions` อ่าน/เขียนได้เฉพาะผู้ที่ล็อกอินแล้ว (RLS role `authenticated`)
