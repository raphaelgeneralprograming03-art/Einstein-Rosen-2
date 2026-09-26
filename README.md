
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Simulação Temporal de Buraco de Minhoca - Tríade MHD+SAMS+SROS</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }
        body, html {
            width: 100%;
            height: 100%;
            overflow: hidden;
            background-color: #000005;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            color: #00ffcc;
            user-select: none;
        }
        canvas {
            display: block;
            width: 100vw;
            height: 100vh;
            position: absolute;
            top: 0;
            left: 0;
            z-index: 1;
        }

        /* HUD Principal */
        #hud {
            position: absolute;
            top: 20px;
            left: 20px;
            z-index: 10;
            background: rgba(0, 5, 15, 0.88);
            padding: 18px;
            border-radius: 8px;
            border: 1px solid #00ffcc;
            box-shadow: 0 0 20px rgba(0, 255, 204, 0.25);
            max-width: 360px;
            backdrop-filter: blur(5px);
        }
        h1 {
            font-size: 13px;
            text-transform: uppercase;
            letter-spacing: 2px;
            margin-bottom: 10px;
            color: #fff;
            border-bottom: 1px solid #00ffcc;
            padding-bottom: 6px;
        }
        .status-item {
            margin-bottom: 6px;
            font-size: 11px;
            display: flex;
            justify-content: space-between;
        }
        .status-label {
            color: #88aacc;
        }
        .status-value {
            color: #fff;
            font-weight: bold;
            font-family: 'Courier New', Courier, monospace;
        }
        
        /* Controles de Estágio no HUD */
        .stage-banner {
            margin-top: 10px;
            padding: 8px;
            background: rgba(0, 255, 204, 0.1);
            border-left: 3px solid #00ffcc;
            font-size: 11px;
            color: #fff;
            font-weight: bold;
        }

        .controls {
            position: absolute;
            bottom: 25px;
            left: 50%;
            transform: translateX(-50%);
            z-index: 10;
            display: flex;
            gap: 10px;
            background: rgba(0, 5, 15, 0.85);
            padding: 10px 15px;
            border-radius: 30px;
            border: 1px solid rgba(0, 255, 204, 0.4);
            backdrop-filter: blur(5px);
        }

        button {
            background: rgba(0, 30, 40, 0.8);
            border: 1px solid #00ffcc;
            color: #00ffcc;
            padding: 6px 14px;
            font-size: 11px;
            font-weight: bold;
            border-radius: 15px;
            cursor: pointer;
            transition: all 0.2s ease;
            text-transform: uppercase;
            letter-spacing: 1px;
        }

        button:hover {
            background: #00ffcc;
            color: #000;
            box-shadow: 0 0 10px #00ffcc;
        }

        button.active {
            background: #00ffcc;
            color: #000;
        }

        .flash {
            animation: pulse 1s infinite alternate;
        }
        @keyframes pulse {
            from { opacity: 0.5; }
            to { opacity: 1; }
        }
    </style>
