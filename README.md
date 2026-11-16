# Khwarizmi ConnectHub

Aplikasi web CRM & customer support. Pelanggan chat lewat widget di website demo,
pesan otomatis menjadi tiket, ditugaskan ke agent, dan tampil realtime di shared inbox.
Dilengkapi SLA, automation, dan webhook simulasi yang aman.

> Status: dalam pengerjaan (Minggu 1)

## Link

- Repository: <https://github.com/framadhanalmubarak-dotcom/customer-support>
- Staging: menyusul
- Website demo (widget chat): menyusul

## Fitur

- Login & akses: role Admin, Supervisor, Agent (izin dicek di backend)
- Pelanggan: kontak, perusahaan, riwayat tiket
- Ticketing: kategori, prioritas, status, assignment, internal notes
- Shared inbox: pesan pelanggan dan balasan agent dalam satu percakapan
- Widget chat untuk website demo
- Realtime: pesan masuk tampil tanpa refresh
- Automation: auto-assignment dan pengingat via background job
- SLA & laporan: deadline respons, tiket telat, backlog, metrik agent

## Tantangan advanced

| Kode | Tantangan | Status |
| --- | --- | --- |
| C1 | Reconnect: ambil pesan yang terlewat tanpa duplikat | Menyusul |
| C2 | Job retry yang aman dan tidak menimbulkan efek ganda | Menyusul |
| C3 | SLA engine (jam kerja Senin-Jumat 09.00-17.00 WIB) | Menyusul |
| C4 | Webhook dengan verifikasi signature dan anti event ganda | Menyusul |

## Stack

- SvelteKit (TypeScript)
- Cloudflare Workers (adapter-cloudflare), D1, Durable Objects, Queues
- Vitest, ESLint, Prettier

## Cara install

Prasyarat: Node.js (versi LTS) dan npm.

```bash
git clone https://github.com/framadhanalmubarak-dotcom/customer-support.git
cd customer-support
npm install
npm run gen
```

## Konfigurasi lingkungan

Belum ada variabel lingkungan yang dibutuhkan. Bagian ini akan diisi seiring pengerjaan
(misalnya secret untuk signature webhook).

File `.env` dan `.dev.vars` tidak boleh di-commit. Jangan menyimpan password,
token, atau data asli di repository.

## Menjalankan aplikasi

```bash
npm run dev
```

Buka <http://localhost:5173>

## Database dan seed data

Menyusul: perintah migration dan seed.

Seed berisi minimal 50 pelanggan, 100 tiket, 3 agent, contoh chat, dan kasus SLA.

## Menjalankan test

```bash
npm run test
```

Target: minimal 15 automated tests yang menguji aturan bisnis, izin akses,
dan edge case penting.

## Akun demo

| Role | Email | Password |
| --- | --- | --- |
| Admin | menyusul | menyusul |
| Supervisor | menyusul | menyusul |
| Agent | menyusul | menyusul |

Akun ini hanya untuk demo, bukan akun asli.

## Dokumentasi

- Kebutuhan: [docs/requirements.md](docs/requirements.md)
- ERD: docs/erd.md (menyusul)
- Kontrak dan dokumentasi API: docs/api.md (menyusul)
- Keputusan teknis: docs/DECISIONS.md (menyusul)

## Penggunaan AI

AI dipakai untuk belajar, mencari referensi, dan review kode. Seluruh kode yang
digunakan sudah dipahami dan dapat dijelaskan. Detail dicatat di docs/DECISIONS.md.
