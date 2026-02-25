# project
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>pink flowers · nadiah</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }
        body {
            background: linear-gradient(145deg, #2a1a2e 0%, #5a3f5a 100%);
            min-height: 100vh;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            font-family: 'Segoe UI', Roboto, sans-serif;
        }
        .canvas-wrapper {
            background: rgba(40, 20, 40, 0.4);
            backdrop-filter: blur(4px);
            border-radius: 3rem 3rem 2rem 2rem;
            padding: 20px 20px 15px 20px;
            box-shadow: 0 25px 40px -10px #180018;
            border: 1px solid #ffb7c5;
        }
        canvas {
            display: block;
            width: 900px;
            height: 450px;
            border-radius: 32px;
            background: radial-gradient(circle at 30% 40%, #ffd9e6, #bf8f9f);
            box-shadow: inset 0 0 30px #ffeef2, 0 10px 18px #2f1422;
            cursor: default;
        }
        .input-area {
            margin-top: 28px;
            display: flex;
            gap: 15px;
            flex-wrap: wrap;
            justify-content: center;
            align-items: center;
        }
        .name-field {
            background: #ffdae6;
            border: none;
            border-radius: 60px;
            padding: 16px 28px;
            font-size: 1.6rem;
            font-weight: 500;
            color: #621f44;
            width: 340px;
            text-align: center;
            letter-spacing: 3px;
            box-shadow: 0 8px 0 #994b73, 0 12px 25px #1f0e1f;
            transition: 0.1s linear;
            font-family: 'Trebuchet MS', cursive, sans-serif;
        }
        .name-field:focus {
            outline: 4px solid #ffa5c3;
            background: #fff0f5;
            transform: translateY(4px);
            box-shadow: 0 4px 0 #994b73, 0 12px 25px #1f0e1f;
        }
        button {
            background: #ff80a5;
            border: none;
            border-radius: 50px;
            padding: 16px 36px;
            font-size: 1.7rem;
            font-weight: bold;
            color: #2d1122;
            box-shadow: 0 7px 0 #b34062, 0 10px 25px #2a0f1f;
            cursor: pointer;
            transition: 0.08s linear;
            display: inline-flex;
            align-items: center;
            gap: 12px;
            letter-spacing: 2px;
        }
        button:active {
            transform: translateY(6px);
            box-shadow: 0 2px 0 #b34062, 0 8px 20px #2a0f1f;
        }
        .hint {
            color: #ffe3f0;
            font-size: 1.3rem;
            margin-top: 20px;
            text-shadow: 2px 2px 0 #462033;
            background: #b26b8c80;
            padding: 10px 40px;
            border-radius: 60px;
            backdrop-filter: blur(3px);
            border: 1px solid #ffb6d1;
        }
        .hint span {
            background: #ffc0d5;
            color: #4d1135;
            padding: 5px 22px;
            border-radius: 50px;
            margin: 0 8px;
            font-weight: 700;
        }
    </style>
</head>
<body>
<div class="canvas-wrapper">
    <canvas id="flowerCanvas" width="900" height="450"></canvas>
</div>

<div class="input-area">
    <input type="text" id="nameInput" class="name-field" placeholder="pink name" value="NADIAH">
    <button id="bloomBtn">🌸 PINK BLOOM 🌸</button>
</div>
<div class="hint">
    <span>✨ nadiah in moving pink flowers ✨</span> — each petal dances
</div>

<script>
(function() {
    const canvas = document.getElementById('flowerCanvas');
    const ctx = canvas.getContext('2d');
    const nameInput = document.getElementById('nameInput');
    const bloomBtn = document.getElementById('bloomBtn');

    // ----- PINK & SOFT parameters -----
    const FLOWER_COUNT = 58;            // extra flowers for a lush name
    const BASE_RADIUS = 13;              // base size
    const STEM_HEIGHT = 22;              
    const PETAL_COUNT = 8;               

    const ANIM_SPEED = 0.005;             // slow drift

    let flowers = [];
    let nameCache = "NADIAH";             
    let time = 0;                        

    // ---------- Flower class (pink edition) ----------
    class Flower {
        constructor(index, total) {
            this.index = index;

            // target position (set later by name grid)
            this.targetX = 0;
            this.targetY = 0;

            // current floating position
            this.currentX = 0;
            this.currentY = 0;

            // drifting personality
            this.phaseX = Math.random() * 100;
            this.phaseY = Math.random() * 100;
            this.wobbleSpeed = 0.15 + Math.random() * 0.3;
            this.wobbleAmount = 3.5 + Math.random() * 6;    

            // PINK PETAL palette – all shades of pink, rose, blush
            const pinkHue = 320 + (index * 9) % 40;          // 320–360 (pink to rose)
            this.petalColor = `hsl(${pinkHue}, 80%, 73%)`;    // soft pink
            this.centerColor = `hsl(${40 + (index * 7) % 30}, 85%, 78%)`; // warm gold / light coral
            this.stemColor = `hsl(${90 + (index * 3) % 25}, 55%, 60%)`;   // fresh green

            // random start position (scattered)
            this.currentX = 150 + Math.random() * 600;
            this.currentY = 100 + Math.random() * 250;
        }

        // update floating motion (drift around target)
        update(deltaTime) {
            const t = deltaTime * this.wobbleSpeed + this.index * 1.5;
            const dx = Math.sin(t * 1.2 + this.phaseX) * this.wobbleAmount
                     + Math.cos(t * 0.8 + this.phaseY) * (this.wobbleAmount * 0.5);
            const dy = Math.cos(t * 1.0 + this.phaseY) * this.wobbleAmount
                     + Math.sin(t * 1.1 + this.phaseX) * (this.wobbleAmount * 0.6);

            const targetDriftX = this.targetX + dx;
            const targetDriftY = this.targetY + dy;

            // smooth follow (gentle lag)
            this.currentX += (targetDriftX - this.currentX) * 0.04;
            this.currentY += (targetDriftY - this.currentY) * 0.04;
        }

        // draw a cute pink flower
        draw(ctx) {
            const x = this.currentX;
            const y = this.currentY;

            // stem
            ctx.beginPath();
            ctx.moveTo(x, y);
            ctx.lineTo(x - 3, y + STEM_HEIGHT);
            ctx.lineTo(x + 4, y + STEM_HEIGHT + 7);
            ctx.strokeStyle = this.stemColor;
            ctx.lineWidth = 4.2;
            ctx.lineCap = 'round';
            ctx.stroke();

            // little leaf
            ctx.beginPath();
            ctx.ellipse(x - 5, y + STEM_HEIGHT - 1, 4, 2, 0.2, 0, Math.PI * 2);
            ctx.fillStyle = `hsl(${95 + (this.index * 4) % 30}, 60%, 65%)`;
            ctx.fill();

            // petals (all pink)
            for (let i = 0; i < PETAL_COUNT; i++) {
                const angle = (i / PETAL_COUNT) * Math.PI * 2 + time * 0.08; 
                const petalLength = BASE_RADIUS * 1.2;
                const petalWidth = BASE_RADIUS * 0.7;

                const offsetX = Math.cos(angle) * petalLength;
                const offsetY = Math.sin(angle) * petalLength;

                ctx.save();
                ctx.translate(x + offsetX * 0.6, y + offsetY * 0.6);
                ctx.rotate(angle);
                ctx.scale(1.0, 0.85 + Math.sin(angle + time) * 0.1);
                ctx.beginPath();
                ctx.ellipse(0, 0, petalWidth, petalWidth * 0.8, 0, 0, Math.PI * 2);
                ctx.fillStyle = this.petalColor;
                ctx.shadowColor = 'rgba(90, 30, 50, 0.4)';
                ctx.shadowBlur = 10;
                ctx.shadowOffsetY = 2;
                ctx.fill();
                ctx.restore();
            }

            // center (pinkish-gold)
            ctx.shadowBlur = 14;
            ctx.beginPath();
            ctx.arc(x, y, BASE_RADIUS * 0.6, 0, 2 * Math.PI);
            ctx.fillStyle = this.centerColor;
            ctx.fill();
            // tiny highlight
            ctx.shadowBlur = 6;
            ctx.beginPath();
            ctx.arc(x - 2, y - 2, BASE_RADIUS * 0.15, 0, 2 * Math.PI);
            ctx.fillStyle = '#fff8f0';
            ctx.fill();

            ctx.shadowBlur = 0;
            ctx.shadowOffsetY = 0;
        }
    }

    // ---------- name to grid (pixel font) ----------
    function getGridForChar(c) {
        c = c.toUpperCase();
        const grids = {
            'A': [ '  ██  ', ' ████ ', '██  ██', '██████', '██  ██' ],
            'B': [ '████ ', '██ ██', '████ ', '██ ██', '████ ' ],
            'C': [ ' ███ ', '██   ', '██   ', '██   ', ' ███ ' ],
            'D': [ '████ ', '██ ██', '██ ██', '██ ██', '████ ' ],
            'E': [ '█████', '██   ', '████ ', '██   ', '█████'],
            'F': [ '█████', '██   ', '████ ', '██   ', '██   '],
            'G': [ ' ███ ', '██   ', '██ ██', '██ ██', ' ███ '],
            'H': [ '██ ██', '██ ██', '█████', '██ ██', '██ ██'],
            'I': [ '███', ' █ ', ' █ ', ' █ ', '███'],
            'J': [ '  ███', '   █ ', '   █ ', '██ █ ', ' ██  '],
            'K': [ '██ █', '██ █', '███ ', '██ █', '██ █'],
            'L': [ '██  ', '██  ', '██  ', '██  ', '████'],
            'M': [ '██  ██', '██████', '██  ██', '██  ██', '██  ██'],
            'N': [ '██  ██', '█████ ', '██ ██ ', '██  ██', '██  ██'],
            'O': [ ' ███ ', '██ ██', '██ ██', '██ ██', ' ███ '],
            'P': [ '████ ', '██ ██', '████ ', '██   ', '██   '],
            'Q': [ ' ███ ', '██ ██', '██ ██', '██ ██', ' ████'],
            'R': [ '████ ', '██ ██', '████ ', '██ █ ', '██ ██'],
            'S': [ ' ███ ', '██   ', ' ███ ', '   ██', '████ '],
            'T': [ '█████', '  █  ', '  █  ', '  █  ', '  █  '],
            'U': [ '██ ██', '██ ██', '██ ██', '██ ██', ' ███ '],
            'V': [ '██ ██', '██ ██', '██ ██', ' ███ ', '  █  '],
            'W': [ '██ ██', '██ ██', '██ ██', '█████', '██ ██'],
            'X': [ '██ ██', ' ███ ', '  █  ', ' ███ ', '██ ██'],
            'Y': [ '██ ██', ' ███ ', '  █  ', '  █  ', '  █  '],
            'Z': [ '█████', '   █ ', '  █  ', ' █   ', '█████'],
            ' ': [ '     ', '     ', '     ', '     ', '     '],
            '0': [ ' ███ ', '██ ██', '██ ██', '██ ██', ' ███ '],
            '1': [ '  █  ', ' ██  ', '  █  ', '  █  ', '█████'],
            '2': [ ' ███ ', '██ ██', '   █ ', '  █  ', '█████'],
            '3': [ '████ ', '    █', ' ███ ', '    █', '████ '],
            '4': [ '██ █ ', '██ █ ', '█████', '   █ ', '   █ '],
            '5': [ '█████', '██   ', '████ ', '   ██', '████ '],
            '6': [ ' ███ ', '██   ', '████ ', '██ ██', ' ███ '],
            '7': [ '█████', '   █ ', '  █  ', ' █   ', '█    '],
            '8': [ ' ███ ', '██ ██', ' ███ ', '██ ██', ' ███ '],
            '9': [ ' ███ ', '██ ██', ' ████', '   ██', ' ███ '],
            '?': [ ' ███ ', '██ ██', '   █ ', '  █  ', '     '],
            '!': [ '  █  ', '  █  ', '  █  ', '     ', '  █  '],
            '.': [ '     ', '     ', '     ', '     ', '  █  '],
        };
        return grids[c] || grids['?'];
    }

    // ----- assign target positions based on name -----
    function arrangeName(name) {
        if (!name) name = "NADIAH";
        const upper = name.toUpperCase().slice(0, 16); // limit length

        const startX = 140;
        const startY = 180;         // vertical center
        const charSpacing = 52;      // space between characters
        const dotSize = 13;           // grid point spacing

        let positions = [];           // list of {x, y} target spots

        // loop through each character
        for (let cIdx = 0; cIdx < upper.length; cIdx++) {
            const ch = upper[cIdx];
            const grid = getGridForChar(ch);
            const rows = 5;
            const cols = grid[0].length;

            for (let row = 0; row < rows; row++) {
                for (let col = 0; col < cols; col++) {
                    if (grid[row][col] === '█') {
                        let x = startX + cIdx * charSpacing + col * dotSize;
                        let y = startY + row * dotSize - 25; // shift up a bit
                        positions.push({ x, y });
                    }
                }
            }
            // small extra space after letter
        }

        // if not enough positions, repeat last ones, but we have many flowers
        // assign each flower to a position (cyclically if needed)
        for (let i = 0; i < flowers.length; i++) {
            const pos = positions[i % positions.length];
            // add tiny variation so they're not perfectly stacked (more natural)
            const varyX = (Math.sin(i * 2.3) * 2) | 0;
            const varyY = (Math.cos(i * 1.7) * 2) | 0;
            flowers[i].targetX = pos.x + varyX;
            flowers[i].targetY = pos.y + varyY;
        }
    }

    // ----- initialize flowers -----
    function initFlowers() {
        flowers = [];
        for (let i = 0; i < FLOWER_COUNT; i++) {
            flowers.push(new Flower(i, FLOWER_COUNT));
        }
        arrangeName(nameCache);
    }

    // ----- animation loop -----
    function animate() {
        time += ANIM_SPEED;

        // update each flower
        for (let f of flowers) {
            f.update(time);
        }

        // draw everything
        ctx.clearRect(0, 0, canvas.width, canvas.height);

        // draw subtle grass / background dots (pinkish)
        ctx.fillStyle = '#ffb8c990';
        for (let i = 0; i < 10; i++) {
            ctx.beginPath();
            ctx.arc(70 + i * 80, 400, 8, 0, 2 * Math.PI);
            ctx.fillStyle = '#ffc0cb40';
            ctx.fill();
        }

        // draw flowers (stems underneath are drawn first, but flower.draw does stem anyway)
        for (let f of flowers) {
            f.draw(ctx);
        }

        // extra floating pink glow particles
        ctx.fillStyle = '#ffb3c6';
        for (let i = 0; i < 12; i++) {
            let x = 100 + (i * 70 + time * 20) % 800;
            let y = 80 + Math.sin(i + time * 2) * 15;
            ctx.beginPath();
            ctx.arc(x, y, 3, 0, 2 * Math.PI);
            ctx.fillStyle = '#ffb0c0';
            ctx.globalAlpha = 0.3;
            ctx.fill();
        }
        ctx.globalAlpha = 1.0;

        requestAnimationFrame(animate);
    }

    // ----- event: update name -----
    function updateName() {
        let newName = nameInput.value.trim();
        if (newName === "") newName = "NADIAH";
        nameCache = newName;
        arrangeName(nameCache);
    }

    // ----- button & input actions -----
    bloomBtn.addEventListener('click', () => {
        updateName();
    });

    nameInput.addEventListener('keypress', (e) => {
        if (e.key === 'Enter') {
            updateName();
        }
    });

    // ----- start everything -----
    initFlowers();
    animate();

    // optional: rearrange on window resize? no, canvas fixed.
    // but we can let it re-arrange with current name
})();
</script>
</body>
</html>
