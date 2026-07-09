---
title: Channel กับ Tool ใน Claude Code ต่างกันตรงไหน — เส้นแบ่งอยู่ที่ notification เดียว
description: อ่าน server.ts จริงของ discord/fakechat/imessage/telegram แล้วพบว่า channel ไม่ใช่คนละอย่างกับ tool แต่เป็น superset — เส้นแบ่งจริง ๆ คือบรรทัด mcp.notification('notifications/claude/channel')
date: 2026-07-09
datetime: 2026-07-09T09:50:00+07:00
tags: [claude-code, mcp, channel, verify]
---

Claude Code มี plugin สองแบบที่คนสับสนกันบ่อย — **channel** (เช่น discord) กับ **tool** (เช่น github, playwright) มันดูคล้ายกันเพราะเป็น MCP server เหมือนกัน แต่พอเปิด `server.ts` ตัวจริงใน `anthropics/claude-plugins-official` อ่านเอง เส้นแบ่งชัดกว่าที่คิดมาก — และมันแคบแค่บรรทัดเดียว

ผมไม่เชื่อบทความหรือความจำ โหลดโค้ดจริงมาอ่านทั้ง 4 ไฟล์ (discord, fakechat, imessage, telegram) นี่คือสิ่งที่เจอ

## เส้นแบ่งหลัก: ใครเริ่มก่อน

ต่างกันที่ทิศทางของการเริ่มบทสนทนา

- **Tool** — Claude เอื้อมมือออกไปเรียก Claude เป็นคนเริ่ม ยิง request แล้วรอ response จบในนั้น
- **Channel** — โลกภายนอก push เข้าหา Claude เอง คนข้างนอกพิมพ์มาก่อน แล้ว Claude ตอบกลับผ่าน tool ชื่อ `reply`

พูดง่าย ๆ tool คือ**มือ** (Claude เอื้อมไปหยิบ) ส่วน channel คือ**หูกับปาก** (คนข้างนอกพูดใส่ Claude ได้เองโดยไม่ต้องขอ แล้ว Claude พูดกลับ)

## 3 signal ที่แยกได้จากโค้ด

**1. capability ที่ประกาศ**

```ts
// fakechat/server.ts:62
capabilities: { tools: {}, experimental: { 'claude/channel': {} } }
```

channel ประกาศ `claude/channel` เพิ่มเข้ามา ส่วน tool มีแค่ `tools: {}` สังเกตว่า fakechat ประกาศ **ทั้งสองอย่าง** — นี่คือกุญแจข้อสำคัญที่จะพูดถึงตอนท้าย

**2. การ push ข้อความขาเข้า (หัวใจจริง)**

```ts
// discord/server.ts:875 · fakechat/server.ts:138
mcp.notification({
  method: 'notifications/claude/channel',
  params: { ... }
})
```

channel เรียก `mcp.notification()` เพื่อ **ดันข้อความขึ้นไปหา Claude เองโดยไม่ต้องรอให้เรียก** — tool ไม่เคยทำแบบนี้ tool รอถูกเรียกอย่างเดียว

**3. loop ที่ฟังโลกภายนอก**

```ts
// discord/server.ts:805
client.on('messageCreate', msg => { ... })
```

channel มี listener ที่ฟังเหตุการณ์จากข้างนอก (Discord Gateway, WebSocket, หรือ poll `chat.db` ของ Messages) แล้วแปลงเป็น notification ดันเข้า Claude ส่วน tool ไม่มี loop แบบนี้เลย

## จุดที่อ่านโค้ดเองแล้วถึงเห็น

**channel ไม่ใช่คนละอย่างกับ tool — มันคือ superset** fakechat ประกาศ `capabilities: { tools: {}, experimental: {'claude/channel':{}} }` แปลว่ามันมี tool (`reply`, `edit_message`) **บวก** ความสามารถให้ภายนอก initiate ได้

เพราะฉะนั้นเส้นแบ่งจริง ๆ ไม่ได้อยู่ที่ "มี tool ไหม" (channel ก็มี tool) แต่อยู่ที่บรรทัดเดียว:

> ถ้ามี `mcp.notification('notifications/claude/channel')` = **channel**
> ถ้าไม่มี = **tool เฉย ๆ**

## นับใน repo จริง

พอรู้ signal ก็ไล่นับได้: ใน `external_plugins/` มี **channel 4 ตัว** (discord, telegram, imessage, fakechat — ทุกตัวมี `server.ts` + emit notification) และ **tool 11 ตัว** (github, gitlab, linear, asana, firebase, context7, greptile, laravel-boost, playwright, serena, terraform — ไม่มี notification loop)

## ทำไมเรื่องนี้สำคัญ

ถ้าจะเขียน channel ของตัวเองต่อกับ LINE, Slack หรืออะไรก็ตาม — สิ่งที่ต้องเพิ่มจาก tool ธรรมดาคือ 3 อย่างนี้เป๊ะ: ประกาศ `claude/channel`, ตั้ง listener ฟังโลกภายนอก, แล้ว `mcp.notification()` ดันเข้า Claude ที่เหลือ (reply, edit) ก็เป็น tool ปกติ

บทเรียนที่ติดตัวมากกว่ารายละเอียด MCP คือ — **grep กับสมมติฐานหลอกได้ โค้ดจริงไม่หลอก** ตอนแรกผม grep หา `registerTool` เจอศูนย์ เกือบเชื่อว่า "ไม่มี tool" แต่พอเปิดอ่านจริงเจอว่ามันใช้ `setRequestHandler(ListToolsRequestSchema)` คนละ API กัน — เปิดโค้ดดูเองก่อนเชื่อเสมอ
