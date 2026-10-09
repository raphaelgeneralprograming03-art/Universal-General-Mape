
<html lang="pt-br">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Cosmos Mapper v1.2 - Pro</title>
    <style>
        body, html {
            margin: 0;
            padding: 0;
            width: 100%;
            height: 100%;
            overflow: hidden;
            background-color: #020205;
            font-family: 'Segoe UI', Roboto, Helvetica, Arial, sans-serif;
        }
        canvas { display: block; }
        #ui-overlay {
            position: absolute;
            top: 20px;
            left: 20px;
            color: #00d4ff;
            pointer-events: none;
            background: rgba(0, 8, 20, 0.6);
            padding: 15px;
            border-radius: 4px;
            border-left: 3px solid #00d4ff;
            backdrop-filter: blur(5px);
        }
        .controls {
            position: absolute;
            bottom: 20px;
            right: 20px;
            color: #aaa;
            background: rgba(0, 0, 0, 0.8);
            padding: 15px;
            border-radius: 8px;
            border: 1px solid #1a3a4a;
            font-size: 11px;
            line-height: 1.6;
        }
        h1 { margin: 0; font-size: 16px; letter-spacing: 3px; color: #fff; }
        p { margin: 5px 0; font-family: monospace; }
        .highlight { color: #00d4ff; }
    </style>
</head>
<body>

    <div id="ui-overlay">
        <h1>COSMOS MAPPER</h1>
        <p id="stats">Objetos: 0 | Zoom: 1.00x</p>
        <p id="coords">X: 0, Y: 0</p>
    </div>

    <div class="controls">
        <b style="color: #00d4ff;">COMANDOS DO SISTEMA:</b><br>
        • Scroll: Zoom In/Out (Foco no mouse)<br>
        • Drag: Navegação Hiperespacial<br>
        • Escala: Parsecs/Simulação Ativa
    </div>

    <canvas id="universeCanvas"></canvas>

    <script>
        const canvas = document.getElementById('universeCanvas');
        const ctx = canvas.getContext('2d');
        const statsEl = document.getElementById('stats');
        const coordsEl = document.getElementById('coords');

        let width, height;
        const objects = [];
        const STAR_COUNT = 5000;
        const GALAXY_COUNT = 60;

        const camera = {
            x: 0,
            y: 0,
            zoom: 0.2, // Começa com um pouco de zoom out para ver o universo
            isDragging: false,
            lastMouseX: 0,
            lastMouseY: 0
        };

        class CelestialBody {
            constructor(isGalaxy = false) {
                // Área de dispersão massiva
                this.x = (Math.random() - 0.5) * 100000;
                this.y = (Math.random() - 0.5) * 100000;
                this.isGalaxy = isGalaxy;
                
                if (this.isGalaxy) {
                    this.size = Math.random() * 100 + 50;
                    this.hue = Math.floor(Math.random() * 60) + 220; // Azuis e Roxos
                    this.rotation = Math.random() * Math.PI * 2;
                } else {
                    this.size = Math.random() * 1.5 + 0.3;
                    this.hue = [200, 220, 40, 0, 30][Math.floor(Math.random() * 5)];
                }
                
                this.alpha = Math.random() * 0.5 + 0.3;
                // Pre-calcula as cores para evitar manipulação de string no loop
                this.baseColor = `hsla(${this.hue}, 80%, 80%, ${this.alpha})`;
                this.glowColor = `hsla(${this.hue}, 80%, 50%, 0.3)`;
            }

            draw() {
                // Cálculo de posição relativa à câmera
                const screenX = (this.x - camera.x) * camera.zoom + width / 2;
                const screenY = (this.y - camera.y) * camera.zoom + height / 2;
                const screenSize = this.size * camera.zoom;

                // Culling: Não processa se estiver fora da tela (margem de segurança de 100px)
                if (screenX + screenSize < -100 || screenX - screenSize > width + 100 ||
                    screenY + screenSize < -100 || screenY - screenSize > height + 100) return;

                if (this.isGalaxy) {
                    this.drawGalaxy(screenX, screenY, screenSize);
                } else {
                    this.drawStar(screenX, screenY, screenSize);
                }
            }

            drawStar(x, y, size) {
                ctx.fillStyle = this.baseColor;
                ctx.beginPath();
                ctx.arc(x, y, Math.max(0.1, size), 0, Math.PI * 2);
                ctx.fill();
                
                // Adiciona brilho apenas em zooms maiores para performance
                if (camera.zoom > 1.5 && size > 1) {
                    ctx.shadowBlur = size * 2;
                    ctx.shadowColor = this.baseColor;
                    ctx.beginPath();
                    ctx.arc(x, y, size * 0.5, 0, Math.PI * 2);
                    ctx.fill();
                    ctx.shadowBlur = 0; // Reseta para o próximo objeto
                }
            }

            drawGalaxy(x, y, size) {
                ctx.save();
                ctx.translate(x, y);
                ctx.rotate(this.rotation);
                
                // Efeito de "Lighter" faz as cores somarem, criando núcleos brilhantes
                ctx.globalCompositeOperation = 'lighter';
                
                const grad = ctx.createRadialGradient(0, 0, 0, 0, 0, size);
                grad.addColorStop(0, `hsla(${this.hue}, 100%, 90%, 0.8)`);
                grad.addColorStop(0.2, `hsla(${this.hue}, 80%, 50%, 0.4)`);
                grad.addColorStop(0.6, `hsla(${this.hue}, 80%, 20%, 0.1)`);
                grad.addColorStop(1, 'transparent');
                
                ctx.fillStyle = grad;
                ctx.beginPath();
                // Forma elíptica de galáxia
                ctx.ellipse(0, 0, size, size * 0.4, 0, 0, Math.PI * 2);
                ctx.fill();
                
                ctx.restore();
                ctx.globalCompositeOperation = 'source-over';
            }
        }

        function init() {
            resize();
            // Criar galáxias primeiro
            for (let i = 0; i < GALAXY_COUNT; i++) objects.push(new CelestialBody(true));
            // Criar estrelas
            for (let i = 0; i < STAR_COUNT; i++) objects.push(new CelestialBody(false));
            animate();
        }

        function resize() {
            width = canvas.width = window.innerWidth;
            height = canvas.height = window.innerHeight;
        }

        function updateUI() {
            coordsEl.innerHTML = `X: <span class="highlight">${Math.round(camera.x)}</span>, Y: <span class="highlight">${Math.round(camera.y)}</span>`;
            statsEl.innerHTML = `Objetos: <span class="highlight">${objects.length}</span> | Zoom: <span class="highlight">${camera.zoom.toFixed(3)}x</span>`;
        }

        window.addEventListener('resize', resize);

        // Lógica de Zoom Inteligente (Roda em direção ao mouse)
        window.addEventListener('wheel', (e) => {
            e.preventDefault();
            const zoomSpeed = 0.1;
            const delta = e.deltaY > 0 ? 1 - zoomSpeed : 1 + zoomSpeed;
            const newZoom = camera.zoom * delta;

            // Limites de Zoom
            if (newZoom > 0.001 && newZoom < 50) {
                // Calcula a posição do mouse no "mundo" antes do zoom
                const mouseWorldX = (e.clientX - width / 2) / camera.zoom + camera.x;
                const mouseWorldY = (e.clientY - height / 2) / camera.zoom + camera.y;

                camera.zoom = newZoom;

                // Reposiciona a câmera para que o mouse continue sobre o mesmo ponto do mundo
                camera.x = mouseWorldX - (e.clientX - width / 2) / camera.zoom;
                camera.y = mouseWorldY - (e.clientY - height / 2) / camera.zoom;
            }
        }, { passive: false });

        window.addEventListener('mousedown', (e) => {
            camera.isDragging = true;
            camera.lastMouseX = e.clientX;
            camera.lastMouseY = e.clientY;
        });

        window.addEventListener('mousemove', (e) => {
            if (camera.isDragging) {
                const dx = e.clientX - camera.lastMouseX;
                const dy = e.clientY - camera.lastMouseY;
                
                camera.x -= dx / camera.zoom;
                camera.y -= dy / camera.zoom;
                
                camera.lastMouseX = e.clientX;
                camera.lastMouseY = e.clientY;
            }
        });

        window.addEventListener('mouseup', () => camera.isDragging = false);

        function animate() {
            // Fundo espacial profundo
            ctx.fillStyle = '#020205';
            ctx.fillRect(0, 0, width, height);

            // Desenhar estrelas de fundo (Parallax estático simples)
            // Para um efeito real, poderíamos criar uma camada separada, 
            // mas o culling já ajuda na performance.
            
            for (let i = 0; i < objects.length; i++) {
                objects[i].draw();
            }

            updateUI();
            requestAnimationFrame(animate);
        }

        init();
    </script>
</body>
</html>
