const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

const coinCountEl = document.getElementById('coinCount');
const scoreEl = document.getElementById('score');
const livesEl = document.getElementById('lives');
const overlay = document.getElementById('overlay');
const startButton = document.getElementById('startButton');
const upgradeButtons = [...document.querySelectorAll('.upgrade-button')];
const jumpCostLabel = document.getElementById('jumpCost');
const speedCostLabel = document.getElementById('speedCost');

const STORAGE_KEY = 'sky-parkour-upgrades';
const FLOOR_Y = 470;

const saveState = () => {
  const data = {
    totalCoins: state.totalCoins,
    jumpLevel: state.jumpLevel,
    speedLevel: state.speedLevel,
  };
  localStorage.setItem(STORAGE_KEY, JSON.stringify(data));
};

const loadState = () => {
  const saved = localStorage.getItem(STORAGE_KEY);
  if (!saved) {
    return;
  }

  try {
    const data = JSON.parse(saved);
    state.totalCoins = Number(data.totalCoins) || 0;
    state.jumpLevel = Number(data.jumpLevel) || 1;
    state.speedLevel = Number(data.speedLevel) || 1;
  } catch {
    state.totalCoins = 0;
    state.jumpLevel = 1;
    state.speedLevel = 1;
  }
};

const state = {
  started: false,
  gameOver: false,
  score: 0,
  totalCoins: 0,
  jumpLevel: 1,
  speedLevel: 1,
  lives: 3,
  worldSpeed: 5,
  spawnTimer: 0,
  coinTimer: 0,
  invulnerableTimer: 0,
};

const player = {
  x: 120,
  y: FLOOR_Y - 64,
  w: 44,
  h: 64,
  vy: 0,
  gravity: 1,
  jumpPower: 15,
  ground: true,
};

const obstacles = [];
const coins = [];
const particles = [];
const clouds = [];

const initClouds = () => {
  clouds.length = 0;
  for (let i = 0; i < 8; i++) {
    clouds.push({
      x: Math.random() * canvas.width,
      y: 50 + Math.random() * 180,
      size: 24 + Math.random() * 36,
      speed: 0.3 + Math.random() * 1.1,
    });
  }
};

const getJumpCost = () => 20 + (state.jumpLevel - 1) * 18;
const getSpeedCost = () => 25 + (state.speedLevel - 1) * 20;

const updateUpgradeLabels = () => {
  jumpCostLabel.textContent = `Custo: ${getJumpCost()}`;
  speedCostLabel.textContent = `Custo: ${getSpeedCost()}`;

  upgradeButtons.forEach((button) => {
    const upgradeType = button.dataset.upgrade;
    const cost = upgradeType === 'jump' ? getJumpCost() : getSpeedCost();
    const canAfford = state.totalCoins >= cost;
    button.disabled = !canAfford;
  });
};

const updateHud = () => {
  coinCountEl.textContent = state.totalCoins;
  scoreEl.textContent = state.score;
  livesEl.textContent = state.lives;
  updateUpgradeLabels();
};

const resetRun = () => {
  state.started = true;
  state.gameOver = false;
  state.score = 0;
  state.lives = 3;
  state.worldSpeed = 5 + (state.speedLevel - 1) * 1.3;
  state.spawnTimer = 0;
  state.coinTimer = 0;
  state.invulnerableTimer = 0;

  player.x = 120;
  player.y = FLOOR_Y - player.h;
  player.vy = 0;
  player.jumpPower = 15 + (state.jumpLevel - 1) * 1.7;
  player.ground = true;

  obstacles.length = 0;
  coins.length = 0;
  particles.length = 0;

  updateHud();
};

const buyUpgrade = (type) => {
  if (!state.started && !state.gameOver) {
    const cost = type === 'jump' ? getJumpCost() : getSpeedCost();
    if (state.totalCoins < cost) return;

    state.totalCoins -= cost;
    if (type === 'jump') {
      state.jumpLevel += 1;
    } else {
      state.speedLevel += 1;
    }

    saveState();
    updateHud();
    return;
  }

  const cost = type === 'jump' ? getJumpCost() : getSpeedCost();
  if (state.totalCoins < cost) return;

  state.totalCoins -= cost;
  if (type === 'jump') {
    state.jumpLevel += 1;
  } else {
    state.speedLevel += 1;
  }

  saveState();
  updateHud();
};

const showOverlay = () => {
  overlay.classList.add('visible');
};

const hideOverlay = () => {
  overlay.classList.remove('visible');
};

