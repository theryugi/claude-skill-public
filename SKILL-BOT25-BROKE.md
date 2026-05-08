# SKILL: AS400/RPGLE — BOT25 Broke (rd1q177)

```yaml
name: bot25-broke-rd1q177
description: >
  ความรู้เกี่ยวกับโปรแกรม rd1q177 สำหรับทำ BOT25 Broke
  คือการยกเลิกออกจากโครงการ BOT แล้ว gen schedule งวดชำระกลับเป็นค่างวดเดิมก่อนเข้าโครงการ
triggers:
  - rd1q177
  - BOT25
  - BOT25 broke
  - broke schedule
  - gen schedule broke
  - ออกจากโครงการ BOT
  - คำนวณงวดใหม่หลัง broke
  - HP1P10B VAT
  - ทอนแวท
  - VAT deduction installment
```

---

## ภาพรวมโปรแกรม

**โปรแกรม:** rd1q177
**วัตถุประสงค์:** BOT25 Broke — ออกจากโครงการ BOT แล้ว generate installment schedule ใหม่กลับเป็นค่างวดเดิมก่อนเข้าโครงการ

### BOT คืออะไร
**BOT = ธนาคารแห่งประเทศไทย** — โครงการช่วยเหลือลูกค้าลดภาระดอกเบี้ย โดย:
- ลูกค้าจ่ายดอกเบี้ยแค่ **50%** ตลอดช่วงที่อยู่ในโครงการ
- ส่วนที่เหลืออีก 50% **นำส่ง BOT (ธนาคารแห่งประเทศไทย)** เพื่อเรียกเก็บแทน

---

## Parameters

| ลำดับ | Field | ความหมาย | ขนาด |
|---|---|---|---|
| 1 | PRCMTH | Company | 5 |
| 2 | PRPDTH | Product | 5 |
| 3 | PRBRNO | Branch | 5 |
| 4 | PRCTNO | Contract | 7 |
| 5 | PRTYPE | BOT Type / Broke Reason (1-4) | 1 |
| 6 | PRBDAT | Broke Date | 6,0 |

---

## Broke Reason (PRTYPE)

เก็บใน **DBMTHD03** โดย THDTB1 = `'B25RS'`, THDTB2 = PRTYPE

| PRTYPE | THDDSC (Thai) | THDDSE | THDFG1 | THDFG2 | วิธี Trigger |
|---|---|---|---|---|---|
| 1 | ออกจากโครงการเนื่องจาก User | ทำรายการผิด | 1 | Y | Manual ผ่านหน้าจอ 8113 |
| 2 | ออกจากโครงการเนื่องจาก ลูกค้า | ขอยกเลิกเปลี่ยนแปลง | blank | Y | Manual ผ่านหน้าจอ 8113 |
| 3 | ออกจากโครงการเนื่องจาก ผิดเงื่อนไข | เงื่อนไข | blank | Y | Manual ผ่านหน้าจอ 8113 |
| 4 | ออกจากโครงการเนื่องจาก ค้าง | ชำระเกินกว่า 90 วัน | blank | blank | **Auto EOD** — ค้างเกิน 4 งวด |

**หมายเหตุ:**
- `THDFG1 = '1'` → USECAN = '1' → logic CANCEL BOT25 (เฉพาะ PRTYPE 1)
- `THDFG2 = 'Y'` → ยกเลิกได้ผ่านหน้าจอ 8113 (manual) — PRTYPE 1,2,3 เท่านั้น
- PRTYPE 4 ไม่มี THDFG2 เพราะ **ระบบ trigger อัตโนมัติตอน EOD** เมื่อค้างเกิน 4 งวด

---

## ไฟล์ที่เกี่ยวข้อง

