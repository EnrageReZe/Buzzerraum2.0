<!DOCTYPE html>
<html lang="de">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Buzzerraum</title>
  <!-- Firebase SDK -->
  <script src="https://www.gstatic.com/firebasejs/9.22.2/firebase-app-compat.js"></script>
  <script src="https://www.gstatic.com/firebasejs/9.22.2/firebase-database-compat.js"></script>
  <style>
    body { display: flex; flex-direction: column; align-items: center; justify-content: flex-start; padding: 20px; margin: 0; font-family: Arial, sans-serif; background: #f0f0f0; position: relative; }
    #join, #lobby { display: none; flex-direction: column; align-items: center; width: 100%; max-width: 400px; }
    #join.active, #lobby.active { display: flex; }
    input, select { width: 100%; padding: 10px; font-size: 1rem; margin-bottom: 10px; box-sizing: border-box; }
    button { padding: 10px 20px; font-size: 1rem; cursor: pointer; margin: 5px; }
    .buzzer { width: 100%; height: 60px; border-radius: 8px; border: none; font-size: 1.2rem; cursor: pointer; background: #e74c3c; color: #fff; transition: transform 0.1s; }
    .buzzer:active { transform: scale(0.95); }
    #status { font-size: 1.2rem; margin: 10px 0; height: 1.2em; }
    #submissionDiv, #evaluationDiv { width: 100%; display: flex; flex-direction: column; align-items: stretch; margin-bottom: 10px; }
    #submissionsList { width: 100%; margin-top: 20px; }
    .submission-item { background: #fff; padding: 8px; border-radius: 4px; margin-bottom: 5px; box-shadow: 0 1px 3px rgba(0,0,0,0.1); }
    /* Scoreboard */
    #scoreboard { position: absolute; top: 20px; right: 20px; width: 180px; background: #fff; padding: 10px; border-radius: 4px; box-shadow: 0 1px 5px rgba(0,0,0,0.2); font-size: 0.9rem; }
    #scoreboard h2 { margin: 0 0 10px 0; font-size: 1rem; text-align: center; }
    .score-entry { display: flex; justify-content: space-between; padding: 2px 0; }
  </style>
</head>
<body>
  <div id="join" class="active">
    <h1>Buzzerraum</h1>
    <select id="roleSelect">
      <option value="player">Spieler</option>
      <option value="host">Spielleiter</option>
    </select>
    <input type="text" id="nameInput" placeholder="Dein Name" />
    <button id="joinBtn">Beitreten</button>
  </div>
  <div id="lobby">
    <h1>Buzzerraum</h1>
    <div id="status">Noch kein Buzz.</div>
    <button class="buzzer" id="buzzBtn">Buzz!</button>
    <button id="resetBtn">Zurücksetzen</button>
    <div id="submissionDiv">
      <input type="text" id="submissionInput" placeholder="Dein Eintrag" />
      <button id="submitBtn">Eintragen</button>
    </div>
    <div id="submissionsList"></div>
    <div id="evaluationDiv" style="display:none;">
      <input type="number" id="pointsInput" placeholder="Punkte vergeben" />
      <div>
        <button id="correctBtn">Richtig</button>
        <button id="wrongBtn">Falsch</button>
      </div>
    </div>
  </div>

  <!-- Scoreboard für alle Teilnehmer -->
  <div id="scoreboard">
    <h2>Punktestand</h2>
    <div id="scoresList">Lade...</div>
  </div>

  <script>
    (function() {
      // Firebase-Konfiguration (bitte anpassen)
      const firebaseConfig = {
        apiKey: "AIzaSyYourAPIKeyHere",
        authDomain: "your-project-id.firebaseapp.com",
        databaseURL: "https://your-project-id-default-rtdb.firebaseio.com",
        projectId: "your-project-id",
        storageBucket: "your-project-id.appspot.com",
        messagingSenderId: "1234567890",
        appId: "1:1234567890:web:abcdef123456"
      };
      firebase.initializeApp(firebaseConfig);
      const db = firebase.database();

      // DOM-Elemente
      const joinEl = document.getElementById('join');
      const lobbyEl = document.getElementById('lobby');
      const roleSelect = document.getElementById('roleSelect');
      const nameInput = document.getElementById('nameInput');
      const joinBtn = document.getElementById('joinBtn');
      const buzzBtn = document.getElementById('buzzBtn');
      const resetBtn = document.getElementById('resetBtn');
      const statusEl = document.getElementById('status');
      const submissionDiv = document.getElementById('submissionDiv');
      const submissionInput = document.getElementById('submissionInput');
      const submitBtn = document.getElementById('submitBtn');
      const submissionsList = document.getElementById('submissionsList');
      const evaluationDiv = document.getElementById('evaluationDiv');
      const pointsInput = document.getElementById('pointsInput');
      const correctBtn = document.getElementById('correctBtn');
      const wrongBtn = document.getElementById('wrongBtn');
      const scoresList = document.getElementById('scoresList');

      let playerName = '';
      let role = 'player';
      let currentBuzz = null;

      // URL-Parameter auslesen für Auto-Join
      const params = new URLSearchParams(window.location.search);
      const urlRole = params.get('role');
      const urlName = params.get('name');
      if (urlRole) {
        role = urlRole;
        roleSelect.value = role;
        nameInput.style.display = (role === 'player') ? 'block' : 'none';
      }
      if (urlName) {
        playerName = urlName;
        nameInput.value = playerName;
      }

      // Funktion zum Anzeigen der Lobby
      function showLobby() {
        joinEl.classList.remove('active');
        lobbyEl.classList.add('active');
        if (role === 'host') {
          buzzBtn.style.display = 'none';
          submissionDiv.style.display = 'none';
        }
      }
      // Auto-Join, wenn Parameter gesetzt
      if (urlRole === 'host' || (urlRole === 'player' && urlName)) {
        showLobby();
      }

      // Rolle wählen
      roleSelect.addEventListener('change', () => {
        role = roleSelect.value;
        nameInput.style.display = (role === 'player') ? 'block' : 'none';
      });

      // Beitreten (manuell)
      joinBtn.addEventListener('click', () => {
        if (role === 'player') {
          const name = nameInput.value.trim();
          if (!name) return alert('Bitte gib deinen Namen ein.');
          playerName = name;
        }
        showLobby();
      });

      // Buzz-Status überwachen
      db.ref('buzz').on('value', snap => {
        const data = snap.val();
        currentBuzz = data;
        if (data) {
          statusEl.textContent = `${data.name} hat als Erster gebuzzert!`;
          buzzBtn.disabled = true;
          if (role === 'host') {
            evaluationDiv.style.display = 'flex';
            pointsInput.value = '';
          }
        } else {
          statusEl.textContent = 'Noch kein Buzz.';
          buzzBtn.disabled = false;
          if (role === 'host') evaluationDiv.style.display = 'none';
        }
      });

      // Buzz drücken
      buzzBtn.addEventListener('click', () => {
        if (role !== 'player' || !playerName) return;
        db.ref('buzz').set({ name: playerName, timestamp: firebase.database.ServerValue.TIMESTAMP });
      });

      // Reset (nur Host)
      resetBtn.addEventListener('click', () => {
        if (role === 'host') {
          db.ref('buzz').remove();
          db.ref('submissions').remove();
          db.ref('buzzResult').remove();
          db.ref('scores').remove();
        }
      });

      // Submission erfassen
      submitBtn.addEventListener('click', () => {
        const text = submissionInput.value.trim();
        if (!text) return alert('Bitte gib einen Eintrag ein.');
        db.ref(`submissions/${playerName}`).set({ text, timestamp: firebase.database.ServerValue.TIMESTAMP });
        submissionInput.disabled = true;
        submitBtn.disabled = true;
      });

      // Submission-Liste (Host) und Sperre (Player)
      db.ref('submissions').on('value', snap => {
        const subs = snap.val() || {};
        if (role === 'player' && playerName && subs[playerName]) {
          submissionInput.value = subs[playerName].text;
          submissionInput.disabled = true;
          submitBtn.disabled = true;
        }
        if (role === 'host') {
          submissionsList.innerHTML = '';
          Object.keys(subs).forEach(name => {
            const item = document.createElement('div');
            item.classList.add('submission-item');
            item.textContent = `${name}: ${subs[name].text}`;
            submissionsList.appendChild(item);
          });
        }
      });

      // Evaluation (nur Host) & Punkte aktualisieren
      const evaluate = correct => {
        if (!currentBuzz) return;
        const pts = parseInt(pointsInput.value, 10);
        if (isNaN(pts)) return alert('Bitte gültige Punktezahl eingeben.');
        // Update buzzResult
        db.ref('buzzResult').set({ name: currentBuzz.name, correct, points: pts, timestamp: firebase.database.ServerValue.TIMESTAMP });
        // Update scores per transaction
        const scoreRef = db.ref(`scores/${currentBuzz.name}`);
        scoreRef.transaction(cur => (cur || 0) + pts);
        evaluationDiv.style.display = 'none';
        statusEl.textContent += ` – ${correct ? 'Richtig' : 'Falsch'} (${pts} Punkte)`;
      };
      correctBtn.addEventListener('click', () => evaluate(true));
      wrongBtn.addEventListener('click', () => evaluate(false));

      // Scoreboard aktualisieren
      db.ref('scores').on('value', snap => {
        const scores = snap.val() || {};
        scoresList.innerHTML = '';
        Object.keys(scores).forEach(name => {
          const div = document.createElement('div');
          div.classList.add('score-entry');
          div.textContent = name;
          const span = document.createElement('span'); span.textContent = scores[name];
          div.appendChild(span);
          scoresList.appendChild(div);
        });
      });
    })();
  </script>
</body>
</html>
