---
title: ถอด Discord channel เหลือแก่น แล้วสลับ transport เป็น MQTT — channel plugin 154 บรรทัดที่ test ผ่าน broker จริง
description: Claude Code channel plugin ไม่ได้ผูกกับ Discord — มันคือสัญญา (MCP Server + claude/channel capability + notification + reply tool) ที่ transport อะไรก็เสียบได้ โพสต์นี้ถอด discord/server.ts 900 บรรทัดเหลือแก่น แล้วสลับ Gateway เป็น MQTT ได้ mqtt-channel 154 บรรทัด พร้อมโค้ดเต็มทุกไฟล์ + end-to-end test ที่รันผ่าน mosquitto จริง (INBOUND publish→notification, OUTBOUND reply→publish)
date: 2026-07-09
datetime: 2026-07-09T14:30:00+07:00
tags: [claude-code, mcp, channel, mqtt, iot, mosquitto, transport]
---

Claude Code channel plugin ตัวจริง (Discord) ยาว 900 บรรทัด แต่ **ส่วนที่เป็น "channel" จริง ๆ มีนิดเดียว** ที่เหลือคือ auth/access control ที่ Discord บังคับให้มี โพสต์นี้พิสูจน์ว่า channel เป็น **สัญญา ไม่ใช่ Discord** — ถอดแก่นออกมาแล้วสลับ transport จาก Discord Gateway เป็น **MQTT** ได้ channel plugin ตัวใหม่ 154 บรรทัด แล้ว test ผ่าน broker จริง โค้ดครบทุกไฟล์อยู่ในหน้านี้

## ก่อนสลับ — อ่านว่า Discord "รับ" ยังไง

เปิด `discord/server.ts` ตัวจริง ส่วนรับ message เข้ามาคือ:

```ts
// discord/server.ts — inbound
client.on('messageCreate', msg => {
  if (msg.author.bot) return                 // กัน loop
  handleInbound(msg).catch(...)
})
// ปลายทางของ handleInbound:
mcp.notification({
  method: 'notifications/claude/channel',
  params: {
    content: msg.content,
    meta: { chat_id, message_id: msg.id, user: msg.author.username, user_id: msg.author.id, ts: ... },
  },
})
```

ส่วนส่งออก คือ `ch.send({ content })` เท่านั้นเอง ทุกอย่างที่เหลือ (gate, access.json, pairing, permission, static mode) เป็นชั้น **auth** ไม่ใช่ชั้น transport

**แก่นของ channel มี 4 ชิ้น:**
1. MCP `Server` + capability `experimental: { 'claude/channel': {} }`
2. inbound → `mcp.notification({ method: 'notifications/claude/channel', params: { content, meta } })`
3. outbound → tool ชื่อ `reply`
4. `connect(new StdioServerTransport())`

Discord เอา discord.js มาต่อ 4 ชิ้นนี้ เราแค่เปลี่ยนตัวต่อเป็น MQTT

## การสลับ — Gateway → MQTT

| | Discord | MQTT |
|---|---------|------|
| inbound | `client.on('messageCreate', ...)` | `client.on('message', (topic, payload) => ...)` (subscribe) |
| outbound | `ch.send({ content })` | `client.publish(OUT_TOPIC, payload)` |
| identity | `msg.author.id` (มี user จริง) | ไม่มี — topic ไม่มีตัวตน → auth ไปไว้ที่ broker |
| notification | เหมือนกันเป๊ะ | เหมือนกันเป๊ะ |

`notifications/claude/channel` **ไม่แตะเลย** — นั่นคือจุดที่พิสูจน์ว่า Claude ไม่รู้ด้วยซ้ำว่าอีกฝั่งเป็น Discord หรือ MQTT

## โค้ดเต็ม — `server.ts` (154 บรรทัด)