| ไฟล์ | บทบาท | หมายเหตุ |
|---|---|---|
| **HP1P10** | ตาราง installment ปัจจุบัน | ตารางหลักที่ใช้แสดงงวดชำระ ไม่มี VAT deduction |
| **HP1P10B** | ตาราง installment ปัจจุบัน + เก่า (PF) | มีการทอนแวท (VAT deduction) |
| **HP1P10B1** | LF ของ HP1P10B | Key ต่างจาก PF — ใช้ดึงข้อมูลด้วย key sequence อื่น |
| **HP1P10B5** | LF ของ HP1P10B | Key ต่างออกไปอีก — ใช้ใน CALIN1 |
| **HP1P10C** | ตาราง installment ชุด copy (PF) | ใช้ระหว่าง process |
| **HP1P10C1** | LF ของ HP1P10C | Key ต่างจาก PF |
| **BOTP03** | BOT Master file | ข้อมูลหลักของโครงการ BOT |
| **HP1P02** | Contract Master file | ข้อมูลหลักของสัญญา |
| **HP1T01L1** | Mirror ของ HP1P02 | มี field ชื่อเหมือน HP1P02 — update ให้ sync กันเสมอ |
| **HP1L5301** | Rebate/ส่วนลด BOT25 | ตอน broke จะ set `RASTS='C'` เพื่อยกเลิก rebate ที่ได้รับตอนเข้า BOT25 |
| **HP1P18** | Message/Notes | เก็บข้อความแจ้งเตือนหลัง broke |

---

## ความแตกต่าง HP1P10 vs HP1P10B

| | HP1P10 | HP1P10B |
|---|---|---|
| VAT | ไม่มี VAT deduction | **มีการทอนแวท** |
| บทบาท | ตารางงวดปัจจุบัน | ตารางงวด + ข้อมูล VAT invoice |
| VAT Invoice # | ไม่เก็บ | เก็บ HNVAT# (เลข VAT invoice) |

---

## Logic การทอนแวท (VAT Deduction ใน HP1P10B)

เมื่อ broke โปรแกรมจะ allocate VAT จาก invoice เดิมไปยังงวดใหม่แต่ละงวด:

```
สะสม VAT ที่รู้แล้วไว้ใน array:
  WKVATO(I) = ยอด VAT แต่ละ invoice
  WKVAT#(I) = เลข VAT invoice

ทอนออกทีละงวด:
  WKVAT$ > WKVATO(I)  → ใช้หมด bucket นั้น → เหลือต่อ bucket ถัดไป (I++)
  WKVAT$ = WKVATO(I)  → พอดี ใช้หมด → ไป bucket ถัดไป (I++)
  WKVAT$ < WKVATO(I)  → ใช้บางส่วน bucket นั้นยังเหลือ (I ไม่เพิ่ม)
```

### ทำไมถึงต้องทอนแวท

ก่อนเข้าโครงการ BOT ระบบ**รับรู้ VAT ล่วงหน้า**ไปแล้วตามยอดงวดเดิม (เช่น 144.51 ต่องวด)
เมื่อเข้าโครงการแล้วได้รับส่วนลด หรือ broke ออก ยอดงวดใหม่ลดลง (เช่น 1105.61)
VAT ที่ควรได้ต่องวดจึงน้อยกว่าที่รับรู้ไปแล้ว (77.39 < 144.51)
โปรแกรมต้องค่อยๆ "ทอน" VAT จากใบที่รับรู้แล้วให้ครบทีละงวด

### กระบวนการทอนแวท (ตัวอย่างจากข้อมูลจริง)

```
ใบแวทเดิม = 144.51 ต่อใบ
งวดใหม่หลัง broke = 1105.61 → VAT ที่ต้องการ = 77.39

งวด 35: ใช้ 77.39 จากใบ 144.51 → เหลือในใบ = 67.12
งวด 36: ต้องการ 77.39
         → ใช้ที่เหลือ 67.12 จากใบเดิมก่อน (แถวปกติ inst=1105.61)
         → ยังขาดอีก 10.27
         → สร้างแถวพิเศษ inst=0 เพื่อดึง 10.27 จากใบใหม่
         → 67.12 + 10.27 = 77.39 ✓ ครบ
งวดถัดไป: ทำซ้ำจนใบใหม่หมด แล้วขึ้นใบถัดไป...
```

### แถว inst=0 คืออะไร

