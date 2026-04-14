# Zira AI — Onboarding Flow Detail

## Halaman 1: Onboarding Name

**Endpoint:** `POST /onboarding/name`

```html
<!-- Tampilan UI -->
<div class="onboarding-card">
  <div class="zira-logo"></div>
  <h1>Hei! Siapa namamu?</h1>
  <p>Zira akan menyapamu dengan nama ini setiap saat.</p>
  <input type="text" id="user-name" placeholder="Masukkan namamu..." />
  <button onclick="submitName()">Lanjut →</button>
</div>
```

```python
# Backend
@router.post("/onboarding/name")
async def save_name(payload: dict, current_user: User = Depends(get_current_user)):
    name = payload["name"]
    # Update full_name di tabel Users
    # Buat record UserMemory baru (topics kosong dulu)
    db.query(User).filter(User.user_id == current_user.user_id).update({"full_name": name})
    memory = UserMemory(user_id=current_user.user_id, topics=[], custom_instruction="")
    db.add(memory)
    db.commit()
    return {"status": "ok", "redirect": "/onboarding/topics"}
```

---

## Halaman 2: Onboarding Topics

**Endpoint:** `POST /onboarding/topics`

```html
<div class="onboarding-card">
  <h1>Apa yang sering kamu kerjakan?</h1>
  <p>Pilih minimal 2 topik agar Zira bisa menyesuaikan diri untukmu.</p>
  <div class="topics-grid">
    <!-- Chip yang bisa dipilih/deselect -->
    <button class="topic-chip" data-topic="Developer">👨‍💻 Developer</button>
    <button class="topic-chip" data-topic="Designer">🎨 Designer</button>
    <button class="topic-chip" data-topic="Pelajar/Mahasiswa">📚 Pelajar</button>
    <button class="topic-chip" data-topic="Peneliti">🔬 Peneliti</button>
    <button class="topic-chip" data-topic="Entrepreneur">🚀 Entrepreneur</button>
    <button class="topic-chip" data-topic="Marketing">📢 Marketing</button>
    <button class="topic-chip" data-topic="Data Analyst">📊 Data Analyst</button>
    <button class="topic-chip" data-topic="Content Creator">✍️ Content Creator</button>
    <button class="topic-chip" data-topic="Penulis">📝 Penulis</button>
    <button class="topic-chip" data-topic="Lainnya">💡 Lainnya</button>
  </div>
  <p id="min-warning" style="display:none;color:red">Pilih minimal 2 topik</p>
  <button onclick="submitTopics()">Lanjut →</button>
</div>
```

```python
@router.post("/onboarding/topics")
async def save_topics(payload: dict, current_user: User = Depends(get_current_user)):
    topics = payload["topics"]
    if len(topics) < 2:
        raise HTTPException(400, "Minimal 2 topik harus dipilih")

    # Generate custom instruction via Claude
    instruction = await generate_custom_instruction(current_user.full_name, topics)

    db.query(UserMemory).filter(UserMemory.user_id == current_user.user_id).update({
        "topics": topics,
        "custom_instruction": instruction
    })
    db.commit()
    return {"status": "ok", "redirect": "/onboarding/hello"}
```

---

## Halaman 3: Onboarding Hello

**Endpoint:** `GET /onboarding/hello` (stream AI greeting)

```python
@router.get("/onboarding/hello/stream")
async def hello_stream(current_user: User = Depends(get_current_user)):
    memory = db.query(UserMemory).filter(UserMemory.user_id == current_user.user_id).first()

    greeting_prompt = f"""Kamu adalah Zira AI. Perkenalkan dirimu kepada {current_user.full_name} yang baru saja bergabung.
Sebutkan bahwa kamu tahu ia tertarik pada: {', '.join(memory.topics)}.
Sebutkan fitur utama secara singkat: Zira Code, Zira Image, Zira Auto, Zira Research.
Gunakan bahasa yang hangat, singkat (maks 4 kalimat), dan antusias."""

    def generate():
        with client.messages.stream(
            model=MODEL,
            max_tokens=300,
            messages=[{"role": "user", "content": greeting_prompt}]
        ) as stream:
            for text in stream.text_stream:
                yield f"data: {json.dumps({'text': text})}\n\n"
        yield "data: [DONE]\n\n"

    return StreamingResponse(generate(), media_type="text/event-stream")
```

```html
<!-- UI: tampilkan streaming greeting + tombol Skip -->
<div class="onboarding-card hello-card">
  <div id="greeting-text"></div>   <!-- Di-stream dari AI -->
  <div class="hello-actions">
    <button onclick="window.location='/chat'" class="btn-primary">Mulai Chat →</button>
    <a href="/chat" class="skip-link">Lewati</a>
  </div>
</div>
```
