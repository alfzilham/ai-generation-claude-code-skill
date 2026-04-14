---
name: zira-ai
description: >
  Skill utama untuk membangun aplikasi Zira AI — AI chat assistant berbasis FastAPI (Python) + Vanilla JS.
  Gunakan skill ini SETIAP KALI bekerja di repo Zira AI, termasuk saat: membuat endpoint FastAPI baru,
  membangun komponen UI Chat, mengintegrasikan Anthropic Claude API (streaming/extended thinking),
  membuat alur onboarding, membangun modul Zira Labs (Code/Image/Auto/Research), mengerjakan dashboard
  monetisasi (Xendit/DANA/BSI), setup database schema, atau mengerjakan fitur apapun yang disebutkan
  dalam mainmap Zira AI. Juga trigger saat user menyebut "Zira", "endpoint chat", "onboarding",
  "streaming AI", "n8n webhook", "Zira Labs", atau "dashboard revenue".
---

# Zira AI — Master Skill

Skill ini adalah panduan lengkap untuk membangun Zira AI. Baca seluruh file ini sebelum menulis kode apapun.

## Arsitektur Sistem

```
zira-ai/
├── backend/
│   ├── main.py              # FastAPI entry point
│   ├── routers/
│   │   ├── auth.py          # Login, signup, Google OAuth
│   │   ├── chat.py          # Chat endpoints + streaming
│   │   ├── onboarding.py    # Onboarding flow
│   │   ├── labs.py          # Zira Labs modules
│   │   └── dashboard.py     # Revenue & analytics
│   ├── models/
│   │   └── database.py      # SQLAlchemy models
│   ├── services/
│   │   ├── claude_service.py   # Anthropic API wrapper
│   │   ├── memory_service.py   # AI memory management
│   │   └── payment_service.py  # Xendit integration
│   └── utils/
│       └── auth.py          # JWT helpers
├── frontend/
│   ├── index.html           # Entry point (redirect to login/chat)
│   ├── pages/
│   │   ├── login.html
│   │   ├── signup.html
│   │   ├── onboarding.html
│   │   └── chat.html        # Main chat UI
│   ├── css/
│   │   └── style.css
│   └── js/
│       ├── auth.js
│       ├── chat.js
│       ├── onboarding.js
│       └── labs.js
├── .env
├── requirements.txt
└── README.md
```

---

## Tech Stack

| Layer | Teknologi |
|---|---|
| Backend | Python 3.10+, FastAPI, Uvicorn |
| AI | Anthropic SDK (`anthropic`) — model `claude-sonnet-4-20250514` |
| Frontend | HTML5, CSS3, Vanilla JavaScript |
| Database | PostgreSQL (prod) / SQLite (dev) via SQLAlchemy |
| Auth | JWT + Google OAuth2 |
| Automation | n8n (webhook trigger dari FastAPI) |
| Payment | Xendit SDK (Python) |
| Icons | Bootstrap Icons |
| Font | Plus Jakarta Sans (UI), JetBrains Mono (code blocks) |
| Deploy | Railway (connect GitHub repo) |

---

## Branding & UI Rules

Selalu terapkan ini di semua komponen UI:

```css
/* Warna utama */
--primary-cyan: #00FFFF;       /* tombol utama, highlight, garis aktif */
--deep-space: #1A1A1B;         /* sidebar, background elemen khusus */

/* Font */
--font-ui: 'Plus Jakarta Sans', sans-serif;
--font-code: 'JetBrains Mono', 'Fira Code', monospace;

/* Shape */
border-radius: 12px;           /* semua komponen — rounded, tidak tajam */
box-shadow: 0 2px 8px rgba(0,0,0,0.08);  /* soft shadow */
```

**Ikon:** Gunakan `bootstrap-icons` CDN. Contoh: `<i class="bi bi-send-fill"></i>`

**Dark/Light mode:** Dukung keduanya. Gunakan CSS custom properties, bukan nilai hardcode.

---

## Database Schema