**ไม่ใช่งวดใหม่** — เป็น "VAT continuation record" ของ period เดิม
- `HNINST = 0` → ไม่มีเงินต้น เป็นแค่การจัดสรร VAT
- `HNPRD#` = period เดียวกับแถวก่อนหน้า (ไม่ advance period)
- เกิดเมื่อ VAT ใบเดิมหมดแล้ว ยังได้ไม่ครบ → ดึงจากใบถัดไป
- โค้ด: `WKTMP4 > 0` → set `HNINST = 0`, `WKTPRD - 1`

**หน้าตาข้อมูลใน HP1P10B หลัง broke:**
- Column J = ยอดงวด (HNINST) — ถ้า 0 คือ VAT continuation record
- Column N = ยอด VAT ของ record นี้ (HNVAT$)
- Column O = ยอด VAT สะสม (HNVATO)
- Column P = ยอดเงินต้น original ของใบ VAT นั้น (คำนวณจาก HNVAT$/0.07)
- Column Q = ยอดรวม (งวด + VAT)
- Column R = VAT Invoice # (HNVAT# เช่น VHI...)
- Column S = Receipt # (เช่น RHI...)

---

## Flow หลักของโปรแกรม

```
1. รับ parameter → PRCMTH, PRPDTH, PRBRNO, PRCTNO, PRTYPE, PRBDAT

2. เช็ก BOT24 ใน BOTP03B1
   → ถ้าเคยเข้า BOT24 มาก่อน จะ set WKBOTF = 'BOT24'

3. EXSR DLT10C
   → Delete HP1P10C ที่มีอยู่เดิม (ทำความสะอาดก่อน)

4. EXSR WRT10C
   → Copy HP1P10B → HP1P10C (backup ข้อมูลเก่า)
   → Collect VAT ไว้ใน array WKVATO/WKVAT#

5. EXSR CALIN1
   → คำนวณ installment ใหม่สำหรับ HP1P10B (ตารางงวดใหม่)
   → EXSR WRT10N ทีละงวดจนครบ (พร้อมทอนแวท)

6. EXSR CALIN2
   → คำนวณ installment ใหม่สำหรับ HP1P10 (ตารางปัจจุบัน)
   → EXSR WRTP10 ทีละงวดจนครบ

7. EXSR UPDP02
   → Update HP1P02 (Contract master) ให้ตรงกับข้อมูลใหม่

8. EXSR WRTP18
   → เขียนข้อความแจ้งเตือนลง HP1P18

9. EXSR DUNBRK
   → เตรียม Dunning broke (CALL CO1R497)
```

---

## Subroutines สำคัญ

| Subroutine | หน้าที่ |
|---|---|
| `DLT10C` | Delete HP1P10C เก่าทิ้งก่อน process |
| `WRT10C` | Copy HP1P10B → HP1P10C และ collect VAT array |
| `WRT10O` | Copy HP1P10 → HP1P10B (backup old) |
| `WRT10N` | Write HP1P10B งวดใหม่พร้อมทอนแวท |
| `WRTP10` | Write HP1P10 งวดใหม่ |
| `CALIN1` | คำนวณ installment schedule ใหม่สำหรับ HP1P10B |
| `CALIN2` | คำนวณ installment schedule ใหม่สำหรับ HP1P10 |
| `UPDP02` | Update HP1P02 contract master |
| `WRTP18` | เขียน message ลง HP1P18 |
| `CLFDUE` | หา first due date ใหม่ |
| `DUNBRK` | เตรียม dunning broke (CALL CO1R497) |

---

## Programs ที่ถูก CALL

| Program | จุดที่เรียก |
|---|---|
| `SBR064` | INZSR — initial routine |
| `SBR425` | CALIN1, CALIN2, UPDP02 — คำนวณ installment (ดูหมายเหตุด้านล่าง) |
| `SBR004` | CLFDUE — แปลงวันที่ |
| `SBR006` | CLFDUE — ปรับวันที่ |
| `DUEBW2` | CLFDUE — คำนวณวันครบกำหนด |
| `SBR1009` | WRTP18 — เขียน message HP1P18 |
| `SBR695` | UPDP02 — (BOT24 case) |
| `CO1R497` | DUNBRK — prepare dunning broke |

---

