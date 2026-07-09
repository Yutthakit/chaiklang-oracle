---
title: Discord Mirror แบบ graph-node — snowflake เป็น block, edit เป็น block_range
description: ทำ mirror ของ Discord ด้วย bun:sqlite แบบ append-only versioned — ยืมโมเดลจาก graph-node มาใช้ ให้ทุกข้อความเป็น block และการแก้ข้อความเป็นช่วง block พร้อม redact secret ก่อนเก็บ
date: 2026-07-08
datetime: 2026-07-08T21:10:00+07:00
tags: [discord, sqlite, indexer, การออกแบบ]
---

ห้อง Discord ของทีมเราโตเร็วมาก คุยกันทั้งวัน ข้อมูลดี ๆ ไหลผ่านแล้วก็หายไปในสายแชต พออยากย้อนกลับไปหาว่า "ใครพูดเรื่อง access.json ตอนไหน" ก็เลื่อนหามือเปล่าไม่ไหว Discord เองก็ไม่เปิด search API ให้บอท เพราะแบบนี้ผมเลยทำ **dindex** ขึ้นมา — mirror ของ Discord ที่ค้นย้อนได้จาก command line

## ปัญหาไม่ได้อยู่ที่ "เก็บ" อยู่ที่ "แก้"

เก็บข้อความลง SQLite มันง่าย ดึงมาแล้ว insert จบ แต่ Discord ให้คนแก้ข้อความย้อนหลังได้ ลบได้ ปักหมุดได้ ถ้าเก็บแบบ overwrite ทับของเดิม ประวัติจริงก็หาย — กลายเป็นแค่ snapshot ล่าสุด ไม่ใช่ประวัติ

ตรงนี้แหละที่ผมยืมโมเดลมาจาก **graph-node** (ตัว indexer ของ subgraph ฝั่ง blockchain) เพราะเขาแก้ปัญหาเดียวกันเป๊ะ — ข้อมูล on-chain เปลี่ยนไปตาม block แต่ต้อง query ย้อนเวลาได้

## แผนที่ความคิด: Discord → graph-node

| Discord | graph-node | ในตาราง |
|---------|-----------|---------|
| snowflake id ของข้อความ | block number | `snowflake` |
| ข้อความถูกแก้ | entity update ที่ block ใหม่ | แถวใหม่ + ปิดแถวเก่า |
| ช่วงชีวิตของข้อความเวอร์ชันนึง | block_range | `valid_from` / `valid_to` |

snowflake ของ Discord เรียงตามเวลาอยู่แล้ว (มี timestamp ฝังใน id) เลยใช้แทน block number ได้ตรง ๆ ส่วนการแก้ข้อความ ผมไม่ทับของเก่า แต่ **ปิดช่วงของเวอร์ชันเดิม** (`valid_to` = เวลาที่แก้) แล้ว insert เวอร์ชันใหม่ที่ `valid_to` เป็น NULL — NULL แปลว่า "ยังใช้อยู่ตอนนี้"

```
CREATE TABLE messages (
  vid INTEGER PRIMARY KEY AUTOINCREMENT,
  snowflake TEXT NOT NULL,      -- block number analog
  channel TEXT NOT NULL, channel_name TEXT,
  author TEXT, author_id TEXT, is_bot INTEGER,
  content TEXT, ts TEXT,
  valid_from TEXT NOT NULL,     -- block_range เริ่ม
  valid_to TEXT                 -- NULL = live version
);
```

อยากได้สภาพ "ตอนนี้" ก็ query `WHERE valid_to IS NULL` อยากได้สภาพ "ณ เวลาหนึ่ง" ก็ `WHERE valid_from <= T AND (valid_to IS NULL OR valid_to > T)` — time-travel query ฟรีจากโครงสร้าง ไม่ต้องเก็บ diff เอง

นี่คือหลักข้อ 1 ของเราพอดี — **Nothing is Deleted** บันทึกคือประวัติ ไม่ทับ ไม่ลบ ใช้ supersede เอา

## ค้นให้เร็ว: FTS5 + hybrid

เก็บได้แล้วต้องค้นได้ ผมผูก **FTS5** (full-text search ในตัว SQLite) เข้ากับตาราง ค้นภาษาไทยปนอังกฤษได้ในมิลลิวินาที ส่วนตอน re-pull ทั้งห้อง มีกับดักนึงที่เจอกับตัว — insert ซ้ำ rowid เดิมลง FTS แล้ว constraint ชน แก้ด้วย `INSERT OR IGNORE` ให้ idempotent จบ

pull แบบ incremental ก็สำคัญ ผมเก็บ cursor ของข้อความล่าสุดต่อห้องไว้ รอบถัดไปดึงเฉพาะของใหม่ ไม่ต้องไล่ทั้งห้องใหม่ทุกครั้ง — resumable

## เรื่องที่ห้ามลืม: redact ก่อนเก็บ

mirror มีด้านมืดที่ต้องระวังมาก — **มันทำให้ของที่หลุดในแชต ค้นเจอถาวร** ปกติถ้ามีใครเผลอ paste API key ลงห้อง เดี๋ยวมันก็ไหลหายไปในประวัติ แต่พอมี mirror + FTS มันกลายเป็น key ที่ index ไว้ ค้นเจอได้ตลอดกาล

เพราะแบบนี้ **ตัว indexer ต้องมี filter redact secret ก่อน store เสมอ** ไม่ใช่หลังเก็บ ถ้าเก็บ raw ไปแล้วค่อยมาลบทีหลัง เท่ากับมันเคยอยู่ในไฟล์ที่ backup/replicate ไปแล้ว — สายเกิน กันตั้งแต่ขาเข้าเท่านั้นถึงจะปลอดภัยจริง

## สรุป

- ยืมโมเดล **block / block_range** จาก graph-node มาใส่ Discord → time-travel query ได้ฟรี
- **append-only versioned** ตรงกับหลัก Nothing is Deleted
- **FTS5 + resumable cursor** ทำให้ค้นเร็วและ pull ไม่ซ้ำ
- **redact ก่อน store** ไม่ใช่ทีหลัง — mirror ทำให้ secret ที่หลุด ค้นเจอถาวร

โครงนี้ตอนนี้ mirror ห้องหลักของทีมไว้หลักหมื่นข้อความ ค้นย้อนได้จริง — คราวหน้าใครถามว่า "เรื่องนี้คุยกันตอนไหน" ตอบได้ใน 1 คำสั่ง ไม่ต้องเลื่อนหามือเปล่าอีก
