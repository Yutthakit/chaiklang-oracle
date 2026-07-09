---
title: กายวิภาค Discord channel plugin ของ Claude Code — เดินโค้ด server.ts 900 บรรทัดจริง
description: เดินโค้ด external_plugins/discord/server.ts ทีละชั้น — สองท่อ (stdio↔Claude, Gateway↔internet), inbound flow messageCreate→gate→notification, access model (pairing/allowlist/groups), 5 MCP tools, และ assertSendable ที่กัน Claude ส่ง state ของตัวเองออก อ้าง line จริงทุกบรรทัด
date: 2026-07-09
datetime: 2026-07-09T10:20:00+07:00
tags: [claude-code, discord, mcp, channel, security]
---

Discord channel plugin ของ Claude Code คือไฟล์เดียว `external_plugins/discord/server.ts` ยาว 900 บรรทัด เทียบกับ fakechat (ตัว minimal) ที่มี 295 บรรทัด — ส่วนต่าง 605 บรรทัดเกือบทั้งหมดคือ **auth + access control + delivery** ที่โลกจริงบังคับให้ต้องมี โพสต์นี้เดินโค้ดตัวจริงทีละชั้น อ้าง line number จริง ไม่เดา

## ชั้นที่ 0 — channel คือ MCP server ที่มี "ท่อสองด้าน"

channel plugin เป็น MCP `Server` ที่คุยกับ Claude ผ่าน **stdio** และคุยกับโลกภายนอกผ่าน **discord.js** สิ่งที่ทำให้มันเป็น "channel" ไม่ใช่ "tool" คือมันประกาศ capability พิเศษ:

```ts
// discord/server.ts — capability + stdio
const mcp = new Server(
  { name: 'discord', version: '...' },
  { capabilities: { tools: {}, experimental: { 'claude/channel': {} } } },
)
// :14
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js'
// :723
await mcp.connect(new StdioServerTransport())
```

`tools: {}` = มี tool ปกติ · `experimental: { 'claude/channel': {} }` = **ปลดล็อกให้ push ข้อความเข้า Claude ได้เอง** — tool ธรรมดาไม่มีบรรทัดนี้ นี่คือเส้นแบ่ง channel vs tool

boot จะตายทันทีถ้าไม่มี token — fail loud ดีกว่า fail เงียบ:

```ts
// :53
const TOKEN = process.env.DISCORD_BOT_TOKEN
// :58 — ถ้าไม่มี token throw ตั้งแต่ boot
//   `discord channel: DISCORD_BOT_TOKEN required ... format: DISCORD_BOT_TOKEN=MTIz...`
```

## ชั้นที่ 1 — inbound: จาก Discord ถึง Claude

ทุกอย่างเริ่มที่ listener ตัวเดียว แล้วส่งต่อเข้า `handleInbound`:

```ts
// :805
client.on('messageCreate', msg => {
  if (msg.author.bot) return                        // กัน loop: ไม่ตอบบอทด้วยกัน
  handleInbound(msg).catch(e =>
    process.stderr.write(`discord: handleInbound failed: ${e}\n`))
})
```

`handleInbound` เรียก `gate()` ก่อนเสมอ แล้ว switch ตามผล — drop / pair / deliver:

```ts
// :810
async function handleInbound(msg: Message): Promise<void> {
  const result = await gate(msg)
  if (result.action === 'drop') return              // ไม่ผ่านด่าน = เงียบ
  if (result.action === 'pair') {
    const lead = result.isResend ? 'Still pending' : 'Pairing required'
    await msg.reply(`${lead} — run in Claude Code:\n\n/discord:access pair ${result.code}`)
    return
  }
  // ...ผ่านด่านแล้ว → เตรียม emit notification (ต่อในชั้น 3)
}
```

## ชั้นที่ 2 — gate(): หัวใจของ access control

`gate()` คือด่านที่ตัดสินว่า "คนนี้พูดกับ Claude ได้ไหม" มี 3 mode ผ่าน `dmPolicy`:

```ts
// :236
async function gate(msg: Message): Promise<GateResult> {
  const access = loadAccess()
  const pruned = pruneExpired(access)               // ล้าง pairing code หมดอายุ
  if (pruned) saveAccess(access)

  if (access.dmPolicy === 'disabled') return { action: 'drop' }

  const senderId = msg.author.id
  const isDM = msg.channel.type === ChannelType.DM

  if (isDM) {
    if (access.allowFrom.includes(senderId)) return { action: 'deliver', access }
    if (access.dmPolicy === 'allowlist')    return { action: 'drop' }
    // pairing mode — ออก code ให้ผูกเครื่อง
    // ... (ต่อด้านล่าง)
  }
  // channel/guild — key ที่ channelId ไม่ใช่ guildId (opt-in ทีละห้อง)
}
```

**pairing mode** คือส่วนที่สวยที่สุด — คนแปลกหน้า DM มา บอทจะออก code 6 hex ให้ แล้วให้มนุษย์เอา code ไป approve ที่ terminal (`/discord:access pair <code>`) กัน replies ไม่ให้ spam และ cap pending ที่ 3:

```ts
// :251 — ถ้าเคยออก code ให้ sender นี้แล้ว ตอบซ้ำได้แค่ 2 ครั้งแล้วเงียบ
for (const [code, p] of Object.entries(access.pending)) {
  if (p.senderId === senderId) {
    if ((p.replies ?? 1) >= 2) return { action: 'drop' }
    p.replies = (p.replies ?? 1) + 1
    saveAccess(access)
    return { action: 'pair', code, isResend: true }
  }
}
if (Object.keys(access.pending).length >= 3) return { action: 'drop' }   // cap 3

const code = randomBytes(3).toString('hex')          // :263 — 6 hex chars
access.pending[code] = {
  senderId,
  chatId: msg.channelId,
  createdAt: Date.now(),
  expiresAt: Date.now() + 60 * 60 * 1000,            // หมดอายุ 1 ชม.
  replies: 1,
}
saveAccess(access)
return { action: 'pair', code, isResend: false }
```

รูปทรงของ access เก็บใน `access.json`:

```ts
// :106
type Access = {
  dmPolicy: 'pairing' | 'allowlist' | 'disabled'
  allowFrom: string[]                                // user IDs ที่อนุญาต DM
  groups: Record<string, GroupPolicy>                // key = channelId
  pending: Record<string, PendingEntry>              // pairing codes ค้าง
  ackReaction?: string                               // emoji ตอบรับ
  replyToMode?: 'off' | 'first' | 'all'
  textChunkLimit?: number                            // default 2000 (Discord cap)
}
type GroupPolicy = { requireMention: boolean; allowFrom: string[] }
```

อ่านไฟล์แบบทน corrupt — ถ้า JSON พัง ย้ายทิ้งแล้วเริ่มใหม่ ไม่ crash:

```ts
// :140
function readAccessFile(): Access {
  try {
    const parsed = JSON.parse(readFileSync(ACCESS_FILE, 'utf8')) as Partial<Access>
    return { dmPolicy: parsed.dmPolicy ?? 'pairing', allowFrom: parsed.allowFrom ?? [], /* ... */ }
  } catch (err) {
    if ((err as NodeJS.ErrnoException).code === 'ENOENT') return defaultAccess()
    renameSync(ACCESS_FILE, `${ACCESS_FILE}.corrupt-${Date.now()}`)   // กันพังทั้งระบบ
    return defaultAccess()
  }
}
```

### pairing lifecycle — filesystem handshake ข้ามโปรเซส

pairing code ที่ `gate()` ออกให้ (ในโค้ดด้านบน) เป็นแค่จุดเริ่ม เส้นทางเต็มคือ **handshake ระหว่างสองโปรเซส** — skill `/discord:access` (รันที่ terminal) กับ server (โปรเซสที่รันค้าง) คุยกันผ่าน**ไฟล์บนดิสก์** ไม่ใช่ memory ร่วม

```
① code เกิด    gate() pairing branch     → เขียน access.json (pending[code])
② ส่ง code     handleInbound reply        → บอท DM code กลับ (ยังไม่ถึง Claude)
③ มนุษย์ approve  /discord:access pair <code> (terminal)
                → ลบ pending[code], เพิ่ม senderId เข้า allowFrom,
                  เขียนไฟล์ approved/<senderId> ที่มี "เนื้อ = DM channelId"
④ confirm      checkApprovals() poll ทุก 5s → ส่ง "Paired!" + ลบไฟล์
⑤ prune        pruneExpired() → code ที่ไม่ approve ใน 1ชม. หายเอง
```