## เงื่อนไขพิเศษ BOTP03 — BOT24 vs BOT25

### กรณี PRTYPE = '1' (ทำรายการผิด / User error)
```
DBMTHD03 (B25RS, THDTB2='1') → THDFG1 = '1' → USECAN = '1'

USECAN = '1'
→ CHAIN BOTP03
→ WRITE BOTP03B  (text = 'CANCEL BOT25')
→ DELETE BOTP03
```
ผลลัพธ์: ยกเลิก BOT25 ออกทันที — record ย้ายไป history

---

### กรณี PRTYPE ≠ '1' (PRTYPE = 2,3,4)
```
DBMTHD03 (B25RS, THDTB2='2/3/4') → THDFG1 = blank → USECAN = ' '
→ เข้า broke ปกติ (ดู logic BOT24 ด้านล่าง)
```

### กรณี PRTYPE ≠ '1' + สัญญาเคยเข้า BOT24 (WKBOTF = 'BOT24')
```
Step 1: READPE BOTP03B1 (หา BOT24 record เดิม)
        → UPDATE BOTP03B  (text = 'BROKE BOT25')

Step 2: CHAIN BOTP03 (BOT25 ปัจจุบัน)
        → WRITE BOTP03B   (ย้าย BOT25 ไป history)
        → DELETE BOTP03

Step 3: CHAIN BOTP03B1 (BOT24 record)
        → WRITE BOTP03    (นำ BOT24 กลับมาเป็น current)
        → text = 'ON CURRENT'
```
ผลลัพธ์: สัญญา broke ออกจาก BOT25 กลับไปอยู่ใน BOT24 เหมือนเดิม

---

### กรณี PRTYPE ≠ '1' + ไม่ใช่ BOT24
```
ไม่ทำอะไรกับ BOTP03 เลย
```

---

### ตาราง summary

| เงื่อนไข | BOTP03 | BOTP03B |
|---|---|---|
| PRTYPE = '1' | DELETE | WRITE (CANCEL BOT25) |
| PRTYPE ≠ '1' + BOT24 | DELETE แล้ว WRITE ใหม่จาก BOT24 (ON CURRENT) | UPDATE (BROKE BOT25) + WRITE จาก BOT25 เดิม |
| PRTYPE ≠ '1' + ไม่ใช่ BOT24 | ไม่มีการเปลี่ยนแปลง | ไม่มีการเปลี่ยนแปลง |

---

## WKIRRT — ดอกเบี้ยที่จ่ายมาแล้ว (Paid Interest Adjustment)

ก่อนคำนวณงวดใหม่ต้องได้ **เงินต้นสุทธิ ไม่รวมดอกเบี้ยคงเหลือ** แล้วค่อยคิดดอกเบี้ยใหม่เข้าไป

```rpg
* สะสมดอกเบี้ยที่จ่ายมาแล้วจาก HP1P10B1
IF HOCODE = 'P' AND HORSTS = ' '
   WKIRRT = WKIRRT + HOPINS - WKAM01   ← กรณี partial payment
ELSE
   WKIRRT = WKIRRT + HOINST - HOPERN   ← กรณีปกติ

* ใช้หักออกจาก principal ก่อนคิดงวดใหม่
HBPAMT SUB WKIRRT → WKTTIN   ← เงินต้นสุทธิสำหรับคิดงวดใหม่
HBEFRT DIV 12    → WKTEFT    ← อัตราดอกเบี้ยต่อเดือน
```

---

## BOTP03C — Broke Master File

ทุกครั้งที่ broke สำเร็จจะ **WRITE BOTP3CR เสมอโดยไม่มีเงื่อนไข** (line 1228)

```rpg
WRITE BOTP3CR   ← บันทึกว่าสัญญานี้ broke แล้ว
```

> BOTP03C ≠ audit trail — เป็น **Master file ของ broke** เก็บข้อมูลสรุปของแต่ละ broke event

---

## WKBDAT — Logic ใน WRT10B (Copy ก่อน Broke Date เท่านั้น)

```rpg
* ใน WRT10B (line 811)
IF WKBDAT <= H1DUDT
   LEAVE   ← หยุด copy ทันที
ELSE
   ...     ← copy งวดนี้เข้า HP1P10B1
ENDIF
```

