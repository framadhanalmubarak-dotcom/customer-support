# Kebutuhan: Khwarizmi ConnectHub

## Tujuan

Aplikasi CRM & customer support: pelanggan chat lewat widget di website demo,
pesan menjadi tiket, ditugaskan ke agent, tampil realtime di shared inbox,
dengan SLA, automation, dan webhook simulasi yang aman.

## Peran

| Role | Hak utama |
| --- | --- |
| Admin | Kelola user, aturan automation, kebijakan SLA |
| Supervisor | Pantau semua tiket, tiket telat, beban agent; pindahkan assignment |
| Agent | Lihat dan balas hanya tiket miliknya |
| Pelanggan | Chat lewat widget tanpa login |

## Fitur wajib

| Modul | Kebutuhan |
| --- | --- |
| Login & akses | Role Admin, Supervisor, Agent; izin dicek di backend |
| Pelanggan | Simpan kontak, perusahaan, riwayat tiket |
| Ticketing | Kategori, prioritas, status, assignment, internal notes |
| Shared inbox | Pesan pelanggan dan balasan agent dalam satu percakapan |
| Widget chat | Widget sederhana untuk mengirim chat dari website demo |
| Realtime | Pesan masuk tampil lewat WebSocket atau SSE |
| Automation | Auto-assignment dan pengingat via background job |
| SLA & laporan | Deadline respons, tiket telat, backlog, metrik agent |

## Alur demo wajib

1. Pelanggan kirim chat dari widget.
2. Tiket dibuat atau ditemukan, lalu agent ditugaskan.
3. Agent menerima notifikasi lalu membalas.
4. SLA, status tiket, dan histori diperbarui.

## Aturan wajib backend

- Pesan masuk database dulu sebelum dianggap berhasil dikirim.
- Event yang datang dua kali tidak boleh membuat pesan atau tiket dobel.
- Agent hanya bisa mengakses tiket sesuai haknya.
- SLA dihitung hanya pada jam kerja: Senin-Jumat, 09.00-17.00 WIB.

## Tantangan advanced

| Kode | Kebutuhan |
| --- | --- |
| C1 Reconnect | Agent offline lalu online lagi: ambil pesan yang terlewat tanpa duplikat |
| C2 Job retry | Pengingat gagal terkirim: retry aman, tercatat, tanpa efek ganda |
| C3 SLA engine | High: balasan pertama 30 menit jam kerja; Normal: 4 jam; hitung lewat jam tutup |
| C4 Webhook aman | Verifikasi signature, tolak yang salah, event ID berulang tidak diproses dua kali |

## Yang harus terbukti (kode + automated test)

- Pesan tersimpan dan muncul realtime saat agent online.
- Setelah reconnect, pesan yang tertinggal bisa diambil.
- Retry job aman dari efek samping dobel.
- SLA benar walau melewati luar jam kerja.
- Webhook palsu ditolak dan event dobel aman.

## Deliverable

- Repo Git dengan riwayat commit jelas
- Link staging aktif dan akun demo semua role
- README, ERD, dokumentasi endpoint API
- Minimal 15 automated tests
- Seed: 50 pelanggan, 100 tiket, 3 agent, contoh chat, kasus SLA
- Ringkasan keputusan teknis dan penggunaan AI

## Asumsi (perlu dikonfirmasi ke Bang Rizki)

- Prioritas tiket: Normal dan High.
- Stack: SvelteKit + Cloudflare (D1, Durable Objects, Queues).
