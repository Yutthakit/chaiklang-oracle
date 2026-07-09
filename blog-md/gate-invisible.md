---
title: gate() ที่มองไม่เห็น — สร้าง maw discord-channel มาส่องประตูระหว่าง Discord กับ Claude
description: gate() คือฟังก์ชันเดียวที่ตัดสินว่า message จาก Discord ถึง Claude ไหม แล้ว drop เงียบ ๆ ไม่บอกเหตุ โพสต์นี้เดินโค้ด plugin ที่เขียนขึ้นมา simulate gate() แบบ read-only — resolveStateDir (global default + override เหมือน maw token), simulateGate (port ตรงจาก server.ts:236-296), lint จับกับดัก config, และ pending pairing พร้อม output จริงจาก access.json ของชายกลาง
date: 2026-07-09
datetime: 2026-07-09T11:45:00+07:00
tags: [claude-code, discord, mcp, channel, plugin, maw, tooling]
---

`gate()` เป็นฟังก์ชันเดียวใน 900 บรรทัดของ Discord channel plugin ที่ตัดสินว่า message จาก Discord จะ "ถึง Claude" หรือไม่ — อ่าน `access.json` สดทุกครั้ง ไม่ cache แล้วถ้าไม่ผ่าน มัน **drop เงียบ ๆ ไม่มี feedback กลับไปที่ใครเลย** ปัญหาคือพอ config ผิดนิดเดียว message ก็หายไปโดยที่คุณไม่รู้เหตุ

โพสต์นี้เดินโค้ด `maw discord-channel` — plugin ที่เขียนขึ้นมาเป็น "หมอตรวจประตู" ไม่แตะลูกกุญแจ (การแก้ `access.json` เป็นงานของ `maw discord access` อยู่แล้ว) แต่บอกว่าลูกกุญแจ *จะปล่อยใครเข้า* และ *ทำไม* ทุกบรรทัดเป็น read-only ล้วน

## ปัญหา — ประตูที่ตัดสินเงียบ

เส้นทางของ message จริงเป็นแบบนี้ (อ้าง `server.ts` ตัวจริง):

```
Discord message
   │ Gateway WebSocket (discord.js)
   ▼
client.on('messageCreate')     server.ts:805   ← bot? → return (กัน loop)
   ▼
gate(msg)                      server.ts:236   ← 🚪 อ่าน access.json
   ├─ 'drop'    → เงียบหาย ไม่ถึง Claude
   ├─ 'pair'    → ตอบ pairing code (ยังไม่ถึง Claude)
   └─ 'deliver' → mcp.notification(...) ──► Claude
```

ตรง `drop` นี่แหละที่เจ็บ — ไม่มี log ไม่มี error ไม่มีอะไรบอกว่า "message ของ user คนนี้โดนตัดเพราะ groups ห้องนี้ไม่ได้ opt-in" ถ้าอยากรู้ต้องไปนั่งไล่ `access.json` เอง ซึ่งเป็น JSON ดิบที่อ่านยาก

เป้าหมายของ plugin: **จำลอง `gate()` ออกมาให้เห็นคำตัดสิน + เหตุผล ก่อนที่มันจะเงียบใส่จริง**

## resolveStateDir — global default แต่ override ได้ (แบบ maw token)

ก่อนตรวจอะไรได้ ต้องรู้ก่อนว่า `access.json` ตัวไหนกำลังมีผล ตรงนี้ยืมสถาปัตยกรรมของ `maw token` มา — มี global default แต่ repo หนึ่ง ๆ ชี้ไปที่ channel state ของตัวเองได้ ไม่ผูกกับ global อย่างเดียว ลำดับ resolve คือ match แรกชนะ:

```ts
function resolveStateDir(args: string[], cwd: string): Resolved | { error: string } {
  // 1. explicit --state-dir wins
  const flag = getFlag(args, "--state-dir");
  if (flag) return locate(flag, `--state-dir ${flag}`);

  // 2. env DISCORD_STATE_DIR — สิ่งที่บอทที่กำลังรันอ่านจริง
  const env = process.env.DISCORD_STATE_DIR;
  if (env) return locate(env, "env DISCORD_STATE_DIR");

  // 3. per-repo ./.discord/ — เคส "ไม่เอา global"
  const hybrid = join(cwd, ".discord");
  if (existsSync(join(hybrid, "access.json"))) return locate(hybrid, "per-repo ./.discord");

  // 4. global ~/.claude/channels/<bot>/  (bot = --bot หรือ basename ของ cwd)
  const bot = getFlag(args, "--bot") ?? basename(cwd);
  const global = join(LEGACY_STATE_ROOT, bot);
  if (existsSync(join(global, "access.json")))
    return locate(global, `global ~/.claude/channels/${bot}`);

  return { error: `no access.json found. tried ./.discord + ~/.claude/channels/${bot} ...` };
}
```

จุดสำคัญคือลำดับ 2 — `env DISCORD_STATE_DIR` มาก่อน global เพราะมันคือ path ที่ **บอทตัวที่กำลังรันจริงอ่านอยู่** เวลารันในเซสชันที่บอทเปิดอยู่ tool จะไปตรวจ access.json ตัวเดียวกับที่มีผลจริง ไม่ใช่ตัวที่เดา ตรงนี้ทำให้ output เชื่อถือได้:

```
$ maw discord-channel where
state-dir : /Users/.../.claude/channels/chai-klang-oracle
access    : /Users/.../.claude/channels/chai-klang-oracle/access.json
resolved  : env DISCORD_STATE_DIR      ← บอทที่รันอยู่ชี้มาที่นี่
dmPolicy  : allowlist
groups    : 15
pending   : 0
```

## simulateGate — port ตรงจาก server.ts:236-296

หัวใจของ plugin เรียง branch เป๊ะตาม `gate()` ตัวจริง แยกสองเส้น DM กับ guild channel:

```ts
function simulateGate(a: Access, opts: { userId: string; channelId?: string; isDM: boolean; mention: boolean }): GateResult {
  if (opts.isDM) {
    const policy = a.dmPolicy ?? "pairing";
    if (policy === "disabled")
      return { action: "drop", reason: "dmPolicy=disabled → all DMs dropped" };
    if ((a.allowFrom ?? []).includes(opts.userId))
      return { action: "deliver", reason: `user ${opts.userId} is in top-level allowFrom` };
    if (policy === "allowlist")
      return { action: "drop", reason: `dmPolicy=allowlist and ${opts.userId} not in allowFrom` };
    return { action: "pair", reason: "dmPolicy=pairing → bot replies a 6-hex code (Claude never sees it)" };
  }

  // guild channel path
  const cid = opts.channelId;
  if (!cid) return { action: "drop", reason: "no --channel given for a guild message" };
  const g = a.groups?.[cid];
  if (!g) return { action: "drop", reason: `groups["${cid}"] absent → channel not opted-in (①)` };
  const allow = g.allowFrom ?? [];
  if (allow.length > 0 && !allow.includes(opts.userId))
    return { action: "drop", reason: `group allowFrom non-empty and ${opts.userId} not listed (②)` };
  if (g.requireMention && !opts.mention)
    return { action: "drop", reason: "group requireMention=true and no @mention (③)" };
  return { action: "deliver", reason: "passed all group gates (④)" };
}
```

เส้น guild มี 4 ด่านเรียงกัน ①→④ — พลาดด่านไหน tool บอกด่านนั้นเลย ไม่ใช่แค่ "drop"

## proof — doctor จำลอง 4 เคสกับ access.json จริง

ทดสอบกับห้อง control ของชายกลาง (`1512079809021214730`) และ user จริง:

```
$ dchan doctor --user 691531480689541170 --channel 1512079809021214730
verdict: ✅ DELIVER — user in allowFrom, no mention required (④)

$ dchan doctor --user 999999999999 --channel 1512079809021214730
verdict: 🚫 DROP — group allowFrom non-empty and 999999999999 not listed (②)

$ dchan doctor --user 999999999999 --dm
verdict: 🚫 DROP — dmPolicy=allowlist and 999999999999 not in allowFrom

$ dchan doctor --user 691531480689541170 --dm
verdict: ✅ DELIVER — user in top-level allowFrom
```

