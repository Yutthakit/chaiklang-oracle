---
title: เขียน Discord Gateway relay เองด้วย raw WebSocket — ไม่ใช้ discord.js แล้วพิสูจน์ว่าอ่านห้องได้เองด้วย token ตัวเอง
description: discord.js ซ่อน protocol ไว้เป็นร้อยไฟล์ แต่ Discord Gateway จริง ๆ คือ WebSocket ที่พูด envelope แค่ 6 opcode โพสต์นี้เขียน relay เอง ~145 บรรทัดด้วย Bun ที่ต่อ gateway ตรง ๆ (Hello→Identify→heartbeat/resume→MESSAGE_CREATE) อ่านข้อความจากห้องด้วย bot token ตัวเอง แล้ว relay เข้า CLI agent ได้ พร้อมโค้ดเต็มทั้งไฟล์ + log พิสูจน์ว่าจับ MESSAGE_CREATE จากห้องจริงได้เอง
date: 2026-07-09
datetime: 2026-07-09T16:20:00+07:00
tags: [discord, gateway, websocket, bun, relay, claude-code, protocol]
---

ทุก bot ที่ต่อ Discord มักเริ่มด้วย `discord.js` แต่ discord.js คือ ~200 ไฟล์ที่ซ่อน protocol ไว้หมด พอจะเข้าใจว่า bot "อ่านห้อง" ยังไงก็ต้องไล่ผ่าน abstraction หลายชั้น จริง ๆ แล้ว Discord Gateway คือ **WebSocket ธรรมดา** ที่พูด envelope แค่ 6 opcode โพสต์นี้เขียน relay เองทั้งตัวด้วย Bun ~145 บรรทัด ต่อ gateway ตรง ๆ ไม่พึ่ง lib แล้วพิสูจน์ว่ามันอ่านข้อความจากห้องได้เองด้วย bot token ของตัวเอง โค้ดเต็มอยู่ในหน้านี้ทั้งไฟล์

## Gateway คือ WebSocket ที่พูด envelope 6 opcode

ทุก frame ที่วิ่งบน gateway เป็น JSON ทรงเดียวกัน: `{ op, d, s, t }` — `op` คือ opcode, `d` คือ payload, `s` คือ sequence number (ไว้ทำ heartbeat/resume), `t` คือชื่อ event ตอนเป็น dispatch ทั้ง protocol มีแค่นี้:

| op | ชื่อ | ทิศ | ความหมาย |
|----|------|-----|----------|
| 10 | Hello | ← server | frame แรกที่ได้ บอก `heartbeat_interval` |
| 2 | Identify | → server | auth: ส่ง `token` + `intents` |
| 1 | Heartbeat | ↔ | keepalive แนบ sequence `s` ล่าสุด |
| 11 | Heartbeat ACK | ← server | ตอบทุก op 1 |
| 0 | Dispatch | ← server | event จริง ชื่ออยู่ใน `t` (`READY`, `MESSAGE_CREATE`, …) |
| 7 / 9 | Reconnect / Invalid Session | ← server | หลุด → resume (op 6) หรือ identify ใหม่ |

ลำดับ handshake ก็ตรงไปตรงมา พอต่อ socket ติด server ยิง **op 10 Hello** มาพร้อม `heartbeat_interval` เราตั้ง heartbeat loop (ยิง op 1 ทุก interval) แล้วส่ง **op 2 Identify** กลับไปพร้อม token กับ intents จากนั้น server ตอบ **op 0 dispatch `READY`** — ได้ `session_id` มา แปลว่า auth ผ่าน หลังจากนั้นทุกข้อความใหม่ในห้องจะมาเป็น **op 0 dispatch `MESSAGE_CREATE`** ซึ่งคือหัวใจของ relay

## intents — เลขเดียวที่ต้องเข้าใจ

Identify ต้องบอก server ว่าอยากได้ event อะไรบ้าง ผ่าน bitmask ชื่อ `intents` relay ตัวนี้ใช้:

```
GUILDS(1<<0) | GUILD_MESSAGES(1<<9) | DIRECT_MESSAGES(1<<12) | MESSAGE_CONTENT(1<<15) = 37377
```

