# AI Agent Event Scheduling System

## 1. Overview

**AI Agent Event Scheduling** คือระบบ AI Agent สำหรับจัดการตารางนัดหมาย โดย AI สามารถเข้าใจคำสั่งภาษาธรรมชาติ ตรวจสอบ Calendar ค้นหาเวลาว่าง ประเมินความเหมาะสม และสร้างหรือปรับเปลี่ยน Event ให้ผู้ใช้

ตัวอย่าง:

> “นัดประชุมทีม AI สัปดาห์หน้า 1 ชั่วโมง ช่วงบ่าย ถ้ามีเวลาว่าง”

ระบบจะทำงานเป็น:

```text
User Request
      ↓
Intent Detection
      ↓
Constraint Extraction
      ↓
Context Engineering
      ↓
Calendar Availability
      ↓
Candidate Slot Generation
      ↓
Slot Ranking
      ↓
Decision
      ↓
Human Approval / Auto Booking
      ↓
Create Event
      ↓
Notification
      ↓
Memory
```

---

# 2. System Architecture

```text
                         ┌───────────────┐
                         │     USER      │
                         └───────┬───────┘
                                 │
                                 ▼
                       ┌──────────────────┐
                       │    AI AGENT      │
                       │   Scheduler      │
                       └────────┬─────────┘
                                │
                 ┌──────────────┼──────────────┐
                 ▼              ▼              ▼
          Intent Agent    Context Engine    Memory
                 │              │              │
                 └──────────────┼──────────────┘
                                ▼
                       ┌──────────────────┐
                       │ Planning Engine  │
                       └────────┬─────────┘
                                │
                                ▼
                       ┌──────────────────┐
                       │ Calendar Agent   │
                       └────────┬─────────┘
                                │
                 ┌──────────────┼──────────────┐
                 ▼              ▼              ▼
             Calendar      Availability     Timezone
                 │              │              │
                 └──────────────┼──────────────┘
                                ▼
                       ┌──────────────────┐
                       │ Decision Engine  │
                       └────────┬─────────┘
                                │
                    ┌───────────┴───────────┐
                    ▼                       ▼
              Ask User                 Auto Book
                    │                       │
                    └───────────┬───────────┘
                                ▼
                       ┌──────────────────┐
                       │  Event Service   │
                       └────────┬─────────┘
                                │
                 ┌──────────────┼──────────────┐
                 ▼              ▼              ▼
             Calendar         Email       Notification
```

---

# 3. User Intent

AI ต้องสามารถเข้าใจคำสั่ง Natural Language เช่น

* นัดประชุมพรุ่งนี้
* นัดประชุมวันจันทร์หน้า
* นัดช่วงบ่าย
* นัด 1 ชั่วโมง
* หาเวลาที่ทุกคนว่าง
* เลื่อนประชุม
* ยกเลิกประชุม
* นัดใหม่
* หาช่วงเวลาที่เหมาะที่สุด
* อย่านัดประชุมติดกัน
* นัดเฉพาะเวลาทำงาน

ตัวอย่าง Input:

```text
นัดประชุมทีม AI สัปดาห์หน้า
1 ชั่วโมง
ช่วงบ่าย
ถ้าทุกคนว่าง
```

AI แปลงเป็น Structured Intent:

```json
{
  "intent": "create_event",
  "title": "ประชุมทีม AI",
  "duration": 60,
  "date_range": "next_week",
  "preferred_time": "afternoon",
  "participants": [
    "AI Team"
  ],
  "require_all_available": true
}
```

---

# 4. Context Engineering

AI Agent ไม่ควรส่งข้อมูล Calendar ทั้งหมดเข้า LLM

ควรสร้าง **Scheduling Context** เฉพาะข้อมูลที่จำเป็น

```text
User Profile
     +
Calendar
     +
Availability
     +
Participants
     +
Timezone
     +
Working Hours
     +
Preferences
     +
Existing Events
     ↓
Scheduling Context
     ↓
LLM
```

ตัวอย่าง:

```json
{
  "timezone": "Asia/Bangkok",
  "working_hours": {
    "start": "09:00",
    "end": "17:00"
  },
  "preferred_meeting_time": {
    "start": "14:00",
    "end": "16:00"
  },
  "avoid_back_to_back": true,
  "minimum_break": 15
}
```

---

# 5. Calendar Agent

Calendar Agent ทำหน้าที่เชื่อมต่อ Calendar API

## Tools

```text
get_calendar_events()
check_availability()
find_free_slots()
create_event()
update_event()
cancel_event()
add_attendee()
send_invitation()
```

Architecture:

```text
              LLM
               │
               ▼
          Tool Calling
               │
               ▼
       Calendar Service
               │
       ┌───────┴────────┐
       ▼                ▼
Google Calendar    Microsoft Graph
```

---

# 6. Scheduling Engine

Scheduling Engine ทำหน้าที่ค้นหาและจัดอันดับเวลาที่เหมาะสม

ตัวอย่าง Candidate Slots:

