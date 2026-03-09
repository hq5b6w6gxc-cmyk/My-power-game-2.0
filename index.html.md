<!DOCTYPE html>  
<html lang="uk">  
<head>  
    <meta charset="UTF-8">  
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">  
    <title>ПОТУЖНО-МЕТР: COMPACT EDITION</title>  
    <style>  
        body {  
            background-color: #050505; color: white;  
            font-family: 'Impact', sans-serif;  
            display: flex; flex-direction: column; align-items: center; justify-content: center;  
            height: 100vh; margin: 0; overflow: hidden; touch-action: none;  
        }  
  
        /* Зменшення всього контенту на 25% через масштаб */  
        .main-wrapper {  
            display: flex;  
            flex-direction: column;  
            align-items: center;  
            justify-content: center;  
            transform: scale(0.75); /* Масштаб 75% від оригіналу */  
            width: 100%;  
        }  
  
        #audioStatus {  
            background: #ffcc00; color: black; padding: 15px; width: 100%;  
            text-align: center; position: fixed; top: 0; font-weight: bold; z-index: 1000;  
            cursor: pointer; font-size: 1.1rem; border-bottom: 5px solid #f00;  
        }  
  
        .rad-btn {  
            padding: 10px 20px; border: 3px solid #00ff00; background: transparent;  
            color: #00ff00; font-weight: bold; border-radius: 8px; margin-bottom: 15px;  
            text-transform: uppercase;  
        }  
  
        .rad-on { background: #00ff00; color: black; box-shadow: 0 0 30px #00ff00; }  
  
        .status-label { font-size: 1.5rem; color: #ffcc00; height: 2rem; margin-bottom: 10px; }  
          
        .meter-container {   
            width: 320px; height: 45px; border: 4px solid #333;   
            border-radius: 12px; background: #111; overflow: hidden;   
        }  
  
        #fill { height: 100%; width: 0%; background: linear-gradient(90deg, #0f0, #ff0, #f00); }  
  
        .percentage { font-size: 6rem; font-weight: bold; margin: 5px 0; }  
  
        .btn-hold {  
            width: 200px; height: 200px; border-radius: 50%;  
            border: 10px solid #ffcc00; background: #1a1a1a;  
            color: #ffcc00; font-size: 2rem; font-weight: bold;  
            box-shadow: 0 10px 0 #665200;  
        }  
  
        .alarm-active { animation: flash 0.1s infinite, shake 0.1s infinite; }  
        @keyframes shake { 0% { transform: translate(6px, 6px); } 50% { transform: translate(-6px, -6px); } }  
        @keyframes flash { 0% { background-color: #050505; } 50% { background-color: #600; } }  
    </style>  
</head>  
<body id="mainBody">  
  
    <div id="audioStatus" onclick="unlockAudio()">🔊 НАТИСНИ ТУТ (АКТИВАЦІЯ СИСТЕМИ)</div>  
  
    <div class="main-wrapper">  
        <button id="radToggle" class="rad-btn" onclick="toggleRad()">РАДІАЦІЯ: ВИМК</button>  
  
        <div class="status-label" id="statusLabel">Готовий?</div>  
        <div class="meter-container"><div id="fill"></div></div>  
        <div id="percentage" class="percentage">0%</div>  
        <button id="mainBtn" class="btn-hold">ТРИМАЙ!</button>  
    </div>  
  
    <script>  
        let power = 0;  
        let isHolding = false;  
        let alarmTriggered = false;  
        let radiationEnabled = false;  
        let voiceTriggered = false;  
        let audioCtx = null;  
        let masterGain = null;  
        let sirenOsc = null;  
        let lastClickTime = 0;  
  
        function speak(text) {  
            window.speechSynthesis.cancel();  
            const msg = new SpeechSynthesisUtterance(text);  
            msg.lang = 'uk-UA';  
            msg.volume = 1.0;  
            msg.rate = 0.9;  
            window.speechSynthesis.speak(msg);  
        }  
  
        function unlockAudio() {  
            if (!audioCtx) {  
                audioCtx = new (window.AudioContext || window.webkitAudioContext)();  
                masterGain = audioCtx.createGain();  
                masterGain.connect(audioCtx.destination);  
            }  
            audioCtx.resume();  
            const msg = new SpeechSynthesisUtterance("");  
            window.speechSynthesis.speak(msg);  
              
            document.getElementById('audioStatus').style.background = "#00ff00";  
            document.getElementById('audioStatus').innerText = "✅ СИСТЕМА ГОТОВА";  
            setTimeout(() => document.getElementById('audioStatus').style.display = 'none', 1000);  
        }  
  
        function toggleRad() {  
            radiationEnabled = !radiationEnabled;  
            const btn = document.getElementById('radToggle');  
            btn.innerText = radiationEnabled ? "РАДІАЦІЯ: УВІМК" : "РАДІАЦІЯ: ВИМК";  
            btn.classList.toggle('rad-on');  
        }  
  
        function playGeiger() {  
            if (!audioCtx || !radiationEnabled || !isHolding || alarmTriggered) return;  
            const osc = audioCtx.createOscillator();  
            const g = audioCtx.createGain();  
            osc.type = 'square';  
            osc.frequency.setValueAtTime(150 + (power * 6), audioCtx.currentTime);  
            g.gain.setValueAtTime(1.5, audioCtx.currentTime);  
            g.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + 0.04);  
            osc.connect(g); g.connect(masterGain);  
            osc.start(); osc.stop(audioCtx.currentTime + 0.05);  
        }  
  
        function startSiren() {  
            const osc = audioCtx.createOscillator();  
            const lfo = audioCtx.createOscillator();  
            const lfoG = audioCtx.createGain();  
            const sG = audioCtx.createGain();  
            osc.type = 'sawtooth'; osc.frequency.value = 110;  
            lfo.frequency.value = 0.4; lfoG.gain.value = 45;  
            lfo.connect(lfoG); lfoG.connect(osc.frequency);  
            osc.connect(sG); sG.connect(masterGain);  
            sG.gain.setValueAtTime(0, audioCtx.currentTime);  
            sG.gain.linearRampToValueAtTime(0.9, audioCtx.currentTime + 1.5);  
            osc.start(); lfo.start();  
            sirenOsc = { osc, lfo, sG };  
        }  
  
        function update() {  
            if (alarmTriggered) return;  
            if (isHolding) {  
                power += (power > 85 ? 0.3 : 0.7);  
                if (radiationEnabled && audioCtx) {  
                    let delay = 0.35 - (power / 100) * 0.33;   
                    if (audioCtx.currentTime > lastClickTime + delay) {  
                        playGeiger();  
                        lastClickTime = audioCtx.currentTime;  
                    }  
                }  
                if (power >= 92 && !voiceTriggered) {  
                    voiceTriggered = true;  
                    speak("Увага! Ти надпотужний. Твоя потужність у небі зафіксована!");  
                }  
            } else {  
                power -= 0.3;  
                if (power < 80) voiceTriggered = false;   
            }  
  
            if (power < 0) power = 0;  
            if (power >= 100) { power = 100; triggerAlarm(); }  
  
            document.getElementById('fill').style.width = power + '%';  
            document.getElementById('percentage').innerText = Math.floor(power) + '%';  
              
            let label = "Слабенький дрищ";  
            if (power > 25) label = "Нормальний хлопчик";  
            if (power > 50) label = "Потужний";  
            if (power > 85) label = "ОБЕРЕЖНО";  
            if (power > 92) label = "НАДПОТУЖНИЙ!";  
            document.getElementById('statusLabel').innerText = label;  
  
            if (power > 50) document.body.style.animation = `shake ${(110-power)/130}s infinite`;  
            else document.body.style.animation = "none";  
        }  
  
        function triggerAlarm() {  
            alarmTriggered = true;  
            document.body.classList.add('alarm-active');  
            document.getElementById('mainBtn').innerText = "СКИДАННЯ";  
            startSiren();  
        }  
  
        const start = (e) => {   
            e.preventDefault(); if (!audioCtx) unlockAudio();  
            if (alarmTriggered) {   
                power=0; alarmTriggered=false; voiceTriggered=false;  
                document.body.classList.remove('alarm-active');   
                document.getElementById('mainBtn').innerText="ТРИМАЙ!";   
                if (sirenOsc) { sirenOsc.osc.stop(); sirenOsc = null; }  
                return;   
            }   
            isHolding = true;   
        };  
        const stop = () => isHolding = false;  
  
        const mainBtn = document.getElementById('mainBtn');  
        mainBtn.addEventListener('touchstart', start); mainBtn.addEventListener('touchend', stop);  
        mainBtn.addEventListener('mousedown', start); window.addEventListener('mouseup', stop);  
        setInterval(update, 30);  
    </script>  
</body>  
</html>  