Gunakan SQLAlchemy. Berikut model lengkap:

```python
# models/database.py
from sqlalchemy import Column, String, Boolean, Integer, Enum, Text, TIMESTAMP, ForeignKey, ARRAY
from sqlalchemy.dialects.postgresql import UUID
import uuid

class User(Base):
    __tablename__ = "users"
    user_id      = Column(UUID, primary_key=True, default=uuid.uuid4)
    full_name    = Column(String, nullable=False)
    email        = Column(String, unique=True, nullable=False)
    auth_provider = Column(Enum("Google", "Email", name="auth_provider"))
    profile_picture = Column(String)  # URL

class UserMemory(Base):
    __tablename__ = "user_memory"
    memory_id    = Column(UUID, primary_key=True, default=uuid.uuid4)
    user_id      = Column(UUID, ForeignKey("users.user_id"))
    topics       = Column(ARRAY(String))       # ["Developer", "AI", "Design"]
    custom_instruction = Column(Text)          # Di-generate otomatis dari topics

class Chat(Base):
    __tablename__ = "chats"
    chat_id      = Column(UUID, primary_key=True, default=uuid.uuid4)
    user_id      = Column(UUID, ForeignKey("users.user_id"), nullable=True)  # NULL = incognito
    title        = Column(String)              # Di-generate AI dari pesan pertama
    is_incognito = Column(Boolean, default=False)
    created_at   = Column(TIMESTAMP)

class Message(Base):
    __tablename__ = "messages"
    message_id   = Column(UUID, primary_key=True, default=uuid.uuid4)
    chat_id      = Column(UUID, ForeignKey("chats.chat_id"))
    role         = Column(Enum("user", "assistant", name="message_role"))
    content      = Column(Text)
    used_lab_module = Column(String)           # "Zira Code", "Zira Image", dll — nullable
    created_at   = Column(TIMESTAMP)

class DashboardTransaction(Base):
    __tablename__ = "dashboard_transactions"
    transaction_id   = Column(UUID, primary_key=True, default=uuid.uuid4)
    user_id          = Column(UUID, ForeignKey("users.user_id"))
    amount           = Column(Integer)         # Dalam Rupiah, misal: 1000000
    status           = Column(Enum("Pending", "Success", name="tx_status"))
    withdrawal_method = Column(String)         # "DANA", "BSI"
```

---

## Anthropic Claude API

### Setup dasar

```python
# services/claude_service.py
import anthropic
import os

client = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])
MODEL = "claude-sonnet-4-20250514"
```

### Streaming response (endpoint utama chat)

```python
from fastapi import APIRouter
from fastapi.responses import StreamingResponse
import json

router = APIRouter()

@router.post("/chat/stream")
async def stream_chat(payload: dict):
    user_message = payload["message"]
    history      = payload.get("history", [])       # list {role, content}
    memory       = payload.get("custom_instruction", "")
    model_choice = payload.get("model", MODEL)
    use_thinking = payload.get("extended_thinking", False)

    system_prompt = f"""Kamu adalah Zira AI, asisten AI yang cerdas dan ramah.
{f"Instruksi khusus pengguna: {memory}" if memory else ""}"""

    messages = history + [{"role": "user", "content": user_message}]

    kwargs = {
        "model": model_choice,
        "max_tokens": 8000,
        "system": system_prompt,
        "messages": messages,
    }

    if use_thinking:
        kwargs["thinking"] = {"type": "enabled", "budget_tokens": 5000}

    def generate():
        with client.messages.stream(**kwargs) as stream:
            for text in stream.text_stream:
                yield f"data: {json.dumps({'text': text})}\n\n"
        yield "data: [DONE]\n\n"

    return StreamingResponse(generate(), media_type="text/event-stream")
```

### Generate chat title otomatis

```python
async def generate_chat_title(first_message: str) -> str:
    response = client.messages.create(
        model=MODEL,
        max_tokens=20,
        messages=[{
            "role": "user",
            "content": f"Buat judul singkat (maks 5 kata) untuk percakapan yang dimulai dengan: '{first_message}'. Hanya balas judulnya saja, tanpa tanda kutip."
        }]
    )
    return response.content[0].text.strip()
```

