<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Voice Timer App</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }

        body {
            font-family: 'Arial', sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            display: flex;
            justify-content: center;
            align-items: center;
            color: white;
        }

        .app-container {
            background: rgba(255, 255, 255, 0.1);
            backdrop-filter: blur(10px);
            border-radius: 20px;
            padding: 40px;
            text-align: center;
            box-shadow: 0 20px 40px rgba(0,0,0,0.3);
            max-width: 400px;
            width: 90%;
        }

        h1 {
            font-size: 2.5em;
            margin-bottom: 20px;
            text-shadow: 2px 2px 4px rgba(0,0,0,0.3);
        }

        .status {
            font-size: 1.2em;
            margin-bottom: 30px;
            padding: 15px;
            border-radius: 10px;
            background: rgba(255,255,255,0.2);
        }

        .timer-display {
            font-size: 4em;
            font-weight: bold;
            margin: 30px 0;
            text-shadow: 3px 3px 6px rgba(0,0,0,0.5);
            min-height: 100px;
            display: flex;
            align-items: center;
            justify-content: center;
        }

        .listening {
            animation: pulse 1.5s infinite;
            color: #ffeb3b;
        }

        @keyframes pulse {
            0% { transform: scale(1); }
            50% { transform: scale(1.05); }
            100% { transform: scale(1); }
        }

        .controls {
            margin: 20px 0;
        }

        button {
            background: rgba(255,255,255,0.2);
            border: none;
            color: white;
            padding: 15px 30px;
            margin: 10px;
            border-radius: 50px;
            font-size: 1.1em;
            cursor: pointer;
            transition: all 0.3s ease;
            backdrop-filter: blur(10px);
        }

        button:hover {
            background: rgba(255,255,255,0.3);
            transform: translateY(-2px);
        }

        button:active {
            transform: translateY(0);
        }

        .commands {
            margin-top: 30px;
            font-size: 0.9em;
            opacity: 0.8;
            line-height: 1.6;
        }

        .hidden {
            display: none;
        }
    </style>
