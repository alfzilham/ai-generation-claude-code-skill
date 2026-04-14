# Zira AI — Chat UI Components

## CSS Variables & Base Theme

```css
:root {
  --primary-cyan: #00FFFF;
  --deep-space: #1A1A1B;
  --font-ui: 'Plus Jakarta Sans', sans-serif;
  --font-code: 'JetBrains Mono', 'Fira Code', monospace;

  /* Light mode */
  --bg-primary: #ffffff;
  --bg-sidebar: #f5f5f5;
  --text-primary: #1a1a1b;
  --text-secondary: #6b7280;
  --border: #e5e7eb;
}

[data-theme="dark"] {
  --bg-primary: #0d0d0e;
  --bg-sidebar: #1A1A1B;
  --text-primary: #f9fafb;
  --text-secondary: #9ca3af;
  --border: #2d2d2e;
}

* { font-family: var(--font-ui); }
code, pre { font-family: var(--font-code); }
```

---

## Layout Utama (chat.html)

```html
<div class="app-layout">
  <!-- LEFT SIDEBAR -->
  <aside class="sidebar" id="sidebar">
    <!-- User profile -->
    <div class="user-profile">
      <img src="" alt="" class="avatar" id="user-avatar" />
      <div>
        <div class="user-name" id="user-name-display"></div>
        <div class="user-email" id="user-email-display"></div>
      </div>
    </div>

    <!-- Search history -->
    <div class="search-wrapper">
      <i class="bi bi-search"></i>
      <input type="text" placeholder="Cari history..." id="history-search" />
    </div>

    <!-- History button -->
    <button class="sidebar-btn" onclick="openHistoryLightbox()">
      <i class="bi bi-clock-history"></i> History
    </button>

    <div class="sidebar-divider"></div>

    <!-- Zira Labs shortcuts -->
    <div class="labs-label">Zira Labs</div>
    <button class="sidebar-btn lab-btn" data-lab="code">
      <i class="bi bi-code-slash"></i> Code
    </button>
    <button class="sidebar-btn lab-btn" data-lab="design">
      <i class="bi bi-palette"></i> Design
    </button>
    <button class="sidebar-btn lab-btn" data-lab="image">
      <i class="bi bi-image"></i> Image
    </button>
    <button class="sidebar-btn lab-btn" data-lab="vid">
      <i class="bi bi-camera-video"></i> Vid
    </button>
    <button class="sidebar-btn lab-btn" data-lab="auto">
      <i class="bi bi-lightning-charge"></i> Auto
    </button>
    <button class="sidebar-btn lab-btn" data-lab="research">
      <i class="bi bi-search-heart"></i> Research
    </button>

    <div class="sidebar-divider"></div>

    <button class="sidebar-btn" onclick="window.open('/cli-docs')">
      <i class="bi bi-terminal"></i> Zira CLI Docs
    </button>
    <button class="sidebar-btn" onclick="window.open('/feedback')">
      <i class="bi bi-chat-dots"></i> Feedback
    </button>
  </aside>

  <!-- MAIN CONTENT -->
  <main class="chat-main">
    <!-- TOP BAR -->
    <header class="top-bar" id="top-bar">
      <div class="chat-title-area">
        <div class="ai-logo">
          <img src="/assets/zira-logo.png" alt="Zira" />
        </div>
        <span class="chat-title" id="chat-title">Obrolan Baru</span>
      </div>
      <div class="top-actions">
        <button class="icon-btn" onclick="createNewChat()" title="Chat Baru">
          <i class="bi bi-pencil-square"></i>
        </button>
        <button class="icon-btn" onclick="toggleIncognito()" title="Incognito" id="incognito-btn">
          <i class="bi bi-incognito"></i>
        </button>
        <!-- Settings dropdown -->
        <div class="dropdown">
          <button class="icon-btn" onclick="toggleSettingsDropdown()">
            <i class="bi bi-gear"></i>
          </button>
          <div class="dropdown-menu" id="settings-dropdown">
            <a onclick="openSettings()">⚙️ Pengaturan</a>
            <a onclick="openLanguage()">🌐 Bahasa</a>
            <a onclick="openFAQ()">❓ Bantuan</a>
            <hr/>
            <a onclick="logout()" class="danger">🚪 Logout</a>
          </div>
        </div>
      </div>
    </header>

    <!-- MESSAGES AREA -->
    <div class="messages-area" id="messages-area">
      <!-- Greeting default -->
      <div class="greeting-message">
        <h2>What shall we do, <span id="greeting-name">User</span>?</h2>
      </div>
    </div>

    <!-- INPUT AREA -->
    <footer class="input-area">
      <div class="input-meta">
        <div class="user-chip">
          <i class="bi bi-person-circle"></i>
          <span id="input-user-name"></span>
          <span id="input-user-email" class="email-text"></span>
        </div>
        <div class="input-controls">
          <!-- Extended thinking toggle -->
          <button class="control-btn" id="thinking-btn" onclick="toggleThinking()" title="Extended Thinking">
            <i class="bi bi-lightbulb"></i>
            <span class="control-label">Extended</span>
          </button>
          <!-- Model selector -->
          <select class="model-select" id="model-select">
            <option value="claude-sonnet-4-20250514">Sonnet 4</option>
            <option value="claude-opus-4-20250514">Opus 4</option>
            <option value="claude-haiku-4-5-20251001">Haiku 4.5</option>
          </select>
          <!-- Voice input -->
          <button class="control-btn" id="voice-btn" onclick="startVoiceInput()" title="Voice Input">
            <i class="bi bi-mic"></i>
          </button>
        </div>
      </div>
      <div class="input-row">
        <textarea
          id="chat-input"
          placeholder="Tulis pesanmu..."
          rows="1"
          onkeydown="handleInputKey(event)"
          oninput="autoResize(this)"
        ></textarea>
        <button class="send-btn" onclick="sendMessage()" id="send-btn">
          <i class="bi bi-send-fill"></i>
        </button>
      </div>
    </footer>
  </main>
</div>
```

