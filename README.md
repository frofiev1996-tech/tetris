<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Мини-мессенджер</title>
<style>
  body{
    font-family:sans-serif;
    background:#031026;
    color:#fff;
    display:flex;
    flex-direction:column;
    align-items:center;
    justify-content:flex-start;
    min-height:100vh;
    padding:10px;
  }
  h1{margin-bottom:10px;}
  #chat{
    width:100%;
    max-width:400px;
    height:400px;
    border:1px solid #00d4ff;
    border-radius:10px;
    padding:10px;
    overflow-y:auto;
    margin-bottom:10px;
    background:#071025;
  }
  #message{
    width:calc(100% - 90px);
    padding:10px;
    border-radius:5px;
    border:none;
    font-size:16px;
  }
  button{
    padding:10px;
    border:none;
    border-radius:5px;
    background:#00d4ff;
    color:#000;
    font-weight:bold;
    margin-left:5px;
  }
  .msg{margin-bottom:8px;}
  .user{color:#00d4ff;}
</style>
</head>
<body>

<h1>Мини-мессенджер</h1>
<div id="chat"></div>
<div style="display:flex;width:100%;max-width:400px;">
  <input type="text" id="message" placeholder="Напиши сообщение...">
  <button onclick="sendMessage()">Отправить</button>
</div>

<!-- Firebase SDK -->
<script src="https://www.gstatic.com/firebasejs/9.23.0/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/9.23.0/firebase-database-compat.js"></script>

<script>
// --- Настройка Firebase ---
// Замените на свои данные из Firebase
const firebaseConfig = {
  apiKey: "ВАШ_API_KEY",
  authDomain: "ВАШ_PROJECT_ID.firebaseapp.com",
  databaseURL: "https://ВАШ_PROJECT_ID.firebaseio.com",
  projectId: "ВАШ_PROJECT_ID",
  storageBucket: "ВАШ_PROJECT_ID.appspot.com",
  messagingSenderId: "ВАШ_SENDER_ID",
  appId: "ВАШ_APP_ID"
};

firebase.initializeApp(firebaseConfig);
const db = firebase.database();

// --- Случайный ник ---
const username = 'Пользователь' + Math.floor(Math.random()*1000);

// --- Отправка сообщения ---
function sendMessage(){
  const msg = document.getElementById('message').value.trim();
  if(msg==='') return;
  db.ref('messages').push({user:username,text:msg});
  document.getElementById('message').value='';
}

// --- Получение сообщений ---
db.ref('messages').on('child_added', snapshot=>{
  const data = snapshot.val();
  const div = document.createElement('div');
  div.className = 'msg';
  div.innerHTML = `<span class="user">${data.user}:</span> ${data.text}`;
  document.getElementById('chat').appendChild(div);
  document.getElementById('chat').scrollTop = document.getElementById('chat').scrollHeight;
});
</script>
</body>
</html>
