<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>Montanha Russa de Bolinhas</title>
  <style>
    body {
      display: flex;
      flex-direction: column;
      align-items: center;
      font-family: Arial, sans-serif;
      background: #f0f8ff;
      color: #333;
      padding: 20px;
    }
    h1 { font-size: 1.8rem; margin: 0.5rem; }
    #timer {
      font-size: 2rem;
      margin: 10px 0;
      font-weight: bold;
    }
    #instructions {
      text-align: center;
      margin-bottom: 1rem;
    }
    #chart-container {
      background: #fff;
      border: 1px solid #888;
      border-radius: 8px;
      padding: 10px;
    }
    canvas {
      display: block;
      margin: auto;
    }
    .info { margin-top: 0.5rem; font-size: 1.05rem; }
    .btn {
      margin-top: 1rem;
      padding: 0.4rem 0.8rem;
      font-size: 0.95rem;
      border: none;
      border-radius: 6px;
      background: #007bff;
      color: #fff;
      cursor: pointer;
    }
    .btn:disabled {
      background: #ccc;
      cursor: default;
    }
    #results { margin-top: 1.5rem; text-align: center; }
    .input-names input {
      margin: 0.2rem;
      padding: 0.3rem;
    }
  </style>
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>
</head>
<body>
  <h1>🎢 Montanha Russa de Bolinhas</h1>

  <div class="input-names">
    <input type="text" id="name1" placeholder="Nome competidor 1">
    <input type="text" id="name2" placeholder="Nome competidor 2">
    <input type="text" id="name3" placeholder="Nome competidor 3">
  </div>

  <div id="timer">00:00.00</div>
  <div id="instructions">
    <p>1 ▶ iniciar • 4 ▶ finalizar</p>
  </div>

  <div id="chart-container">
    <canvas id="chart" width="600" height="400"></canvas>
  </div>

  <div class="info" id="avgSpeed"></div>
  <button class="btn" id="nextBtn" disabled>Próximo Teste</button>
  <button class="btn" id="savePDF" style="display:none">Salvar PDF</button>
  <div id="results"></div>

  <script>
    const ctx = document.getElementById('chart').getContext('2d');
    const timerEl = document.getElementById('timer');
    let chart;
    let startTime = null;
    let timerInterval = null;
    let times = {};
    const DIST = 9.4;
    let trials = [];
    let current = 1;

    function formatTime(ms) {
      const total = Math.floor(ms);
      const m = Math.floor(total / 60000);
      const s = Math.floor((total % 60000) / 1000);
      const cs = Math.floor((total % 1000) / 10);
      return `${String(m).padStart(2,'0')}:${String(s).padStart(2,'0')}.${String(cs).padStart(2,'0')}`;
    }

    function startTimer() {
      startTime = performance.now();
      timerInterval = setInterval(() => {
        const elapsed = performance.now() - startTime;
        timerEl.textContent = formatTime(elapsed);
      }, 30);
    }

    function stopTimer() {
      clearInterval(timerInterval);
    }

    function initChart(dataset = []) {
      const data = {
        labels: [0, 2.35, 7.05, 9.4],
        datasets: dataset
      };
      if (chart) chart.destroy();
      chart = new Chart(ctx, {
        type: 'line',
        data: data,
        options: {
          responsive: false,
          scales: {
            x: {
              title: { display: true, text: 'Distância (m)' },
              min: 0, max: 9.4
            },
            y: {
              title: { display: true, text: 'Tempo (s)' },
              beginAtZero: true
            }
          }
        }
      });
    }

    function updatePlot() {
      const name = document.getElementById(`name${current}`).value || `Teste ${current}`;
      const dset = [{
        label: name,
        data: [0, times.t2, times.t3, times.t4],
        borderColor: ['blue','yellow','red'][current-1] || 'gray',
        backgroundColor: 'transparent',
        tension: 0.3
      }];
      initChart(dset);
      const avg = (DIST / times.t4).toFixed(2);
      document.getElementById('avgSpeed').textContent = `Velocidade média: ${avg} m/s`;
      document.getElementById('nextBtn').disabled = false;
    }

    document.addEventListener('keydown', e => {
      if (e.key === '1' && !startTime) {
        startTimer();
        times = {};
        document.getElementById('avgSpeed').textContent = '';
        document.getElementById('nextBtn').disabled = true;
      }
      if (startTime && e.key === '4') {
        times.t2 = (performance.now() - startTime) / 1000 * 0.25;
        times.t3 = (performance.now() - startTime) / 1000 * 0.75;
        times.t4 = ((performance.now() - startTime) / 1000).toFixed(2);
        stopTimer();
        updatePlot();
      }
    });

    document.getElementById('nextBtn').addEventListener('click', () => {
      trials.push({ ...times, trial: current, name: document.getElementById(`name${current}`).value });
      if (current < 3) {
        current++;
        startTime = null;
        timerEl.textContent = '00:00.00';
        times = {};
        document.getElementById('nextBtn').disabled = true;
      } else showFinalResults();
    });

    function showFinalResults() {
      const colors = ['blue','yellow','red'];
      const dsets = trials.map((t,i)=>({
        label: t.name || `Teste ${t.trial}`,
        data: [0,t.t2,t.t3,t.t4],
        borderColor: colors[i],
        backgroundColor: 'transparent',
        tension: 0.3
      }));
      initChart(dsets);

      const sortedTime = [...trials].sort((a,b)=> a.t4 - b.t4);
      const sortedVel = [...trials].sort((a,b)=> (DIST/b.t4) - (DIST/a.t4));

      let html = '<h2>🏁 Resultados Finais</h2>';
      html += '<h3>Menores Tempos</h3><ol>' + sortedTime.map(t=>`<li>${t.name || 'Teste '+t.trial}: ${t.t4}s</li>`).join('') + '</ol>';
      html += '<h3>Maiores Velocidades</h3><ol>' + sortedVel.map(t=>`<li>${t.name || 'Teste '+t.trial}: ${(DIST/t.t4).toFixed(2)} m/s</li>`).join('') + '</ol>';

      document.getElementById('results').innerHTML = html;
      document.getElementById('nextBtn').disabled = true;
      document.getElementById('savePDF').style.display = 'inline-block';
    }

    document.getElementById('savePDF').addEventListener('click', () => {
      const { jsPDF } = window.jspdf;
      const pdf = new jsPDF();

      pdf.text("Montanha Russa de Bolinhas - Resultados", 10, 10);
      trials.forEach((t, i) => {
        pdf.text(`${t.name || 'Teste '+t.trial}: t4=${t.t4}s, Veloc. média=${(DIST/t.t4).toFixed(2)}m/s`, 10, 20 + i * 10);
      });

      const canvas = document.getElementById("chart");
      const imgData = canvas.toDataURL("image/png", 1.0);
      pdf.addImage(imgData, 'PNG', 10, 50, 180, 120);

      pdf.save("resultados_montanha_russa.pdf");
    });

    initChart();
  </script>
</body>
</html>
