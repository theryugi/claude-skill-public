# SKILL: AS400/RPGLE — ยกเลิกใบเสร็จไม่ได้ (C01R116 / ERR21)

```yaml
name: cancel-receipt-err21
description: >
  วินิจฉัยและแก้ปัญหายกเลิกใบเสร็จ / temporary receipt ไม่ได้บนระบบ AS400/RPGLE
  หน้าจอ C01R116 (Cancel Temp. & Received Collection) ขึ้น ERR21
  สาเหตุหลัก: GTABLE ขาด config CKLAS ทำให้โปรแกรมหา latest date ได้เป็น 0
triggers:
  - C01R116
  - Cancel Temp. & Received Collection
  - Cancel Receipt
  - ยกเลิกใบเสร็จไม่ได้
  - ยกเลิก temporary receipt ไม่ได้
  - ERR21
  - โปรแกรมมองว่าไม่ใช่ใบล่าสุด
  - latest date เป็น 0
  - วันที่ที่โปรแกรมหาได้เป็น 0
  - GTABLE
  - CKLAS
  - parameter ขาด
```

---

## รูปแบบการตอบเมื่อเจอเคสนี้

เมื่อผู้ใช้รายงานอาการที่ตรงกับ trigger ข้างต้น ให้ตอบตาม structure ต่อไปนี้เสมอ:

---

### 1. สรุปอาการ

> ระบบขึ้น **ERR21** ตอนพยายามยกเลิกใบเสร็จ หรือ temporary receipt บนหน้าจอ **C01R116 (Cancel Temp. & Received Collection)** และไม่สามารถดำเนินการต่อได้

---

### 2. อธิบายสาเหตุ

โปรแกรมจะตรวจสอบว่าใบเสร็จที่กำลังจะยกเลิก **เป็นใบล่าสุด (latest)** หรือไม่ก่อนอนุญาตให้ยกเลิก

กระบวนการที่โปรแกรมทำ:
1. ดึง **latest date** ของใบเสร็จจาก config ในตาราง `GTABLE` โดยใช้ `TDTYPE = 'CKLAS'`
2. นำวันที่ที่ได้ไปเทียบกับวันที่ของใบเสร็จที่ต้องการยกเลิก
3. ถ้าตรงกัน → อนุญาตให้ยกเลิก
4. ถ้าไม่ตรงกัน → ขึ้น **ERR21** และหยุดการทำงาน

---

### 3. Root Cause — วันที่ที่โปรแกรมหาได้เป็น 0

> **โปรแกรมหา latest date ได้เป็น `0` เพราะไม่พบ `TDTYPE = 'CKLAS'` ใน `GTABLE`**
> เมื่อเอา `0` ไปเทียบกับวันที่ใบเสร็จจริง ค่าไม่ตรงกัน โปรแกรมจึงมองว่าใบเสร็จนี้ **ไม่ใช่ใบล่าสุด** และปฏิเสธการยกเลิก

สรุปสั้น: `GTABLE` ขาด config `CKLAS` → latest date = 0 → ไม่ตรงกับวันที่ใบเสร็จ → ERR21

---

### 4. Checklist ก่อนแก้ไข

- [ ] ระบุ **library ของ environment** ที่มีปัญหา (อาจไม่ใช่ `SSKKLIB4` เสมอไป)
- [ ] ตรวจสอบว่า `CKLAS` มีอยู่ใน `GTABLE` ของ library เป้าหมายหรือยัง
- [ ] ตรวจสอบ `CKLAS` จาก library ต้นทาง (SSPARMLIB) เพื่อเตรียม insert
- [ ] ระวัง **duplicate key** หากมี record อยู่แล้ว

---

### 5. SQL ตรวจสอบ (ทำก่อนเสมอ)

**เช็ก library เป้าหมายว่ามี CKLAS หรือยัง:**
```sql
SELECT *
FROM SSKKLIB4/GTABLE
WHERE TDTYPE = 'CKLAS';
```
> ถ้าได้ผลลัพธ์ว่าง (0 rows) → ยืนยันว่า missing และต้อง insert

**เช็ก library ต้นทางว่ามีข้อมูลให้ copy:**
```sql
SELECT *
FROM SSPARMLIB/GTABLE
WHERE TDTYPE = 'CKLAS';
```
> ถ้ามีผลลัพธ์ → พร้อม insert ได้

---

### 6. SQL แก้ไข (เฉพาะกรณี missing จริงเท่านั้น)

```sql
INSERT INTO SSKKLIB4/GTABLE
SELECT *
FROM SSPARMLIB/GTABLE
WHERE TDTYPE = 'CKLAS';
```

> **แทนที่ `SSKKLIB4` ด้วย library ของ environment จริงที่มีปัญหา**

---

### 7. ข้อควรระวัง

> **ห้ามรัน INSERT ทันทีโดยไม่เช็ก**

| ความเสี่ยง | วิธีป้องกัน |
|---|---|
| Insert ผิด library | ระบุ library ให้ถูกต้องตาม environment จริง |
| Duplicate key error | รัน SELECT เช็กก่อนเสมอ ถ้ามีอยู่แล้วไม่ต้อง insert |
| ข้อมูลใน SSPARMLIB ไม่ครบ | เช็ก source ก่อนว่ามีข้อมูลจริง |
| กระทบ production | ทดสอบใน test environment ก่อนหากเป็นไปได้ |

---

## ข้อมูล Reference

| หัวข้อ | รายละเอียด |
|---|---|
| หน้าจอ | C01R116 — Cancel Temp. & Received Collection |
| Error code | ERR21 |
| ตารางที่เกี่ยวข้อง | GTABLE |
| Key ที่ขาด | `TDTYPE = 'CKLAS'` |
| Library ต้นทาง (ตัวอย่าง) | SSPARMLIB |
| Library เป้าหมาย (ตัวอย่าง) | SSKKLIB4 |

---

## หมายเหตุสำหรับ AI

- เมื่อเจอ trigger ใดใน YAML frontmatter ให้ตอบตาม structure ข้างต้นทันที
- ถามผู้ใช้เสมอว่า library เป้าหมายคือ library ใด อย่า assume ว่าเป็น `SSKKLIB4` เสมอไป
- ถ้าผู้ใช้แสดง SQL result ที่ 0 rows จาก GTABLE → ยืนยัน root cause และให้ INSERT SQL ได้เลย
- ถ้าผู้ใช้แสดง SQL result ที่มีข้อมูลอยู่แล้ว → ตรวจสอบว่า record ถูกต้องหรือไม่ก่อนสรุป