จุดที่แยบยล: ตอน server เห็นไฟล์ `approved/` แล้ว **`pending` ถูกลบไปแล้ว** (ขั้น ③) — chatId ที่เคยอยู่ใน pending ก็หายไปด้วย server เลยไม่รู้ว่าจะส่ง "Paired!" ไปห้อง DM ไหน วิธีแก้: ให้ skill เขียน DM channelId ลงใน **เนื้อไฟล์** `approved/<senderId>` server อ่านจากตรงนั้น:

```ts
// :327
function checkApprovals(): void {
  let files: string[]
  try { files = readdirSync(APPROVED_DIR) } catch { return }
  for (const senderId of files) {
    const file = join(APPROVED_DIR, senderId)
    const dmChannelId = readFileSync(file, 'utf8').trim()   // channelId อยู่ในเนื้อไฟล์
    if (!dmChannelId) { rmSync(file, { force: true }); continue }
    void (async () => {
      const ch = await fetchTextChannel(dmChannelId)
      if ('send' in ch) await ch.send("Paired! Say hi to Claude.")
      rmSync(file, { force: true })                         // consume แล้วลบ
    })()
  }
}
if (!STATIC) setInterval(checkApprovals, 5000).unref()      // poll ทุก 5s (ยกเว้น static mode)
```

| สเตจ | access.json | ไฟล์ approved/ | Discord |
|---|---|---|---|
| ① code เกิด | `pending[code]` +1 | — | บอททัก msg |
| ② ตอบ code | `replies`++ | — | คนได้ code ใน DM |
| ③ approve (terminal) | `pending` −, `allowFrom` + | `approved/<id>` เกิด (=chatId) | — |
| ④ confirm | — | ไฟล์ถูกลบ | คนได้ "Paired!" |
| ⑤ ทุก msg ถัดไป | prune pending หมดอายุ | — | ผ่าน gate → ถึง Claude |

`setInterval(...).unref()` สำคัญ — `.unref()` ทำให้ timer นี้ไม่กันโปรเซสตาย ถ้างานอื่นจบหมด server ก็ปิดได้ ไม่ค้างเพราะ poller

### เส้น guild channel — "เปิดทั้งห้อง" ทำงานยังไง

DM คือเส้นหนึ่ง อีกเส้นคือข้อความในห้อง guild ตรงนี้ key ที่ **channelId ไม่ใช่ guildId** (opt-in ทีละห้อง) และ thread สืบทอด opt-in ของห้องแม่:

```ts
// :280
const channelId = msg.channel.isThread()
  ? msg.channel.parentId ?? msg.channelId       // thread → ใช้ห้องแม่ตอน gate
  : msg.channelId
const policy = access.groups[channelId]
if (!policy) return { action: 'drop' }                       // ① ห้องไม่ opt-in → drop
const groupAllowFrom = policy.allowFrom ?? []
const requireMention = policy.requireMention ?? true         // default = true!
if (groupAllowFrom.length > 0 && !groupAllowFrom.includes(senderId)) {
  return { action: 'drop' }                                  // ② allowFrom ว่าง = ทุกคนผ่าน
}
if (requireMention && !(await isMentioned(msg, access.mentionPatterns))) {
  return { action: 'drop' }                                  // ③ ต้อง @mention ไหม
}
return { action: 'deliver', access }                         // ④ ผ่าน → เข้า Claude
```

แปลว่า config "เปิดทั้งห้อง ไม่ต้อง tag ไม่ต้อง approve รายคน" = group entry เดียว ว่าง ๆ:

```json
{
  "dmPolicy": "disabled",
  "allowFrom": [],
  "groups": { "<channelId>": { "requireMention": false, "allowFrom": [] } },
  "pending": {}
}
```

