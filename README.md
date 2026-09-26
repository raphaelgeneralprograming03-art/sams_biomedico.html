
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>SROS // TRIADE SYSTEM - Módulo SAMS JARVIS</title>
    <style>
        * { box-sizing: border-box; margin: 0; padding: 0; }
        body, html { width: 100%; height: 100%; overflow: hidden; background-color: #080404; font-family: 'Courier New', Courier, monospace; color: #ff3333; }
        #canvas-container { width: 100%; height: 100%; position: absolute; top: 0; left: 0; z-index: 1; }
        canvas { display: block; width: 100%; height: 100%; }
        .hud-overlay { position: absolute; top: 20px; left: 20px; z-index: 10; pointer-events: none; text-shadow: 0 0 5px #ff3333; background: rgba(16, 4, 4, 0.9); padding: 15px; border: 1px solid #ff3333; border-radius: 4px; box-shadow: 0 0 15px rgba(255, 51, 51, 0.2); max-width: 320px; }
        h1 { font-size: 14px; margin-bottom: 8px; border-bottom: 1px solid #ff3333; padding-bottom: 4px; text-transform: uppercase; }
        .telemetry-item { font-size: 11px; margin: 5px 0; display: flex; justify-content: space-between; }
        .status-active { color: #00ff66; animation: blink 1.5s infinite; }
        .status-alert { color: #ff3333; animation: blink 0.8s infinite; }
        .btn-action { position: absolute; bottom: 20px; left: 20px; z-index: 10; background: #220707; border: 1px solid #ff3333; color: #ff3333; padding: 10px 20px; font-family: inherit; font-size: 11px; cursor: pointer; text-shadow: 0 0 3px #ff3333; box-shadow: 0 0 10px rgba(255,51,51,0.1); border-radius: 4px; pointer-events: auto; }
        .btn-action:hover { background: #ff3333; color: #080404; font-weight: bold; }
        .legend { position: absolute; top: 20px; right: 20px; z-index: 10; background: rgba(16, 4, 4, 0.9); border: 1px solid #ff3333; padding: 10px; font-size: 10px; border-radius: 4px; }
        .legend-item { margin: 4px 0; display: flex; align-items: center; }
        .dot { width: 8px; height: 8px; border-radius: 50%; margin-right: 8px; display: inline-block; }
        @keyframes blink { 0%, 100% { opacity: 1; } 50% { opacity: 0.4; } }
    </style>
</head>
<body>

    <div id="canvas-container">
        <canvas id="samsCanvas"></canvas>
    </div>

    <div class="hud-overlay">
        <h1>SROS // SAMS BIOMEDICAL</h1>
        <div class="telemetry-item"><span>MONITOR BIOMETRICO:</span> <span id="status-sams" class="status-active">NOMINAL</span></div>
        <div class="telemetry-item"><span>FREQUENCIA CARD.:</span> <span id="bpm-val">72 BPM</span></div>
        <div class="telemetry-item"><span>OXIGENAÇÃO (SpO₂):</span> <span id="spo2-val">98%</span></div>
        <div class="telemetry-item"><span>INJECÇÕES RESTANTES:</span> <span id="inj-val">05/05</span></div>
        <div class="telemetry-item"><span>DOSAGEM RECENTE:</span> <span id="dose-val">NENHUMA</span></div>
    </div>

    <div class="legend">
        <div class="legend-item"><span class="dot" style="background:#ff3333;"></span>Eletrocardiograma Ativo</div>
        <div class="legend-item"><span class="dot" style="background:#00ff66;"></span>Vaporizadores Médicos Prontos</div>
        <div class="legend-item"><span class="dot" style="background:#ffffff;"></span>Visão Interna do Capacete</div>
    </div>

    <button class="btn-action" id="trigger-arrhythmia">INDUZIR ARRITMIA CARDÍACA GRAVE</button>

    <script>
        const canvas = document.getElementById('samsCanvas');
        const ctx = canvas.getContext('2d');

        function redimensionar() {
            canvas.width = window.innerWidth;
            canvas.height = window.innerHeight;
        }
        window.addEventListener('resize', redimensionar);
        redimensionar();

        // Parâmetros do SAMS
        let modoArritmia = false;
        let bpm = 72;
        let spo2 = 98;
        let injecoes = 5;
        let doseRecente = "NENHUMA";
        let cooldownMed = 0;
        let pulsoHUD = 0;

        // Histórico de pontos para desenhar a linha do ECG
        const ecgPontos = [];
        const maxPontos = 100;
        let ecgX = 0;

        // Partículas das injeções médicas (Vaporizadores intramusculares)
        const particulasMedicas = [];

        const bpmEl = document.getElementById('bpm-val');
        const spo2El = document.getElementById('spo2-val');
        const injEl = document.getElementById('inj-val');
        const doseEl = document.getElementById('dose-val');
        const statusEl = document.getElementById('status-sams');
        const btnToggle = document.getElementById('trigger-arrhythmia');

        btnToggle.addEventListener('click', () => {
            modoArritmia = !modoArritmia;
            btnToggle.innerText = modoArritmia ? "ESTABILIZAR RITMO CARDÍACO" : "INDUZIR ARRITMIA CARDÍACA GRAVE";
        });

        function gerarPontoECG() {
            ecgX += 1;
            let tempo = ecgX * 0.2;
            let baseBPM = modoArritmia ? 165 : 72;
            
            let ciclo = (tempo * (baseBPM / 60)) % (Math.PI * 2);
            let y = 0;

            if (ciclo < 0.3) y = Math.sin(ciclo * Math.PI / 0.3) * -4; 
            else if (ciclo >= 0.4 && ciclo < 0.5) y = (ciclo - 0.4) * 15; 
            else if (ciclo >= 0.5 && ciclo < 0.65) y = -50 + (Math.random() * (modoArritmia ? 20 : 0)); 
            else if (ciclo >= 0.65 && ciclo < 0.8) y = 15; 
            else if (ciclo >= 0.9 && ciclo < 1.3) y = Math.sin((ciclo - 0.9) * Math.PI / 0.4) * -8; 

            if (modoArritmia) {
                y += (Math.random() - 0.5) * 12;
            }

            ecgPontos.push(y);
            if (ecgPontos.length > maxPontos) {
                ecgPontos.shift();
            }
        }

        function dispararInjecao() {
            if (injecoes > 0 && cooldownMed <= 0) {
                injecoes--;
                cooldownMed = 250; 
                doseRecente = "ESTABILIZADOR METABÓLICO";
                
                for (let i = 0; i < 40; i++) {
                    particulasMedicas.push({
                        x: canvas.width / 2 + (Math.random() - 0.5) * 120,
                        y: canvas.height / 2 + 120,
                        vx: (Math.random() - 0.5) * 4,
                        vy: -Math.random() * 3 - 1,
                        vida: 1.0,
                        tamanho: 3 + Math.random() * 4
                    });
                }
            }
        }

        function draw() {
            // Fundo escuro do HUD
            ctx.fillStyle = '#080404';
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            const centerX = canvas.width / 2;
            const centerY = canvas.height / 2;
            pulsoHUD += 0.025;

            // Lógica biométrica
            if (modoArritmia) {
                bpm = Math.min(176, bpm + 1.2);
                spo2 = Math.max(85, spo2 - 0.08);
                statusEl.innerText = "CRÍTICO: ARRITMIA";
                statusEl.className = "status-alert";
                
                if (bpm > 150 && cooldownMed <= 0 && injecoes > 0) {
                    dispararInjecao();
                }
            } else {
                bpm = Math.max(72, bpm - 0.8);
                spo2 = Math.min(98, spo2 + 0.15);
                if (bpm <= 75) {
                    statusEl.innerText = "NOMINAL";
                    statusEl.className = "status-active";
                }
            }

            if (cooldownMed > 0) {
                cooldownMed--;
                if (cooldownMed === 1) modoArritmia = false; 
            }

            // Atualização dos textos de telemetria
            bpmEl.innerText = `${Math.round(bpm)} BPM`;
            spo2El.innerText = `${Math.round(spo2)}%`;
            injEl.innerText = `0${injecoes}/05`;
            doseEl.innerText = doseRecente;

            gerarPontoECG();

            // VISÃO INTERNA DO CAPACETE (ESTILO IRON MAN HUD)
            ctx.strokeStyle = modoArritmia ? 'rgba(255, 51, 51, 0.25)' : 'rgba(0, 255, 102, 0.18)';
            ctx.lineWidth = 3;
            
            let raioViseira = Math.min(canvas.width, canvas.height) * 0.42;
            ctx.beginPath();
            ctx.arc(centerX, centerY, raioViseira, 0.15 * Math.PI, 0.85 * Math.PI);
            ctx.stroke();
            ctx.beginPath();
            ctx.arc(centerX, centerY, raioViseira, 1.15 * Math.PI, 1.85 * Math.PI);
            ctx.stroke();

            // Mira holográfica central
            ctx.strokeStyle = modoArritmia ? 'rgba(255, 51, 51, 0.5)' : 'rgba(0, 255, 102, 0.4)';
            ctx.lineWidth = 1.5;
            ctx.beginPath();
            ctx.arc(centerX, centerY - 40, 25 + Math.sin(pulsoHUD) * 2, 0, Math.PI * 2);
            ctx.stroke();
            
            // Cantoneiras holográficas
            let d = 55 + Math.sin(pulsoHUD) * 3;
            ctx.beginPath();
            ctx.moveTo(centerX - d, centerY - 80); ctx.lineTo(centerX - d - 10, centerY - 80); ctx.lineTo(centerX - d - 10, centerY - 60);
            ctx.moveTo(centerX + d, centerY - 80); ctx.lineTo(centerX + d + 10, centerY - 80); ctx.lineTo(centerX + d + 10, centerY - 60);
            ctx.stroke();

            // DESENHO DA LINHA DO ECG
            let ecgYBase = centerY - 40;
            ctx.lineWidth = 2.5;
            ctx.strokeStyle = modoArritmia ? '#ff3333' : '#00ff66';
            ctx.shadowBlur = 10;
            ctx.shadowColor = modoArritmia ? '#ff3333' : '#00ff66';
            ctx.beginPath();
            
            let inicioX = centerX - (maxPontos * 2.5) / 2;

            for (let i = 0; i < ecgPontos.length; i++) {
                let x = inicioX + (i * 2.5);
                let y = ecgYBase + ecgPontos[i];
                if (i === 0) ctx.moveTo(x, y);
                else ctx.lineTo(x, y);
            }
            ctx.stroke();
            ctx.shadowBlur = 0;

            // PAINEL INFERIOR DE INJETORES MÉDICOS (VAPORIZADORES)
            ctx.strokeStyle = modoArritmia ? '#ff3333' : '#00ff66';
            ctx.lineWidth = 1.5;
            
            let totalSlots = 5;
            let larguraSlot = 28;
            let espacamento = 12;
            let larguraTotal = (totalSlots * larguraSlot) + ((totalSlots - 1) * espacamento);
            let startX = centerX - (larguraTotal / 2);
            let startY = centerY + 90;

            for (let i = 0; i < totalSlots; i++) {
                let sx = startX + i * (larguraSlot + espacamento);
                
                // Moldura da ampola de injeção
                ctx.strokeStyle = i < injecoes ? '#00ff66' : 'rgba(255, 51, 51, 0.4)';
                ctx.strokeRect(sx, startY, larguraSlot, 40);

                if (i < injecoes) {
                    ctx.fillStyle = 'rgba(0, 255, 102, 0.4)';
                    ctx.fillRect(sx + 3, startY + 10, larguraSlot - 6, 26);
                }
            }

            // PARTICULAS DE VAPORIZAÇÃO MÉDICA
            for (let i = particulasMedicas.length - 1; i >= 0; i--) {
                let p = particulasMedicas[i];
                p.x += p.vx;
                p.y += p.vy;
                p.vida -= 0.015;

                if (p.vida <= 0) {
                    particulasMedicas.splice(i, 1);
                    continue;
                }

                ctx.fillStyle = `rgba(0, 255, 102, ${p.vida})`;
                ctx.shadowColor = '#00ff66';
                ctx.shadowBlur = 6;
                ctx.beginPath();
                ctx.arc(p.x, p.y, p.tamanho, 0, Math.PI * 2);
                ctx.fill();
                ctx.shadowBlur = 0;
            }

            requestAnimationFrame(draw);
        }

        // Inicializa a renderização
        draw();
    </script>
</body>
</html>