</head>
<body>
    <div class="app-container">
        <h1>🗣️ Voice Timer</h1>
        
        <div class="status" id="status">Say "START" to begin listening...</div>
        
        <div class="timer-display" id="timerDisplay">00:00:00</div>
        
        <div class="controls">
            <button onclick="startListening()">🎤 Start Listening</button>
            <button onclick="stopTimer()" id="stopBtn" class="hidden">⏹️ Stop</button>
        </div>
        
        <div class="commands">
            <strong>Voice Commands:</strong><br>
            - "START" - Start timer<br>
            - "STOP" - Stop timer<br>
            - "SET 5 MINUTES" or "5 MINUTES" - Set timer<br>
            - Numbers like "10 SECONDS", "2 MINUTES", etc.
        </div>
    </div>

    <script>
        let recognition;
        let timerInterval;
        let timeLeft = 0;
        let isRunning = false;
        let isListening = false;

        // Initialize Speech Recognition
        function initSpeechRecognition() {
            if (!('webkitSpeechRecognition' in window) && !('SpeechRecognition' in window)) {
                document.getElementById('status').innerHTML = '❌ Speech recognition not supported in this browser';
                return false;
            }

            const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
            recognition = new SpeechRecognition();
            recognition.continuous = true;
            recognition.interimResults = false;
            recognition.lang = 'en-US';

            recognition.onstart = function() {
                isListening = true;
                updateStatus('🎤 Listening... Say "START", "STOP", or "SET X MINUTES"');
                document.getElementById('status').classList.add('listening');
            };

            recognition.onresult = function(event) {
                const command = event.results[event.results.length - 1][0].transcript.toUpperCase().trim();
                handleVoiceCommand(command);
            };

            recognition.onerror = function(event) {
                updateStatus('❌ Error: ' + event.error);
                isListening = false;
                document.getElementById('status').classList.remove('listening');
            };

            recognition.onend = function() {
                isListening = false;
                document.getElementById('status').classList.remove('listening');
                if (isRunning) {
                    updateStatus('⏳ Timer running... Say "STOP" to stop');
                } else {
                    updateStatus('👂 Ready. Click microphone or say "START"');
                }
            };

            return true;
        }

        function handleVoiceCommand(command) {
            console.log('Heard:', command);
            
            if (command.includes('START') && !isRunning) {
                startTimer();
            } else if (command.includes('STOP') && isRunning) {
                stopTimer();
            } else if (command.match(/SET\s+(\d+(?:\.\d+)?)\s*(MINUTE|MINUTES|SECOND|SECONDS)/i) || 
                       command.match(/(\d+(?:\.\d+)?)\s*(MINUTE|MINUTES|SECOND|SECONDS)/i)) {
                setTimerFromCommand(command);
            } else {
                updateStatus(`Did you say "${command}"? Try "START", "STOP", or "SET 5 MINUTES"`);
            }
        }

        function setTimerFromCommand(command) {
            const match = command.match(/(\d+(?:\.\d+)?)\s*(MINUTE|MINUTES|SECOND|SECONDS)/i);
            if (match) {
                const value = parseFloat(match[1]);
                const unit = match[2].toLowerCase();
                
                timeLeft = unit.includes('minute') ? value * 60 : value;
                
                if (timeLeft > 0) {
                    stopTimer();
                    updateDisplay();
                    updateStatus(`⏰ Timer set to ${value} ${unit.includes('minute') ? 'minutes' : 'seconds'}`);
                }
            }
        }

        function startTimer() {
            if (timeLeft === 0) {
                timeLeft = 1; // Default 1 second if no time set
            }
            isRunning = true;
            document.getElementById('stopBtn').classList.remove('hidden');
            
            timerInterval = setInterval(() => {
                timeLeft--;
                updateDisplay();
                
                if (timeLeft <= 0) {
                    stopTimer();
                    timerComplete();
                }
            }, 1000);
            
            updateStatus('⏳ Timer started! Say "STOP" to stop');
        }

        function stopTimer() {
            isRunning = false;
            clearInterval(timerInterval);
            document.getElementById('stopBtn').classList.add('hidden');
            updateStatus('⏹️ Timer stopped');
        }

        function timerComplete() {
            updateStatus('✅ TIME\'S UP! 🎉');
            
            // Vibrate if supported
            if (navigator.vibrate) {
                navigator.vibrate([200, 100, 200, 100, 200, 100, 500]);
            }
            
            // Audio beep (works even if muted)
            const audioContext = new (window.AudioContext || window.webkitAudioContext)();
            const oscillator = audioContext.createOscillator();
            const gainNode = audioContext.createGain();
            
            oscillator.connect(gainNode);
            gainNode.connect(audioContext.destination);
            
            oscillator.frequency.value = 800;
            oscillator.type = 'sine';
            
            gainNode.gain.setValueAtTime(0.3, audioContext.currentTime);
            gainNode.gain.exponentialRampToValueAtTime(0.01, audioContext.currentTime + 1);
            
            oscillator.start(audioContext.currentTime);
            oscillator.stop(audioContext.currentTime + 1);
        }

        function updateDisplay() {
            const hours = Math.floor(timeLeft / 3600);
            const minutes = Math.floor((timeLeft % 3600) / 60);
            const seconds = timeLeft % 60;
            
            document.getElementById('timerDisplay').textContent = 
                `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${seconds.toString().padStart(2, '0')}`;
        }

        function updateStatus(message) {
            document.getElementById('status').textContent = message;
        }

        function startListening() {
            if (!recognition) {
                if (!initSpeechRecognition()) return;
            }
            recognition.start();
        }

        // Auto-start listening on page load (mobile-friendly)
        window.onload = function() {
            updateStatus('👋 Click microphone to start or tap screen to enable voice');
            // Auto-request permission on first interaction
            document.addEventListener('click', function enableVoice() {
                initSpeechRecognition();
                document.removeEventListener('click', enableVoice);
            }, { once: true });
        };

        // Prevent accidental page close during timer
        window.addEventListener('beforeunload', function(e) {
            if (isRunning) {
                e.preventDefault();
                e.returnValue = 'Timer is running!';
            }
        });
    </script>
</body>
</html>
