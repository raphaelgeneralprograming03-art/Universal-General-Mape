
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
        canvas { display: block; }
        #ui-overlay {
            position: absolute;
            top: 20px;
            left: 20px;
            color: #00d4ff;
            pointer-events: none;
            text-shadow: 0 0 5px rgba(0,212,255,0.5);
            background: rgba(0, 0, 0, 0.3);
            padding: 10px;
            border-radius: 5px;
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
        <h1>COSMOS MAPPER v1.1</h1>
        <p id="stats">Objetos: 0 | Zoom: 1.00x</p>
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

        let width, height;
        const objects = [];
        const STAR_COUNT = 4000;
        const GALAXY_COUNT = 40;

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
                // Área de dispersão aumentada para exploração
                this.x = (Math.random() - 0.5) * 80000;
                this.y = (Math.random() - 0.5) * 80000;
                this.size = isGalaxy ? Math.random() * 60 + 30 : Math.random() * 2 + 0.5;
                this.color = this.generateColor(isGalaxy);
                this.isGalaxy = isGalaxy;
                this.rotation = Math.random() * Math.PI * 2;
            }

            generateColor(isGalaxy) {
                const hues = isGalaxy ? [220, 260, 300] : [200, 220, 40, 0];
                const h = hues[Math.floor(Math.random() * hues.length)];
                const s = 70;
                const l = isGalaxy ? 50 : 90;
                return `hsla(${h}, ${s}%, ${l}%, ${Math.random() * 0.5 + 0.5})`;
            }

            draw() {
                const screenX = (this.x - camera.x) * camera.zoom + width / 2;
                const screenY = (this.y - camera.y) * camera.zoom + height / 2;
                const screenSize = this.size * camera.zoom;

                // Culling: Não desenha se estiver fora da visão
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
                
                // Brilho otimizado (apenas se o zoom for alto e o objeto for visível)
                if (camera.zoom > 3 && size > 1) {
                    ctx.beginPath();
                    ctx.arc(x, y, size * 2, 0, Math.PI * 2);
                    ctx.fillStyle = this.color.replace('%, 0.', '%, 0.1'); // Versão mais transparente
                    ctx.fill();
                }
            }

            drawGalaxy(x, y, size) {
                ctx.save();
                ctx.translate(x, y);
                ctx.rotate(this.rotation);
                
                const gradient = ctx.createRadialGradient(0, 0, 0, 0, 0, size);
                gradient.addColorStop(0, this.color);
                gradient.addColorStop(0.5, this.color.replace('1)', '0.2)'));
                gradient.addColorStop(1, 'transparent');
                
                ctx.fillStyle = gradient;
                ctx.beginPath();
                ctx.ellipse(0, 0, size, size * 0.4, 0, 0, Math.PI * 2);
                ctx.fill();
                
                ctx.restore();
            }
        }

        function init() {
            resize();
            // Criar primeiro galáxias (fundo) e depois estrelas (frente)
            for (let i = 0; i < GALAXY_COUNT; i++) objects.push(new CelestialBody(true));
            for (let i = 0; i < STAR_COUNT; i++) objects.push(new CelestialBody(false));
            animate();
        }

        function resize() {
            width = canvas.width = window.innerWidth;
            height = canvas.height = window.innerHeight;
        }

        function updateCoordsUI() {
            coordsEl.innerText = `X: ${Math.round(camera.x)}, Y: ${Math.round(camera.y)}`;
            statsEl.innerText = `Objetos: ${objects.length} | Zoom: ${camera.zoom.toFixed(2)}x`;
        }

        window.addEventListener('resize', resize);

        window.addEventListener('wheel', (e) => {
            e.preventDefault();
            const zoomSpeed = 0.15;
            const delta = e.deltaY > 0 ? 1 - zoomSpeed : 1 + zoomSpeed;
            
            const mouseWorldX = (e.clientX - width / 2) / camera.zoom + camera.x;
            const mouseWorldY = (e.clientY - height / 2) / camera.zoom + camera.y;

            const newZoom = camera.zoom * delta;
            if (newZoom > 0.005 && newZoom < 100) {
                camera.zoom = newZoom;
                camera.x = mouseWorldX - (e.clientX - width / 2) / camera.zoom;
                camera.y = mouseWorldY - (e.clientY - height / 2) / camera.zoom;
            }
            updateCoordsUI();
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
            updateCoordsUI();
        });

        window.addEventListener('mouseup', () => camera.isDragging = false);

        function animate() {
            // Limpa o canvas e redefine o alpha e sombras (importante!)
            ctx.fillStyle = '#05050a';
            ctx.fillRect(0, 0, width, height);
            ctx.shadowBlur = 0; 
            ctx.globalAlpha = 1;

            // Renderiza objetos
            for (const obj of objects) {
                obj.draw();
            }

            requestAnimationFrame(animate);
        }

        init();
    </script>
</body>
</html>
