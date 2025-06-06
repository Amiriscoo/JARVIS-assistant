import os
from flask import Flask, render_template_string, request, jsonify
import webbrowser
import subprocess
import threading
import time
import openai

# === CONFIG ===
OPENAI_API_KEY = "your_openai_api_key_here"  # <-- Put your key here

app = Flask(__name__)

openai.api_key = OPENAI_API_KEY

HTML = """
<!DOCTYPE html>
<html>
<head>
<title>MCU JARVIS Replica</title>
<style>
  body {
    margin:0; background:#00001a; color:#0ff; font-family:'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    overflow:hidden;
  }
  #container {
    position: absolute; top:50%; left:50%; transform: translate(-50%, -50%);
    width: 600px; max-width: 90vw; background: rgba(0, 255, 255, 0.1); border-radius: 25px;
    padding: 20px; box-shadow: 0 0 20px #0ff;
  }
  h1 {
    font-weight: 900; text-align:center; margin-bottom: 10px;
  }
  #status {
    font-size: 18px; margin-bottom: 15px; text-align:center;
  }
  #chatbox {
    background: #002233; border-radius: 15px; padding: 15px; height: 300px; overflow-y: auto;
    font-size: 16px; line-height: 1.4;
  }
  .user-msg {
    text-align: right; margin: 10px 0; color: #0ff;
  }
  .jarvis-msg {
    text-align: left; margin: 10px 0; color: #66ffff;
  }
  button {
    background: #0ff; color: #003; border:none; padding: 10px 20px; border-radius: 10px;
    font-weight: bold; cursor: pointer; font-size: 18px; margin-top: 10px;
    width: 100%;
  }
</style>
</head>
<body>
<div id="container">
  <h1>JARVIS - MCU Replica</h1>
  <div id="status">Say "Jarvis" to wake me up.</div>
  <div id="chatbox"></div>
  <button onclick="startWakeWord()">Start Listening</button>
</div>

<script>
  const status = document.getElementById('status');
  const chatbox = document.getElementById('chatbox');

  let recognizing = false;
  let recognition;
  let commandRecognition;

  if (!('webkitSpeechRecognition' in window)) {
    status.textContent = "Your browser doesn't support Speech Recognition. Use Chrome or Edge.";
  } else {
    recognition = new webkitSpeechRecognition();
    recognition.continuous = false;
    recognition.interimResults = false;
    recognition.lang = 'en-US';

    recognition.onstart = () => {
      recognizing = true;
      status.textContent = 'Listening for hotword "Jarvis"...';
    };

    recognition.onerror = (event) => {
      status.textContent = 'Error: ' + event.error;
      recognizing = false;
    };

    recognition.onend = () => {
      recognizing = false;
      status.textContent = 'Say "Jarvis" to wake me up.';
    };

    recognition.onresult = (event) => {
      let spoken = event.results[0][0].transcript.toLowerCase();
      console.log('Spoken:', spoken);
      if (spoken.includes('jarvis')) {
        status.textContent = 'Hotword detected! Listening for your command...';
        listenCommand();
      } else {
        status.textContent = 'Waiting for "Jarvis"...';
      }
    };
  }

  function startWakeWord() {
    if (recognizing) {
      recognition.stop();
      return;
    }
    recognition.start();
  }

  function listenCommand() {
    commandRecognition = new webkitSpeechRecognition();
    commandRecognition.continuous = false;
    commandRecognition.interimResults = false;
    commandRecognition.lang = 'en-US';

    commandRecognition.onstart = () => {
      status.textContent = 'Listening for your command...';
    };
    commandRecognition.onerror = (e) => {
      status.textContent = 'Error listening: ' + e.error;
    };
    commandRecognition.onend = () => {
      status.textContent = 'Say "Jarvis" to wake me up.';
    };
    commandRecognition.onresult = (event) => {
      let command = event.results[0][0].transcript;
      addMessage(command, 'user-msg');
      status.textContent = 'Processing...';
      sendCommand(command);
    };
    commandRecognition.start();
  }

  function addMessage(text, cls) {
    let div = document.createElement('div');
    div.textContent = text;
    div.className = cls;
    chatbox.appendChild(div);
    chatbox.scrollTop = chatbox.scrollHeight;
  }

  function sendCommand(command) {
    fetch('/command', {
      method: 'POST',
      headers: {'Content-Type': 'application/json'},
      body: JSON.stringify({command})
    }).then(response => response.json())
      .then(data => {
        addMessage(data.response, 'jarvis-msg');
        speak(data.response);
        status.textContent = 'Say "Jarvis" to wake me up.';
      });
  }

  function speak(text) {
    if ('speechSynthesis' in window) {
      let utterance = new SpeechSynthesisUtterance(text);
      utterance.rate = 1.1;
      utterance.pitch = 1.2;
      window.speechSynthesis.speak(utterance);
    }
  }
</script>
</body>
</html>
"""

@app.route('/')
def home():
    return render_template_string(HTML)

@app.route('/command', methods=['POST'])
def command():
    data = request.json
    cmd = data.get('command', '').lower()

    # Special commands you can customize:
    if 'open youtube' in cmd:
        threading.Thread(target=open_website, args=("https://youtube.com",)).start()
        response = "Opening YouTube for you."
    elif 'open google' in cmd:
        threading.Thread(target=open_website, args=("https://google.com",)).start()
        response = "Opening Google."
    elif 'time' in cmd:
        response = time.strftime("The time is %I:%M %p")
    elif 'date' in cmd:
        response = time.strftime("Today is %B %d, %Y")
    elif 'your name' in cmd:
        response = "I am JARVIS, your personal AI assistant."
    elif 'hello' in cmd or 'hi' in cmd:
        response = "Hello! How can I help you today?"
    elif 'shutdown' in cmd:
        response = "Goodbye! Shutting down."
        shutdown_server()
    else:
        # Use OpenAI GPT to answer any question (if key is set)
        if OPENAI_API_KEY and OPENAI_API_KEY != "your_openai_api_key_here":
            try:
                completion = openai.Completion.create(
                    engine="text-davinci-003",
                    prompt=f"Answer like JARVIS from MCU: {cmd}",
                    max_tokens=100,
                    temperature=0.7,
                )
                response = completion.choices[0].text.strip()
            except Exception as e:
                response = "Sorry, I couldn't get a response from my brain."
        else:
            response = "I don't understand that yet. Please ask something else."

    return jsonify({'response': response})

def open_website(url):
    # Open URL in default browser
    webbrowser.open(url)

def shutdown_server():
    func = request.environ.get('werkzeug.server.shutdown')
    if func:
        func()

if __name__ == '__main__':
    print("Starting JARVIS MCU Replica...")
    app.run(host='0.0.0.0', port=5000)