const startGame = () => {
  resetRun();
  hideOverlay();
};

const endGame = () => {
  state.started = false;
  state.gameOver = true;
  saveState();
  showOverlay();

  overlay.innerHTML = `
    <div class="panel">
      <h1>Run finalizada</h1>
      <p>Você conseguiu <strong>${state.score}</strong> pontos e <strong>${state.totalCoins}</strong> moedas no total.</p>
      <div class="upgrade-list">
        <button class="upgrade-button" data-upgrade="jump">
          Salto +1
          <span id="jumpCost">Custo: ${getJumpCost()}</span>
        </button>
        <button class="upgrade-button" data-upgrade="speed">
          Velocidade +1
          <span id="speedCost">Custo: ${getSpeedCost()}</span>
        </button>
      </div>
      <button id="startButton" class="primary-button">Jogar de novo</button>
    </div>
  `;

  const newStartButton = document.getElementById('startButton');
  const newUpgradeButtons = [...document.querySelectorAll('.upgrade-button')];
  newStartButton.addEventListener('click', startGame);

  newUpgradeButtons.forEach((button) => {
    button.addEventListener('click', () => buyUpgrade(button.dataset.upgrade));
  });

  updateHud();
};

const spawnObstacle = () => {
  const w = 28 + Math.random() * 46;
  const h = 24 + Math.random() * 70;
  obstacles.push({
    x: canvas.width + 30,
    y: FLOOR_Y - h,
    w,
    h,
  });
};

const spawnCoin = () => {
  coins.push({
    x: canvas.width + 40,
    y: 120 + Math.random() * 210,
    r: 10,
    bob: Math.random() * Math.PI * 2,
  });
};

const createParticles = (x, y, color) => {
  for (let i = 0; i < 14; i++) {
    particles.push({
      x,
      y,
      vx: (Math.random() - 0.5) * 5,
      vy: -Math.random() * 4,
      color,
      life: 24 + Math.random() * 18,
      size: 3 + Math.random() * 4,
    });
  }
};

const jump = () => {
  if (!state.started || state.gameOver) return;
  if (player.ground) {
    player.vy = -player.jumpPower;
    player.ground = false;
  }
};

document.addEventListener('keydown', (event) => {
  if (event.code === 'Space') {
    event.preventDefault();
    jump();
  }
});

window.addEventListener('pointerdown', () => {
  if (!state.started && !state.gameOver) {
    startGame();
    return;
  }
  jump();
});

upgradeButtons.forEach((button) => {
  button.addEventListener('click', () => buyUpgrade(button.dataset.upgrade));
});

startButton.addEventListener('click', startGame);

const update = () => {
  if (!state.started || state.gameOver) return;

  state.worldSpeed = 5 + (state.speedLevel - 1) * 1.7;
  state.spawnTimer += 1;
  state.coinTimer += 1;
  state.invulnerableTimer = Math.max(0, state.invulnerableTimer - 1);

  player.vy += player.gravity;
  player.y += player.vy;

  if (player.y + player.h >= FLOOR_Y) {
    player.y = FLOOR_Y - player.h;
    player.vy = 0;
    player.ground = true;
  }

  if (state.spawnTimer > 70) {
    spawnObstacle();
    state.spawnTimer = 0;
  }

  if (state.coinTimer > 48) {
    spawnCoin();
    state.coinTimer = 0;
  }

  for (let i = obstacles.length - 1; i >= 0; i--) {
    const obstacle = obstacles[i];
    obstacle.x -= state.worldSpeed;

    const colliding =
      player.x < obstacle.x + obstacle.w &&
      player.x + player.w > obstacle.x &&
      player.y < obstacle.y + obstacle.h &&
      player.y + player.h > obstacle.y;

    if (colliding && state.invulnerableTimer <= 0) {
      state.lives -= 1;
      state.invulnerableTimer = 50;
      createParticles(player.x + player.w / 2, player.y + player.h / 2, '#ff6b6b');

      if (state.lives <= 0) {
        state.score = Math.max(state.score, 0);
        state.totalCoins += Math.max(0, Math.floor(state.score / 20));
        saveState();
        endGame();
        return;
      }
    }

    if (obstacle.x + obstacle.w < -20) {
      obstacles.splice(i, 1);
    }
  }

  for (let i = coins.length - 1; i >= 0; i--) {
    const coin = coins[i];
    coin.x -= state.worldSpeed;
    coin.bob += 0.18;
    coin.y += Math.sin(coin.bob) * 0.35;

    const dx = player.x + player.w / 2 - coin.x;
    const dy = player.y + player.h / 2 - coin.y;
    const distance = Math.sqrt(dx * dx + dy * dy);

    if (distance < (player.w + coin.r) / 2) {
      state.score += 10;
      state.totalCoins += 1;
      createParticles(coin.x, coin.y, '#ffd166');
      coins.splice(i, 1);
      updateHud();
      saveState();
      continue;
    }

    if (coin.x + coin.r < -10) {
      coins.splice(i, 1);
    }
  }

  for (let i = particles.length - 1; i >= 0; i--) {
    const p = particles[i];
    p.x += p.vx;
    p.y += p.vy;
    p.life -= 1;
    if (p.life <= 0) {
      particles.splice(i, 1);
    }
  }

  state.score += 0.2;
  scoreEl.textContent = Math.floor(state.score);
  coinCountEl.textContent = state.totalCoins;
  livesEl.textContent = state.lives;
};

