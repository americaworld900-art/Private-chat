private-chat/
 ├── server.js
 └── public/
      └── index.html
      const express = require("express");
const http = require("http");
const { Server } = require("socket.io");
const bcrypt = require("bcryptjs");
const { randomUUID } = require("crypto");

const app = express();
const server = http.createServer(app);
const io = new Server(server);

app.use(express.json());
app.use(express.static("public"));

const rooms = new Map();

// Create private chat room
app.post("/create-room", async (req, res) => {
    const { password } = req.body;

    if (!password || password.length < 6) {
        return res.status(400).json({
            error: "Password कम्तीमा 6 अक्षरको हुनुपर्छ।"
        });
    }

    const roomId = randomUUID().slice(0, 8);
    const passwordHash = await bcrypt.hash(password, 12);

    rooms.set(roomId, {
        passwordHash,
        messages: []
    });

    res.json({ roomId });
});

// Join private room
io.on("connection", (socket) => {

    socket.on("joinRoom", async ({ roomId, password }) => {
        const room = rooms.get(roomId);

        if (!room) {
            return socket.emit("errorMessage", "Chat room भेटिएन।");
        }

        const valid = await bcrypt.compare(
            password,
            room.passwordHash
        );

        if (!valid) {
            return socket.emit("errorMessage", "गलत पासवर्ड!");
        }

        socket.join(roomId);
        socket.data.roomId = roomId;

        socket.emit("joined", room.messages);
    });

    // Send message
    socket.on("sendMessage", (message) => {
        const roomId = socket.data.roomId;
        const room = rooms.get(roomId);

        if (!room || typeof message !== "string") return;

        const cleanMessage = message.trim().slice(0, 2000);
        if (!cleanMessage) return;

        const data = {
            message: cleanMessage,
            time: new Date().toLocaleTimeString()
        };

        room.messages.push(data);

        if (room.messages.length > 100) {
            room.messages.shift();
        }

        io.to(roomId).emit("newMessage", data);
    });
});

server.listen(3000, () => {
    console.log("Chat website running at http://localhost:3000");
});
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">

<title>Private Chat</title>

<script src="/socket.io/socket.io.js"></script>

<style>
* {
    box-sizing: border-box;
}

body {
    margin: 0;
    font-family: Arial, sans-serif;
    background: #101827;
    color: white;
    display: flex;
    justify-content: center;
    align-items: center;
    min-height: 100vh;
}

.container {
    width: 95%;
    max-width: 450px;
    background: #1e293b;
    padding: 25px;
    border-radius: 18px;
    box-shadow: 0 10px 40px #0006;
}

h2 {
    text-align: center;
    color: #38bdf8;
}

input {
    width: 100%;
    padding: 14px;
    margin: 10px 0;
    border: none;
    border-radius: 9px;
    background: #334155;
    color: white;
    font-size: 15px;
}

button {
    width: 100%;
    padding: 13px;
    border: none;
    border-radius: 9px;
    background: #0284c7;
    color: white;
    font-size: 16px;
    cursor: pointer;
    margin-top: 10px;
}

#chatPage {
    display: none;
}

#messages {
    height: 350px;
    overflow-y: auto;
    background: #0f172a;
    padding: 12px;
    border-radius: 10px;
    margin: 15px 0;
}

.message {
    background: #334155;
    padding: 10px;
    margin: 8px 0;
    border-radius: 9px;
    overflow-wrap: anywhere;
}

.time {
    display: block;
    font-size: 10px;
    color: #94a3b8;
    margin-top: 5px;
}

.error {
    color: #f87171;
    text-align: center;
}
</style>
</head>

<body>

<div class="container">

    <h2>🔐 Private Chat</h2>

    <div id="loginPage">

        <div id="createSection">
            <h3>Create Your Chat</h3>

            <input
                type="password"
                id="newPassword"
                placeholder="Create chat password"
            >

            <button onclick="createRoom()">
                Create Private Chat
            </button>
        </div>

        <hr>

        <div id="joinSection">
            <h3>Join Chat</h3>

            <input
                type="password"
                id="joinPassword"
                placeholder="Enter chat password"
            >

            <button onclick="joinRoom()">
                Join Chat
            </button>
        </div>

        <p class="error" id="error"></p>
    </div>

    <div id="chatPage">

        <h3>💬 Your Private Chat</h3>

        <p id="shareLink"></p>

        <button onclick="copyLink()">Copy Chat Link</button>

        <div id="messages"></div>

        <input
            id="messageInput"
            placeholder="Type your message..."
            onkeydown="if(event.key==='Enter') sendMessage()"
        >

        <button onclick="sendMessage()">Send Message</button>

    </div>

</div>

<script>
const socket = io();

let roomId = new URLSearchParams(
    window.location.search
).get("room");

let connected = false;

if (roomId) {
    document.getElementById("createSection").style.display = "none";
}

async function createRoom() {

    const password =
        document.getElementById("newPassword").value;

    const response = await fetch("/create-room", {
        method: "POST",
        headers: {
            "Content-Type": "application/json"
        },
        body: JSON.stringify({ password })
    });

    const result = await response.json();

    if (!response.ok) {
        showError(result.error);
        return;
    }

    roomId = result.roomId;

    window.history.replaceState(
        {},
        "",
        "?room=" + roomId
    );

    document.getElementById("createSection").style.display = "none";

    document.getElementById("joinPassword").value = password;

    joinRoom();
}

function joinRoom() {

    const password =
        document.getElementById("joinPassword").value;

    if (!roomId || !password) {
        showError("Please enter the password.");
        return;
    }

    socket.emit("joinRoom", {
        roomId,
        password
    });
}

socket.on("joined", (history) => {

    connected = true;

    document.getElementById("loginPage").style.display = "none";
    document.getElementById("chatPage").style.display = "block";

    document.getElementById("shareLink").textContent =
        "Share this link: " + window.location.href;

    document.getElementById("messages").innerHTML = "";

    history.forEach(addMessage);
});

socket.on("errorMessage", (message) => {
    showError(message);
});

socket.on("newMessage", (data) => {
    addMessage(data);
});

function sendMessage() {

    if (!connected) return;

    const input = document.getElementById("messageInput");
    const message = input.value.trim();

    if (!message) return;

    socket.emit("sendMessage", message);

    input.value = "";
}

function addMessage(data) {

    const div = document.createElement("div");
    div.className = "message";

    const content = document.createElement("span");
    content.textContent = data.message;

    const time = document.createElement("small");
    time.className = "time";
    time.textContent = data.time;

    div.appendChild(content);
    div.appendChild(time);

    const messages = document.getElementById("messages");
    messages.appendChild(div);

    messages.scrollTop = messages.scrollHeight;
}

function copyLink() {
    navigator.clipboard.writeText(window.location.href);
    alert("Chat link copied!");
}

function showError(message) {
    document.getElementById("error").textContent = message;
}
</script>

</body>
</html>
npm init -y
npm install express socket.io bcryptjs
node server.js
npm init -y
npm install express socket.io bcryptjs
node server.js
