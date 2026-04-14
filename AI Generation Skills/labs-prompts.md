# Zira AI — Labs System Prompts

Setiap modul Zira Labs menggunakan system prompt tambahan yang di-inject bersama `custom_instruction` user.
Simpan `used_lab_module` ke kolom `Messages.used_lab_module` saat lab aktif.

---

## Zira Code

```python
LAB_CODE_PROMPT = """Kamu adalah Zira Code, asisten coding expert dari Zira AI.
- Selalu berikan kode yang bersih, well-commented, dan siap pakai.
- Saat menjelaskan kode, gunakan format markdown dengan syntax highlighting.
- Jika ada bug, identifikasi akar masalah dan berikan solusi konkret.
- Dukung semua bahasa pemrograman populer.
- Sertakan contoh penggunaan untuk setiap fungsi atau kelas yang kamu buat."""
```

Frontend: Aktifkan `highlight.js` untuk syntax highlighting di code blocks.

```html
<!-- Tambah di chat.html -->
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.9.0/styles/github-dark.min.css">
<script src="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.9.0/highlight.min.js"></script>
<script>
// Setelah render markdown, jalankan highlight
function highlightCode() {
  document.querySelectorAll('pre code').forEach(el => hljs.highlightElement(el));
}
</script>
```

---

## Zira Design

```python
LAB_DESIGN_PROMPT = """Kamu adalah Zira Design, asisten desain UI/UX dari Zira AI.
- Bantu pengguna dengan konsep desain, color palette, typography, dan layout.
- Berikan saran berdasarkan prinsip desain modern (Material, Human Interface, dll).
- Saat mendeskripsikan desain, sertakan nilai CSS, kode HTML/CSS jika relevan.
- Pahami branding Zira AI: primary cyan #00FFFF, rounded corners, soft shadows."""
```

---

## Zira Image

```python
LAB_IMAGE_PROMPT = """Kamu adalah Zira Image, asisten generasi gambar dari Zira AI.
- Bantu user membuat prompt yang baik untuk image generation.
- Jelaskan parameter seperti style, aspect ratio, lighting, mood.
- Terjemahkan deskripsi Indonesia ke prompt Bahasa Inggris yang optimal."""
```

Backend: Integrasi image generation API.

```python
# routers/labs.py
import replicate  # pip install replicate

@router.post("/labs/image/generate")
async def generate_image(payload: dict):
    prompt = payload["prompt"]

    # Opsi 1: Replicate (Stable Diffusion / FLUX)
    output = replicate.run(
        "black-forest-labs/flux-schnell",
        input={"prompt": prompt, "num_outputs": 1}
    )
    return {"image_url": output[0]}

    # Opsi 2: OpenAI DALL-E (jika lebih familiar)
    # import openai
    # response = openai.images.generate(model="dall-e-3", prompt=prompt, n=1)
    # return {"image_url": response.data[0].url}
```

---

## Zira Vid

```python
LAB_VID_PROMPT = """Kamu adalah Zira Vid, asisten konten video dari Zira AI.
- Bantu buat script video, storyboard, dan deskripsi konten.
- Berikan saran struktur video: hook, isi, call-to-action.
- Pahami berbagai format: YouTube, TikTok, Reels, presentasi."""
```

---

## Zira Auto (n8n Integration)

```python
LAB_AUTO_PROMPT = """Kamu adalah Zira Auto, asisten otomatisasi dari Zira AI.
- Bantu user mendeskripsikan workflow otomatisasi yang ingin mereka buat.
- Tanyakan: trigger apa yang diinginkan, aksi apa yang harus dilakukan, kondisi apa yang berlaku.
- Setelah user konfirmasi, trigger workflow via n8n webhook."""
```

Backend flow:

```python
@router.post("/labs/auto/execute")
async def execute_automation(payload: dict, current_user: User = Depends(get_current_user)):
    # 1. Kirim ke Claude untuk parse intent user
    parsed = await parse_automation_intent(payload["description"])

    # 2. Trigger n8n webhook
    webhook_url = os.environ["N8N_WEBHOOK_URL"]
    async with httpx.AsyncClient() as http:
        result = await http.post(webhook_url, json={
            "user_id": str(current_user.user_id),
            "workflow_type": parsed["type"],
            "parameters": parsed["parameters"]
        })

    return {"status": "Workflow berhasil dijalankan", "result": result.json()}
```

---

## Zira Research

```python
LAB_RESEARCH_PROMPT = """Kamu adalah Zira Research, asisten riset mendalam dari Zira AI.
- Lakukan analisis komprehensif terhadap topik yang diberikan.
- Gunakan extended thinking untuk menghasilkan jawaban yang mendalam.
- Strukturkan output: Ringkasan → Analisis → Temuan Kunci → Rekomendasi.
- Sertakan sumber/referensi jika relevan."""
```

Backend: Aktifkan `extended_thinking` dan `max_tokens` tinggi untuk Research.

```python
@router.post("/labs/research/query")
async def research_query(payload: dict, current_user: User = Depends(get_current_user)):
    def generate():
        with client.messages.stream(
            model=MODEL,
            max_tokens=16000,
            system=LAB_RESEARCH_PROMPT,
            thinking={"type": "enabled", "budget_tokens": 10000},
            messages=[{"role": "user", "content": payload["query"]}]
        ) as stream:
            for text in stream.text_stream:
                yield f"data: {json.dumps({'text': text})}\n\n"
        yield "data: [DONE]\n\n"

    return StreamingResponse(generate(), media_type="text/event-stream")
```

---

## Cara Inject Lab ke Endpoint Chat Utama

```python
# Dalam claude_service.py
LAB_PROMPTS = {
    "Zira Code":     LAB_CODE_PROMPT,
    "Zira Design":   LAB_DESIGN_PROMPT,
    "Zira Image":    LAB_IMAGE_PROMPT,
    "Zira Vid":      LAB_VID_PROMPT,
    "Zira Auto":     LAB_AUTO_PROMPT,
    "Zira Research": LAB_RESEARCH_PROMPT,
}

def build_system_prompt(custom_instruction: str, lab_module: str = None) -> str:
    base = "Kamu adalah Zira AI, asisten AI yang cerdas dan ramah."
    if custom_instruction:
        base += f"\n\nInstruksi pengguna: {custom_instruction}"
    if lab_module and lab_module in LAB_PROMPTS:
        base += f"\n\n{LAB_PROMPTS[lab_module]}"
    return base
```
