# ERD: Khwarizmi ConnectHub

Database: Cloudflare D1 (SQLite). Semua waktu disimpan dalam UTC; perhitungan jam kerja
SLA dilakukan di zona waktu Asia/Jakarta.

## Diagram

```mermaid
erDiagram
    USERS ||--o{ TICKETS : "menangani"
    USERS ||--o{ TICKET_ASSIGNMENTS : "ditugaskan"
    USERS ||--o{ MESSAGES : "membalas"
    USERS ||--o{ NOTIFICATION_JOBS : "penerima"
    USERS ||--o{ AUTOMATION_RULES : "membuat"
    CUSTOMERS ||--o{ TICKETS : "punya"
    CUSTOMERS ||--o{ CONVERSATIONS : "memulai"
    TICKETS ||--o{ TICKET_ASSIGNMENTS : "riwayat"
    TICKETS ||--o{ CONVERSATIONS : "berisi"
    TICKETS ||--o{ NOTIFICATION_JOBS : "memicu"
    CONVERSATIONS ||--o{ MESSAGES : "berisi"
    SLA_POLICIES ||--o{ TICKETS : "mengatur"

    USERS {
        integer id PK
        text name
        text email UK
        text password_hash
        text role "admin | supervisor | agent"
        integer is_active
        integer created_at
    }
    CUSTOMERS {
        integer id PK
        text name
        text email
        text phone
        text company
        text widget_token UK "identitas pelanggan di widget"
        integer created_at
    }
    TICKETS {
        integer id PK
        integer customer_id FK
        integer assigned_agent_id FK "agent saat ini"
        integer sla_policy_id FK
        text subject
        text category
        text priority "normal | high"
        text status "open | pending | resolved | closed"
        integer first_response_due_at "deadline SLA"
        integer first_responded_at
        integer created_at
        integer updated_at
        integer resolved_at
    }
    TICKET_ASSIGNMENTS {
        integer id PK
        integer ticket_id FK
        integer agent_id FK
        integer assigned_by FK "null kalau otomatis"
        text assignment_type "auto | manual"
        integer assigned_at
        integer unassigned_at
    }
    CONVERSATIONS {
        integer id PK
        integer ticket_id FK
        integer customer_id FK
        text channel "widget"
        integer created_at
    }
    MESSAGES {
        integer id PK "berurutan, jadi cursor reconnect (C1)"
        integer conversation_id FK
        integer sender_user_id FK "null kalau pelanggan"
        text sender_type "customer | agent | system"
        text message_type "message | internal_note"
        text body
        text client_message_id "anti duplikat"
        integer created_at
    }
    SLA_POLICIES {
        integer id PK
        text priority UK
        integer first_response_minutes "30 untuk high, 240 untuk normal"
        text business_start "09:00"
        text business_end "17:00"
        text business_days "1,2,3,4,5"
        text timezone "Asia/Jakarta"
        integer is_active
    }
    AUTOMATION_RULES {
        integer id PK
        integer created_by FK
        text name
        text type "auto_assign | reminder"
        text conditions "JSON"
        text action "JSON"
        integer is_active
        integer created_at
    }
    NOTIFICATION_JOBS {
        integer id PK
        integer ticket_id FK
        integer user_id FK
        text type
        text payload "JSON"
        text idempotency_key UK "anti efek ganda (C2)"
        text status "pending | processing | sent | failed | dead"
        integer attempts
        integer max_attempts
        integer next_attempt_at
        text last_error
        integer created_at
        integer sent_at
    }
    WEBHOOK_EVENTS {
        integer id PK
        text event_id UK "anti event ganda (C4)"
        text source
        integer signature_valid
        text payload "JSON"
        text status "received | processed | rejected"
        integer received_at
        integer processed_at
    }
```

## Keputusan desain

- **Satu ticket punya satu conversation utama.** Pesan baru dari pelanggan yang masih punya tiket
  terbuka ditempelkan ke tiket itu; kalau tidak ada, dibuat tiket baru.
- **Internal notes disimpan di tabel `messages`** dengan `message_type = internal_note`.
  Pelanggan tidak pernah menerima baris bertipe ini.
- **`tickets.assigned_agent_id`** menyimpan penanggung jawab saat ini (cepat untuk filter inbox),
  sedangkan `ticket_assignments` menyimpan riwayat perpindahan.
- **Waktu disimpan sebagai UTC** (integer epoch milidetik).

## Index

| Tabel | Index | Untuk apa |
| --- | --- | --- |
| users | UNIQUE(email) | Login |
| customers | UNIQUE(widget_token) | Mengenali pelanggan widget |
| tickets | (assigned_agent_id, status) | Inbox agent |
| tickets | (status, priority) | Filter tiket |
| tickets | (first_response_due_at) | Tiket telat dan backlog |
| ticket_assignments | (ticket_id), (agent_id) | Riwayat dan beban agent |
| messages | (conversation_id, id) | Ambil pesan setelah ID terakhir (C1) |
| messages | UNIQUE(conversation_id, client_message_id) | Cegah pesan dobel |
| notification_jobs | UNIQUE(idempotency_key) | Cegah efek ganda (C2) |
| notification_jobs | (status, next_attempt_at) | Cari job yang siap dicoba ulang |
| webhook_events | UNIQUE(event_id) | Cegah event diproses dua kali (C4) |

## Hubungan dengan tantangan

| Tantangan | Didukung oleh |
| --- | --- |
| C1 Reconnect | `messages.id` berurutan; client kirim ID terakhir, server balas semua pesan setelahnya |
| C2 Job retry | `idempotency_key`, `status`, `attempts`, `next_attempt_at`, `last_error` |
| C3 SLA engine | `sla_policies`, `tickets.first_response_due_at`, `first_responded_at` |
| C4 Webhook aman | `webhook_events.event_id` unik dan `signature_valid` |