### Generate custom instruction dari topics onboarding

```python
async def generate_custom_instruction(name: str, topics: list[str]) -> str:
    topics_str = ", ".join(topics)
    response = client.messages.create(
        model=MODEL,
        max_tokens=200,
        messages=[{
            "role": "user",
            "content": f"Buat system instruction singkat untuk AI assistant bernama Zira agar menyesuaikan respons untuk pengguna bernama {name} yang tertarik pada: {topics_str}. Tulis dalam 2-3 kalimat, sudut pandang orang kedua."
        }]
    )
    return response.content[0].text.strip()
```

---

## Alur Onboarding

Ikuti urutan ini dengan ketat. Redirect otomatis setelah setiap langkah berhasil.

```
/signup atau /login
    ↓ sukses
/onboarding/name     → user input nama → simpan ke UserMemory
    ↓ klik "Lanjut"
/onboarding/topics   → pilih min. 2 topik → generate custom_instruction
    ↓ klik "Lanjut"
/onboarding/hello    → Zira menyapa + perkenalan fitur → bisa di-skip
    ↓
/chat                → halaman utama
```

**Topik tersedia (untuk onboarding):**
`Developer`, `Designer`, `Penulis`, `Pelajar/Mahasiswa`, `Peneliti`, `Entrepreneur`, `Marketing`, `Data Analyst`, `Content Creator`, `Lainnya`

**Deteksi login otomatis:** Cek JWT di localStorage. Jika valid → redirect langsung ke `/chat`. Jika user baru (belum punya `UserMemory`) → redirect ke onboarding.

---

## Struktur Chat UI

### Left Sidebar (disembunyikan saat Incognito)

```
[Avatar] Nama User
         user@email.com
─────────────────────
🔍 Search history...
📋 History (lightbox)
─────────────────────
💻 Zira Code
🎨 Zira Design
🖼️ Zira Image
🎬 Zira Vid
⚡ Zira Auto
🔬 Zira Research
─────────────────────
📖 Zira CLI Docs
💬 Feedback
[Admin only: 🛠️ Feedback Admin]
```

### Main Chat Area

```
[AI Logo] [Chat Title]                    [New Chat] [Incognito]
──────────────────────────────────────────────────────────────
                    area percakapan
                  (render Markdown)
──────────────────────────────────────────────────────────────
[User] [email]         [Extended 🧠] [Model AI ▼] [🎤]
[Input message...                              ] [Send ➤]
```

### Top Navigation (disembunyikan saat Incognito)

```
[⚙️ Settings ▼]  →  Pengaturan | Bahasa | Bantuan | ─── | Logout
```

### Incognito Mode

Saat aktif:
- Sembunyikan: left sidebar, top nav, user profile, email
- `is_incognito = true` di tabel Chats → tidak simpan history
- Ganti ikon incognito dengan ikon khusus di posisi yang sama
- Tampilkan hanya chat area bersih

---

## Zira Labs — Modul-modul

Setiap Lab dipanggil dari sidebar. Saat aktif, simpan ke kolom `used_lab_module` di Messages.

| Modul | Fungsi | Implementasi |
|---|---|---|
| Zira Code | Coding assistant + syntax highlight | System prompt khusus coding + highlight.js |
| Zira Design | Design assistant | System prompt + contoh visual |
| Zira Image | Generate gambar | Integrasi image generation API (Replicate/DALL-E) |
| Zira Vid | Video assistant | Deskripsi & script video |
| Zira Auto | Otomatisasi via n8n | POST ke n8n webhook → trigger workflow |
| Zira Research | Riset mendalam | Tool use: web search + extended thinking |

### n8n Integration (Zira Auto)

