<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Calendário de Folgas</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      background: #f5f5f5;
      margin: 0;
      padding: 10px;
      color: #333;
    }

    h1 {
      text-align: center;
      margin-bottom: 10px;
    }

    #calendar {
      display: grid;
      grid-template-columns: repeat(7, 1fr);
      gap: 5px;
      max-width: 700px;
      margin: 20px auto;
      text-align: center;
    }

    .weekday {
      font-weight: bold;
      background: #007BFF;
      color: white;
      padding: 8px 0;
      border-radius: 4px;
    }

    .day {
      background: rgb(255, 254, 254);
      border: 1px solid #ccc;
      height: 90px;
      padding: 5px;
      border-radius: 5px;
      cursor: pointer;
      transition: background 0.3s;
      position: relative;
      font-size: 14px;
    }

    .day:hover {
      background: #e0f7fa;
    }

    .occupied {
      background: #ffe082 !important;
    }

    .team {
      background: #a5d6a7 !important;
    }

    .day small {
      display: block;
      font-size: 12px;
      color: #444;
    }

    #names-section {
      max-width: 700px;
      margin: 30px auto;
      background: rgb(255, 255, 255);
      padding: 15px;
      border-radius: 8px;
      border: 1px solid #ccc;
    }

    .group {
      display: flex;
      flex-wrap: wrap;
      gap: 10px;
      margin-bottom: 15px;
    }

    .name {
      flex: 1 1 45%;
      border: 1px solid #be1313;
      padding: 8px;
      border-radius: 5px;
      background: #f9f9f9;
    }

    .name span {
      font-weight: bold;
    }

    #popup, #teamPopup {
      position: fixed;
      top: 50%;
      left: 50%;
      transform: translate(-50%, -50%);
      background: rgb(255, 255, 255);
      border: 1px solid #ccc;
      border-radius: 8px;
      padding: 20px;
      box-shadow: 0 0 10px rgba(0,0,0,0.2);
      display: none;
      z-index: 10;
      max-width: 300px;
      width: 90%;
    }

    #popup h3, #teamPopup h3 {
      margin-top: 0;
    }

    #popup select, #popup button, #teamPopup button {
      width: 100%;
      padding: 8px;
      margin-top: 10px;
      border-radius: 5px;
      border: none;
      cursor: pointer;
      font-weight: bold;
    }

    #saveDay {
      background: #28a745;
      color: white;
    }

    #saveDay:hover {
      background: #1e7e34;
    }

    #removeDay {
      background: #dc3545;
      color: white;
    }


    #overlay {
      position: fixed;
      top: 0; left: 0;
      width: 100%; height: 100%;
      background: rgba(0,0,0,0.4);
      display: none;
      z-index: 5;
    }

    #buttons {
      text-align: center;
      margin-top: 30px;
    }

    button {
      padding: 10px 20px;
      margin: 5px;
      border: none;
      border-radius: 5px;
      cursor: pointer;
      color: rgb(255, 255, 255);
    }

    #generatePDF { background: #007BFF; }
    #generatePDF:hover { background: #0056b3; }

    #addTeamDay { background: #28a745; }
    #addTeamDay:hover { background: #1e7e34; }

    @media (max-width: 600px) {
      .day {
        height: 70px;
        font-size: 12px;
      }
    }
  </style>
</head>
<body>

  <h1 id="monthYear"></h1>

  <div id="calendar"></div>

  <div id="names-section">
    <h2>Auxiliares</h2>
    <div class="group" id="auxiliares"></div>

    <h2>Roçadores</h2>
    <div class="group" id="rocadores"></div>
  </div>

  <div id="buttons">
    <button id="addTeamDay">Adicionar folga da equipe</button>
    <button id="generatePDF">Gerar PDF</button>
  </div>

  <div id="overlay"></div>

  <div id="popup">
    <h3 id="popupTitle">Escolha o auxiliar e o roçador</h3>
    <select id="auxSelect">
      <option value="">Selecione um auxiliar</option>
    </select>
    <select id="rocSelect">
      <option value="">Selecione um roçador</option>
    </select>
    <button id="saveDay">Salvar</button>
    <button id="removeDay">Remover folga</button>
  </div>

  <div id="teamPopup">
    <h3>Adicionar folga da equipe</h3>
    <input type="number" id="teamDayInput" placeholder="Digite o dia (ex: 15)" min="1" max="31" style="width:100%;padding:8px;margin-top:10px;">
    <button id="saveTeamDay">Salvar</button>
  </div>

  <script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>
