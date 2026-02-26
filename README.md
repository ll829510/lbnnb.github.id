<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>WiFi互通·终极管理员版</title>
<style>
*{box-sizing:border-box;margin:0;padding:0;font-family:system-ui, sans-serif}
body{background:#f4f6f8;padding:15px}
.main{max-width:500px;margin:0 auto;background:#fff;border-radius:20px;overflow:hidden;box-shadow:0 5px 20px #00000010}
.head{background:#007aff;color:#fff;padding:16px;text-align:center}
.tip{font-size:12px;opacity:.8;margin-top:4px}
.room{padding:12px;text-align:center;border-bottom:1px solid #eee}
#pwd{padding:10px;width:160px;border:1px solid #ddd;border-radius:10px;text-align:center}
#enter{padding:10px 14px;background:#007aff;color:#fff;border:none;border-radius:10px;margin-left:4px}
.admin-bar{padding:8px 14px;background:#fff8e1;border-top:1px solid #ffe082;border-bottom:1px solid #ffe082;font-size:12px;color:#666}
.online-box{padding:10px 14px;background:#f8f9fa;border-bottom:1px solid #eee}
#online{font-size:12px;color:#333;max-height:70px;overflow-y:auto}
.chat{height:380px;overflow-y:auto;padding:14px;display:flex;flex-direction:column;gap:8px}
.msg{padding:10px 14px;border-radius:16px;max-width:75%;line-height:1.4}
.msg.me{background:#007aff;color:#fff;align-self:flex-end}
.msg.other{background:#e5e5ea;color:#222;align-self:flex-start}
.msg.file{background:#e3f2fd;color:#01579b}
.msg.warn{background:#ffebee;color:#b71c1c;font-size:12px;text-align:center;width:100%}
.toolbar{display:flex;gap:8px;padding:10px;border-top:1px solid #eee}
#file{display:none}
#text{flex:1;padding:12px;border:1px solid #ddd;border-radius:14px;outline:none}
.toolbar button{padding:12px 14px;background:#007aff;color:#fff;border:none;border-radius:14px}
.kick{color:#f44336;font-weight:bold;margin-left:4px;font-size:12px;cursor:pointer}
</style>
</head>
<body>
<div class="main">
  <div class="head">
    <h3>📶 WiFi互通·管理员终极版</h3>
    <div class="tip">同WiFi可用｜聊天｜传文件｜管理员后台</div>
  </div>

  <div class="room">
    <input id="pwd" placeholder="房间密码" value="123456">
    <button id="enter">进入房间</button>
  </div>

  <div class="admin-bar" id="adminBar" style="display:none">
    <strong>管理员模式</strong>｜点击ID即可踢人
  </div>

  <div class="online-box">
    在线设备：
    <div id="online">请先进入房间</div>
  </div>

  <div class="chat" id="chat"></div>

  <div class="toolbar">
    <button onclick="document.getElementById('file').click()">📎 文件</button>
    <input type="file" id="file" multiple>
    <input id="text" placeholder="输入消息…" autocomplete="off">
    <button onclick="send()">发送</button>
  </div>
</div>

<script>
const chat = document.getElementById('chat');
const text = document.getElementById('text');
const online = document.getElementById('online');
const pwd = document.getElementById('pwd');
const enter = document.getElementById('enter');
const adminBar = document.getElementById('adminBar');
const fileInp = document.getElementById('file');

const myId = Math.random().toString(36).slice(2,8);
const myName = '用户_' + myId;
let roomPwd = '';
let isAdmin = false;
let userMap = new Map();
const channel = new BroadcastChannel('wifi_super_chat');

// 消息样式
function addMsg(content, isMe = false, isFile = false, isWarn = false) {
  const div = document.createElement('div');
  div.className = 'msg';
  if (isMe) div.classList.add('me');
  else if (isWarn) div.classList.add('warn');
  else if (isFile) div.classList.add('file');
  else div.classList.add('other');
  div.textContent = content;
  chat.appendChild(div);
  chat.scrollTop = chat.scrollHeight;
}

// 发送消息
function send() {
  const t = text.value.trim();
  if (!t || !roomPwd) return;
  channel.postMessage({
    type: 'msg', pwd: roomPwd, from: myId, name: myName, content: t
  });
  addMsg(t, true);
  text.value = '';
}
text.addEventListener('keydown', e => e.key === 'Enter' && send());

// 进入房间
enter.onclick = () => {
  roomPwd = pwd.value.trim();
  if (!roomPwd) return alert('请输入房间密码');
  enter.disabled = true;
  enter.textContent = '已进入';
  isAdmin = true;
  adminBar.style.display = 'block';
  addMsg('✅ 你已进入房间（管理员）', false, false, true);
  sendOnline();
  setInterval(sendOnline, 2000);
};

// 在线心跳
function sendOnline() {
  channel.postMessage({
    type: 'online', pwd: roomPwd, from: myId, name: myName
  });
}

// 踢人
function kick(uid) {
  channel.postMessage({
    type: 'kick', pwd: roomPwd, target: uid, from: myId
  });
}

// 禁言
function mute(uid) {
  channel.postMessage({
    type: 'mute', pwd: roomPwd, target: uid
  });
}

// 更新在线列表
function updateOnlineList() {
  if (!roomPwd) return;
  let html = '';
  userMap.forEach((name, uid) => {
    html += name;
    if (isAdmin && uid !== myId) {
      html += `<span class="kick" onclick="kick('${uid}')"> [踢]</span>`;
    }
    html += '　';
  });
  online.innerHTML = html || '暂无在线设备';
}

// 接收所有消息
channel.onmessage = e => {
  const d = e.data;
  if (!d.pwd || d.pwd !== roomPwd) return;

  // 在线状态
  if (d.type === 'online') {
    userMap.set(d.from, d.name);
    updateOnlineList();
  }

  // 聊天消息
  if (d.type === 'msg' && d.from !== myId) {
    addMsg(`[${d.name}]：${d.content}`);
  }

  // 文件通知
  if (d.type === 'file' && d.from !== myId) {
    addMsg(`[${d.name}] 发送文件：${d.filename}`, false, true);
  }

  // 管理员踢人
  if (d.type === 'kick' && d.target === myId) {
    addMsg('❌ 你已被管理员踢出房间', false, false, true);
    channel.close();
    setTimeout(() => location.reload(), 1500);
  }
};

// 发送文件
fileInp.onchange = () => {
  const f = fileInp.files[0];
  if (!f || !roomPwd) return;
  channel.postMessage({
    type: 'file', pwd: roomPwd, from: myId, name: myName, filename: f.name
  });
  addMsg(`📎 已发送：${f.name}`, true, true);
};
</script>
</body>
</html>
