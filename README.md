<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>SROS // TRIADE SYSTEM - Módulo SAMS</title>
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
        <div class="legend-item"><span class="dot" style="background:#ffffff;"></span>Disparo Automático de Dose</div>
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

        // Histórico de pontos para desenhar a linha do ECG
        const ecgPontos = [];
        const maxPontos = 150;
        let ecgX = 0;

        // Partículas das injeções médicas (Vaporizadores intramusculares)
        const particulasMedicas = [];

        const bpmEl = document.getElementById('bpm-val');
        const spo2El = document.getElementById('spo2-val');
        const injEl = document.getElementById('inj-val');
        const doseEl = document.getElementById('dose-val');
        const statusEl = document.getElementById('status-sams');

        function gerarPontoECG() {
            ecgX += 1;
            let tempo = ecgX * 0.15;
            let baseBPM = modoArritmia ? 165 : 72;
            
            // Frequência do batimento baseada no BPM atual
            let ciclo = (tempo * (baseBPM / 60)) % (Math.PI * 2);
            let y = 0;

            // Simulação da onda do complexo P-Q-R-S-T cardíaco
            if (ciclo < 0.3) y = Math.sin(ciclo * Math.PI / 0.3) * -5; // Onda P
            else if (ciclo >= 0.4 && ciclo < 0.5) y = (ciclo - 0.4) * 20; // Q
            else if (ciclo >= 0.5 && ciclo < 0.65) y = -70 + (Math.random() * (modoArritmia ? 25 : 0)); // R
            else if (ciclo >= 0.65 && ciclo < 0.8) y = 25; // S
            else if (ciclo >= 0.9 && ciclo < 1.3) y = Math.sin((ciclo - 0.9) * Math.PI / 0.4) * -12; // Onda T

            // Adiciona ruído se estiver em arritmia cardíaca
            if (modoArritmia) {
                y += (Math.random() - 0.5) * 15;
            }

            ecgPontos.push(y);
            if (ecgPontos.length > maxPontos) {
                ecgPontos.shift();
            }
        }

        function dispararInjecao() {
            if (injecoes > 0 && cooldownMed <= 0) {
                injecoes--;
                cooldownMed = 300; // Impede disparos simultâneos
                doseRecente = "ESTABILIZADOR METABÓLICO";
                
                // Dispara efeito visual de spray médico de contramedida
                for (let i = 0; i < 30; i++) {
                    particulasMedicas.push({
                        x: canvas.width / 2 + (Math.random() - 0.5) * 40,
                        y: canvas.height * 0.75,
                        vx: (Math.random() - 0.5) * 4,
                        vy: -Math.random() * 3 - 1,
                        vida: 1.0,
                        tamanho: 3 + Math.random() * 3
                    });
                }
            }
        }

        function draw() {
            // Limpa com fundo escuro hospitalar/militar
            ctx.fillStyle = '#100404';
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            const centerX = canvas.width / 2;
            const centerY = canvas.height / 2;

            // Atualiza lógica dos sinais biométricos computados pelo SAMS
            if (modoArritmia) {
                bpm = Math.min(178, bpm + 2);
                spo2 = Math.max(86, spo2 - 0.1);
                statusEl.innerText = "CRÍTICO: ARRITMIA";
                statusEl.className = "status-alert";
                
                // Resposta Autônoma do Traje SAMS: Aplica o medicamento se o quadro for severo
                if (bpm > 150 && cooldownMed <= 0) {
                    dispararInjecao();
                }
            } else {
                bpm = Math.max(72, bpm - 1);
                spo2 = Math.min(98, spo2 + 0.2);
                if (bpm === 72) {
                    statusEl.innerText = "NOMINAL";
                    statusEl.className = "status-active";
                }
            }

            if (cooldownMed > 0) {
                cooldownMed--;
                if (cooldownMed === 1) modoArritmia = false; // O remédio faz efeito
            }

            // Gera e avança o traçado do monitor cardíaco
            gerarPontoECG();

            // 1. DESENHAR A GRADE E MONITOR DO ELETROCARDIOGRAMA (ECG)
            ctx.strokeStyle = modoArritmia ? 'rgba(255, 51, 51, 0.1)' : 'rgba(0, 255, 102, 0.1)';
            ctx.lineWidth = 1;
            
            // Desenha linhas de fundo do osciloscópio
            let ecgYBase = centerY - 60;
            ctx.beginPath();
            ctx.moveTo(0, ecgYBase);
            ctx.lineTo(canvas.width, ecgYBase);
            ctx.stroke();
            
            // Desenha a linha dinâmica do batimento cardíaco
            ctx.lineWidth = 2.5;
            ctx.strokeStyle = modoArritmia ? '#ff3333' : '#00ff66';
            ctx.beginPath();
            
            let inicioX = centerX - (maxPontos * 2) / 2;

            for (let i = 0; i < ecgPontos.length; i++) {
                let x = inicioX + (i * 2);
                let y = ecgYBase + ecgPontos[i];
                if (i === 0) ctx.moveTo(x, y);
                else ctx.lineTo(x, y);
            }
            ctx.stroke();

            // 2. DESENHAR MÓDULO FÍSICO DE MICROINJETORES (Parte inferior)
            ctx.strokeStyle = modoArritmia ? '#ff3333' : '#00ff66';
            ctx.lineWidth = 2;
            ctx.fillStyle = 'rgba(255, 51, 51, 0.05)';
            ctx.fillRect(centerX - 60, centerY + 100, 120, 50);
            ctx.strokeRect(centerX - 60, centerY + 100, 120, 50);

            // Desenha 5 slots de ampolas médicas
            for(let i = 0; i < 5; i++) {
                ctx.fillStyle = i < injecoes ? '#00ff66' : '#333333';
                ctx.fillRect(centerX - 50 + (i * 20), centerY + 115, 12, 20);
            }

            // 3. RENDERIZAR NUVEM DE VAPORIZAÇÃO DA INJEÇÃO MÉDICA
            particulasMedicas.forEach((p, idx) => {
                p.x += p.vx;
                p.y += p.vy;
                p.vida -= 0.015;

                ctx.fillStyle = `rgba(255, 255, 255, ${p.vida})`;
                ctx.beginPath();
                ctx.arc(p.x, p.y, p.tamanho, 0, Math.PI * 2);
                ctx.fill();

                if (p.vida <= 0) {
                    particulasMedicas.splice(idx, 1);
                }
            });

            // 4. ATUALIZAR HUD DE TELEMETRIA