</head>
<body>

    <canvas id="wormholeCanvas"></canvas>

    <!-- HUD de Telemetria Técnica -->
    <div id="hud">
        <h1>Métrica Einstein-Rosen HD</h1>
        <div class="status-item"><span class="status-label">FASE DA VIAGEM:</span> <span class="status-value flash" id="phase-val" style="color: #00ff55;">ORIGEM (PRESENTE)</span></div>
        <div class="status-item"><span class="status-label">COORD. TEMPORAL:</span> <span class="status-value" id="time-val">2026.09.26</span></div>
        <div class="status-item"><span class="status-label">REATOR MHD:</span> <span class="status-value" id="mhd-val">143.00 Eq/Plck</span></div>
        <div class="status-item"><span class="status-label">ESTABILIZADOR SAMS:</span> <span class="status-value" id="sams-val">FLUIDO ATIVO</span></div>
        <div class="status-item"><span class="status-label">LASERS SROS:</span> <span class="status-value" id="sros-val">100.0% SINCRONIZADO</span></div>

        <div class="stage-banner" id="stage-desc">
            SISTEMA EM ÓRBITA TERRESTRE (ERA CONTEMPORÂNEA).
        </div>
    </div>

    <!-- Painel de Seleção Manual de Fases -->
    <div class="controls">
        <button id="btn-0" class="active" onclick="setStage(0)">1. Origem (Presente)</button>
        <button id="btn-1" onclick="setStage(1)">2. Entrada (Portal)</button>
        <button id="btn-2" onclick="setStage(2)">3. Túnel HIPERESPAÇO</button>
        <button id="btn-3" onclick="setStage(3)">4. Saída (Ruptura)</button>
        <button id="btn-4" onclick="setStage(4)">5. Destino (Futuro)</button>
        <button id="btn-auto" onclick="toggleAutoPlay()" style="border-color: #ffaa00; color: #ffaa00;">Modo: Sequencial</button>
    </div>

    <script>
        const canvas = document.getElementById('wormholeCanvas');
        const ctx = canvas.getContext('2d');

        function resize() {
            canvas.width = window.innerWidth;
            canvas.height = window.innerHeight;
        }
        window.addEventListener('resize', resize);
        resize();

        // --- ESTADOS DA NARRATIVA TEMPORAL ---
        // 0: Origem (Terra Presente + Galáxia)
        // 1: Abertura do Buraco de Minhoca de Entrada + Injeção da Nave
        // 2: Trânsito no Túnel de Einstein-Rosen (Hiperespaço)
        // 3: Abertura do Buraco de Minhoca de Saída (Emergência de Luz)
        // 4: Destino (Terra do Futuro Pós-Singularidade + Galáxia do Futuro)
        let currentStage = 0;
        let stageProgress = 0; // 0.0 a 1.0 dentro da fase
        let autoPlay = true;
        let rotationAngle = 0;

        // --- PARTICULAS DO TÚNEL DE MINHOCA ---
        const particleCount = 300;
        const particles = [];
        for (let i = 0; i < particleCount; i++) {
            let angle = Math.random() * Math.PI * 2;
            let radius = 60 + Math.random() * 220;
            particles.push({
                x: Math.cos(angle) * radius,
                y: Math.sin(angle) * radius,
                z: Math.random() * 800,
                type: Math.random() > 0.5 ? 'sros' : 'energy',
                size: 1.5 + Math.random() * 2.5
            });
        }

        // --- ESTRELAS DE FUNDO DA GALÁXIA ---
        const starCount = 400;
        const stars = [];
        for(let i = 0; i < starCount; i++) {
            stars.push({
                x: (Math.random() - 0.5) * 2000,
                y: (Math.random() - 0.5) * 2000,
                z: Math.random() * 1000,
                size: Math.random() * 1.8 + 0.5,
                color: Math.random() > 0.3 ? '#ffffff' : (Math.random() > 0.5 ? '#88ccff' : '#ffddaa')
            });
        }

        // --- NAVE ESPACIAL ---
        const ship = {
            x: 0,
            y: 0,
            z: 0,
            scale: 1,
            angle: 0
        };

        // --- DESENHO DA TERRA DO PRESENTE (ESTÁGIO 0 E 1) ---
        function drawPresentEarth(centerX, centerY, radius) {
            ctx.save();
            ctx.translate(centerX, centerY);

            // Atmosfera Azul Suave
            let atmGrad = ctx.createRadialGradient(0, 0, radius * 0.95, 0, 0, radius * 1.25);
            atmGrad.addColorStop(0, 'rgba(0, 180, 255, 0.4)');
            atmGrad.addColorStop(1, 'rgba(0, 180, 255, 0)');
            ctx.fillStyle = atmGrad;
            ctx.beginPath(); ctx.arc(0, 0, radius * 1.25, 0, Math.PI * 2); ctx.fill();

            // Globo Terrestre Oceânico
            let oceanGrad = ctx.createRadialGradient(-radius*0.3, -radius*0.3, 5, 0, 0, radius);
            oceanGrad.addColorStop(0, '#1a5f7a');
            oceanGrad.addColorStop(0.7, '#082032');
            oceanGrad.addColorStop(1, '#010a12');
            ctx.fillStyle = oceanGrad;
            ctx.beginPath(); ctx.arc(0, 0, radius, 0, Math.PI * 2); ctx.fill();

            // Continentes Procedurais
            ctx.fillStyle = '#2c5d3b';
            let time = Date.now() * 0.0003;
            for(let i = 0; i < 5; i++) {
                let cx = Math.sin(time + i * 1.5) * (radius * 0.5);
                let cy = Math.cos(time * 0.8 + i) * (radius * 0.4);
                ctx.beginPath();
                ctx.arc(cx, cy, radius * 0.35, 0, Math.PI * 2);
                ctx.fill();
            }

            // Nuvens
            ctx.fillStyle = 'rgba(255, 255, 255, 0.35)';
            for(let i = 0; i < 4; i++) {
                let cx = Math.cos(time * 1.2 + i * 2) * (radius * 0.6);
                let cy = Math.sin(time + i) * (radius * 0.5);
                ctx.beginPath();
                ctx.arc(cx, cy, radius * 0.28, 0, Math.PI * 2);
                ctx.fill();
            }

            ctx.restore();
        }

        // --- DESENHO DA TERRA DO FUTURO (ESTÁGIO 4 - SÉCULO XXXI) ---
        function drawFutureEarth(centerX, centerY, radius) {
            ctx.save();
            ctx.translate(centerX, centerY);

            // Anel Orbital Cyberpunk / Escudo Energético
            ctx.strokeStyle = 'rgba(0, 255, 204, 0.6)';
            ctx.lineWidth = 2;
            ctx.beginPath();
            ctx.ellipse(0, 0, radius * 1.6, radius * 0.4, -0.2, 0, Math.PI * 2);
            ctx.stroke();

            // Atmosfera Dourada / Ciano de Alta Tecnologia
            let atmGrad = ctx.createRadialGradient(0, 0, radius * 0.9, 0, 0, radius * 1.35);
            atmGrad.addColorStop(0, 'rgba(0, 255, 204, 0.5)');
            atmGrad.addColorStop(0.5, 'rgba(180, 0, 255, 0.2)');
            atmGrad.addColorStop(1, 'rgba(0, 0, 0, 0)');
            ctx.fillStyle = atmGrad;
            ctx.beginPath(); ctx.arc(0, 0, radius * 1.35, 0, Math.PI * 2); ctx.fill();

            // Núcleo Terrestre Futurista
            let oceanGrad = ctx.createRadialGradient(-radius*0.2, -radius*0.2, 5, 0, 0, radius);
            oceanGrad.addColorStop(0, '#0f2b46');
            oceanGrad.addColorStop(0.8, '#05101e');
            oceanGrad.addColorStop(1, '#02050b');
            ctx.fillStyle = oceanGrad;
            ctx.beginPath(); ctx.arc(0, 0, radius, 0, Math.PI * 2); ctx.fill();

            // Megacidades / Rede de Luzes no Lado Noturno (Ouro e Neon)
            let time = Date.now() * 0.0005;
            ctx.fillStyle = '#ffaa00';
            for(let i = 0; i < 40; i++) {
                let lx = Math.sin(i * 99 + time) * (radius * 0.85);
                let ly = Math.cos(i * 33 + time) * (radius * 0.85);
                if (Math.hypot(lx, ly) < radius * 0.95) {
                    ctx.fillRect(lx, ly, 2, 2);
                }
            }

            // Escudo Geodésico MHD
            ctx.strokeStyle = 'rgba(255, 0, 180, 0.25)';
            ctx.lineWidth = 1;
            for(let r = 20; r < radius; r += 25) {
                ctx.beginPath();
                ctx.arc(0, 0, r, 0, Math.PI * 2);
                ctx.stroke();
            }

            ctx.restore();
        }

        // --- DESENHO DA NAVE ESPACIAL (CRUZADOR TRÍADE) ---
        function drawShip(x, y, scale, rotation, thrusterActive = true) {
            ctx.save();
            ctx.translate(x, y);
            ctx.rotate(rotation);
            ctx.scale(scale, scale);

            // Rastro dos Propulsores MHD
            if (thrusterActive) {
                let gradThruster = ctx.createLinearGradient(-30, 0, -70, 0);
                gradThruster.addColorStop(0, '#00ffcc');
                gradThruster.addColorStop(0.5, '#ff00b4');
                gradThruster.addColorStop(1, 'rgba(0,0,0,0)');
                ctx.fillStyle = gradThruster;
                ctx.beginPath();
                ctx.moveTo(-20, -6);
                ctx.lineTo(-65 - Math.random() * 15, 0);
                ctx.lineTo(-20, 6);
                ctx.closePath();
                ctx.fill();
            }

            // Casco Principal Sleek Sci-Fi
            ctx.fillStyle = '#d0e0e8';
            ctx.strokeStyle = '#00ffcc';
            ctx.lineWidth = 1.5;

            ctx.beginPath();
            ctx.moveTo(30, 0);      // Nariz
            ctx.lineTo(-15, -15);   // Asa Esquerda
            ctx.lineTo(-10, -5);    // Corpo
            ctx.lineTo(-25, -8);    // Leme Esquerdo
            ctx.lineTo(-20, 0);     // Centro Popa
            ctx.lineTo(-25, 8);     // Leme Direito
            ctx.lineTo(-10, 5);     // Corpo
            ctx.lineTo(-15, 15);    // Asa Direita
            ctx.closePath();
            ctx.fill();
            ctx.stroke();

            // Cockpit / Núcleo de Energia SROS (Ciano Brilhante)
            ctx.fillStyle = '#00ffcc';
            ctx.beginPath();
            ctx.arc(5, 0, 4, 0, Math.PI * 2);
            ctx.fill();

            // Anéis Magnéticos MHD nas Asas
            ctx.strokeStyle = '#ff00b4';
            ctx.beginPath();
            ctx.arc(-10, -10, 3, 0, Math.PI * 2);
            ctx.arc(-10, 10, 3, 0, Math.PI * 2);
            ctx.stroke();

            ctx.restore();
        }

        // --- VÓRTICE DE ENTRADA / SAÍDA DO BURACO DE MINHOCA ---
        function drawWormholePortal(centerX, centerY, radius, intensity, isExit = false) {
            ctx.save();
            ctx.translate(centerX, centerY);

            let mainColor = isExit ? 'rgba(255, 200, 50, ' : 'rgba(0, 255, 204, ';
            let secColor  = isExit ? 'rgba(0, 255, 150, ' : 'rgba(255, 0, 180, ';

            // Anel de Lente Gravitacional (Distorção do Espaço-Tempo)
            for (let i = 5; i > 0; i--) {
                ctx.strokeStyle = mainColor + (0.15 * intensity) + ')';
                ctx.lineWidth = 2 * i;
                ctx.beginPath();
                ctx.arc(0, 0, radius + i * 12, 0, Math.PI * 2);
                ctx.stroke();
            }

            // Disco de Acresção em Espiral
            ctx.rotate(rotationAngle * (isExit ? -2 : 2));
            for (let a = 0; a < Math.PI * 2; a += Math.PI / 6) {
                let grad = ctx.createRadialGradient(0, 0, 10, 0, 0, radius);
                grad.addColorStop(0, mainColor + intensity + ')');
                grad.addColorStop(0.6, secColor + (intensity * 0.6) + ')');
                grad.addColorStop(1, 'rgba(0,0,0,0)');

                ctx.fillStyle = grad;
                ctx.beginPath();
                ctx.moveTo(0, 0);
                ctx.arc(0, 0, radius, a, a + 0.3);
                ctx.closePath();
                ctx.fill();
            }

            // Horizonte de Eventos Negro Absoluto no Centro
            ctx.fillStyle = '#000000';
            ctx.beginPath();
            ctx.arc(0, 0, radius * 0.45, 0, Math.PI * 2);
            ctx.fill();
            ctx.strokeStyle = isExit ? '#ffffff' : '#00ffcc';
            ctx.lineWidth = 2;
            ctx.stroke();

            ctx.restore();
        }

        // --- LAÇO PRINCIPAL DE RENDERIZAÇÃO ---
        function render() {
            // Limpa o canvas com rastro dinâmico dependendo da fase
            if (currentStage === 2) {
                ctx.fillStyle = 'rgba(0, 0, 8, 0.18)'; // Rastro hiperespacial
            } else {
                ctx.fillStyle = 'rgba(1, 4, 12, 0.4)';
            }
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            const cx = canvas.width / 2;
            const cy = canvas.height / 2;

            rotationAngle += 0.01;
            stageProgress += 0.003;

            // Transição Automática de Estágios
            if (stageProgress >= 1.0) {
                stageProgress = 0;
                if (autoPlay) {
                    currentStage = (currentStage + 1) % 5;
                    updateUI();
                }
            }

            // --- RENDERIZAÇÃO BASEADA NO ESTÁGIO SELECIONADO ---
            switch (currentStage) {

                // =========================================================
                // ESTÁGIO 1: ORIGEM - TERRA PRESENTE & DEEP SPACE
                // =========================================================
                case 0:
                    // Renderiza estrelas fixas
                    stars.forEach(s => {
                        ctx.fillStyle = s.color;
                        ctx.fillRect(cx + s.x * 0.8, cy + s.y * 0.8, s.size, s.size);
                    });

                    // Terra do Presente posicionada à esquerda
                    drawPresentEarth(cx - canvas.width * 0.22, cy, 140);

                    // Nave navegando suavemente em órbita
                    ship.x = cx - canvas.width * 0.05 + Math.sin(Date.now() * 0.001) * 30;
                    ship.y = cy + Math.cos(Date.now() * 0.001) * 20;
                    drawShip(ship.x, ship.y, 1.1, 0.1);
                    break;

                // =========================================================
                // ESTÁGIO 2: ABERTURA DO PORTAL DE ENTRADA & INJEÇÃO
                // =========================================================
                case 1:
                    stars.forEach(s => {
                        ctx.fillStyle = s.color;
                        ctx.fillRect(cx + s.x * 0.8, cy + s.y * 0.8, s.size, s.size);
                    });

                    drawPresentEarth(cx - canvas.width * 0.28, cy, 110);

                    // Vórtice do Buraco de Minhoca se abrindo no centro-direito
                    let portalRadius = Math.min(180, stageProgress * 220);
                    let portalX = cx + canvas.width * 0.18;
                    drawWormholePortal(portalX, cy, portalRadius, Math.min(1, stageProgress * 1.5), false);

                    // Nave acelerando em direção ao centro do portal
                    ship.x = (cx - canvas.width * 0.1) + (portalX - (cx - canvas.width * 0.1)) * stageProgress;
                    ship.y = cy;
                    let shipScale = Math.max(0.2, 1.2 * (1 - stageProgress * 0.8));
                    let angleToPortal = Math.atan2(cy - ship.y, portalX - ship.x);

                    drawShip(ship.x, ship.y, shipScale, angleToPortal);
                    break;

                // =========================================================
                // ESTÁGIO 3: TRÂNSITO PELO TÚNEL DE EINSTEIN-ROSEN (HIPERESPAÇO)
                // =========================================================
                case 2:
                    // Renderiza o túnel tridimensional de partículas fluindo
                    particles.sort((a, b) => b.z - a.z);

                    for (let i = 0; i < particleCount; i++) {
                        let p = particles[i];
                        p.z -= 12; // Velocidade hiperespacial acelerada

                        if (p.z <= 10) {
                            p.z = 800;
                            let angle = Math.random() * Math.PI * 2;
                            let radius = 60 + Math.random() * 220;
                            p.x = Math.cos(angle) * radius;
                            p.y = Math.sin(angle) * radius;
                        }

                        let factor = 260 / p.z;
                        let cosR = Math.cos(rotationAngle);
                        let sinR = Math.sin(rotationAngle);
                        let rotX = p.x * cosR - p.y * sinR;
                        let rotY = p.x * sinR + p.y * cosR;

                        let sx = cx + rotX * factor;
                        let sy = cy + rotY * factor;
                        let sz = p.size * factor;

                        if (sx > 0 && sx < canvas.width && sy > 0 && sy < canvas.height) {
                            ctx.fillStyle = (p.type === 'sros') ? 
                                `rgba(0, 255, 238, ${1 - p.z/800})` : 
                                `rgba(255, 0, 180, ${1 - p.z/800})`;
                            ctx.beginPath();
                            ctx.arc(sx, sy, sz, 0, Math.PI * 2);
                            ctx.fill();
                        }
                    }

                    // Nave posicionada no centro de estabilização MHD do túnel
                    let shakeX = (Math.random() - 0.5) * 3;
                    let shakeY = (Math.random() - 0.5) * 3;
                    drawShip(cx + shakeX, cy + shakeY, 0.9, 0, true);
                    break;

                // =========================================================
                // ESTÁGIO 4: ABERTURA DO BURACO DE MINHOCA DE SAÍDA (RUPTURA)
                // =========================================================
                case 3:
                    // Fundo transicionando do túnel para o novo espaço
                    let flareSize = stageProgress * canvas.width * 1.2;

                    // Exibe estrelas do futuro surgindo ao fundo
                    stars.forEach(s => {
                        ctx.fillStyle = s.color;
                        ctx.fillRect(cx + s.x, cy + s.y, s.size * 1.2, s.size * 1.2);
                    });

                    // Vórtice de Saída Erompendo em Luz Branca e Dourada
                    drawWormholePortal(cx, cy, Math.min(250, stageProgress * 300), 1.0, true);

                    // Luz Intensa de Ruptura Espaço-Temporal
                    let brightGrad = ctx.createRadialGradient(cx, cy, 10, cx, cy, Math.max(15, flareSize));
                    brightGrad.addColorStop(0, 'rgba(255, 255, 255, ' + Math.min(1, stageProgress * 1.8) + ')');
                    brightGrad.addColorStop(0.5, 'rgba(0, 255, 204, ' + (0.5 * stageProgress) + ')');
                    brightGrad.addColorStop(1, 'rgba(0,0,0,0)');
                    ctx.fillStyle = brightGrad;
                    ctx.beginPath(); ctx.arc(cx, cy, Math.max(15, flareSize), 0, Math.PI * 2); ctx.fill();

                    // Nave emergindo do centro da ruptura
                    let exitScale = Math.min(1.2, 0.1 + stageProgress * 1.3);
                    drawShip(cx, cy, exitScale, 0);
                    break;

                // =========================================================
                // ESTÁGIO 5: DESTINO - TERRA DO FUTURO (SÉCULO XXXI)
                // =========================================================
                case 4:
                    // Universo Futurista (Estrelas Brilhantes + Nebulosa Violeta)
                    let nebGrad = ctx.createRadialGradient(cx + 200, cy - 100, 50, cx + 200, cy - 100, 500);
                    nebGrad.addColorStop(0, 'rgba(150, 0, 255, 0.15)');
                    nebGrad.addColorStop(1, 'rgba(0,0,0,0)');
                    ctx.fillStyle = nebGrad;
                    ctx.fillRect(0, 0, canvas.width, canvas.height);

                    stars.forEach(s => {
                        ctx.fillStyle = s.color;
                        ctx.fillRect(cx + s.x * 0.9, cy + s.y * 0.9, s.size * 1.3, s.size * 1.3);
                    });

                    // Terra do Futuro posicionada à direita
                    drawFutureEarth(cx + canvas.width * 0.22, cy, 150);

                    // Nave concluindo a travessia e entrando na órbita do futuro
                    ship.x = cx - canvas.width * 0.12 + (stageProgress * 80);
                    ship.y = cy - 30 + Math.sin(stageProgress * Math.PI) * 20;
                    drawShip(ship.x, ship.y, 1.2, -0.05);
                    break;
            }

            // Atualiza telemetria contínua do HUD
            updateDynamicTelemetry();

            requestAnimationFrame(render);
        }

        // --- ATUALIZAÇÃO DE INTERFACE E TELEMETRIA ---
        function updateUI() {
            // Atualiza destaque dos botões
            for (let i = 0; i < 5; i++) {
                let btn = document.getElementById(`btn-${i}`);
                if (i === currentStage) btn.classList.add('active');
                else btn.classList.remove('active');
            }

            const phaseVal = document.getElementById('phase-val');
            const timeVal  = document.getElementById('time-val');
            const descVal  = document.getElementById('stage-desc');

            switch (currentStage) {
                case 0:
                    phaseVal.innerText = "1. ORIGEM (PRESENTE)";
                    phaseVal.style.color = "#00ff55";
                    timeVal.innerText = "2026.09.26 (ERA ATUAL)";
                    descVal.innerText = "Nave alinhada em órbita terrestre. Sistema MHD carregando singularidade.";
                    break;
                case 1:
                    phaseVal.innerText = "2. ABERTURA (ENTRADA)";
                    phaseVal.style.color = "#00e5ff";
                    timeVal.innerText = "2026.09.26 -> COLAPSO";
                    descVal.innerText = "MHD+SAMS+SROS rasgam a malha espaço-tempo. Entrada na garganta de Einstein-Rosen.";
                    break;
                case 2:
                    phaseVal.innerText = "3. HIPERESPAÇO (TÚNEL)";
                    phaseVal.style.color = "#ff00b4";
                    timeVal.innerText = "DESLOCAMENTO TEMPORAL";
                    descVal.innerText = "Trânsito em velocidade superlumínica através da ponte quântica.";
                    break;
                case 3:
                    phaseVal.innerText = "4. RUPTURA (SAÍDA)";
                    phaseVal.style.color = "#ffaa00";
                    timeVal.innerText = "SÉCULO XXXI (EMERGÊNCIA)";
                    descVal.innerText = "Colapso do horizonte de eventos de saída. Descompressão métrica bem-sucedida.";
                    break;
                case 4:
                    phaseVal.innerText = "5. DESTINO (TERRA DO FUTURO)";
                    phaseVal.style.color = "#00ffcc";
                    timeVal.innerText = "3026.09.26 (+1000 ANOS)";
                    descVal.innerText = "Chegada à Terra Pós-Singularidade. Escudo orbital e megagrid energéticos ativos.";
                    break;
            }
        }

        function updateDynamicTelemetry() {
            const live = Date.now() * 0.003;
            document.getElementById('mhd-val').innerText = (143.0 + Math.sin(live) * 1.2).toFixed(2) + " Eq/Plck";
            document.getElementById('sros-val').innerText = (99.98 + Math.cos(live) * 0.02).toFixed(3) + "% SINCRONIZADO";
        }

        function setStage(stageIndex) {
            currentStage = stageIndex;
            stageProgress = 0;
            updateUI();
        }

        function toggleAutoPlay() {
            autoPlay = !autoPlay;
            const btn = document.getElementById('btn-auto');
            if (autoPlay) {
                btn.innerText = "Modo: Sequencial";
                btn.style.borderColor = "#ffaa00";
                btn.style.color = "#ffaa00";
            } else {
                btn.innerText = "Modo: Manual";
                btn.style.borderColor = "#888";
                btn.style.color = "#888";
            }
        }

        // Inicia a execução da simulação
        render();
    </script>
</body>
</html>