<script>
  const { jsPDF } = window.jspdf;

  const calendar = document.getElementById("calendar");
  const popup = document.getElementById("popup");
  const teamPopup = document.getElementById("teamPopup");
  const overlay = document.getElementById("overlay");
  const auxSelect = document.getElementById("auxSelect");
  const rocSelect = document.getElementById("rocSelect");
  const saveDay = document.getElementById("saveDay");
  const removeDay = document.getElementById("removeDay");
  const monthYearTitle = document.getElementById("monthYear");
  const addTeamDay = document.getElementById("addTeamDay");
  const saveTeamDay = document.getElementById("saveTeamDay");
  const teamDayInput = document.getElementById("teamDayInput");

  const auxiliares = ["Rafael 06","Joaquin","Edmilson","Josias","Sandro"];
  const rocadores = ["Pitoco","Renato","Gustavo","Ricardo","Maguinho"];

  let folgas = JSON.parse(localStorage.getItem("folgas")) || {};
  let folgasEquipe = JSON.parse(localStorage.getItem("folgasEquipe")) || [];

  const current = new Date();
  const currentMonth = current.getMonth();
  const currentYear = current.getFullYear();
  const monthNames = ["Janeiro","Fevereiro","Março","Abril","Maio","Junho","Julho","Agosto","Setembro","Outubro","Novembro","Dezembro"];

  monthYearTitle.textContent = `${monthNames[currentMonth]} ${currentYear}`;

  const auxSection = document.getElementById("auxiliares");
  const rocSection = document.getElementById("rocadores");

  function renderNames() {
    auxSection.innerHTML = "";
    rocSection.innerHTML = "";
    auxiliares.forEach(a => {
      const div = document.createElement("div");
      div.className = "name";
      const diaFolga = Object.keys(folgas).find(d => folgas[d].auxiliar === a);
      div.innerHTML = `<span>${a}</span> ${diaFolga ? "- folga dia " + diaFolga : ""}`;
      auxSection.appendChild(div);
    });
    rocadores.forEach(r => {
      const div = document.createElement("div");
      div.className = "name";
      const diaFolga = Object.keys(folgas).find(d => folgas[d].rocador === r);
      div.innerHTML = `<span>${r}</span> ${diaFolga ? "- folga dia " + diaFolga : ""}`;
      rocSection.appendChild(div);
    });
  }

  function renderCalendar() {
    calendar.innerHTML = "";
    const weekdays = ["Dom","Seg","Ter","Qua","Qui","Sex","Sáb"];
    weekdays.forEach(w => {
      const div = document.createElement("div");
      div.className = "weekday";
      div.textContent = w;
      calendar.appendChild(div);
    });

    const firstDay = new Date(currentYear, currentMonth, 1).getDay();
    const daysInMonth = new Date(currentYear, currentMonth + 1, 0).getDate();

    for (let i = 0; i < firstDay; i++) {
      const empty = document.createElement("div");
      calendar.appendChild(empty);
    }

    for (let i = 1; i <= daysInMonth; i++) {
      const div = document.createElement("div");
      div.className = "day";
      div.textContent = i;

      if (folgasEquipe.includes(i)) {
        div.classList.add("team");
        div.innerHTML += `<small>Folga da equipe</small>`;
      } else if (folgas[i] && (folgas[i].auxiliar || folgas[i].rocador)) {
        div.classList.add("occupied");
        div.innerHTML += `<small>${folgas[i].auxiliar || ''}<br>${folgas[i].rocador || ''}</small>`;
      }

      div.addEventListener("click", () => openPopup(i));
      calendar.appendChild(div);
    }
  }

  function openPopup(day) {
    popup.style.display = "block";
    overlay.style.display = "block";
    popup.setAttribute("data-day", day);
    document.getElementById("popupTitle").textContent = `Dia ${day}`;

    auxSelect.innerHTML = `<option value="">Selecione um auxiliar</option>`;
    rocSelect.innerHTML = `<option value="">Selecione um roçador</option>`;

    const usadosAux = Object.keys(folgas).filter(d => d != day).map(d => folgas[d].auxiliar).filter(Boolean);
    const usadosRoc = Object.keys(folgas).filter(d => d != day).map(d => folgas[d].rocador).filter(Boolean);

    auxiliares.filter(a => !usadosAux.includes(a) || folgas[day]?.auxiliar === a).forEach(a => {
      const opt = document.createElement("option");
      opt.value = a;
      opt.textContent = a;
      if (folgas[day]?.auxiliar === a) opt.selected = true;
      auxSelect.appendChild(opt);
    });

    rocadores.filter(r => !usadosRoc.includes(r) || folgas[day]?.rocador === r).forEach(r => {
      const opt = document.createElement("option");
      opt.value = r;
      opt.textContent = r;
      if (folgas[day]?.rocador === r) opt.selected = true;
      rocSelect.appendChild(opt);
    });

    removeDay.style.display = (folgas[day]?.auxiliar || folgas[day]?.rocador) ? "block" : "none";
  }

  saveDay.addEventListener("click", () => {
    const day = popup.getAttribute("data-day");
    const aux = auxSelect.value;
    const roc = rocSelect.value;

    if (aux || roc) {
      folgas[day] = { auxiliar: aux || null, rocador: roc || null };
    } else {
      delete folgas[day];
    }

    localStorage.setItem("folgas", JSON.stringify(folgas));
    closePopup();
    renderCalendar();
    renderNames();
  });

  removeDay.addEventListener("click", () => {
    const day = popup.getAttribute("data-day");
    delete folgas[day];
    localStorage.setItem("folgas", JSON.stringify(folgas));
    closePopup();
    renderCalendar();
    renderNames();
  });

  addTeamDay.addEventListener("click", () => {
    teamPopup.style.display = "block";
    overlay.style.display = "block";
  });

  // ✅ Agora permite cancelar folga da equipe
  saveTeamDay.addEventListener("click", () => {
    const dia = parseInt(teamDayInput.value);
    if (!isNaN(dia)) {
      if (folgasEquipe.includes(dia)) {
        folgasEquipe = folgasEquipe.filter(d => d !== dia); // remove
      } else {
        folgasEquipe.push(dia); // adiciona
      }
      localStorage.setItem("folgasEquipe", JSON.stringify(folgasEquipe));
      renderCalendar();
    }
    closePopup();
  });

  overlay.addEventListener("click", closePopup);
  function closePopup() {
    popup.style.display = "none";
    teamPopup.style.display = "none";
    overlay.style.display = "none";
  }

  // PDF permanece igual
  document.getElementById("generatePDF").addEventListener("click", () => {
    const pdf = new jsPDF('p', 'mm', 'a4');
    let y = 20;

    pdf.setFontSize(16);
    pdf.text(monthYearTitle.textContent, 105, y, { align: 'center' });
    y += 10;

    const weekdays = ["Dom","Seg","Ter","Qua","Qui","Sex","Sáb"];
    let startX = 10, cellW = 27, cellH = 15;
    pdf.setFontSize(10);

    weekdays.forEach((w, i) => {
      pdf.setFillColor(0,123,255);
      pdf.setTextColor(255,255,255);
      pdf.rect(startX + i*cellW, y, cellW, cellH, "F");
      pdf.text(w, startX + i*cellW + cellW/2, y + 10, { align: 'center' });
    });

    y += cellH;
    const firstDay = new Date(currentYear, currentMonth, 1).getDay();
    const daysInMonth = new Date(currentYear, currentMonth + 1, 0).getDate();

    let dayPos = firstDay;
    pdf.setTextColor(0,0,0);

    for (let d = 1; d <= daysInMonth; d++) {
      let aux = folgas[d]?.auxiliar || "";
      let roc = folgas[d]?.rocador || "";

      if (folgasEquipe.includes(d)) {
        pdf.setFillColor(165,214,167);
      } else if (aux || roc) {
        pdf.setFillColor(255,224,130);
      } else {
        pdf.setFillColor(255,255,255);
      }

      pdf.rect(startX + dayPos*cellW, y, cellW, cellH, "F");
      pdf.setFontSize(10);
      pdf.text(String(d), startX + dayPos*cellW + 2, y + 6);

      pdf.setFontSize(7);
      if (folgasEquipe.includes(d)) {
        pdf.text("Equipe", startX + dayPos*cellW + cellW/2, y + 10, { align: 'center' });
      } else {
        if (aux) pdf.text(aux, startX + dayPos*cellW + cellW/2, y + 9, { align: 'center' });
        if (roc) pdf.text(roc, startX + dayPos*cellW + cellW/2, y + 12, { align: 'center' });
      }

      dayPos++;
      if (dayPos > 6) {
        dayPos = 0;
        y += cellH;
      }
    }

    y += cellH + 10;
    pdf.setFontSize(14);
    pdf.text("Auxiliares", 10, y);
    y += 6;

    pdf.setFontSize(10);
    auxiliares.forEach(a => {
      const diaFolga = Object.keys(folgas).find(d => folgas[d].auxiliar === a);
      pdf.text(`${a} ${diaFolga ? "- folga dia " + diaFolga : ""}`, 15, y);
      y += 5;
    });

    y += 8;
    pdf.setFontSize(14);
    pdf.text("Roçadores", 10, y);
    y += 6;

    pdf.setFontSize(10);
    rocadores.forEach(r => {
      const diaFolga = Object.keys(folgas).find(d => folgas[d].rocador === r);
      pdf.text(`${r} ${diaFolga ? "- folga dia " + diaFolga : ""}`, 15, y);
      y += 5;
    });

    pdf.save("folgas.pdf");
  });

  renderCalendar();
  renderNames();
</script>


 
</body>
</html>