```ts
#!/usr/bin/env bun
/**
 * Minimal MQTT channel for Claude Code.
 * Same channel contract as the Discord plugin — only the transport is swapped.
 */
import { Server } from '@modelcontextprotocol/sdk/server/index.js'
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js'
import { ListToolsRequestSchema, CallToolRequestSchema } from '@modelcontextprotocol/sdk/types.js'
import mqtt from 'mqtt'

const URL = process.env.MQTT_URL ?? 'mqtt://localhost:1883'
const IN_TOPIC = process.env.MQTT_IN_TOPIC ?? 'claude/in'
const OUT_TOPIC = process.env.MQTT_OUT_TOPIC ?? 'claude/out'

let seq = 0
const nextId = () => `m${Date.now()}-${++seq}`

const mcp = new Server(
  { name: 'mqtt-channel', version: '0.1.0' },
  {
    capabilities: { tools: {}, experimental: { 'claude/channel': {} } },
    instructions:
      `The sender reads MQTT (an external bridge / device), not this session. Anything you want ` +
      `them to see must go through the reply tool — it publishes to "${OUT_TOPIC}".\n\n` +
      `Messages arrive as <channel source="mqtt" chat_id="..." message_id="..." user="...">. ` +
      `chat_id is the inbound topic (or the payload's chat_id). Reply with the reply tool, passing chat_id back.`,
  },
)

const client = mqtt.connect(URL, {
  username: process.env.MQTT_USERNAME,
  password: process.env.MQTT_PASSWORD,
})

// ── tools ─────────────────────────────────────────────────────────────
mcp.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: 'reply',
      description: `Publish a message to MQTT (topic "${OUT_TOPIC}"). Pass chat_id and text.`,
      inputSchema: {
        type: 'object',
        properties: { chat_id: { type: 'string' }, text: { type: 'string' } },
        required: ['chat_id', 'text'],
      },
    },
  ],
}))

mcp.setRequestHandler(CallToolRequestSchema, async req => {
  try {
    if (req.params.name !== 'reply')
      return { content: [{ type: 'text', text: `unknown: ${req.params.name}` }], isError: true }

    const a = (req.params.arguments ?? {}) as Record<string, unknown>
    const id = nextId()
    const payload = JSON.stringify({
      id,
      chat_id: String(a.chat_id ?? ''),
      text: String(a.text ?? ''),
      from: 'assistant',
      ts: new Date().toISOString(),
    })
    await new Promise<void>((resolve, reject) =>
      client.publish(OUT_TOPIC, payload, { qos: 1 }, err => (err ? reject(err) : resolve())),
    )
    return { content: [{ type: 'text', text: `sent (id: ${id})` }] }
  } catch (err) {
    return {
      content: [{ type: 'text', text: `${req.params.name}: ${err instanceof Error ? err.message : err}` }],
      isError: true,
    }
  }
})

// ── inbound: MQTT → Claude (mirrors discord messageCreate → notification) ──
client.on('message', (topic, payload) => {
  const raw = payload.toString()
  let text = raw, chatId = topic, user = 'mqtt', userId = topic, msgId = nextId()

  try {
    const j = JSON.parse(raw)
    if (j && typeof j === 'object') {
      if (j.from === 'assistant') return // ignore our own echoes — loop guard
      text = String(j.text ?? j.content ?? raw)
      chatId = String(j.chat_id ?? topic)
      user = String(j.user ?? 'mqtt')
      userId = String(j.user_id ?? user)
      if (j.id) msgId = String(j.id)
    }
  } catch { /* not JSON → treat as raw text */ }

  void mcp
    .notification({
      method: 'notifications/claude/channel',
      params: {
        content: text || '(no text)',
        meta: { chat_id: chatId, message_id: msgId, user, user_id: userId, ts: new Date().toISOString(), topic },
      },
    })
    .catch(e => process.stderr.write(`mqtt-channel: deliver failed: ${e}\n`))
})

client.on('connect', () => {
  client.subscribe(IN_TOPIC, { qos: 1 }, err => {
    if (err) return process.stderr.write(`mqtt-channel: subscribe ${IN_TOPIC} failed: ${err}\n`)
    process.stderr.write(`mqtt-channel: connected ${URL} — in:${IN_TOPIC} out:${OUT_TOPIC}\n`)
  })
})
client.on('error', e => process.stderr.write(`mqtt-channel: ${e}\n`))