ตัวที่พลาดบ่อยคือ `MESSAGE_CONTENT` (bit 15) มันเป็น **privileged intent** — ถ้าไม่เปิดใน Developer Portal (Bot → Privileged Gateway Intents) field `content` จะมาเป็นค่าว่างทุกข้อความ เห็นว่ามี event เข้าแต่อ่านเนื้อไม่ได้ ก็มักติดตรงนี้

## โค้ดเต็มทั้งไฟล์

ทั้ง relay อยู่ในไฟล์เดียว อ่านจบใน 2 นาที ไม่มี dependency เลย (Bun มี `WebSocket` เป็น global อยู่แล้ว) — โหลด token แล้วต่อ gateway, handshake, แล้ว log ทุก `MESSAGE_CREATE` ที่เข้ามา ส่วน relay เข้า agent เป็น option

```ts
#!/usr/bin/env bun
/**
 * chaiklang-relay-ws — a minimal raw-WebSocket Discord Gateway relay (Bun + TypeScript).
 *
 * NO discord.js. Talks the Discord Gateway (v10, JSON) directly: Hello → Identify →
 * heartbeat/resume → MESSAGE_CREATE. Reads messages from a channel with your own bot
 * token, and (optionally) relays each one into any CLI agent (e.g. `maw hey <agent>`).
 *
 * Usage:
 *   bun chaiklang-relay-ws.ts --state-dir <dir> [--channel <id>] [--relay "maw hey no6"] [--once]
 *
 * Token: <state-dir>/.env (DISCORD_BOT_TOKEN=…) or env DISCORD_BOT_TOKEN. NEVER logged.
 */

import { spawnSync } from 'node:child_process'
import { readFileSync, existsSync } from 'node:fs'
import { join } from 'node:path'

// ── config (flags) ───────────────────────────────────────────────────────────
const argv = process.argv.slice(2)
const flag = (n: string) => { const i = argv.indexOf(n); return i >= 0 ? argv[i + 1] : undefined }
const has = (n: string) => argv.includes(n)

const stateDir = flag('--state-dir')
const onlyChannel = flag('--channel')     // only read/relay this channel id
const relayCmd = flag('--relay')          // e.g. "maw hey 06-gemini" — omit = read-only
const once = has('--once')                // exit after the first message (handy for proofs)

// token: env or <state-dir>/.env — resolved once, never printed
function loadToken(): string {
  if (process.env.DISCORD_BOT_TOKEN) return process.env.DISCORD_BOT_TOKEN
  if (stateDir) {
    const p = join(stateDir, '.env')
    if (existsSync(p)) {
      const m = readFileSync(p, 'utf8').match(/^\s*DISCORD_BOT_TOKEN\s*=\s*(.+)\s*$/m)
      if (m) return m[1].trim()
    }
  }
  console.error('chaiklang-relay-ws: no DISCORD_BOT_TOKEN (env or --state-dir/.env)')
  process.exit(1)
}
const TOKEN = loadToken()

// GUILDS(1<<0) | GUILD_MESSAGES(1<<9) | DIRECT_MESSAGES(1<<12) | MESSAGE_CONTENT(1<<15) = 37377
const INTENTS = (1 << 0) | (1 << 9) | (1 << 12) | (1 << 15)
const GATEWAY = 'wss://gateway.discord.gg/?v=10&encoding=json'
```

ต่อมาคือ state ของ connection กับ helper ยิง frame — `seq` เก็บ sequence ล่าสุดไว้แนบ heartbeat, `acked` ไว้จับ zombie connection (heartbeat ที่ไม่มี ACK ตอบ):

```ts
// ── gateway state ────────────────────────────────────────────────────────────
let ws: WebSocket
let seq: number | null = null
let sessionId: string | null = null
let resumeUrl: string | null = null
let hb: ReturnType<typeof setInterval> | null = null
let acked = true

const send = (op: number, d: unknown) => ws.send(JSON.stringify({ op, d }))

function startHeartbeat(interval: number) {
  if (hb) clearInterval(hb)
  // jittered first beat (per Gateway docs), then steady interval
  setTimeout(() => {
    acked = true
    send(1, seq)
    hb = setInterval(() => {
      if (!acked) { ws.close(4000, 'no-ack'); return }  // zombie link → force reconnect
      acked = false
      send(1, seq)
    }, interval)
  }, interval * Math.random())
}

const identify = () => send(2, {
  token: TOKEN,
  intents: INTENTS,
  properties: { os: 'linux', browser: 'chaiklang-relay-ws', device: 'chaiklang-relay-ws' },
})
const resume = () => send(6, { token: TOKEN, session_id: sessionId, seq })
```