---

## JavaScript: Kirim Pesan & Streaming

```javascript
// chat.js
let chatHistory = [];
let activeLab = null;
let extendedThinking = false;
let currentChatId = null;

async function sendMessage() {
  const input = document.getElementById('chat-input');
  const text = input.value.trim();
  if (!text) return;

  input.value = '';
  autoResize(input);

  // Tambah pesan user ke UI
  appendMessage('user', text);
  chatHistory.push({ role: 'user', content: text });

  // Generate title saat pesan pertama
  if (chatHistory.length === 1 && !isIncognito) {
    generateAndSetTitle(text);
  }

  // Buat placeholder untuk AI response
  const aiMessageEl = appendMessage('assistant', '');
  const contentEl = aiMessageEl.querySelector('.message-content');

  // Tampilkan indikator thinking jika aktif
  if (extendedThinking) showThinkingIndicator(contentEl);

  try {
    const response = await fetch('/api/chat/stream', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'Authorization': `Bearer ${getToken()}` },
      body: JSON.stringify({
        message: text,
        history: chatHistory.slice(0, -1),  // Tanpa pesan terakhir
        extended_thinking: extendedThinking,
        model: document.getElementById('model-select').value,
        used_lab_module: activeLab
      })
    });

    const reader = response.body.getReader();
    const decoder = new TextDecoder();
    let fullResponse = '';

    hideThinkingIndicator(contentEl);

    while (true) {
      const { done, value } = await reader.read();
      if (done) break;
      const chunk = decoder.decode(value);
      const lines = chunk.split('\n');
      for (const line of lines) {
        if (line.startsWith('data: ') && line !== 'data: [DONE]') {
          const data = JSON.parse(line.slice(6));
          fullResponse += data.text;
          contentEl.innerHTML = renderMarkdown(fullResponse);
        }
      }
    }

    chatHistory.push({ role: 'assistant', content: fullResponse });

  } catch (err) {
    contentEl.innerHTML = '<span class="error">Gagal terhubung ke Zira AI.</span>';
  }
}

function appendMessage(role, content) {
  const area = document.getElementById('messages-area');
  const el = document.createElement('div');
  el.className = `message message-${role}`;
  el.innerHTML = `
    <div class="message-avatar">
      ${role === 'assistant' ? '<img src="/assets/zira-logo.png" />' : '<i class="bi bi-person-circle"></i>'}
    </div>
    <div class="message-content">${renderMarkdown(content)}</div>
  `;
  area.appendChild(el);
  area.scrollTop = area.scrollHeight;
  return el;
}

function handleInputKey(e) {
  if (e.key === 'Enter' && !e.shiftKey) {
    e.preventDefault();
    sendMessage();
  }
}

function autoResize(el) {
  el.style.height = 'auto';
  el.style.height = Math.min(el.scrollHeight, 200) + 'px';
}

function toggleThinking() {
  extendedThinking = !extendedThinking;
  document.getElementById('thinking-btn').classList.toggle('active', extendedThinking);
}
```

---

## Incognito Mode

```javascript
let isIncognito = false;

function toggleIncognito() {
  isIncognito = !isIncognito;
  document.getElementById('sidebar').style.display = isIncognito ? 'none' : 'flex';
  document.getElementById('top-bar').style.display = isIncognito ? 'none' : 'flex';
  document.querySelector('.user-chip').style.display = isIncognito ? 'none' : 'flex';

  // Ganti ikon incognito
  const btn = document.getElementById('incognito-btn');
  btn.innerHTML = isIncognito
    ? '<i class="bi bi-eye-slash-fill"></i>'
    : '<i class="bi bi-incognito"></i>';
  btn.classList.toggle('active', isIncognito);

  // Buat chat baru mode incognito
  currentChatId = null;
  chatHistory = [];
  document.getElementById('messages-area').innerHTML = '';
}
```

---

## Voice Input

```javascript
function startVoiceInput() {
  if (!('webkitSpeechRecognition' in window || 'SpeechRecognition' in window)) {
    alert('Browser tidak mendukung voice input.');
    return;
  }
  const SR = window.SpeechRecognition || window.webkitSpeechRecognition;
  const recognition = new SR();
  recognition.lang = 'id-ID';  // Default Indonesia, bisa disesuaikan
  recognition.continuous = false;
  recognition.interimResults = false;

  const btn = document.getElementById('voice-btn');
  btn.classList.add('recording');

  recognition.onresult = (e) => {
    document.getElementById('chat-input').value = e.results[0][0].transcript;
    btn.classList.remove('recording');
  };
  recognition.onerror = () => btn.classList.remove('recording');
  recognition.start();
}
```