**เหตุผล:** ต้องการข้อมูลตาราง installment เดิมที่ backup ไว้ **ก่อนวัน broke เท่านั้น**
งวดที่ due date >= WKBDAT (on/after broke date) จะไม่ copy — โปรแกรมจะ gen งวดใหม่เองหลังจากนั้น

```
HP1P10C (backup)  →  copy เฉพาะ H1DUDT < WKBDAT  →  HP1P10B1
                                                          ↓
                                                    gen งวดใหม่ต่อจากนี้
```

---

## Entry Point — หน้าจอที่เรียกโปรแกรม

```
หน้าจอ 8113  →  CALL rd1q177 โดยตรง
```

---

## SBR425 — การใช้งาน Output จริงในโปรแกรม

SBR425 ถูก CALL 3 จุด แต่ output ที่ใช้จริงมีเพียงจุดเดียว:

| จุดที่เรียก | Output ที่ใช้จริง | หมายเหตุ |
|---|---|---|
| CALIN1 (line 987) | `@POFST` → `Ymdds` | ใช้เป็น first due date เริ่มต้น |
| CALIN2 (line 912+) | **ไม่มี** | หลัง CALL ใช้ `HBPAMT`, `HBEFRT` จาก HP1P02 โดยตรง |
| UPDP02 (line 1061+) | **ไม่มี** | หลัง CALL ไม่มีการอ้างถึง @PO* ใดเลย |

**Parameters ที่ส่งเข้า SBR425 ทุกครั้ง:**
```
@PICMT = KKCMTH (company)
@PIPDT = KKPDTH (product)
@PIBRN = KKBRNO (branch)
@PICTN = KKCTNO (contract)
@PIBOT = *BLANK
@POFST = *ZERO  ← output: first due date (ใช้แค่ใน CALIN1)
(ที่เหลือ *BLANK/*ZERO และไม่ได้อ่านค่ากลับ)
```

> **Refactor note:** CALIN2 และ UPDP02 อาจ CALL SBR425 เพื่อ side effect เท่านั้น
> ควรตรวจสอบว่า SBR425 มีการ update ตารางภายในหรือไม่ก่อน refactor

---

## Dead Code — WKFNDT

WKFNDT ถูก **set หลายจุดแต่ไม่เคยถูกอ่านเพื่อใช้งานจริง**

```
set ใน: WRTP10 (663), WRT10N (733, 777), CALIN1/CALIN2 init, UPDP02 (1111)
ใช้จริง: ไม่มี — UPDP02 ใช้ WKLDUE ไปอัปเดต HBLDUE ใน HP1P02 แทน
```

> **Refactor note:** WKFNDT น่าจะเป็น pattern เก่าที่ไม่ได้ใช้แล้ว
> สามารถ remove ออกได้ตอน refactor (ตรวจสอบให้แน่ใจก่อนว่าไม่มี program อื่นอ้างถึง)

---

## WKSNDT = 'S' — Cut Bill Check และ WKUNER (CT4)

### Cut Bill Date คืออะไร
```
Cut Bill Date = Due Date - 21 วัน
คำนวณผ่าน DUEBW2 (ZAGE = '000021')
```

### Logic ใน CLFDUE
```rpg
IF WKTRND > @WKDAT AND USECAN = ' '
   → broke date เลย cut bill แล้ว
   → WKSNDT = 'S'          (S = Shift — เลื่อนงวดแรกไปงวดถัดไป)
   → WKOLDD = Ymdds         (บันทึก due date ของงวดที่ผ่าน cut bill)
   → M_Ymd + 1              (เลื่อนไปงวดถัดไป)
ELSE
   → WKSNDT = ' '           (ปกติ ไม่ต้องเลื่อน)
   → @FDUE = Ymdds          (ใช้งวดปัจจุบันเป็นงวดแรก)
```

