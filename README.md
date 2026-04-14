<div align="center">

<!-- Logo & Nama -->
<img src="frontend/assets/zira-logo.png" alt="Zira AI Logo" width="80" />

# Zira AI

**Your personal AI assistant — smarter, faster, built for you.**

[![Python](https://img.shields.io/badge/Python-3.10+-3776AB?style=flat-square&logo=python&logoColor=white)](https://python.org)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.111-009688?style=flat-square&logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com)
[![Anthropic](https://img.shields.io/badge/Claude-Sonnet_4-D97757?style=flat-square&logo=anthropic&logoColor=white)](https://anthropic.com)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15-336791?style=flat-square&logo=postgresql&logoColor=white)](https://postgresql.org)
[![Deploy](https://img.shields.io/badge/Deploy-Railway-7B00D4?style=flat-square&logo=railway&logoColor=white)](https://railway.app)
[![License](https://img.shields.io/badge/License-MIT-22C55E?style=flat-square)](LICENSE)

[Demo](#) · [Dokumentasi](#dokumentasi) · [Laporkan Bug](issues) · [Request Fitur](issues)

</div>

---

## Tentang Zira AI

Zira AI adalah aplikasi AI chat assistant yang dipersonalisasi, dibangun di atas **Claude Anthropic API** dengan backend **FastAPI (Python)** dan frontend **Vanilla JavaScript**. Zira dirancang untuk mengenali siapa penggunanya — nama, minat, dan pekerjaan — lalu menyesuaikan setiap respons secara otomatis.

### Fitur Utama

| Fitur | Deskripsi |
|---|---|
| 🧠 **AI Memory** | Zira mengingat nama dan topik minat kamu dari onboarding |
| ⚡ **Streaming Response** | Jawaban AI muncul real-time, kata per kata |
| 💡 **Extended Thinking** | Mode berpikir mendalam untuk pertanyaan kompleks |
| 🔒 **Incognito Chat** | Percakapan tanpa riwayat, tanpa data tersimpan |
| 🎤 **Voice Input** | Kirim pesan dengan suara, otomatis jadi teks |
| 🤖 **Model Selector** | Pilih antara Claude Opus, Sonnet, atau Haiku |
| 🧪 **Zira Labs** | 6 modul khusus: Code, Design, Image, Vid, Auto, Research |
| 💰 **Dashboard Revenue** | Analitik user + monetisasi via Xendit → DANA/BSI |
| 🌐 **Multi-bahasa** | Antarmuka tersedia dalam 11 bahasa |

---

## Tech Stack

```
Backend    →  Python 3.10+  ·  FastAPI  ·  Uvicorn  ·  SQLAlchemy
AI         →  Anthropic Claude (claude-sonnet-4-20250514)
Frontend   →  HTML5  ·  CSS3  ·  Vanilla JavaScript
Database   →  PostgreSQL (prod)  ·  SQLite (dev)
Auth       →  JWT  ·  Google OAuth2
Automation →  n8n (webhook)
Payment    →  Xendit  →  DANA / BSI
Deploy     →  Railway (auto-deploy dari GitHub)
```

---

## Struktur Proyek

```
zira-ai/
├── backend/
│   ├── main.py                   # FastAPI app entry point
│   ├── routers/
│   │   ├── auth.py               # Login, signup, Google OAuth2
│   │   ├── chat.py               # Chat + streaming endpoint
│   │   ├── onboarding.py         # Alur onboarding 3 tahap
│   │   ├── labs.py               # Zira Labs (Code, Image, Auto, dll)
│   │   └── dashboard.py          # Revenue & analytics
│   ├── models/
│   │   └── database.py           # SQLAlchemy ORM models
│   ├── services/
│   │   ├── claude_service.py     # Anthropic API wrapper + streaming
│   │   ├── memory_service.py     # AI memory management
│   │   └── payment_service.py    # Xendit disbursement
│   └── utils/
│       └── auth.py               # JWT helpers
├── frontend/
│   ├── index.html                # Entry (redirect ke login/chat)
│   ├── pages/
│   │   ├── login.html
│   │   ├── signup.html
│   │   ├── onboarding.html       # Name → Topics → Hello
│   │   └── chat.html             # Main chat UI
│   ├── css/
│   │   └── style.css             # Global styles + dark/light mode
│   └── js/
│       ├── auth.js
│       ├── chat.js               # Streaming, incognito, voice
│       ├── onboarding.js
│       └── labs.js               # Zira Labs modules
├── .env.example
├── requirements.txt
├── alembic.ini
└── README.md
```

---

## Instalasi & Setup Lokal

### Prasyarat

- Python 3.10+
- Node.js LTS (untuk n8n)
- PostgreSQL (atau gunakan SQLite untuk dev)
- API Key: Anthropic, Xendit, Google OAuth

### 1. Clone & Setup Environment

```bash
git clone https://github.com/username/zira-ai.git
cd zira-ai

python -m venv zira_env
source zira_env/bin/activate      # Windows: zira_env\Scripts\activate

pip install -r requirements.txt
```

### 2. Konfigurasi Environment

```bash
cp .env.example .env
```

Edit `.env` dan isi semua variabel:

```env
# AI
ANTHROPIC_API_KEY=sk-ant-...

# Database
DATABASE_URL=postgresql://user:password@localhost/ziradb

# Auth
SECRET_KEY=your-super-secret-jwt-key
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...

# Payment
XENDIT_API_KEY=xnd_...

# Automation
N8N_WEBHOOK_URL=http://localhost:5678/webhook/...

# App
ENVIRONMENT=development
FRONTEND_URL=http://localhost:3000
```

### 3. Setup Database

```bash
# Buat database PostgreSQL
createdb ziradb

# Jalankan migration
alembic upgrade head
```

### 4. Jalankan Backend

```bash
uvicorn backend.main:app --reload --port 8000
```

Backend berjalan di `http://localhost:8000`
Dokumentasi API otomatis tersedia di `http://localhost:8000/docs`

### 5. Jalankan Frontend

```bash
# Buka langsung di browser atau gunakan live server
open frontend/index.html

# Atau dengan VS Code Live Server / Python simple server:
cd frontend && python -m http.server 3000
```

### 6. (Opsional) Setup n8n untuk Zira Auto

```bash
npm install -g n8n
n8n
# Buka http://localhost:5678
# Buat Webhook Node → salin URL ke .env sebagai N8N_WEBHOOK_URL
```

---

## Alur Pengguna

```
Buka App
   ↓
[Sudah login?] ──Ya──→ Chat UI
   ↓ Tidak
Login / Sign Up (Email atau Google)
   ↓
Onboarding 1: Masukkan Nama
   ↓
Onboarding 2: Pilih Topik (min. 2)
   ↓
Onboarding 3: Zira menyapa (bisa skip)
   ↓
Chat UI — siap digunakan
```

---

## Zira Labs

Akses dari sidebar kiri. Setiap lab mengaktifkan mode AI khusus:

| Lab | Ikon | Fungsi |
|---|---|---|
| Zira Code | `</>`| Coding assistant + syntax highlighting |
| Zira Design | 🎨 | UI/UX & design consultant |
| Zira Image | 🖼️ | AI image generation |
| Zira Vid | 🎬 | Script & storyboard video |
| Zira Auto | ⚡ | Otomatisasi via n8n workflow |
| Zira Research | 🔬 | Riset mendalam + extended thinking |

---

## Database Schema

```
Users ──────┬── UserMemory (topics, custom_instruction)
            ├── Chats ───── Messages (role, content, lab_module)
            └── DashboardTransactions (amount, status, method)
```

---

## Deploy ke Railway

1. Push repo ke GitHub
2. Buka [railway.app](https://railway.app) → **New Project** → **Deploy from GitHub**
3. Pilih repo `zira-ai`
4. Tambahkan semua environment variables dari `.env`
5. Railway otomatis build dan deploy

---

## Kontribusi

Pull request sangat disambut! Untuk perubahan besar, buka issue terlebih dahulu.

```bash
git checkout -b feat/nama-fitur
git commit -m "feat: tambah fitur X"
git push origin feat/nama-fitur
```

Baca [CONTRIBUTING.md](CONTRIBUTING.md) untuk panduan lengkap.

---

## Lisensi

Didistribusikan di bawah lisensi MIT. Lihat [LICENSE](LICENSE) untuk detail.

---

<div align="center">

Dibuat dengan ❤️ oleh tim Zira AI

</div>
