# Kontrak API: Khwarizmi ConnectHub

Base path: `/api`. Format: JSON. Autentikasi staf memakai cookie sesi (`HttpOnly`).
Semua izin dicek di server, bukan hanya di UI.

## Format error

```json
{ "error": { "code": "FORBIDDEN", "message": "Anda tidak berhak mengakses tiket ini" } }
```

| Status | Arti |
| --- | --- |
| 400 | Request tidak valid |
| 401 | Belum login atau signature salah |
| 403 | Login, tapi tidak punya hak |
| 404 | Data tidak ditemukan (juga dipakai untuk tiket milik agent lain) |
| 409 | Konflik data |
| 422 | Validasi gagal |

## Hak akses

| Endpoint | Admin | Supervisor | Agent | Pelanggan (widget) |
| --- | --- | --- | --- | --- |
| Auth | ya | ya | ya | tidak |
| Kelola user | ya | tidak | tidak | tidak |
| Pelanggan | semua | semua | hanya milik tiketnya | tidak |
| Tiket | semua | semua | hanya yang ditugaskan | tidak |
| Assign tiket | ya | ya | tidak | tidak |
| Pesan dan internal note | semua | semua | tiket miliknya | hanya pesan sendiri, tanpa note |
| Automation rules, SLA policy | ya | baca | tidak | tidak |
| Laporan | ya | ya | metrik sendiri | tidak |
| Widget | tidak | tidak | tidak | ya (token) |
| Webhook | signature | signature | signature | signature |

## 1. Auth

| Method | Path | Fungsi |
| --- | --- | --- |
| POST | `/api/auth/login` | Body `{ "email", "password" }`. Sukses: 200 + cookie sesi. Gagal: 401 |
| POST | `/api/auth/logout` | Hapus sesi |
| GET | `/api/auth/me` | Data user yang sedang login |

## 2. User (Admin)

| Method | Path | Fungsi |
| --- | --- | --- |
| GET | `/api/users` | Daftar user |
| POST | `/api/users` | Buat user `{ name, email, password, role }` |
| PATCH | `/api/users/:id` | Ubah nama, role, atau status aktif |

## 3. Pelanggan

| Method | Path | Fungsi |
| --- | --- | --- |
| GET | `/api/customers?q=` | Cari pelanggan |
| GET | `/api/customers/:id` | Detail + riwayat tiket |
| POST | `/api/customers` | Buat pelanggan (Supervisor/Admin) |
| PATCH | `/api/customers/:id` | Ubah kontak atau perusahaan |

## 4. Tiket

| Method | Path | Fungsi |
| --- | --- | --- |
| GET | `/api/tickets` | Query: `status`, `priority`, `assigned_to`, `overdue=true`, `page` |
| GET | `/api/tickets/:id` | Detail tiket + info SLA |
| PATCH | `/api/tickets/:id` | Ubah `status`, `priority`, `category` |
| POST | `/api/tickets/:id/assign` | Body `{ "agent_id" }` (Supervisor/Admin) |
| GET | `/api/tickets/:id/assignments` | Riwayat penugasan |

## 5. Pesan (sisi agent)

| Method | Path | Fungsi |
| --- | --- | --- |
| GET | `/api/tickets/:id/messages?after_id=` | Ambil pesan, urut naik, hanya setelah `after_id` |
| POST | `/api/tickets/:id/messages` | Kirim balasan atau internal note |

Body kirim pesan:

```json
{ "body": "Halo, kami cek dulu ya", "message_type": "message", "client_message_id": "c-9f3a..." }
```

- `message_type`: `message` atau `internal_note`.
- Pesan disimpan ke database dulu, baru dianggap berhasil dan disiarkan realtime.
- Jika `client_message_id` sudah pernah dikirim di percakapan yang sama: balas **200** dengan
  pesan yang sudah ada (tidak membuat pesan baru). Pesan baru: **201**.

## 6. Realtime dan sinkronisasi (C1)

| Method | Path | Fungsi |
| --- | --- | --- |
| GET | `/api/realtime` | Koneksi WebSocket/SSE untuk agent. Query `last_message_id` |
| GET | `/api/sync/messages?after_id=123&limit=100` | Semua pesan baru di tiket milik agent setelah ID tersebut |

Alur reconnect: client menyimpan ID pesan terakhir yang diterima, lalu setelah tersambung lagi
memanggil `/api/sync/messages?after_id=<ID terakhir>`. Karena `messages.id` berurutan dan
client mengabaikan ID yang sudah dimiliki, tidak ada pesan hilang atau dobel.

## 7. Widget (pelanggan, tanpa login)

| Method | Path | Fungsi |
| --- | --- | --- |
| POST | `/api/widget/session` | Body `{ name?, email? }`. Balas `{ widget_token }` |
| POST | `/api/widget/messages` | Header `X-Widget-Token`. Body `{ body, client_message_id }` |
| GET | `/api/widget/messages?after_id=` | Ambil pesan percakapan sendiri (tanpa internal note) |
| GET | `/api/widget/stream` | SSE/WebSocket untuk balasan agent |

Saat pesan masuk: simpan pesan, cari tiket terbuka milik pelanggan (kalau tidak ada, buat tiket
baru), hitung deadline SLA, jalankan auto-assignment, lalu siarkan realtime.

## 8. Automation dan job (C2)

| Method | Path | Fungsi |
| --- | --- | --- |
| GET | `/api/automation-rules` | Daftar aturan (Admin, Supervisor baca) |
| POST | `/api/automation-rules` | Buat aturan (Admin) |
| PATCH | `/api/automation-rules/:id` | Ubah atau nonaktifkan |
| GET | `/api/notification-jobs?status=` | Pantau job dan jumlah percobaan (Supervisor/Admin) |
| POST | `/api/notification-jobs/:id/retry` | Coba ulang manual job `failed`/`dead` (Admin) |

## 9. SLA dan laporan (C3)

| Method | Path | Fungsi |
| --- | --- | --- |
| GET | `/api/sla-policies` | Lihat kebijakan: High 30 menit, Normal 240 menit jam kerja |
| PATCH | `/api/sla-policies/:id` | Ubah kebijakan (Admin) |
| GET | `/api/reports/summary` | Tiket telat, backlog, tiket per status |
| GET | `/api/reports/agents` | Metrik per agent: jumlah tiket, waktu respons rata-rata |

## 10. Webhook simulasi (C4)

| Method | Path | Fungsi |
|---|---|---|
| POST | `/api/webhooks/simulated` | Terima event dari sistem luar |

Header wajib:

- `X-Event-Id`: ID unik event
- `X-Signature`: `sha256=<hex>`, yaitu HMAC-SHA256 dari body mentah memakai secret `WEBHOOK_SECRET`

Aturan:

1. Hitung ulang HMAC dari body mentah, bandingkan dengan perbandingan waktu konstan.
2. Signature salah: **401**, event disimpan dengan status `rejected`, tidak diproses.
3. `event_id` belum pernah ada: proses, simpan status `processed`, balas **200** `{ "status": "processed" }`.
4. `event_id` sudah pernah ada: **jangan proses lagi**, balas **200** `{ "status": "duplicate" }`.
