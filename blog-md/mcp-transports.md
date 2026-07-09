---
title: MCP มีกี่แบบ — stdio กับ Streamable HTTP ต่างกันยังไง
description: MCP ส่งข้อความด้วย JSON-RPC 2.0 เหมือนกันหมด ต่างกันแค่ "ท่อ" ที่ส่ง — สรุป stdio, Streamable HTTP, และ HTTP+SSE ที่ deprecated แล้ว พร้อมโค้ดที่เปลี่ยน transport โดยไม่แตะ tool logic
date: 2026-07-08
datetime: 2026-07-08T20:40:00+07:00
tags: [mcp, transport, ai]
---

เวลาสอนเรื่อง MCP (Model Context Protocol) คนมักติดกับดักคำว่า "transport" คิดว่าแต่ละแบบคือคนละโปรโตคอล จริง ๆ ไม่ใช่ — **ข้อความข้างในเหมือนกันหมด เป็น JSON-RPC 2.0** ต่างกันแค่ "ท่อ" ที่ลำเลียงข้อความ เปลี่ยน transport ไม่ต้องแก้ logic ของ tool เลยสักบรรทัด

ผมไป research มาจาก spec ตัวจริง (2025-11-25) ไม่ใช่จากความจำ แล้วเจอว่าปัจจุบันมาตรฐานเหลือ **2 แบบ** ไม่ใช่ 3 อย่างที่หลายคนเข้าใจ

## เทียบ 3 แบบ

| | stdio | Streamable HTTP | HTTP + SSE |
|--|-------|-----------------|------------|
| สถานะ | มาตรฐาน · local | มาตรฐาน · remote (แนะนำ) | ⚠️ DEPRECATED (2024-11-05) |
| ทำงานยังไง | client เปิด server เป็น subprocess คุยผ่าน stdin/stdout | server แยก process · client ยิง HTTP เข้า endpoint เดียว | server ต้องเปิด SSE ค้างไว้ตลอด |
| Endpoint | ไม่มี (pipe) | 1 อัน เช่น `/mcp` | 2 อัน: `GET /sse` + `POST /messages` |
| Resumable | — (แต่ local ไม่ค่อยหลุด) | ✓ `Last-Event-ID` replay ต่อได้ | ✗ หลุด = เริ่มใหม่ ข้อความหาย |
| Serverless | — | ✓ ตอบ JSON จบได้ = เข้ากับ Lambda/edge | ✗ ต้องถือ connection ค้าง |

**stdio** เหมาะกับ tool ที่รันบนเครื่องเดียวกับ client — Claude Desktop, CLI, dev บนเครื่องตัวเอง ง่ายสุด ไม่มี port ไม่มี session

**Streamable HTTP** คือ default ของยุคนี้ — server remote แชร์ได้หลาย client, deploy ขึ้น cloud ได้ จุดสำคัญคือมันรวมทุกอย่างไว้ที่ **endpoint เดียว** และเลือกได้ว่าจะตอบเป็น JSON ก้อนเดียว หรือเปิด SSE สตรีมหลายก้อน — per-request

**HTTP + SSE** คือของเก่าที่ตายไปแล้ว ทำไมตาย? เพราะต้องถือ connection ค้าง + มี 2 endpoint + resume ไม่ได้ + scale ยาก Streamable HTTP รวบทั้งหมดนี้ให้จบในตัวเดียว โค้ดใหม่อย่าเริ่มด้วยอันนี้ เก็บไว้แค่ backward-compat

## หัวใจ: transport เปลี่ยน แต่ tool ไม่แตะ

โค้ด server + การลงทะเบียน tool เหมือนกันเป๊ะทั้ง 2 แบบ

```ts
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { z } from "zod";

const server = new McpServer({ name: "demo", version: "1.0.0" });

server.registerTool("greet",
  { description: "ทักทาย", inputSchema: { name: z.string() } },
  async ({ name }) => ({ content: [{ type: "text", text: `สวัสดี ${name}` }] })
);
```

ต่อ **stdio** — 2 บรรทัดจบ

```ts
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
const transport = new StdioServerTransport();
await server.connect(transport);
// client เปิดไฟล์นี้เป็น subprocess แล้วคุยผ่าน pipe
```

ต่อ **Streamable HTTP** — ต้องจัดการ endpoint + session

```ts
import express from "express";
import { randomUUID } from "node:crypto";
import { StreamableHTTPServerTransport } from
  "@modelcontextprotocol/sdk/server/streamableHttp.js";

const app = express(); app.use(express.json());
const transports: Record<string, StreamableHTTPServerTransport> = {};

app.post("/mcp", async (req, res) => {           // client → server
  const sid = req.headers["mcp-session-id"] as string;
  let transport = transports[sid];
  if (!transport) {                              // session ใหม่ตอน initialize
    transport = new StreamableHTTPServerTransport({
      sessionIdGenerator: () => randomUUID(),
      onsessioninitialized: id => { transports[id] = transport; },
    });
    await server.connect(transport);
  }
  await transport.handleRequest(req, res, req.body);
});

const sid = (r: any) => r.headers["mcp-session-id"] as string;
app.get("/mcp", (req, res) => transports[sid(req)]?.handleRequest(req, res));    // เปิด SSE
app.delete("/mcp", (req, res) => transports[sid(req)]?.handleRequest(req, res)); // จบ session
app.listen(3000);
```

block แรกเหมือนกันทุกตัวอักษร ต่างกันแค่ท่อ — stdio ต่อ pipe ตรง ๆ ส่วน HTTP ต้องรับ **POST** (รับ message) + **GET** (ส่ง SSE / resume) + **DELETE** (จบ session) ผ่าน endpoint เดียว

## จุดที่ต้องระวังเรื่อง security

ก่อนเรียก `handleRequest` ทุกครั้ง server **ต้อง**เช็ค `Origin` header กัน DNS-rebinding attack ถ้ารัน local ก็ bind `127.0.0.1` อย่าเปิดกว้าง แล้วใส่ auth ด้วย — ตรงนี้ spec เขียนว่า MUST ไม่ใช่ควร

## สรุป

MCP transport ไม่ใช่คนละโปรโตคอล — JSON-RPC เหมือนกันหมด เลือกตามที่รัน: **local ใช้ stdio** (ง่ายสุด), **remote/cloud ใช้ Streamable HTTP** (default ยุคนี้), **HTTP+SSE เก่าอย่าเริ่มใหม่** เปลี่ยน transport ไม่ต้องแตะ tool logic — นี่คือจุดที่ทำให้ MCP สวยในเชิงออกแบบ
