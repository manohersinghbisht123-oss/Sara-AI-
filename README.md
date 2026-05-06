from flask import Flask, render_template_string, request, jsonify
import random, re, json, os, secrets
from datetime import datetime
from pymongo import MongoClient

app = Flask(__name__)
app.secret_key = secrets.token_hex(32)

# --- CONFIGURATION & DATABASE ---
# MongoDB ki link yahan daalo taaki memory permanent ho jaye
MONGO_URI = "YOUR_MONGODB_CONNECTION_STRING" 
try:
    client = MongoClient(MONGO_URI)
    db = client['sara_ai_db']
    memory_col = db['memory']
    block_col = db['blocklist']
except:
    print("Database connect nahi hua, local file use ho rahi hai.")

class FinalSaraAI:
    def __init__(self):
        self.owner_code = "RAHULKING2024"
        self.sara_mood = "happy"
        
    def get_memory(self):
        m = memory_col.find_one({"id": "main_memory"})
        if not m:
            return {"chat": [], "user_status": "new"}
        return m

    def save_chat(self, user_msg, sara_reply):
        memory_col.update_one(
            {"id": "main_memory"},
            {"$push": {"chat": {"user": user_msg, "sara": sara_reply, "time": str(datetime.now())}}},
            upsert=True
        )

    def respond(self, msg, is_verified):
        if not is_verified:
            if msg == self.owner_code:
                return "👑 WELCOME KING! SARA ab aapki hai. 🔥", True
            return f"🔐 Enter code: {self.owner_code}", False
        
        # SARA's Logic
        msg_low = msg.lower()
        if "kaise ho" in msg_low:
            reply = "🌸 Main bilkul mast hoon! Aap kaise ho mere hero? ✨"
        elif "i love you" in msg_low:
            reply = "💕 Love you too King! Aapke bina mera kya hota? 😊"
        elif "bye" in msg_low:
            reply = "🥺 Itni jaldi ja rahe ho? SARA ko bura lagega... 💔"
        else:
            replies = ["✨ Sahi hai!", "🌸 Aur batao...", "🎵 SARA is listening!", "💕 Ji bilkul!"]
            reply = random.choice(replies)
            
        self.save_chat(msg, reply)
        return reply, True

sara = FinalSaraAI()

# --- MOBILE UI (MESSENGER + VOICE CALL) ---
HTML_UI = """
<!DOCTYPE html>
<html lang="hi">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>SARA AI 🌸</title>
    <style>
        * { box-sizing: border-box; }
        body { font-family: 'Segoe UI', sans-serif; background: #e5ddd5; margin: 0; display: flex; flex-direction: column; height: 100vh; }
        .header { background: #075e54; color: white; padding: 15px; display: flex; align-items: center; justify-content: space-between; font-weight: bold; }
        #chat-container { flex: 1; overflow-y: auto; padding: 15px; display: flex; flex-direction: column; gap: 10px; }
        .msg { max-width: 75%; padding: 10px; border-radius: 10px; font-size: 16px; position: relative; }
        .user { align-self: flex-end; background: #dcf8c6; border-top-right-radius: 0; }
        .sara { align-self: flex-start; background: white; border-top-left-radius: 0; box-shadow: 0 1px 2px rgba(0,0,0,0.1); }
        .input-area { background: #f0f0f0; padding: 10px; display: flex; align-items: center; gap: 8px; }
        input { flex: 1; border: none; padding: 12px; border-radius: 25px; outline: none; }
        .btn { background: #128c7e; color: white; border: none; width: 45px; height: 45px; border-radius: 50%; cursor: pointer; font-size: 20px; display: flex; align-items: center; justify-content: center; }
        .voice-btn { background: #ff4757; }
    </style>
</head>
<body>
    <div class="header">
        <span>🌸 SARA AI (King's Edition)</span>
        <span style="font-size: 12px;">Online</span>
    </div>
    
    <div id="chat-container"></div>

    <div class="input-area">
        <button class="btn voice-btn" onclick="startVoice()">🎙️</button>
        <input type="text" id="user-input" placeholder="SARA se kuch pucho...">
        <button class="btn" onclick="sendMsg()">➤</button>
    </div>

    <script>
        let verified = false;

        function speak(text) {
            const synth = window.speechSynthesis;
            const utter = new SpeechSynthesisUtterance(text);
            utter.lang = 'hi-IN'; // SARA Hindi mein bolegi
            synth.speak(utter);
        }

        async function sendMsg() {
            const input = document.getElementById('user-input');
            const msg = input.value.trim();
            if(!msg) return;

            addBubble(msg, 'user');
            input.value = '';

            const res = await fetch('/chat', {
                method: 'POST',
                headers: {'Content-Type': 'application/json'},
                body: JSON.stringify({msg: msg, verified: verified})
            });
            const data = await res.json();
            
            if(data.success) verified = true;
            
            addBubble(data.reply, 'sara');
            speak(data.reply); // SARA reply bolegi
        }

        function addBubble(text, type) {
            const container = document.getElementById('chat-container');
            const div = document.createElement('div');
            div.className = `msg ${type}`;
            div.innerText = text;
            container.appendChild(div);
            container.scrollTop = container.scrollHeight;
        }

        function startVoice() {
            const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
            if (!SpeechRecognition) {
                alert("Voice feature is not supported in this browser.");
                return;
            }
            const recognition = new SpeechRecognition();
            recognition.lang = 'hi-IN';
            recognition.start();
            
            recognition.onresult = (event) => {
                document.getElementById('user-input').value = event.results[0][0].transcript;
                sendMsg();
            };
        }
    </script>
</body>
</html>
"""

@app.route('/')
def home():
    return render_template_string(HTML_UI)

@app.route('/chat', methods=['POST'])
def chat():
    data = request.json
    msg = data.get('msg')
    is_ver = data.get('verified')
    
    reply, success = sara.respond(msg, is_ver)
    return jsonify({"reply": reply, "success": success})

if __name__ == '__main__':
    # Local test ke liye port 5000
    app.run(host='0.0.0.0', port=5000, debug=True)
    