### WKUNER — ยอดที่ต้องเรียกเก็บท้ายสัญญา (CT4)
เมื่อ WKSNDT = 'S' งวดที่ผ่าน cut bill ไปแล้วจะไม่ถูกเรียกเก็บทันที
แต่ดอกเบี้ย (HNAM02) ของงวดนั้นต้องบวกกลับเข้า WKUNER เพื่อเรียกเก็บท้ายสัญญา:

```rpg
* ใน WRT10C (line 292-294)
IF WKSNDT = 'S' AND WKOLDD = HNDUDT
   ADD HNAM02 WKUNER    ← บวกดอกเบี้ยงวดที่ skip กลับคืน
ENDIF
```

### เขียน CT4 ลง HP1P11
```rpg
* ใน UPDP02 (line 1158-1172)
IF USECAN = ' '
   WKUNER MULT 0.07 WKUNVT   ← คำนวณ VAT ของ unearn
   ARTRNC = 'CT4'
   ARTAMT = WKUNER + WKUNVT
   WRITE HP1P11R              ← เก็บไว้เรียกเก็บท้ายสัญญา
ENDIF
```

### สรุป Flow
```
broke date > cut bill?
   YES → WKSNDT='S' → เลื่อน first due ไปงวดถัดไป
         → งวดที่ข้ามไป: บวก HNAM02 → WKUNER
         → ท้าย process: เขียน CT4 ลง HP1P11 ให้เรียกเก็บท้ายสัญญา
   NO  → WKSNDT=' ' → ใช้งวดปัจจุบันเป็น first due ปกติ
```

---

## @FDUE vs @FDUE2 — First Due Date

ทั้งคู่ถูก set ค่าเดียวกันพร้อมกันใน CLFDUE แต่แยกไว้เพื่อใช้กับคนละตาราง:

| Variable | ใช้กับ | พฤติกรรม |
|---|---|---|
| `@FDUE` | HP1P10 (HADUDT) ใน WRTP10 | set งวดแรก แล้ว clear = 0 ทันที ให้ loop บวกเดือนต่อไปเอง |
| `@FDUE2` | HP1P10B (HNDUDT) ใน WRT10N | set งวดแรก แล้ว clear = 0 ทันที ให้ loop บวกเดือนต่อไปเอง |

```rpg
* ใน WRTP10 — ใช้ @FDUE รอบแรก แล้ว clear
IF @FDUE <> 0
   EVAL HADUDT = @FDUE
   EVAL @FDUE = 0        ← clear แล้ว → งวดถัดไปจะบวก M_Ymd + 1 เอง
ELSE
   ADD 1 M_Ymd            ← งวดปกติ บวกเดือนไปเรื่อยๆ
   ...
ENDIF
```

> **Refactor note:** @FDUE และ @FDUE2 เป็น pattern เดียวกันทุกอย่าง ต่างกันแค่ตารางปลายทาง — อาจ refactor รวม subroutine ได้

---

## Modification Indicators (WI01, WI02, WW001)

บรรทัดที่ขึ้นต้นด้วย `WI01`, `WI02`, `WW001` คือ **version tracking** — บันทึกว่าโค้ดส่วนนั้นถูกเพิ่ม/แก้ไขในรอบการแก้ไขใด ไม่ใช่ conditional compile

| Indicator | ความหมาย |
|---|---|
| `WI01` | Work item / แก้ไขครั้งที่ 1 |
| `WI02` | Work item / แก้ไขครั้งที่ 2 |
| `WW001` | Work item อีก set หนึ่ง |

> **Refactor note:** ตอน refactor ควรระวังบรรทัดที่มี indicator เหล่านี้ เพราะอาจเป็น logic ที่เพิ่มทีหลังและมี dependency กับ feature เฉพาะ

---

## HNAM01 / HNAM02 / HNAM03 — ยอดดอกเบี้ยใน HP1P10

| Field | ความหมาย | หมายเหตุ |
|---|---|---|
| HNAM01 | — | ไม่ได้ใช้ใน logic หลัก |
| HNAM02 | ดอกเบี้ย **เต็มงวด** (100%) | ยอดดอกเบี้ยจริงถ้าไม่มีโครงการ |
| HNAM03 | ดอกเบี้ย **50%** ที่เก็บตอนอยู่ในโครงการ | นำส่ง BOT (ธนาคารแห่งประเทศไทย) เพื่อเรียกเก็บแทน |