await mcp.connect(new StdioServerTransport())
```

จุดที่ต่างจาก Discord ตรง inbound: payload ของ MQTT เป็น **string ดิบหรือ JSON ก็ได้** — ถ้าเป็น JSON `{ text, chat_id, user, id }` ก็ map เข้า meta, ถ้าเป็น string เฉย ๆ ก็เอาทั้งก้อนเป็น content เหมาะกับ sensor/device ที่ยิง MQTT ดิบ ๆ · `from: "assistant"` ถูก ignore กัน loop เวลามีคน bridge in↔out

## ทำให้เป็น channel plugin — 2 ไฟล์ config

`.mcp.json` (Claude Code start ผ่าน stdio):

```json
{
  "mcpServers": {
    "mqtt-channel": {
      "command": "bun",
      "args": ["run", "--cwd", "${CLAUDE_PLUGIN_ROOT}", "--shell=bun", "--silent", "start"]
    }
  }
}
```

`package.json`:

```json
{
  "name": "claude-channel-mqtt",
  "type": "module",
  "bin": "./server.ts",
  "scripts": { "start": "bun install --no-summary && bun server.ts" },
  "dependencies": { "@modelcontextprotocol/sdk": "^1.0.0", "mqtt": "^5.10.0" }
}
```

## Setup broker — Mosquitto localhost

`mosquitto.conf`:

```text
listener 1883 127.0.0.1
allow_anonymous true
```

รัน:

```bash
brew install mosquitto            # ถ้ายังไม่มี
mosquitto -c mosquitto.conf &     # localhost:1883, anonymous (dev only)
```

## test เต็ม — รันผ่าน broker จริง ไม่ mock

test harness spawn `server.ts` เป็น subprocess แล้วคุยกับมันแบบ **MCP client จริงผ่าน stdio** (handshake → tools/list → tools/call) พร้อม pub/sub MQTT จริงผ่าน broker ทดสอบสองทาง:

```ts
// test.ts (ตัดส่วน assert หลัก)
// 1. MCP handshake
send({ jsonrpc: '2.0', id: 1, method: 'initialize', params: {...} })
await waitFor(m => m.id === 1 && m.result, 'initialize')
send({ jsonrpc: '2.0', method: 'notifications/initialized' })

// 2. tools/list ต้องมี reply
send({ jsonrpc: '2.0', id: 2, method: 'tools/list' })
// → ['reply']

// 3. INBOUND — publish claude/in → รอ notifications/claude/channel
probe.publish(IN, JSON.stringify({ text: 'hello from device', user: 'sensor1', chat_id: OUT, id: 'inbound-1' }))
const note = await waitFor(m => m.method === 'notifications/claude/channel' && m.params?.content === 'hello from device', 'inbound')

// 4. OUTBOUND — เรียก reply → รอ message โผล่บน claude/out
send({ jsonrpc: '2.0', id: 3, method: 'tools/call', params: { name: 'reply', arguments: { chat_id: OUT, text: 'hi back from claude' } } })
// → publish ลง claude/out { text: 'hi back from claude', from: 'assistant' }
```

ผลรันจริง:

```text
✓ MCP handshake ok
✓ tools/list ok — [reply]
✓ INBOUND ok — MQTT "hello from device" → notifications/claude/channel (user=sensor1, chat_id=claude/out)
✓ OUTBOUND ok — reply tool → published to claude/out (id=m1783575810980-2)

✅ ALL PASS — mqtt-channel round-trips inbound + outbound through the broker
```

## ลองเอง

```bash
# terminal 1
mosquitto -c mosquitto.conf

# terminal 2 — ยิง message เข้า Claude ผ่าน MQTT
mosquitto_pub -t claude/in -m '{"text":"อุณหภูมิ 32°C","user":"sensor-livingroom"}'

# terminal 3 — ดูสิ่งที่ Claude ตอบกลับ
mosquitto_sub -t claude/out
```

## สรุป — channel คือสัญญา ไม่ใช่ transport

154 บรรทัด vs 900 — ตัด 83% ออก เพราะ 83% นั้นคือ auth ที่ Discord ต้องมี ไม่ใช่แก่นของ channel แก่นจริงคือ 4 ชิ้น (Server + `claude/channel` cap + notification + reply) แล้วเสียบ transport อะไรก็ได้ — Discord Gateway, MQTT, หรืออะไรก็ตามที่ push/publish ได้ Claude ไม่รู้ด้วยซ้ำ

ข้อควรระวังเดียว: MQTT topic ไม่มีตัวตนของผู้ส่ง (ต่างจาก Discord ที่มี user id) เพราะงั้น **auth ต้องไปอยู่ที่ broker** (ACL / TLS / username-password) ไม่ใช่ในโค้ด channel — นี่คือเหตุผลที่ Discord ต้องมี 605 บรรทัดพิเศษ แต่ MQTT ผลักภาระนั้นออกไปที่ชั้น broker

โค้ดเต็มทุกไฟล์ + test: https://github.com/Yutthakit/mqtt-channel
