<!DOCTYPE html>
<html lang="tr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Lebap Drive: Sayat v2.0</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Rajdhani:wght@600&display=swap');
        body { margin: 0; overflow: hidden; background: #000; font-family: 'Rajdhani', sans-serif; touch-action: none; color: white; }
        
        #ui-overlay { position: absolute; width: 100%; height: 100%; background: rgba(0,0,0,0.95); display: flex; flex-direction: column; align-items: center; z-index: 100; overflow-y: auto; padding: 20px 0; }
        .stats-bar { position: absolute; top: 15px; left: 15px; font-size: 26px; color: #ffd700; z-index: 110; text-shadow: 0 0 10px rgba(255,215,0,0.5); pointer-events: none; }
        .car-card { width: 90%; background: #111; border: 2px solid #333; margin: 8px; padding: 15px; border-radius: 12px; display: flex; justify-content: space-between; align-items: center; cursor: pointer; transition: 0.3s; pointer-events: auto; }
        .car-card.selected { border-color: #00f0ff; box-shadow: 0 0 15px #00f0ff; background: #1a1a1a; }
        
        .mod-box { width: 85%; background: #151515; padding: 15px; border-radius: 12px; border: 1px solid #444; margin-top: 15px; }
        #game-hud { position: absolute; width: 100%; height: 100%; pointer-events: none; display: none; }
        .speed-panel { position: absolute; bottom: 170px; right: 25px; text-align: right; }
        .drift-score { position: absolute; top: 70px; width: 100%; text-align: center; color: #ffae00; font-size: 35px; display: none; text-shadow: 0 0 15px #ffae00; }

        .controls { position: absolute; bottom: 30px; width: 100%; display: flex; justify-content: space-between; padding: 0 20px; box-sizing: border-box; }
        .btn { width: 75px; height: 75px; background: rgba(255,255,255,0.08); border: 2px solid #00f0ff; border-radius: 50%; display: flex; align-items: center; justify-content: center; pointer-events: auto; font-size: 16px; }
        .pedals { display: flex; flex-direction: column; gap: 15px; align-items: flex-end; }
        
        #start-btn { margin-top: 25px; padding: 18px 60px; background: #00f0ff; color: #000; border: none; border-radius: 35px; font-weight: bold; font-size: 22px; cursor: pointer; pointer-events: auto; }
    </style>
</head>
<body>

<div class="stats-bar">PARA: $<span id="money-val">0</span></div>

<div id="ui-overlay">
    <h1 style="color: #00f0ff; letter-spacing: 3px;">LEBAP DRIVE: GARAJ</h1>
    <div id="car-list" style="width: 100%; display: flex; flex-direction: column; align-items: center;"></div>
    
    <div class="mod-box">
        <h3 style="margin-top:0">MODİFİYE</h3>
        <label>RENK: </label><input type="color" id="carColor" value="#ffffff">
        <label style="margin-left:20px">BASIKLIK: </label><input type="range" id="suspHeight" min="0.2" max="0.8" step="0.05" value="0.4">
    </div>

    <button id="start-btn" onclick="startGame()">SÜRÜŞE BAŞLA</button>
</div>

<div id="game-hud">
    <div id="drift-ui" class="drift-score">DRIFT: <span id="drift-val">0</span></div>
    <div class="speed-panel">
        <div id="speedo" style="font-size: 65px; color: #00f0ff; font-weight: 900;">0</div>
        <div id="car-name-display" style="font-size: 20px; letter-spacing: 2px;">TOYOTA</div>
    </div>
    <div class="controls">
        <div style="display:flex; gap:15px;"><div class="btn" id="lBtn">SOL</div><div class="btn" id="rBtn">SAĞ</div></div>
        <div class="pedals">
            <div class="btn" id="dBtn" style="width:110px; height:45px; border-radius:10px; border-color:orange; color:orange;">DRIFT</div>
            <div style="display:flex; gap:15px;">
                <div class="btn" id="bBtn" style="background:rgba(255,0,0,0.15); border-color:red">FREN</div>
                <div class="btn" id="gBtn" style="height:120px; background:rgba(0,255,0,0.15); width:85px; border-radius:20px; border-color:lime">GAZ</div>
            </div>
        </div>
    </div>
</div>

<script>
const CARS = [
    { name: "Toyota Camry 2007", price: 0, power: 0.7, drift: 0.1 },
    { name: "Toyota Avalon 2012", price: 3000, power: 0.75, drift: 0.08 },
    { name: "BMW M5 E60", price: 10000, power: 1.0, drift: 0.2 },
    { name: "Mercedes S600", price: 12000, power: 0.9, drift: 0.06 },
    { name: "Lada 2107", price: 500, power: 0.45, drift: 0.18 }
];

let money = parseInt(localStorage.getItem('lebap_money')) || 0;
let selectedIdx = 0, scene, camera, renderer, car, speed = 0, rot = 0, driftScore = 0;
let inputs = { f:0, b:0, l:0, r:0, d:0 };

const carListDiv = document.getElementById('car-list');
CARS.forEach((c, i) => {
    const card = document.createElement('div');
    card.className = `car-card ${i === 0 ? 'selected' : ''}`;
    card.innerHTML = `<span>${c.name}</span> <span>${c.price === 0 ? 'AÇIK' : '$'+c.price}</span>`;
    card.onclick = () => {
        document.querySelectorAll('.car-card').forEach(el => el.classList.remove('selected'));
        card.classList.add('selected');
        selectedIdx = i;
    };
    carListDiv.appendChild(card);
});
document.getElementById('money-val').innerText = money;

function startGame() {
    document.getElementById('ui-overlay').style.display = 'none';
    document.getElementById('game-hud').style.display = 'block';
    document.getElementById('car-name-display').innerText = CARS[selectedIdx].name.toUpperCase();
    init();
}

function init() {
    scene = new THREE.Scene();
    scene.background = new THREE.Color(0x020202);
    scene.fog = new THREE.Fog(0x020202, 15, 120);
    
    camera = new THREE.PerspectiveCamera(75, window.innerWidth/window.innerHeight, 0.1, 1000);
    renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    document.body.appendChild(renderer.domElement);

    const bodyGeo = new THREE.BoxGeometry(2, 0.8, 4.5);
    const bodyMat = new THREE.MeshPhongMaterial({ color: document.getElementById('carColor').value });
    car = new THREE.Mesh(bodyGeo, bodyMat);
    car.position.y = parseFloat(document.getElementById('suspHeight').value);
    scene.add(car);

    const roadGeo = new THREE.PlaneGeometry(25, 20000);
    const roadMat = new THREE.MeshStandardMaterial({color: 0x151515});
    const road = new THREE.Mesh(roadGeo, roadMat);
    road.rotation.x = -Math.PI/2;
    scene.add(road);

    scene.add(new THREE.AmbientLight(0xffffff, 0.4));
    const sun = new THREE.DirectionalLight(0xffffff, 0.5);
    sun.position.set(0, 10, 0);
    scene.add(sun);

    setupEvents();
    animate();
}

function setupEvents() {
    const bind = (id, k) => {
        const el = document.getElementById(id);
        el.addEventListener('touchstart', (e) => { e.preventDefault(); inputs[k] = 1; }, {passive: false});
        el.addEventListener('touchend', (e) => { e.preventDefault(); inputs[k] = 0; }, {passive: false});
    };
    bind('gBtn','f'); bind('bBtn','b'); bind('lBtn','l'); bind('rBtn','r'); bind('dBtn','d');
}

function animate() {
    requestAnimationFrame(animate);
    const stats = CARS[selectedIdx];
    
    if(inputs.f) speed += stats.power * 0.18;
    if(inputs.b) speed -= 0.5;
    speed *= 0.965;

    let turnPower = inputs.d ? stats.drift * 2.8 : 0.045;
    if(inputs.l) rot += turnPower * (speed * 0.12);
    if(inputs.r) rot -= turnPower * (speed * 0.12);

    if(inputs.d && Math.abs(speed) > 1.5) {
        driftScore += Math.floor(Math.abs(speed) * 3);
        document.getElementById('drift-ui').style.display = 'block';
        document.getElementById('drift-val').innerText = driftScore;
    } else if (driftScore > 0) {
        money += Math.floor(driftScore / 8);
        localStorage.setItem('lebap_money', money);
        document.getElementById('money-val').innerText = money;
        driftScore = 0;
        setTimeout(() => document.getElementById('drift-ui').style.display = 'none', 800);
    }

    car.position.z -= Math.cos(rot) * speed;
    car.position.x -= Math.sin(rot) * speed;
    car.rotation.y = rot;

    camera.position.set(car.position.x + Math.sin(rot) * 9, 3.5, car.position.z + Math.cos(rot) * 9);
    camera.lookAt(car.position);

    document.getElementById('speedo').innerText = Math.round(Math.abs(speed * 22));
    renderer.render(scene, camera);
}

window.onresize = () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
};
</script>
</body>
</html>