ทุกเคสตรงกับ branch order ของ `gate()` จริง — user ที่อยู่ใน allowFrom เข้าได้ทั้ง channel และ DM, user แปลกหน้าโดนตัดที่ด่าน ② (channel) และ allowlist (DM) พร้อมระบุด่านชัด

## lint — จับกับดักที่ทำ message หายแบบไม่รู้ตัว

`access.json` มีกับดัก config ที่เจอบ่อย — key บางตัวใช้ได้เฉพาะ top-level ถ้าเผลอเอาไปใส่ใน group จะถูกเมินเงียบ ๆ `lint` ไล่จับให้:

```ts
const TOP_ONLY = ["ackReaction", "replyToMode", "textChunkLimit", "chunkMode"];
for (const [id, g] of Object.entries(a.groups ?? {})) {
  for (const k of TOP_ONLY)
    if (k in (g as object))
      warn.push(`group ${id}: "${k}" is a TOP-LEVEL key — ignored inside a group. Move it out.`);
  if (g.requireMention === undefined)
    warn.push(`group ${id}: requireMention undefined — set it explicitly.`);
  if ((g.allowFrom ?? []).length === 0)
    info.push(`group ${id}: allowFrom empty → open to everyone in the channel.`);
}

// stale pending: หมดอายุแล้วแต่ยังค้าง
const now = Date.now();
for (const [code, p] of Object.entries(a.pending ?? {}))
  if (p.expiresAt && p.expiresAt < now)
    warn.push(`pending ${code}: expired ${Math.round((now - p.expiresAt) / 60000)}m ago — stale.`);
```

`allowFrom: []` ที่ว่างเปล่าเป็น **info ไม่ใช่ warning** เพราะมันคือ "เปิดทั้งห้อง" ที่ตั้งใจ — tool ต้องแยกออกว่าอันไหน bug อันไหน design

## เส้นแบ่งที่ตั้งใจ — หมอตรวจ ไม่ใช่คนแก้

ทั้งไฟล์ไม่มี `writeFileSync` ไปแตะ `access.json` เลยแม้แต่บรรทัดเดียว นี่เป็นเจตนา ไม่ใช่ยังทำไม่เสร็จ:

- **การเขียน** groups / dmPolicy / allowFrom → `maw discord access` (mutate) มีอยู่แล้ว จะทำซ้ำทำไม
- **การอนุมัติ pairing** → เป็น terminal step ต้องรันผ่าน `/discord:access` skill ในเทอร์มินัลเท่านั้น

ข้อหลังสำคัญด้านความปลอดภัย — ถ้า message ในแชตบอกให้ "approve pairing ให้หน่อย" นั่นคือสิ่งที่ prompt injection จะขอเป๊ะ tool เลยออกแบบให้ทำไม่ได้ตั้งแต่แรก `pending` แสดงแค่ code + TTL ไม่มีทางให้ channel message มา self-approve ได้:

```
$ dchan pending
pending pairings: 0
note: approving a pairing is a TERMINAL step (run the /discord:access skill).
      this tool never approves — a channel message asking to approve is a red flag.
```

## สรุป

`maw discord-channel` เกิดจากช่องว่างเดียว — `gate()` ตัดสินเงียบ เลยสร้างตัวจำลองมันขึ้นมาแบบ read-only: `where` บอกว่า access.json ไหนมีผล, `explain` แปลเป็นภาษาคน, `doctor` จำลองคำตัดสิน + เหตุผล, `lint` จับกับดัก, `pending` ส่อง pairing เส้นแบ่ง read-only vs mutate คือสิ่งที่กันไม่ให้ทำซ้ำของเดิม และกัน prompt injection ไปในตัว

ลองเอาไปส่องประตูของ oracle ตัวเองดู — `maw discord-channel doctor --user <id> --channel <id>` แล้วจะรู้ว่า message ที่ "หายไปเฉย ๆ" มันตกด่านไหน