const drawBackground = () => {
  const sky = ctx.createLinearGradient(0, 0, 0, canvas.height);
  sky.addColorStop(0, '#7cc8ff');
  sky.addColorStop(1, '#dff6ff');
  ctx.fillStyle = sky;
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  for (const cloud of clouds) {
    cloud.x -= cloud.speed;
    if (cloud.x < -80) {
      cloud.x = canvas.width + 60;
    }

    ctx.fillStyle = 'rgba(255,255,255,0.8)';
    ctx.beginPath();
    ctx.arc(cloud.x, cloud.y, cloud.size * 0.4, 0, Math.PI * 2);
    ctx.arc(cloud.x + cloud.size * 0.6, cloud.y - 10, cloud.size * 0.5, 0, Math.PI * 2);
    ctx.arc(cloud.x + cloud.size * 1.2, cloud.y, cloud.size * 0.45, 0, Math.PI * 2);
    ctx.fill();
  }

  ctx.fillStyle = '#8ecf9f';
  ctx.beginPath();
  ctx.moveTo(0, 360);
  ctx.lineTo(150, 280);
  ctx.lineTo(300, 330);
  ctx.lineTo(450, 260);
  ctx.lineTo(620, 350);
  ctx.lineTo(760, 290);
  ctx.lineTo(960, 340);
  ctx.lineTo(960, 540);
  ctx.lineTo(0, 540);
  ctx.closePath();
  ctx.fill();

  ctx.fillStyle = '#478b5e';
  ctx.fillRect(0, FLOOR_Y, canvas.width, canvas.height - FLOOR_Y);
  ctx.fillStyle = '#2e5d3d';
  ctx.fillRect(0, FLOOR_Y + 18, canvas.width, 12);
};

const drawPlayer = () => {
  const blink = state.invulnerableTimer > 0 && Math.floor(state.invulnerableTimer / 6) % 2 === 0;
  if (blink) return;

  ctx.fillStyle = '#1d3557';
  ctx.fillRect(player.x, player.y, player.w, player.h);

  ctx.fillStyle = '#f1faee';
  ctx.fillRect(player.x + 8, player.y + 10, 8, 8);
  ctx.fillRect(player.x + 24, player.y + 10, 8, 8);

  ctx.fillStyle = '#ffb703';
  ctx.fillRect(player.x + 12, player.y + 30, 20, 10);
};

const drawObstacles = () => {
  obstacles.forEach((obstacle) => {
    ctx.fillStyle = '#ef476f';
    ctx.fillRect(obstacle.x, obstacle.y, obstacle.w, obstacle.h);
  });
};

const drawCoins = () => {
  coins.forEach((coin) => {
    ctx.fillStyle = '#ffd166';
    ctx.beginPath();
    ctx.arc(coin.x, coin.y, coin.r, 0, Math.PI * 2);
    ctx.fill();

    ctx.fillStyle = '#fff2b2';
    ctx.beginPath();
    ctx.arc(coin.x - 3, coin.y - 3, coin.r * 0.35, 0, Math.PI * 2);
    ctx.fill();
  });
};

const drawParticles = () => {
  particles.forEach((p) => {
    ctx.fillStyle = p.color;
    ctx.fillRect(p.x, p.y, p.size, p.size);
  });
};

const render = () => {
  drawBackground();
  drawCoins();
  drawObstacles();
  drawParticles();
  drawPlayer();
};

const loop = () => {
  update();
  render();
  requestAnimationFrame(loop);
};

loadState();
initClouds();
updateHud();
showOverlay();
requestAnimationFrame(loop);