```text
09:00 - 10:00
10:00 - 11:00
13:00 - 14:00
14:00 - 15:00
15:00 - 16:00
16:00 - 17:00
```

ระบบคำนวณคะแนน:

```text
Score =
    Availability
  + User Preference
  + Participant Availability
  + Time Preference
  + Meeting Priority
  - Conflict Penalty
  - Back-to-Back Penalty
  - Outside Working Hours
```

ตัวอย่าง:

| Time  |  Score |
| ----- | -----: |
| 09:00 |     45 |
| 10:00 |     55 |
| 13:00 |     70 |
| 14:00 | **95** |
| 15:00 |     90 |
| 16:00 |     60 |

ดังนั้น AI เลือก:

```text
14:00 - 15:00
Score = 95
```

---

# 7. Multi-Agent Architecture

ระบบขนาดใหญ่สามารถแยกเป็นหลาย Agent

```text
                    Supervisor Agent
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
    Intent Agent      Calendar Agent   Communication Agent
          │                │                │
          ▼                ▼                ▼
     NLP / Context     Availability     Email / Chat
                           │
                           ▼
                    Scheduling Agent
                           │
                           ▼
                     Decision Agent
```

## Intent Agent

รับผิดชอบ:

```text
Natural Language
       ↓
Intent
       ↓
Constraints
```

## Calendar Agent

รับผิดชอบ:

```text
Calendar
   ↓
Events
   ↓
Availability
```

## Scheduling Agent

รับผิดชอบ:

```text
Candidate Slots
       ↓
Ranking
       ↓
Best Slots
```

## Decision Agent

รับผิดชอบ:

```text
Best Candidate
       ↓
Auto Book
   OR
Ask User
```

## Communication Agent

รับผิดชอบ:

```text
Email
Slack
LINE
Push Notification
```

---

# 8. Event State Machine

Event ควรมี Lifecycle

```text
DRAFT
  │
  ▼
PROPOSED
  │
  ▼
PENDING_CONFIRMATION
  │
  ▼
CONFIRMED
  │
  ├───────────────┐
  ▼               ▼
RESCHEDULED    CANCELLED
  │
  ▼
CONFIRMED
  │
  ▼
COMPLETED
```

ตัวอย่าง State:

```text
DRAFT
PROPOSED
PENDING_CONFIRMATION
CONFIRMED
RESCHEDULED
CANCELLED
COMPLETED
```

---

# 9. Database Design

## Users

```text
users
├── id
├── name
├── timezone
└── preferences
```

## Calendars

```text
calendars
├── id
├── user_id
├── provider
└── external_calendar_id
```

## Events

```text
events
├── id
├── calendar_id
├── title
├── description
├── start_time
├── end_time
├── timezone
├── status
└── created_by
```

## Event Attendees

```text
event_attendees
├── event_id
├── user_id
└── response_status
```

## Scheduling Requests

```text
scheduling_requests
├── id
├── user_id
├── intent
├── constraints
├── candidates
└── selected_slot
```

## Agent Runs

```text
agent_runs
├── id
├── agent
├── input
├── output
├── tool_calls
└── status
```

---

# 10. API Design

## Schedule Event

```http
POST /api/agent/schedule
```

Request:

```json
{
  "message": "นัดประชุมทีม AI สัปดาห์หน้า 1 ชั่วโมงช่วงบ่าย"
}
```

Response:

```json
{
  "status": "proposed",
  "title": "ประชุมทีม AI",
  "duration": 60,
  "candidates": [
    {
      "start": "2026-08-24T14:00:00+07:00",
      "end": "2026-08-24T15:00:00+07:00",
      "score": 95
    }
  ]
}
```

---

# 11. Event APIs

```http
GET    /api/events
POST   /api/events
GET    /api/events/:id
PATCH  /api/events/:id
DELETE /api/events/:id
```

Scheduling APIs:

```http
GET  /api/calendar/availability

POST /api/scheduling/find-slots

POST /api/scheduling/confirm

POST /api/scheduling/reschedule
```

---

# 12. Human-in-the-Loop

AI ไม่ควรมีสิทธิ์จองทุก Event โดยอัตโนมัติ

สามารถกำหนดระดับ Autonomy:

```text
Level 1
AI แนะนำเวลา
      ↓
Human Confirm
      ↓
Create Event
```

```text
Level 2
AI เลือกเวลา
      ↓
Human Approval
      ↓
Create Event
```

```text
Level 3
AI เลือกและจองอัตโนมัติ
      ↓
เฉพาะ Event ที่ผ่าน Policy
```

---

# 13. Scheduling Policy

ตัวอย่าง Policy:

```text
IF
    meeting_duration <= 30 minutes
AND
    participants = internal
AND
    within_working_hours
AND
    no_calendar_conflict
AND
    confidence >= 0.90

THEN
    auto_book
ELSE
    ask_user
```

---

# 14. Memory

AI Agent สามารถเรียนรู้ Preference ของผู้ใช้ได้