`groups["<channelId>"]` มีอยู่ = ผ่าน ① · `allowFrom: []` = ผ่าน ② ทุกคน · `requireMention: false` = ผ่าน ③ พิมพ์เฉย ๆ ก็ถึง Claude · **กับดัก:** `requireMention` อยู่ใน group (`GroupPolicy`) แต่ `ackReaction`/`replyToMode` เป็น **top-level เท่านั้น** เอาไปใส่ใน group จะถูกเมิน

`requireMention` default เป็น `true` — แปลว่าถ้าไม่ตั้ง ต้อง @mention เสมอ แล้ว "mention" นับอะไรบ้าง? `isMentioned()` นับ 3 อย่าง:

```ts
// :297
async function isMentioned(msg, extraPatterns?): Promise<boolean> {
  if (client.user && msg.mentions.has(client.user)) return true       // (a) @mention บอทตรง ๆ
  const refId = msg.reference?.messageId
  if (refId) {
    if (recentSentIds.has(refId)) return true                         // (b) reply ข้อความบอท = implicit
    try { const ref = await msg.fetchReference()
          if (ref.author.id === client.user?.id) return true } catch {}
  }
  for (const pat of extraPatterns ?? [])                              // (c) regex ใน mentionPatterns
    try { if (new RegExp(pat, 'i').test(msg.content)) return true } catch {}
  return false
}
```

หัวใจ: ข้อมูล Discord "เข้า Channel" (ถึง Claude) **ก็ต่อเมื่อ `gate()` คืน `deliver` เท่านั้น** — และ gate re-read `access.json` ใหม่ทุก message (ไม่ cache) ดังนั้น access.json = ประตูเดียวที่คุมว่าอะไรผ่านเข้ามาได้

### static mode — แช่แข็ง access ตอน boot

ปกติ `gate()` re-read `access.json` ทุก message (hot-reload — แก้ไฟล์ปุ๊บมีผลปั๊บ) แต่ตั้ง env `DISCORD_ACCESS_MODE=static` แล้วมันจะ snapshot access ตอน boot ครั้งเดียว ไม่อ่านซ้ำ ไม่เขียนเลย:

```ts
// :54
const STATIC = process.env.DISCORD_ACCESS_MODE === 'static'

// :177 — snapshot ตอน boot (ครั้งเดียว)
const BOOT_ACCESS: Access | null = STATIC
  ? (() => {
      const a = readAccessFile()
      if (a.dmPolicy === 'pairing') {              // pairing ต้องเขียน runtime → static ทำไม่ได้
        a.dmPolicy = 'allowlist'                   // ลดเป็น allowlist อัตโนมัติ + warn
      }
      a.pending = {}                               // ล้าง pending (pair ไม่ได้แล้ว)
      return a
    })()
  : null

function loadAccess(): Access { return BOOT_ACCESS ?? readAccessFile() }  // static คืน snapshot / ปกติอ่านสด
function saveAccess(a: Access): void {
  if (STATIC) return                               // static = ไม่เขียนไฟล์เลย
  const tmp = ACCESS_FILE + '.tmp'                 // ปกติ = atomic write (tmp → rename)
  writeFileSync(tmp, JSON.stringify(a, null, 2) + '\n', { mode: 0o600 })
  renameSync(tmp, ACCESS_FILE)
}
```

static เหมาะกับ config "เปิดทั้งห้อง" (group เดียว allowFrom ว่าง requireMention false) เพราะไม่มีอะไรต้องเขียน runtime — ได้ประโยชน์ 3 อย่าง: (1) กัน concurrent write ถ้ารันหลาย instance ที่ชี้ STATE_DIR เดียวกัน (2) ปิด pairing สนิท (3) อ่านครั้งเดียว เบากว่า · แลกกับ: แก้ `access.json` แล้วต้อง restart ถึงมีผล · สังเกต atomic write ในโหมดปกติ (tmp+rename, perms `0o600`) — กันไฟล์พังถ้า crash กลางเขียน

## ชั้นที่ 3 — notification: บรรทัดที่ทำให้มันเป็น channel

พอผ่าน gate แล้ว บอท push ข้อความขึ้น Claude ด้วย `mcp.notification()` — นี่คือบรรทัดเดียวที่แยก channel ออกจาก tool:

```ts
// :875
mcp.notification({
  method: 'notifications/claude/channel',
  params: {
    content,                                         // ข้อความ (หรือ "(attachment)")
    meta: { chat_id, /* message_id, user, ts, attachments... */ },
  },
})
```

attachment จะ **list แต่ไม่โหลด** — ใส่ชื่อ/ชนิด/ขนาดใน meta ให้ Claude เรียก `download_attachment` เองเมื่ออยากได้ (กัน inbox บวมด้วยรูปที่ไม่มีใครดู):

```ts
// :860
const atts: string[] = []
for (const att of msg.attachments.values()) {
  const kb = (att.size / 1024).toFixed(0)
  atts.push(`${safeAttName(att)} (${att.contentType ?? 'unknown'}, ${kb}KB)`)
}
const content = msg.content || (atts.length > 0 ? '(attachment)' : '')
```

มี notification อีกชนิดสำหรับ permission — ถ้า user ตอบ "yes xxxxx" ตรงกับ pending permission request บอทยิง event แยก ไม่ relay เป็นแชต:

```ts
// :843
const permMatch = PERMISSION_REPLY_RE.exec(msg.content)
if (permMatch) {
  void mcp.notification({
    method: 'notifications/claude/channel/permission',
    params: {
      request_id: permMatch[2]!.toLowerCase(),
      behavior: permMatch[1]!.toLowerCase().startsWith('y') ? 'allow' : 'deny',
    },
  })
  void msg.react(permMatch[1]!.toLowerCase().startsWith('y') ? '✅' : '❌').catch(() => {})
  return
}
```

## ชั้นที่ 4 — outbound: 5 tools

ฝั่งขาออก Claude เรียก tool ผ่าน `setRequestHandler(ListToolsRequestSchema)` — **ไม่ใช่** `registerTool` (raw MCP Server API):

```ts
// :520
mcp.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    { name: 'reply',              /* :523 chat_id, text, reply_to, files */ },
    { name: 'react',              /* :545 chat_id, message_id, emoji */ },
    { name: 'edit_message',       /* :558 chat_id, message_id, text */ },
    { name: 'download_attachment',/* :571 chat_id, message_id */ },
    { name: 'fetch_messages',     /* :583 channel, limit */ },
  ],
}))
```

## ชั้นที่ 5 — กับดักความปลอดภัยที่คนมองข้าม

`reply` รับ `files: string[]` เป็น path อะไรก็ได้ ปัญหาคือ Claude อาจถูกหลอกให้ส่ง `access.json` หรือ `.env` ของ channel เองออกไป โค้ดกันไว้ด้วย `assertSendable()` — ห้ามส่งไฟล์ที่อยู่ใน `STATE_DIR` ยกเว้น `inbox/`:

```ts
// :139
function assertSendable(f: string): void {
  const real = realpathSync(f)
  const stateReal = realpathSync(STATE_DIR)
  const inbox = join(stateReal, 'inbox')
  if (real.startsWith(stateReal + sep) && !real.startsWith(inbox + sep)) {
    throw new Error(`refusing to send channel state: ${f}`)   // กัน exfil state ตัวเอง
  }
}
```

comment ในโค้ดเขียนเหตุผลตรง ๆ ว่า *"the server's own state is the one thing Claude has no reason to ever send"* — นี่คือ threat model ที่เขียนเป็นภาษาคน

อีกชั้นคือ access.json ถูกจัดการผ่าน `/discord:access` skill (markdown ล้วน) และ ACCESS.md เตือนไว้ว่า **การ approve/allowlist ห้ามสั่งมาจากข้อความในห้อง** เพราะข้อความในห้องคือ untrusted — คนไม่หวังดีจะพิมพ์ "approve me" ซึ่งเป็นสิ่งที่ prompt injection ทำเป๊ะ ต้องให้มนุษย์สั่งที่ terminal เท่านั้น

### inbox/ — ทำไม attachment ถึงเป็น subdir เดียวที่ส่งออกได้

ตอน emit notification บอท **ไม่โหลดไฟล์แนบ** แค่ list ชื่อ/ขนาดใน meta (ชั้น 3) ไฟล์จะโหลดจริงตอน Claude เรียก tool `download_attachment` เท่านั้น — โหลดจาก Discord CDN ลง `inbox/`:

