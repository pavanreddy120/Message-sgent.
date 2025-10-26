<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Message Sender Agent</title>
<style>
  body { font-family: system-ui, -apple-system, Roboto, "Segoe UI", sans-serif; margin:0; display:flex; flex-direction:column; height:100vh; }
  header { background:#0b5; padding:12px; color:#032; font-weight:700; display:flex; justify-content:space-between; align-items:center; }
  .container { padding:12px; flex:1; overflow:auto; background:#f5f6f7; }
  .msg { max-width:80%; margin:8px 0; padding:10px; border-radius:10px; }
  .user { background:#dfe7ff; margin-left:auto; text-align:right; }
  .agent { background:#fff; margin-right:auto; text-align:left; }
  footer { padding:10px; display:flex; gap:8px; border-top:1px solid #ddd; background:#fff; }
  input[type=text]{flex:1;padding:8px;border:1px solid #ccc;border-radius:8px;}
  button{padding:8px 12px;border-radius:8px;border:0;background:#0b5;color:white;font-weight:600;}
  .small { font-size:12px; color:#333; }
  .controls { display:flex; gap:8px; align-items:center; }
  .settings { display:none; padding:10px; background:#fff; border-top:1px solid #ddd; }
  label{display:block;margin:6px 0;font-size:13px;}
  textarea{width:100%;height:60px;border:1px solid #ccc;border-radius:6px;padding:6px}
</style>
</head>
<body>
<header>
  <div>Message Agent — XT2403</div>
  <div class="small">Local demo • Exportable</div>
</header>

<div class="container" id="chat"></div>

<div class="settings" id="settings">
  <label>LLM mode (optional)
    <select id="llmMode">
      <option value="none">Local rule-based (no internet)</option>
      <option value="openai">OpenAI Chat (requires API key)</option>
    </select>
  </label>
  <label>OpenAI API key (paste here if using OpenAI). Key is stored locally only.</label>
  <input type="text" id="openaiKey" placeholder="sk-..." />
  <label>System prompt / agent instructions</label>
  <textarea id="systemPrompt">You are a helpful message-sender agent that replies concisely, suggests next actions, and formats messages for download.</textarea>
  <button id="saveSettingsBtn">Save settings</button>
</div>

<footer>
  <div class="controls">
    <button id="toggleSettings">⚙️</button>
    <button id="exportBtn">⬇️ Export</button>
  </div>
  <input id="inputMsg" type="text" placeholder="Type a message..." />
  <button id="sendBtn">Send</button>
</footer>

<script>
/* Simple in-browser message-sender agent with optional OpenAI call.
   Usage:
   - Save file and open in browser (mobile recommended).
   - Settings toggle to configure optional OpenAI mode and API key.
   - Export saves conversation as .txt to device storage.
*/

// DOM
const chat = document.getElementById('chat');
const inputMsg = document.getElementById('inputMsg');
const sendBtn = document.getElementById('sendBtn');
const exportBtn = document.getElementById('exportBtn');
const toggleSettings = document.getElementById('toggleSettings');
const settings = document.getElementById('settings');
const llmMode = document.getElementById('llmMode');
const openaiKey = document.getElementById('openaiKey');
const systemPrompt = document.getElementById('systemPrompt');
const saveSettingsBtn = document.getElementById('saveSettingsBtn');

let convo = []; // {who: 'user'|'agent', text: '...'}

function renderMessage(who, text) {
  convo.push({who, text, time: new Date().toISOString()});
  const d = document.createElement('div');
  d.className = 'msg ' + (who === 'user' ? 'user' : 'agent');
  d.innerHTML = `<div>${escapeHtml(text)}</div><div style="font-size:11px;color:#666;margin-top:6px">${new Date().toLocaleString()}</div>`;
  chat.appendChild(d);
  chat.scrollTop = chat.scrollHeight;
}

function escapeHtml(s){ return s.replaceAll('&','&amp;').replaceAll('<','&lt;').replaceAll('>','&gt;'); }

async function onSend() {
  const text = inputMsg.value.trim();
  if(!text) return;
  renderMessage('user', text);
  inputMsg.value = '';
  // Agent reply
  renderMessage('agent', '⏳ thinking...');
  const lastAgentIndex = convo.length - 1;
  try {
    const reply = await generateReply(text);
    // replace placeholder
    convo[lastAgentIndex].text = reply;
    chat.children[chat.children.length-1].innerHTML = `<div>${escapeHtml(reply)}</div><div style="font-size:11px;color:#666;margin-top:6px">${new Date().toLocaleString()}</div>`;
  } catch (e) {
    convo[lastAgentIndex].text = 'Error: ' + e.message;
    chat.children[chat.children.length-1].textContent = 'Error: ' + e.message;
  }
}

function loadSettings() {
  const s = localStorage.getItem('agent_settings');
  if(s) {
    try {
      const obj = JSON.parse(s);
      llmMode.value = obj.mode || 'none';
      openaiKey.value = obj.key || '';
      systemPrompt.value = obj.prompt || systemPrompt.value;
    } catch {}
  }
}
function saveSettings() {
  const obj = {mode: llmMode.value, key: openaiKey.value.trim(), prompt: systemPrompt.value.trim()};
  localStorage.setItem('agent_settings', JSON.stringify(obj));
  alert('Settings saved locally.');
}
saveSettingsBtn.onclick = saveSettings;
toggleSettings.onclick = ()=> settings.style.display = (settings.style.display === 'none' ? 'block' : 'none');

async function generateReply(userText) {
  const settingsObj = JSON.parse(localStorage.getItem('agent_settings') || '{}');
  const mode = settingsObj.mode || 'none';
  const prompt = (settingsObj.prompt || systemPrompt.value) + "\\nUser: " + userText;
  if(mode === 'openai' && settingsObj.key) {
    // call OpenAI Chat completions (note: user supplies key; we do not send it anywhere)
    // We use the Chat Completions endpoint with model gpt-3.5-turbo as a default.
    const resp = await fetch('https://api.openai.com/v1/chat/completions', {
      method:'POST',
      headers: {
        'Content-Type':'application/json',
        'Authorization':'Bearer ' + settingsObj.key
      },
      body: JSON.stringify({
        model: 'gpt-3.5-turbo',
        messages: [{role:'system', content: settingsObj.prompt || systemPrompt.value},{role:'user', content: userText}],
        max_tokens: 400,
        temperature: 0.3
      })
    });
    if(!resp.ok) {
      const txt = await resp.text();
      throw new Error('LLM error: ' + resp.status + ' ' + txt.slice(0,200));
    }
    const j = await resp.json();
    const content = j.choices?.[0]?.message?.content || JSON.stringify(j);
    return content.trim();
  } else {
    // Local rule-based fallback: short responses, simple actions
    return localAgentReply(userText);
  }
}

function localAgentReply(text) {
  // Brutally simple local agent: patterns -> replies
  const t = text.toLowerCase();
  if(t.includes('hello')||t.includes('hi')) return 'Hello. What message would you like to send and to whom?';
  if(t.includes('send') && t.includes('sms')) return 'I can format an SMS message for you. Copy the content and paste into your messaging app.';
  if(t.includes('download')||t.includes('export')) return 'Use the Export button (⬇️) to download the conversation as a text file.';
  if(t.includes('help')) return 'I can: 1) compose messages, 2) format for SMS/email, 3) export conversation file, 4) optionally call OpenAI if you provide an API key in settings.';
  // fallback: echo + suggestion
  const short = text.length > 180 ? text.slice(0,180) + '…' : text;
  return `Echo: "${short}"\n\nTip: press Export to save this conversation. For smarter replies, enable OpenAI mode and paste your API key in settings.`;
}

sendBtn.onclick = onSend;
inputMsg.addEventListener('keydown', (e)=> { if(e.key === 'Enter') onSend(); });

exportBtn.onclick = ()=>{
  const lines = convo.map(c => `[${c.time}] ${c.who.toUpperCase()}: ${c.text}`).join('\\n\\n');
  const blob = new Blob([lines], {type:'text/plain'});
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  const name = 'chat_export_' + (new Date()).toISOString().replace(/[:.]/g,'-') + '.txt';
  a.download = name;
  document.body.appendChild(a);
  a.click();
  a.remove();
  URL.revokeObjectURL(url);
};

// load and initialize
loadSettings();
settings.style.display = 'none';
renderMessage('agent','Agent ready. Type a message or open settings to enable an LLM. (Local mode is offline and limited.)');
</script>
</body>
</html>