```python
# routers/labs.py
import httpx

@router.post("/labs/auto")
async def trigger_automation(payload: dict):
    webhook_url = os.environ["N8N_WEBHOOK_URL"]
    async with httpx.AsyncClient() as http:
        r = await http.post(webhook_url, json={
            "user_id": payload["user_id"],
            "task": payload["task"],
            "parameters": payload.get("parameters", {})
        })
    return {"status": "triggered", "n8n_response": r.json()}
```

---

## Dashboard & Monetisasi

### Kalkulasi Revenue

```python
# 1 user aktif = Rp 1.000.000
def calculate_revenue(active_users: int) -> int:
    return active_users * 1_000_000

# Analytics: user per bulan, per tahun, per negara
@router.get("/dashboard/analytics")
async def get_analytics():
    # Query dari tabel users + sessions
    # Return: monthly_users, yearly_users, users_by_country
    pass
```

### Xendit Withdrawal Flow

```python
# services/payment_service.py
import xendit  # pip install xendit-python

xendit.api_key = os.environ["XENDIT_API_KEY"]

async def create_disbursement(amount: int, method: str, account: str):
    # method: "DANA" atau "BSI"
    disbursement = xendit.Disbursement.create(
        external_id=f"zira-{uuid.uuid4()}",
        amount=amount,
        bank_code=method,     # "DANA" atau kode bank BSI
        account_holder_name="Zira AI Owner",
        account_number=account,
        description="Pencairan Zira AI"
    )
    # Simpan ke dashboard_transactions dengan status Pending
    return disbursement
```

---

## Environment Variables (.env)

```env
# AI
ANTHROPIC_API_KEY=sk-ant-...

# Database
DATABASE_URL=postgresql://user:pass@localhost/ziradb

# Auth
SECRET_KEY=your-jwt-secret-key
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...

# Payment
XENDIT_API_KEY=xnd_...

# n8n
N8N_WEBHOOK_URL=http://localhost:5678/webhook/...

# App
ENVIRONMENT=development
FRONTEND_URL=http://localhost:3000
```

---

## Requirements.txt

```
fastapi==0.111.0
uvicorn[standard]==0.30.0
anthropic>=0.30.0
sqlalchemy==2.0.30
psycopg2-binary==2.9.9
python-jose[cryptography]==3.3.0
passlib[bcrypt]==1.7.4
python-multipart==0.0.9
httpx==0.27.0
xendit-python==5.0.0
python-dotenv==1.0.0
alembic==1.13.1
```

---

## Cara Menjalankan Lokal

```bash
# 1. Clone & setup
python -m venv zira_env
source zira_env/bin/activate   # Windows: zira_env\Scripts\activate
pip install -r requirements.txt

# 2. Copy env
cp .env.example .env
# Edit .env dengan API keys kamu

# 3. Jalankan database migration
alembic upgrade head

# 4. Jalankan backend
uvicorn backend.main:app --reload --port 8000

# 5. (Opsional) Jalankan n8n untuk Zira Auto
npm install -g n8n
n8n  # Buka http://localhost:5678
```

---

## Panduan Penulisan Kode

1. **Selalu gunakan async/await** untuk semua endpoint FastAPI dan database calls.
2. **Streaming responses** harus menggunakan `StreamingResponse` dengan `text/event-stream`.
3. **Frontend fetch streaming** gunakan `ReadableStream` / `EventSource`.
4. **Semua warna UI** harus merujuk ke CSS custom properties (bukan hardcode hex), kecuali `--primary-cyan: #00FFFF` dan `--deep-space: #1A1A1B`.
5. **Incognito chat** → `user_id = NULL` di tabel Chats, jangan simpan Messages.
6. **Error handling** → selalu return `{"error": "pesan"}`  dengan HTTP status yang tepat.
7. **CORS** → konfigurasi di `main.py` untuk izinkan frontend URL dari `.env`.

---

## Referensi Cepat

- Lihat `references/onboarding-flow.md` untuk detail alur UI onboarding
- Lihat `references/chat-ui-components.md` untuk detail komponen HTML/CSS chat
- Lihat `references/labs-prompts.md` untuk system prompt masing-masing Zira Lab
