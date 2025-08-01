<!DOCTYPE html>
<html lang="pl">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>WoW Classic - Wybierz klasę i rasę</title>
  <style>
    body { font-family: Arial, sans-serif; padding: 20px; background: #1a1a1a; color: #f0f0f0; }
    h1 { color: #ffd700; }
    .form-section { margin-bottom: 20px; }
    .checkbox-group { display: flex; flex-wrap: wrap; gap: 10px; }
    label { display: flex; align-items: center; gap: 5px; }
    table { width: 100%; border-collapse: collapse; margin-top: 20px; }
    th, td { border: 1px solid #444; padding: 8px; text-align: center; }
    th { background-color: #333; }
    button { padding: 10px 20px; cursor: pointer; background-color: #444; color: white; border: none; margin-top: 10px; }
    button:hover { background-color: #666; }
    input[type='text'] { padding: 5px; margin-bottom: 10px; width: 200px; }
  </style>
</head>
<body>
  <h1>Wybierz klasę i rasę - WoW Classic</h1>

  <form id="selectionForm">
    <div class="form-section">
      <h3>Nick gracza:</h3>
      <input type="text" id="nickname" placeholder="Wpisz swój nick" required />
    </div>

    <div class="form-section">
      <h3>Klasy:</h3>
      <div class="checkbox-group" id="classCheckboxes"></div>
    </div>

    <div class="form-section">
      <h3>Rasy:</h3>
      <div class="checkbox-group" id="raceCheckboxes"></div>
    </div>

    <button type="submit">Zapisz wybór</button>
    <button type="button" id="resetButton">Resetuj dane</button>
  </form>

  <h2>Wyniki:</h2>
  <table id="resultsTable">
    <thead>
      <tr>
        <th>Nick</th>
        <th>Klasa</th>
        <th>Rasa</th>
      </tr>
    </thead>
    <tbody></tbody>
  </table>

  <h2>Statystyki:</h2>
  <ul id="statsList"></ul>

  <script>
    const wowClasses = [
      'Warrior', 'Paladin', 'Hunter', 'Rogue', 'Priest', 'Shaman',
      'Mage', 'Warlock', 'Druid'
    ];

    const wowRaces = [
      'Human', 'Dwarf', 'Night Elf', 'Gnome', 'Orc', 'Undead',
      'Tauren', 'Troll'
    ];

    const raceClassMap = {
      'Human': ['Warrior', 'Paladin', 'Rogue', 'Priest', 'Mage', 'Warlock'],
      'Dwarf': ['Warrior', 'Paladin', 'Hunter', 'Rogue', 'Priest'],
      'Night Elf': ['Warrior', 'Hunter', 'Rogue', 'Priest', 'Druid'],
      'Gnome': ['Warrior', 'Rogue', 'Mage', 'Warlock'],
      'Orc': ['Warrior', 'Hunter', 'Rogue', 'Shaman', 'Warlock'],
      'Undead': ['Warrior', 'Rogue', 'Priest', 'Mage', 'Warlock'],
      'Tauren': ['Warrior', 'Hunter', 'Shaman', 'Druid'],
      'Troll': ['Warrior', 'Hunter', 'Rogue', 'Priest', 'Shaman', 'Mage', 'Warlock']
    };

    let stats = {};
    let users = [];

    const endpoint = 'https://api.npoint.io/7e6a70afb24e5542671c';

    function loadSharedData() {
      fetch(endpoint)
        .then(res => res.json())
        .then(data => {
          stats = data.stats || {};
          users = data.users || [];
          updateStatsDisplay();
          loadResultsTable();
        });
    }

    function syncSharedData() {
      fetch(endpoint, {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ stats, users })
      });
    }

    const classContainer = document.getElementById('classCheckboxes');
    const raceContainer = document.getElementById('raceCheckboxes');
    const statsList = document.getElementById('statsList');
    const resultsTable = document.querySelector('#resultsTable tbody');

    function createCheckbox(name, groupName) {
      const label = document.createElement('label');
      const checkbox = document.createElement('input');
      checkbox.type = 'checkbox';
      checkbox.name = groupName;
      checkbox.value = name;
      checkbox.addEventListener('change', () => enforceSingleSelection(groupName));
      label.appendChild(checkbox);
      label.appendChild(document.createTextNode(name));
      return label;
    }

    function enforceSingleSelection(groupName) {
      const checkboxes = document.querySelectorAll(`input[name='${groupName}']`);
      checkboxes.forEach(cb => {
        if (cb !== event.target) cb.checked = false;
      });
    }

    function updateStatsDisplay() {
      statsList.innerHTML = '';
      Object.entries(stats).forEach(([key, count]) => {
        const li = document.createElement('li');
        li.textContent = `${key} - ${count}`;
        statsList.appendChild(li);
      });
    }

    function loadResultsTable() {
      resultsTable.innerHTML = '';
      users.forEach(({ nickname, race, cls }) => {
        const row = document.createElement('tr');
        row.innerHTML = `<td>${nickname}</td><td>${cls}</td><td>${race}</td>`;
        resultsTable.appendChild(row);
      });
    }

    wowClasses.forEach(cls => classContainer.appendChild(createCheckbox(cls, 'class')));
    wowRaces.forEach(race => raceContainer.appendChild(createCheckbox(race, 'race')));

    document.getElementById('selectionForm').addEventListener('submit', function(e) {
      e.preventDefault();
      const nickname = document.getElementById('nickname').value.trim();
      const selectedClass = document.querySelector("input[name='class']:checked");
      const selectedRace = document.querySelector("input[name='race']:checked");

      if (!nickname || !selectedClass || !selectedRace) {
        alert('Proszę podać nick, wybrać jedną klasę i jedną rasę.');
        return;
      }

      const allowedClasses = raceClassMap[selectedRace.value];
      if (!allowedClasses.includes(selectedClass.value)) {
        alert(`Rasa ${selectedRace.value} nie może być klasą ${selectedClass.value}.`);
        return;
      }

      const comboKey = `${selectedRace.value} ${selectedClass.value}`;
      stats[comboKey] = (stats[comboKey] || 0) + 1;
      users.push({ nickname, race: selectedRace.value, cls: selectedClass.value });

      updateStatsDisplay();
      loadResultsTable();
      syncSharedData();

      document.getElementById('nickname').value = '';
      selectedClass.checked = false;
      selectedRace.checked = false;
    });

    document.getElementById('resetButton').addEventListener('click', function() {
      const password = prompt('Podaj hasło, aby zresetować dane:');
      if (password === 'Mantas123') {
        if (confirm('Na pewno chcesz zresetować wszystkie dane?')) {
          stats = {};
          users = [];
          updateStatsDisplay();
          loadResultsTable();
          syncSharedData();
        }
      } else {
        alert('Nieprawidłowe hasło.');
      }
    });

    loadSharedData();
  </script>
</body>
</html>