ตัวอย่าง:

```json
{
  "user_id": "user_001",
  "preferences": {
    "preferred_meeting_time": "14:00-16:00",
    "avoid_monday": true,
    "avoid_friday_afternoon": true,
    "avoid_back_to_back": true,
    "default_duration": 60
  }
}
```

ดังนั้นเมื่อผู้ใช้พูดว่า:

> “หาที่เหมาะที่สุดให้เลย”

Agent สามารถใช้ทั้ง Calendar และ User Preference ในการตัดสินใจ

---

# 15. AI Agent Loop

แกนหลักของระบบสามารถออกแบบเป็น Agent Loop:

```text
        ┌──────────────┐
        │    Observe   │
        └──────┬───────┘
               ↓
        ┌──────────────┐
        │    Reason    │
        └──────┬───────┘
               ↓
        ┌──────────────┐
        │     Plan     │
        └──────┬───────┘
               ↓
        ┌──────────────┐
        │     Tool     │
        │    Calling   │
        └──────┬───────┘
               ↓
        ┌──────────────┐
        │    Observe   │
        └──────┬───────┘
               │
               └───────────────► LOOP
```

---

# 16. End-to-End Flow

ตัวอย่าง:

```text
User:

"พรุ่งนี้ช่วยนัดประชุมกับ John
ประมาณ 1 ชั่วโมง
ช่วงบ่าย"
```

### Step 1 — Intent

```text
intent = create_event
```

### Step 2 — Extract Constraints

```text
date = tomorrow
duration = 60 minutes
time = afternoon
participant = John
```

### Step 3 — Context

```text
Timezone
Working Hours
User Preferences
John Availability
Existing Events
```

### Step 4 — Search

```text
Find Available Slots
```

### Step 5 — Ranking

```text
13:00 → 72
14:00 → 95
15:00 → 91
16:00 → 64
```

### Step 6 — Decision

```text
Best Slot = 14:00
```

### Step 7 — Approval

```text
"พบเวลาที่เหมาะที่สุดคือ
พรุ่งนี้ 14:00–15:00

ต้องการจองหรือไม่?"
```

### Step 8 — Tool Call

```text
create_event()
```

### Step 9 — Notification

```text
Send Invitation
Send Notification
```

### Step 10 — Memory

```text
Store:
Meeting scheduled successfully
Preferred time = afternoon
```

---

# 17. Recommended Tech Stack

```text
Frontend
├── Next.js
├── React
└── TailwindCSS

Backend
├── FastAPI
└── Node.js + TypeScript

AI
├── LLM
├── Structured Output
└── Tool Calling

Agent
├── LangGraph
└── Custom Agent Orchestrator

Database
├── PostgreSQL
├── Redis
└── pgvector

Calendar
├── Google Calendar API
└── Microsoft Graph API

Notification
├── Email
├── LINE
├── Slack
└── Push Notification

Infrastructure
├── Docker
├── Kubernetes
└── Cloud
```

---

# 18. Security

ระบบต้องควบคุมสิทธิ์ของ Agent อย่างชัดเจน

```text
User
  ↓
Authentication
  ↓
Authorization
  ↓
Agent Policy
  ↓
Tool Permission
  ↓
Calendar API
```

ควรมี:

* OAuth 2.0
* Role-Based Access Control
* Tool Permission
* Audit Log
* Event Ownership
* Human Approval
* Rate Limiting
* Prompt Injection Protection
* Calendar Data Isolation

---

# 19. Final Architecture

```text
                         USER
                          │
                          ▼
                ┌─────────────────┐
                │   AI Scheduler  │
                └────────┬────────┘
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
           Intent     Context     Memory
           Agent      Engine
              │          │          │
              └──────────┼──────────┘
                         ▼
                  Planning Agent
                         │
                         ▼
                  Calendar Agent
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
          Calendar   Availability  Timezone
              │          │          │
              └──────────┼──────────┘
                         ▼
                 Scheduling Engine
                         │
                         ▼
                  Ranking Engine
                         │
                         ▼
                  Decision Agent
                    │         │
                    ▼         ▼
                Ask User   Auto Book
                    │         │
                    └────┬────┘
                         ▼
                   Event Service
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
          Calendar      Email    Notification
                         │
                         ▼
                       Memory
```

## Core Concept

```text
AI Agent Event Scheduling
=
LLM
+
Intent Understanding
+
Context Engineering
+
Calendar Tools
+
Constraint Satisfaction
+
Scheduling Optimization
+
Memory
+
Human-in-the-Loop
+
Autonomous Action
```

**จุดสำคัญ:** ระบบนี้จึงไม่ใช่เพียง **Calendar Application ที่มี Chatbot** แต่เป็น **Agentic Scheduling System** ที่สามารถ `Understand → Plan → Search → Reason → Act → Verify → Remember` เพื่อจัดการ Event ให้ผู้ใช้แบบกึ่งอัตโนมัติหรืออัตโนมัติเต็มรูปแบบ.
