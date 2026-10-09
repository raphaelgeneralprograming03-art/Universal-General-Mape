
<html lang="pt-br">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Simulador de Mapeamento do Universo</title>
    <style>
        body, html {
            margin: 0;
            padding: 0;
            width: 100%;
            height: 100%;
            overflow: hidden;
            background-color: #05050a;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
        }

        canvas {
            display: block;
        }

        #ui-overlay {
            position: absolute;
            top: 20px;
            left: 20px;
            color: #00d4ff;
            pointer-events: none;
            text-shadow: 0 0 5px rgba(0,212,255,0.5);
        }

        .controls {
            position: absolute;
            bottom: 20px;
            right: 20px;
            color: white;
            background: rgba(0, 0, 0, 0.7);
            padding: 15px;
            border-radius: 8px;
            border: 1px solid #00d4ff;
            font-size: 12px;
        }

        h1 { margin: 0; font-size: 18px; letter-spacing: 2px; }
        p { margin: 5px 0; opacity: 0.8; }
    </style>
</head>
<body>

    <div id="ui-overlay">
        <h1>COSMOS MAPPER v1.0</h1>
        <p id="stats">Objetos: 0 | Zoom: 1.0x</p>
        <p id="coords">X: 0, Y: 0</p>
    </div>

    <div class="controls">
        <b>CONTROLES:</b><br>
        • Scroll: Zoom In/Out<br>
        • Click & Drag: Mover pelo espaço<br>
        • Escala: Anos-luz (Simulado)
    </div>

    <canvas id="universeCanvas"></canvas>

    <script>
        const canvas = document.getElementById('universeCanvas');
        const ctx = canvas.getContext('2d');
        const statsEl = document.getElementById('stats');
        const coordsEl = document.getElementById('coords');

        // Configurações do Universo
        let width, height;
        const objects = [];
        const STAR_COUNT = 5000;
        const GALAXY_COUNT = 50;

        // Estado da Câmera
        const camera = {
            x: 0,
            y: 0,
            zoom: 1,
            isDragging: false,
            lastMouseX: 0,
            lastMouseY: 0
        };

        class CelestialBody {
            constructor(isGalaxy = false) {
                // Coordenadas mundiais massivas
                this.x = (Math.random() - 0.5) * 50000;
                this.y = (Math.random() - 0.5) * 50000;
                this.size = isGalaxy ? Math.random() * 50 + 20 : Math.random() * 2 + 0.5;
                this.color = this.generateColor(isGalaxy);
                this.isGalaxy = isGalaxy;
                this.rotation = Math.random() * Math.PI * 2;
            }

            generateColor(isGalaxy) {
                const hues = isGalaxy ? [200, 280, 320] : [0, 60, 200, 180];
                const h = hues[Math.floor(Math.random() * hues.length)];
                const s = 80;
                const l = isGalaxy ? 60 : 80;
                return `hsla(${h}, ${s}%, ${l}%, ${Math.random() * 0.5 + 0.5})`;
            }

            draw() {
                // Converter coordenadas do mundo para tela
                const screenX = (this.x - camera.x) * camera.zoom + width / 2;
                const screenY = (this.y - camera.y) * camera.zoom + height / 2;
                const screenSize = this.size * camera.zoom;

                // Culling (não desenhar se estiver fora da tela)
                if (screenX + screenSize < 0 || screenX - screenSize > width ||
                    screenY + screenSize < 0 || screenY - screenSize > height) return;

                if (this.isGalaxy) {
                    this.drawGalaxy(screenX, screenY, screenSize);
                } else {
                    this.drawStar(screenX, screenY, screenSize);
                }
            }

            drawStar(x, y, size) {
                ctx.fillStyle = this.color;
                ctx.beginPath();
                ctx.arc(x, y, size, 0, Math.PI * 2);
                ctx.fill();
                
                if (camera.zoom > 2) { // Brilho extra no zoom alto
                    ctx.shadowBlur = size * 2;
                    ctx.shadowColor = this.color;
                }
            }

            drawGalaxy(x, y, size) {
                ctx.save();
                ctx.translate(x, y);
                ctx.rotate(this.rotation);
                
                const gradient = ctx.createRadialGradient(0, 0, 0, 0, 0, size);
                gradient.addColorStop(0, this.color);
                gradient.addColorStop(1, 'transparent');
                
                ctx.fillStyle = gradient;
                ctx.globalAlpha = 0.4;
                
                // Forma elíptica da galáxia
                ctx.beginPath();
                ctx.ellipse(0, 0, size, size / 2, 0, 0, Math.PI * 2);
                ctx.fill();
                
                ctx.restore();
                ctx.globalAlpha = 1;
                ctx.shadowBlur = 0;
            }
        }

        function init() {
            resize();
            // Criar Estrelas
            for (let i = 0; i < STAR_COUNT; i++) objects.push(new CelestialBody(false));
            // Criar Galáxias
            for (let i = 0; i < GALAXY_COUNT; i++) objects.push(new CelestialBody(true));
            
            animate();
        }

        function resize() {
            width = canvas.width = window.innerWidth;
            height = canvas.height = window.innerHeight;
        }

        // Listeners de Input
        window.addEventListener('resize', resize);

        window.addEventListener('wheel', (e) => {
            const zoomSpeed = 0.1;
            const delta = e.deltaY > 0 ? 1 - zoomSpeed : 1 + zoomSpeed;
            
            // Zoom em direção ao mouse
            const mouseWorldX = (e.clientX - width / 2) / camera.zoom + camera.x;
            const mouseWorldY = (e.clientY - height / 2) / camera.zoom + camera.y;

            camera.zoom *= delta;
            camera.zoom = Math.max(0.01, Math.min(camera.zoom, 50)); // Limites de zoom

            camera.x = mouseWorldX - (e.clientX - width / 2) / camera.zoom;
            camera.y = mouseWorldY - (e.clientY - height / 2) / camera.zoom;
        });

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
            coordsEl.innerText = `X: ${Math.round(camera.x)}, Y: ${Math.round(camera.y)}`;
        });

        window.addEventListener('mouseup', () => camera.isDragging = false);

        function animate() {
            ctx.fillStyle = '#05050a';
            ctx.fillRect(0, 0, width, height);

            // Desenhar todos os objetos celestes
            objects.forEach(obj => obj.draw());

            // Atualizar UI
            statsEl.innerText = `Objetos: ${objects.length} | Zoom: ${camera.zoom.toFixed(2)}x`;

            requestAnimationFrame(animate);
        }

        init();
    </script>
</body>
</html>