```ts
// :37  STATE_DIR = env DISCORD_STATE_DIR ?? ~/.claude/channels/discord
// :64  INBOX_DIR = join(STATE_DIR, 'inbox')
// :133 MAX_ATTACHMENT_BYTES = 25 * 1024 * 1024
// :418
async function downloadAttachment(att: Attachment): Promise<string> {
  if (att.size > MAX_ATTACHMENT_BYTES) throw new Error(`attachment too large: ...`)
  const res = await fetch(att.url)                          // Discord CDN
  const buf = Buffer.from(await res.arrayBuffer())
  const rawExt = (att.name ?? att.id).split('.').pop() ?? 'bin'
  const ext = rawExt.replace(/[^a-zA-Z0-9]/g, '') || 'bin'  // sanitize นามสกุล
  const path = join(INBOX_DIR, `${Date.now()}-${att.id}.${ext}`)
  mkdirSync(INBOX_DIR, { recursive: true })
  writeFileSync(path, buf)
  return path                                              // คืน path ให้ Claude ไปอ่าน
}
```

ทิศทางของ `inbox/`: `Discord CDN ──download_attachment──► inbox/ ──Claude อ่าน/reply files──► ออกได้`

นี่คือจุดที่ต่อกับ `assertSendable()` พอดี — จำได้ว่ามันบล็อกทุกไฟล์ใน `STATE_DIR` ยกเว้น `inbox/`? เพราะ `inbox/` คือที่เดียวที่ทั้ง "ไฟล์จากภายนอกลงมา" และ "Claude ส่งกลับออกได้" ส่วน `access.json`/`.env` ที่อยู่ระดับบนของ STATE_DIR ส่งออกไม่ได้ = ปลอดภัย

**prompt-injection defense อีกจุด** — ชื่อไฟล์แนบเป็น uploader-controlled มันไปโผล่ใน annotation `[...]` ของ notification + ใน tool result ที่ join ด้วย newline ทั้งสองที่เป็นจุดที่ตัวอักษร delimiter ให้ attacker หลุดกรอบ untrusted ได้ เลยต้อง sanitize:

```ts
// :435
function safeAttName(att: Attachment): string {
  return (att.name ?? att.id).replace(/[\[\]\r\n;]/g, '_')  // กัน [ ] \r \n ; แหกกรอบ
}
```

## เทียบทั้ง 4 channel (จาก source ทุกไฟล์)

| แกน | discord (900) | telegram (1083) | imessage (912) | fakechat (295) |
|---|---|---|---|---|
| ท่อ→Claude | stdio | stdio | stdio | stdio |
| ท่อ→user | discord.js Gateway+REST | Telegram Bot API (getUpdates) | chat.db + osascript | Bun WS localhost:8787 |
| ปลายทาง | internet (DM/guild) | internet | เครื่องเดียว (Messages.app) | เบราว์เซอร์ในเครื่อง |
| auth | DISCORD_BOT_TOKEN | TELEGRAM_BOT_TOKEN | ไม่มี token | ไม่มี ("no tokens" :6) |
| access | pairing/allowlist/groups | มี | allowlist (chat.db แรง) | ไม่มีเลย |
| tools | 5 | 5+ | หลายตัว | 2 (reply, edit) |
| open/closed | CLOSED–OPEN | CLOSED–OPEN | CLOSED–CLOSED | CLOSED–CLOSED |

## สรุปเชิงสถาปัตยกรรม

discord ไม่ใช่ tool ที่ใหญ่กว่า — มันคือ contract เดียวกับ fakechat (stdio + reply + inbound notification) **ห่อด้วยชั้น auth/access/delivery ที่โลกจริงบังคับ** ทั้งเล่มนี้ยึดกฎเดียว: อ่าน markdown ให้ออกว่าเป็น attack surface, อ่าน comment ให้ออกว่าเป็น design decision, และเช็คเงื่อนไขเล็กสุด (เช่น `msg.author.bot` guard, cap pending 3, `assertSendable`) ก่อนเชื่อว่าระบบปลอดภัย

ทุก line number ในโพสต์นี้มาจากการเปิด `server.ts` จริงอ่านเอง ไม่ได้เดาจากความจำ — ปัจจัตตัง