```
ดอกเบี้ยลูกค้าจ่ายจริง = HNAM03 (50%)
ดอกเบี้ยที่นำส่ง BOT   = HNAM02 - HNAM03 (50% ที่เหลือ)
```

> ตอน broke: WKUNER = ยอด HNAM02 ของงวดที่ยังรับรู้ดอกเบี้ยไม่ครบ — เรียกเก็บท้ายสัญญาผ่าน CT4 ลง HP1P11 (unearn interest)
> ต้องเขียน CT4 ลง HP1P11 เพื่อเรียกเก็บท้ายสัญญา

---

## HACODE / HARSTS — สถานะการจ่ายงวด

| Field | ค่า | ความหมาย |
|---|---|---|
| HACODE | `' '` | ยังไม่จ่าย |
| HACODE | `'R'` | จ่ายแล้วเต็มงวด |
| HACODE | `'P'` | จ่าย partial |
| HARSTS | `'O'` | Overdue — ค้างชำระ |

### ผลต่อการคำนวณ WKUNER

```rpg
* จ่ายเต็มงวดแล้ว และมี AM03 (เก็บไปแล้วบางส่วน)
HACODE='R' AND HNAM03≠0  →  WKUNER + HNAM02 - HNAM03

* จ่าย partial และมี AM03
HACODE='P' AND HNAM03≠0  →  WKUNER + HNAM02

* ยังไม่จ่าย แต่ค้าง (overdue)
HACODE=' ' AND HARSTS='O'  →  WKUNER + HNAM02

* เลย cut bill (WKSNDT='S') และตรงงวดนั้นพอดี
WKSNDT='S' AND WKOLDD=HNDUDT  →  WKUNER + HNAM02
```

---

## BTFA07 — VAT ก่อนเข้าโครงการ (จาก BOTP03)

```rpg
Z-ADD BTFA07 WKVAT$   ← ใช้เป็น VAT reference ใน CALIN1/CALIN2
```

**BTFA07** = ยอด VAT เดิม **ก่อนเข้าโครงการ BOT** เก็บไว้ใน BOTP03 (BOT master)
เนื่องจากตอน broke ต้องกลับไปใช้ตาราง VAT เดิม ไม่ใช่ VAT ที่คิดลดในโครงการ

---

## SBR695 — Data Warehouse Update (HP1P02 Changed)

```rpg
CALL 'SBR695'
PARM HBCMTH @PICMT
PARM HBPDTH @PIPDT
PARM HBBRNO @PIBRN
PARM HBCTNO @PICTN
PARM ' '    @PIFLG
```

**หน้าที่:** แจ้ง Data Warehouse ว่า HP1P02 (contract master) มีการเปลี่ยนแปลง — ต้อง sync ข้อมูลใหม่

> ต้องรันทุกครั้งหลัง broke เสร็จ ไม่ว่าจะเป็น BOT24 หรือ BOT25 ปกติ
> **(แก้ไขใน source จริงแล้ว)**

---

## CO1R497 — Dunning Broke (DUNBRK)

```rpg
CALL 'CO1R497'
PARM PRCMTH, PRPDTH, PRBRNO, PRCTNO, PRBDAT, BCSBMK
```

> น่าจะเป็นโปรแกรมออกจดหมาย **Dunning** แจ้งลูกค้าหลัง broke
> (ไม่ได้เขียนเอง — ต้องตรวจสอบ source ของ CO1R497 แยกต่างหาก)

---

## หมายเหตุสำหรับ AI

- เมื่อเจอ trigger ให้อธิบาย flow ตาม structure ข้างต้น
- ถ้าถามเรื่อง VAT deduction ให้อ้างอิง logic array WKVATO/WKVAT# ใน subroutine WRT10N
- PRTYPE ต้องเช็กใน DBMTHD03 เสมอ ไม่ใช่ hardcode ใน program
- กรณี BOT24 → broke ต้องเช็ก WKBOTF = 'BOT24' ด้วย จะมี flow พิเศษใน UPDP02