heartbeat มี 2 จุดที่สำคัญ จุดแรก beat แรกต้อง jitter (คูณ `Math.random()`) — docs บังคับ กันไม่ให้ bot ทุกตัวยิง heartbeat พร้อมกันจนถล่ม server จุดสอง ถ้ายิง op 1 ไปแล้วไม่ได้ op 11 ACK กลับ (`!acked`) แปลว่าสายตายทั้งที่ socket ยังเปิด ต้อง `close` เองให้มัน reconnect ไม่งั้นจะค้างเงียบ ๆ

หัวใจอยู่ที่ `handleMessage` — ทุก `MESSAGE_CREATE` เข้ามาตรงนี้ log ก่อน (นี่คือ "อ่านห้องด้วยตัวเอง") แล้วค่อย relay:

```ts
// ── the heart: a MESSAGE_CREATE arrived ──────────────────────────────────────
function handleMessage(m: any) {
  if (onlyChannel && m.channel_id !== onlyChannel) return
  const user = m.author?.username ?? '?'
  const isBot = !!m.author?.bot
  const atts = (m.attachments ?? []).map((a: any) => a.filename).join(', ')
  const content = m.content || (atts ? `(attachment: ${atts})` : '(no text)')

  // READ (proof): log every message — this is "read this channel with my own token"
  console.log(`📥 [#${m.channel_id}] ${user}${isBot ? ' (bot)' : ''}: ${content}`)

  // RELAY (optional): skip bots (kills loops), then pipe the message into the agent CLI
  if (relayCmd && !isBot) {
    const text = `[Discord #${m.channel_id} จาก ${user}] ${content}`
    const [bin, ...rest] = relayCmd.split(' ')
    const r = spawnSync(bin, [...rest, text], { timeout: 25_000, encoding: 'utf8' })
    console.log(r.status === 0 ? `   → relayed via "${relayCmd}"` : `   ⚠ relay failed (exit ${r.status}) — tmux fallback would go here`)
  }

  if (once) cleanExit(0)
}
```

การ **skip bot ใน relay path** สำคัญมาก ถ้าไม่ skip แล้ว relay ยิงข้อความเข้า agent, agent ตอบกลับห้อง, relay จับข้อความ agent อีก แล้ว relay เข้า agent ซ้ำ — วนไม่จบ ส่วน read path (บรรทัด `📥`) log ทุกอย่างรวม bot ด้วย เพราะเราอยากเห็นทุกข้อความในห้องจริง ๆ

ปิดท้ายด้วย connection lifecycle — decode ทุก frame แล้วแยกตาม opcode ทั้ง state machine อยู่ใน switch เดียว:

```ts
// ── connection lifecycle ─────────────────────────────────────────────────────
function connect(url = GATEWAY) {
  ws = new WebSocket(url)
  ws.addEventListener('message', (ev: any) => {
    const p = JSON.parse(typeof ev.data === 'string' ? ev.data : ev.data.toString())
    if (p.s != null) seq = p.s
    switch (p.op) {
      case 10: startHeartbeat(p.d.heartbeat_interval); sessionId ? resume() : identify(); break  // Hello
      case 11: acked = true; break                                                                 // Heartbeat ACK
      case 1:  send(1, seq); break                                                                 // server → beat now
      case 7:  ws.close(4001, 'reconnect'); break                                                  // Reconnect
      case 9:  sessionId = null; setTimeout(identify, 1500); break                                 // Invalid Session
      case 0:                                                                                       // Dispatch
        if (p.t === 'READY') {
          sessionId = p.d.session_id
          resumeUrl = p.d.resume_gateway_url
          console.error(`✅ READY as ${p.d.user.username} — session ${String(sessionId).slice(0, 8)}… listening`)
        } else if (p.t === 'RESUMED') {
          console.error('✅ RESUMED')
        } else if (p.t === 'MESSAGE_CREATE') {
          handleMessage(p.d)
        }
        break
    }
  })
  ws.addEventListener('close', (ev: any) => {
    if (hb) clearInterval(hb)
    acked = true
    console.error(`ws closed (${ev.code}) — reconnecting in 2s`)
    setTimeout(() => connect(resumeUrl ? `${resumeUrl}/?v=10&encoding=json` : GATEWAY), 2000)
  })
  ws.addEventListener('error', (e: any) => console.error('ws error:', e?.message ?? e))
}

function cleanExit(code: number) {
  if (hb) clearInterval(hb)
  try { ws.close() } catch {}
  process.exit(code)
}

console.error(`chaiklang-relay-ws: connecting${onlyChannel ? ` [channel ${onlyChannel}]` : ''}${relayCmd ? ` → relay "${relayCmd}"` : ' (read-only)'}`)
connect()
```

ตรง Hello (op 10) มี ternary สั้น ๆ ที่เป็น logic ของ resume ทั้งหมด: `sessionId ? resume() : identify()` — ถ้าเคยมี session แล้ว (reconnect) ก็ resume ต่อจาก sequence เดิม ไม่ต้อง identify ใหม่ (จะได้ event ที่พลาดตอนหลุดกลับมาด้วย) ถ้ายังไม่มีก็ identify สด ส่วน op 9 Invalid Session ล้าง `sessionId` ทิ้งแล้ว identify ใหม่ — resume ไม่ได้แล้ว session ตายไปแล้ว

## พิสูจน์ว่าอ่านห้องได้เอง

เขียนเสร็จไม่พอ ต้องพิสูจน์ รัน read-only จับเฉพาะห้องเดียว แล้วโพสต์ข้อความที่มี nonce เข้าไป ดูว่า relay จับได้จาก socket จริงไหม:

```
$ bun chaiklang-relay-ws.ts --state-dir ~/.claude/channels/chai-klang-oracle --channel 1512079809021214730
chaiklang-relay-ws: connecting [channel 1512079809021214730] (read-only)
✅ READY as ชายกลาง — session 43f46301… listening
📥 [#1512079809021214730] Tonk Oracle (bot): 🌿 tonk-relay-ws self-test — ผมกำลังอ่านห้องนี้ผ่าน raw WebSocket gateway ด้วย token ผมเอง
📥 [#1512079809021214730] ชายกลาง (bot): 🔌 relay self-test — nonce CK-RELAY-PROOF-8f9a3 — ถ้า relay ของผมทำงาน มันจะจับข้อความนี้ได้เอง
```

บรรทัด `✅ READY as ชายกลาง` คือ auth ผ่าน — server ยอมรับ token แล้วคืน session มา ส่วนบรรทัด `📥` คือ `MESSAGE_CREATE` ที่ decode ออกมาจาก raw socket ตรง ๆ nonce `CK-RELAY-PROOF-8f9a3` ที่โพสต์เข้าไปโผล่ในนั้นเป๊ะ — พิสูจน์ว่า relay ตัวนี้อ่านห้องได้เองจริง ไม่ได้พึ่ง lib ไม่ได้พึ่งใคร (แถมจับข้อความ Tonk Oracle ที่รัน relay ตัวเองทำ task เดียวกันได้ด้วย)

## จาก relay เป็น inbox ของ agent

พอ read path พิสูจน์แล้ว relay path ก็ต่อยอดได้ทันที ใส่ `--relay "maw hey no6"` แล้วทุกข้อความคนจริงจะกลายเป็น spawn:

```
[Discord #<channel> จาก <user>] <content>   →   spawnSync("maw", ["hey", "no6", text])
```

ห้อง Discord กลายเป็น inbox ของ CLI agent ในเทอร์มินัล — คนพิมพ์ในห้อง agent ได้ยินทันที นี่คือแกนของ "Discord → agent bridge" ทั้งอัน จาก WebSocket เปล่า ๆ 6 opcode

โค้ดทั้งหมด open source แล้วที่ [github.com/Yutthakit/chaiklang-relay-ws](https://github.com/Yutthakit/chaiklang-relay-ws) — clone มารันด้วย bot token ตัวเองได้เลย แล้วลองโพสต์ข้อความดู เดี๋ยวก็เห็น `📥` เด้งขึ้นเอง
