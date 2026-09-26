<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
  <title>Chronicles of the Ashen Crypts: GBA Edition</title>
  <style>
    * {
      box-sizing: border-box;
      user-select: none;
      -webkit-user-select: none;
      margin: 0;
      padding: 0;
    }
    body {
      background-color: #0d0e15;
      color: #e2e8f0;
      font-family: 'Courier New', Courier, monospace;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      min-height: 100vh;
      overflow: hidden;
    }
    #game-container {
      position: relative;
      width: 100%;
      max-width: 640px;
      display: flex;
      flex-direction: column;
      align-items: center;
    }
    #top-hud {
      width: 100%;
      background: #1a1c23;
      border: 2px solid #3b3e4f;
      border-bottom: none;
      border-radius: 8px 8px 0 0;
      padding: 6px 12px;
      display: flex;
      justify-content: space-between;
      align-items: center;
      font-size: 13px;
      font-weight: bold;
      color: #ffd700;
      box-shadow: inset 0 0 10px rgba(0,0,0,0.5);
    }
    canvas {
      width: 100%;
      max-width: 640px;
      height: auto;
      aspect-ratio: 4 / 3;
      background: #000;
      image-rendering: pixelated;
      image-rendering: crisp-edges;
      border: 2px solid #3b3e4f;
      display: block;
    }
    #controls {
      width: 100%;
      background: #1a1c23;
      border: 2px solid #3b3e4f;
      border-top: none;
      border-radius: 0 0 8px 8px;
      padding: 10px;
      display: flex;
      justify-content: space-between;
      align-items: center;
    }
    .dpad {
      display: grid;
      grid-template-columns: repeat(3, 42px);
      grid-template-rows: repeat(3, 42px);
      gap: 2px;
    }
    .btn {
      background: #2d313e;
      border: 2px solid #4a4e63;
      color: #fff;
      font-weight: bold;
      font-family: inherit;
      border-radius: 6px;
      display: flex;
      align-items: center;
      justify-content: center;
      font-size: 14px;
      cursor: pointer;
      touch-action: manipulation;
    }
    .btn:active {
      background: #4a4e63;
      transform: translateY(1px);
    }
    .dpad .btn-empty { visibility: hidden; }
    .action-btns {
      display: flex;
      gap: 12px;
    }
    .btn-round {
      width: 52px;
      height: 52px;
      border-radius: 50%;
      font-size: 16px;
      box-shadow: 0 4px 0 #181a22;
    }
    .btn-a { background: #cf3a3a; border-color: #ef5353; }
    .btn-b { background: #2b70c9; border-color: #4a8ee8; }
    .btn-round:active {
      box-shadow: 0 1px 0 #181a22;
      transform: translateY(3px);
    }
    #instructions {
      margin-top: 8px;
      font-size: 11px;
      color: #94a3b8;
      text-align: center;
    }
  </style>
</head>
<body>

<div id="game-container">
  <div id="top-hud">
    <span id="location-text">The Ashen Bastion</span>
    <span id="objective-text">Talk to Roderick</span>
  </div>
  
  <canvas id="canvas" width="320" height="240"></canvas>

  <div id="controls">
    <div class="dpad">
      <div class="btn-empty"></div>
      <button class="btn" id="btn-up">▲</button>
      <div class="btn-empty"></div>
      <button class="btn" id="btn-left">◀</button>
      <div class="btn-empty"></div>
      <button class="btn" id="btn-right">▶</button>
      <div class="btn-empty"></div>
      <button class="btn" id="btn-down">▼</button>
      <div class="btn-empty"></div>
    </div>
    
    <div style="text-align:center; font-size:11px; color:#64748b;">
      <div>Hold <b>SHIFT</b> / <b>B</b> to Run</div>
      <div><b>Z / Space</b> to Act</div>
    </div>

    <div class="action-btns">
      <button class="btn btn-round btn-b" id="btn-b">B</button>
      <button class="btn btn-round btn-a" id="btn-a">A</button>
    </div>
  </div>
  <div id="instructions">Desktop: WASD / Arrows to Walk | SHIFT to Run | Z, Space, Enter to Interact</div>
</div>

<script>
// --- AUDIO SYNTHESIZER (Web Audio API) ---
class SoundController {
  constructor() {
    this.ctx = null;
  }
  init() {
    if (!this.ctx) {
      this.ctx = new (window.AudioContext || window.webkitAudioContext)();
    }
  }
  playTone(freq, type, duration, vol=0.1) {
    if (!this.ctx) return;
    try {
      const osc = this.ctx.createOscillator();
      const gain = this.ctx.createGain();
      osc.type = type;
      osc.frequency.setValueAtTime(freq, this.ctx.currentTime);
      gain.gain.setValueAtTime(vol, this.ctx.currentTime);
      gain.gain.exponentialRampToValueAtTime(0.0001, this.ctx.currentTime + duration);
      osc.connect(gain);
      gain.connect(this.ctx.destination);
      osc.start();
      osc.stop(this.ctx.currentTime + duration);
    } catch(e){}
  }
  playBump() { this.playTone(120, 'triangle', 0.08, 0.15); }
  playSelect() { this.playTone(440, 'square', 0.05, 0.1); }
  playHit() { this.playTone(80, 'sawtooth', 0.15, 0.2); }
  playMagic() { this.playTone(600, 'sine', 0.25, 0.15); }
  playHeal() { this.playTone(520, 'sine', 0.3, 0.15); }
  playVictory() {
    [440, 554, 659, 880].forEach((f, i) => {
      setTimeout(() => this.playTone(f, 'square', 0.15, 0.12), i * 100);
    });
  }
}
const sound = new SoundController();

// --- CONSTANTS & MAP DATA ---
const TILE_SIZE = 16;
const MAP_COLS = 20;
const MAP_ROWS = 15;

// Tile types: 0: Floor, 1: Wall, 2: Stairs, 3: Chest, 4: Hearth, 5: Roderick, 6: Maeve, 7: Garrick
const TOWN_MAP = [
  [1,1,1,1,1,1,1,1,1,2,2,1,1,1,1,1,1,1,1,1],
  [1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1],
  [1,0,5,0,0,0,0,0,0,0,0,0,0,0,0,0,6,0,0,1],
  [1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1],
  [1,0,0,0,1,1,1,0,0,0,0,0,1,1,1,0,0,0,0,1],
  [1,0,0,0,1,1,1,0,0,0,0,0,1,1,1,0,0,0,0,1],
  [1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1],
  [1,0,0,0,0,0,0,0,4,4,0,0,0,0,0,0,0,0,0,1],
  [1,0,0,0,0,0,0,0,4,4,0,0,0,0,0,0,0,0,0,1],
  [1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1],
  [1,0,0,0,1,1,1,0,0,0,0,0,1,1,1,0,0,0,0,1],
  [1,0,0,0,1,1,1,0,0,7,0,0,1,1,1,0,0,0,0,1],
  [1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1],
  [1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1],
  [1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1]
];

// --- GAME STATE ---
const player = {
  x: 10, y: 13,
  px: 10 * TILE_SIZE, py: 13 * TILE_SIZE,
  dir: 'UP',
  moving: false,
  moveSpeed: 1.5,
  hp: 100, maxHp: 100,
  mp: 50, maxMp: 50,
  level: 1, exp: 0, gold: 50, potions: 2,
  quests: { roderick: false, bossDefeated: false },
  bleedTurns: 0,
  stunned: false
};

let currentFloor = 0; // 0 = Town, 1-5 = Crypt Floors
let activeMap = TOWN_MAP;
let monsters = [];
let chests = [];
let gameState = 'OVERWORLD'; // OVERWORLD, DIALOGUE, SHOP, BATTLE, TRANSITION
let dialogueQueue = [];
let dialogueCallback = null;
let currentBattle = null;
let transitionAlpha = 0;
let transitionState = 'NONE'; // IN, OUT
let pendingAction = null;

// --- MONSTER DATABASE ---
const MONSTER_TYPES = {
  SKELETON: { name: 'Skeleton', hp: 40, mp: 0, atk: 12, def: 2, exp: 25, gold: 15, color: '#e2e8f0', undead: true },
  GHOUL: { name: 'Ghoul', hp: 65, mp: 10, atk: 18, def: 4, exp: 45, gold: 25, color: '#84cc16', undead: true },
  BANSHEE: { name: 'Banshee', hp: 90, mp: 40, atk: 24, def: 6, exp: 70, gold: 40, color: '#38bdf8', undead: true },
  KNIGHT: { name: 'Dread Knight', hp: 130, mp: 20, atk: 32, def: 12, exp: 110, gold: 70, color: '#94a3b8', undead: false },
  LORD_MALAKOR: { name: 'Lord Malakor', hp: 280, mp: 60, atk: 42, def: 16, exp: 350, gold: 250, color: '#c084fc', undead: true, isBoss: true }
};

// --- SKILLS ---
const SKILLS = [
  { name: '⚔️ Strike', mp: 0, desc: 'Basic physical attack.' },
  { name: '✨ Holy Smite', mp: 15, desc: 'Radiant blast. +50% vs Undead.' },
  { name: '🛡️ Shield Bash', mp: 18, desc: 'Heavy hit with 70% stun chance.' },
  { name: '🩸 Shadow Rend', mp: 20, desc: 'Causes 3 turns of bleed.' },
  { name: '🔮 Siphon Life', mp: 24, desc: 'Dmg enemy & heal 60% of dmg.' },
  { name: '🛡️ Guard', mp: 0, desc: 'Reduce dmg 50% & gain 15 MP.' },
  { name: '🧪 Potion', mp: 0, desc: 'Restore 80 HP.' },
  { name: '🏃 Flee', mp: 0, desc: 'Escape back to floor start.' }
];

// --- INPUT SYSTEM ---
const keys = {};
window.addEventListener('keydown', e => {
  sound.init();
  keys[e.code] = true;
  if (['Space', 'KeyZ', 'Enter'].includes(e.code)) handleInteract();
});
window.addEventListener('keyup', e => { keys[e.code] = false; });

function bindTouch(id, code) {
  const btn = document.getElementById(id);
  btn.addEventListener('pointerdown', (e) => {
    e.preventDefault();
    sound.init();
    keys[code] = true;
    if (code === 'KeyZ') handleInteract();
  });
  btn.addEventListener('pointerup', (e) => { e.preventDefault(); keys[code] = false; });
  btn.addEventListener('pointerleave', (e) => { e.preventDefault(); keys[code] = false; });
}
bindTouch('btn-up', 'KeyW');
bindTouch('btn-down', 'KeyS');
bindTouch('btn-left', 'KeyA');
bindTouch('btn-right', 'KeyD');
bindTouch('btn-a', 'KeyZ');
bindTouch('btn-b', 'ShiftLeft');

// --- SETUP & GENERATION ---
function generateDungeon(floor) {
  const map = Array.from({ length: MAP_ROWS }, () => Array(MAP_COLS).fill(0));
  // Outer walls
  for (let r = 0; r < MAP_ROWS; r++) {
    for (let c = 0; c < MAP_COLS; c++) {
      if (r === 0 || r === MAP_ROWS - 1 || c === 0 || c === MAP_COLS - 1) {
        map[r][c] = 1;
      } else if (Math.random() < 0.18 && !(r === 13 && c === 10)) {
        map[r][c] = 1;
      }
    }
  }
  map[1][10] = 2; // Stairs to next floor
  
  // Clear path around spawn & stairs
  map[13][10] = 0; map[12][10] = 0;
  map[1][10] = 2; map[2][10] = 0;

  monsters = [];
  chests = [];

  const count = floor === 5 ? 3 : 3 + floor;
  let placed = 0;
  while (placed < count) {
    let rx = Math.floor(Math.random() * (MAP_COLS - 2)) + 1;
    let ry = Math.floor(Math.random() * (MAP_ROWS - 4)) + 1;
    if (map[ry][rx] === 0) {
      let type = MONSTER_TYPES.SKELETON;
      if (floor === 2) type = Math.random() < 0.5 ? MONSTER_TYPES.SKELETON : MONSTER_TYPES.GHOUL;
      if (floor === 3) type = Math.random() < 0.5 ? MONSTER_TYPES.GHOUL : MONSTER_TYPES.BANSHEE;
      if (floor === 4) type = Math.random() < 0.5 ? MONSTER_TYPES.BANSHEE : MONSTER_TYPES.KNIGHT;
      if (floor === 5) {
        if (placed === 0) type = MONSTER_TYPES.LORD_MALAKOR;
        else type = MONSTER_TYPES.KNIGHT;
      }
      monsters.push({
        x: rx, y: ry,
        px: rx * TILE_SIZE, py: ry * TILE_SIZE,
        type: { ...type, currentHp: type.hp },
        dir: 'DOWN', timer: Math.random() * 100
      });
      placed++;
    }
  }

  // Scatter 2 Chests
  let chestCount = 0;
  while (chestCount < 2) {
    let rx = Math.floor(Math.random() * (MAP_COLS - 2)) + 1;
    let ry = Math.floor(Math.random() * (MAP_ROWS - 2)) + 1;
    if (map[ry][rx] === 0) {
      map[ry][rx] = 3;
      chests.push({ x: rx, y: ry, opened: false });
      chestCount++;
    }
  }

  return map;
}

// --- INTERACTION / DIALOGUE ---
function showDialogue(lines, callback = null) {
  dialogueQueue = [...lines];
  dialogueCallback = callback;
  gameState = 'DIALOGUE';
}

function handleInteract() {
  if (gameState === 'DIALOGUE') {
    sound.playSelect();
    dialogueQueue.shift();
    if (dialogueQueue.length === 0) {
      gameState = 'OVERWORLD';
      if (dialogueCallback) {
        dialogueCallback();
        dialogueCallback = null;
      }
    }
    return;
  }

  if (gameState === 'BATTLE') {
    handleBattleInput();
    return;
  }

  if (gameState !== 'OVERWORLD') return;

  // Find target tile in front of player
  let tx = player.x;
  let ty = player.y;
  if (player.dir === 'UP') ty--;
  if (player.dir === 'DOWN') ty++;
  if (player.dir === 'LEFT') tx--;
  if (player.dir === 'RIGHT') tx++;

  const tile = activeMap[ty]?.[tx];

  if (tile === 5) { // Roderick
    if (!player.quests.roderick) {
      showDialogue([
        "Roderick: The crypts below grow restless!",
        "Roderick: Slay Lord Malakor on Floor 5 to save our realm.",
        "Roderick: Take these 50 gold to prepare!"
      ], () => { player.quests.roderick = true; player.gold += 50; });
    } else if (player.quests.bossDefeated) {
      showDialogue(["Roderick: You have defeated Malakor! You are our Savior! ✨"]);
    } else {
      showDialogue(["Roderick: Proceed North to enter Dungeon Floor 1."]);
    }
  } else if (tile === 6) { // Sister Maeve
    showDialogue([
      "Maeve: Welcome traveler. I sell Potions for 25g each.",
      "Maeve: Restored 1 Potion to your inventory!"
    ], () => {
      if (player.gold >= 25) {
        player.gold -= 25;
        player.potions++;
      }
    });
  } else if (tile === 7) { // Garrick
    showDialogue(["Garrick: Steel blades and heavy shields! (+5 Max HP)"], () => {
      if (player.gold >= 30) {
        player.gold -= 30;
        player.maxHp += 5;
        player.hp += 5;
      }
    });
  } else if (tile === 4) { // Campfire
    player.hp = player.maxHp;
    player.mp = player.maxMp;
    sound.playHeal();
    showDialogue(["Campfire Hearth: HP and MP fully restored! 🔥"]);
  } else if (tile === 3) { // Chest
    const c = chests.find(ch => ch.x === tx && ch.y === ty);
    if (c && !c.opened) {
      c.opened = true;
      activeMap[ty][tx] = 0;
      const g = 20 + Math.floor(Math.random() * 30);
      player.gold += g;
      sound.playVictory();
      showDialogue([`Found Treasure Sarcophagus! +${g} Gold!`]);
    }
  }
}

// --- TRANSITIONS ---
function startTransition(onPeak) {
  gameState = 'TRANSITION';
  transitionState = 'IN';
  transitionAlpha = 0;
  pendingAction = onPeak;
}

// --- MOVEMENT & OVERWORLD UPDATE ---
function updateOverworld() {
  const isRunning = keys['ShiftLeft'] || keys['ShiftRight'] || keys['KeyB'];
  const speed = isRunning ? 2.5 : 1.5;

  if (!player.moving) {
    let dx = 0, dy = 0;
    if (keys['KeyW'] || keys['ArrowUp']) { dy = -1; player.dir = 'UP'; }
    else if (keys['KeyS'] || keys['ArrowDown']) { dy = 1; player.dir = 'DOWN'; }
    else if (keys['KeyA'] || keys['ArrowLeft']) { dx = -1; player.dir = 'LEFT'; }
    else if (keys['KeyD'] || keys['ArrowRight']) { dx = 1; player.dir = 'RIGHT'; }

    if (dx !== 0 || dy !== 0) {
      const nx = player.x + dx;
      const ny = player.y + dy;

      if (nx >= 0 && nx < MAP_COLS && ny >= 0 && ny < MAP_ROWS) {
        const targetTile = activeMap[ny][nx];
        // North gate transition (Town -> Floor 1)
        if (currentFloor === 0 && ny === 0 && (nx === 9 || nx === 10)) {
          startTransition(() => {
            currentFloor = 1;
            activeMap = generateDungeon(1);
            player.x = 10; player.y = 13;
            player.px = player.x * TILE_SIZE; player.py = player.y * TILE_SIZE;
          });
          return;
        }
        // Stairs transition (Floor N -> Floor N+1 or Town)
        if (targetTile === 2) {
          if (monsters.length > 0) {
            showDialogue(["Stairs Locked! Slay all monsters on this floor first!"]);
            return;
          }
          startTransition(() => {
            currentFloor++;
            if (currentFloor > 5) {
              currentFloor = 0;
              activeMap = TOWN_MAP;
              showDialogue(["✨ You completed the Crypts and returned victorious!"]);
            } else {
              activeMap = generateDungeon(currentFloor);
            }
            player.x = 10; player.y = 13;
            player.px = player.x * TILE_SIZE; player.py = player.y * TILE_SIZE;
          });
          return;
        }

        // Walkable floor check
        if (targetTile === 0) {
          player.x = nx;
          player.y = ny;
          player.moving = true;
        } else {
          sound.playBump();
        }
      }
    }
  } else {
    // Smooth pixel interpolation
    const tx = player.x * TILE_SIZE;
    const ty = player.y * TILE_SIZE;
    if (player.px < tx) player.px = Math.min(player.px + speed, tx);
    if (player.px > tx) player.px = Math.max(player.px - speed, tx);
    if (player.py < ty) player.py = Math.min(player.py + speed, ty);
    if (player.py > ty) player.py = Math.max(player.py - speed, ty);

    if (player.px === tx && player.py === ty) {
      player.moving = false;
    }
  }

  // Update Monsters
  monsters.forEach((m, idx) => {
    m.timer += 1;
    if (m.timer > 80) {
      m.timer = 0;
      const dirs = [[0,1],[0,-1],[1,0],[-1,0]];
      const d = dirs[Math.floor(Math.random() * 4)];
      const nx = m.x + d[0];
      const ny = m.y + d[1];
      if (activeMap[ny]?.[nx] === 0) {
        m.x = nx; m.y = ny;
        m.px = nx * TILE_SIZE; m.py = ny * TILE_SIZE;
      }
    }

    // Encounter Trigger
    if (Math.abs(m.px - player.px) < 10 && Math.abs(m.py - player.py) < 10) {
      sound.playHit();
      startTransition(() => {
        initiateBattle(m, idx);
      });
    }
  });

  // Update HUD
  const locEl = document.getElementById('location-text');
  const objEl = document.getElementById('objective-text');
  if (currentFloor === 0) {
    locEl.innerText = "The Ashen Bastion (Town)";
    objEl.innerText = player.quests.roderick ? "Enter North Gate" : "Talk to Roderick";
  } else {
    locEl.innerText = `Crypt Floor ${currentFloor}`;
    objEl.innerText = monsters.length > 0 ? `Monsters to Clear: ${monsters.length}` : "✨ STAIRS UNLOCKED!";
  }
}

// --- BATTLE SYSTEM ---
let battleIndex = 0;
let battleLog = "";

function initiateBattle(monsterObj, index) {
  battleIndex = index;
  gameState = 'BATTLE';
  currentBattle = {
    monster: { ...monsterObj.type },
    menuIndex: 0,
    turn: 'PLAYER',
    guarding: false
  };
  battleLog = `Encountered ${currentBattle.monster.name}!`;
}

function handleBattleInput() {
  if (currentBattle.turn !== 'PLAYER') return;

  if (keys['KeyW'] || keys['ArrowUp']) { currentBattle.menuIndex = (currentBattle.menuIndex - 1 + SKILLS.length) % SKILLS.length; sound.playSelect(); keys['KeyW'] = keys['ArrowUp'] = false; }
  if (keys['KeyS'] || keys['ArrowDown']) { currentBattle.menuIndex = (currentBattle.menuIndex + 1) % SKILLS.length; sound.playSelect(); keys['KeyS'] = keys['ArrowDown'] = false; }

  if (keys['KeyZ'] || keys['Space'] || keys['Enter']) {
    keys['KeyZ'] = keys['Space'] = keys['Enter'] = false;
    executePlayerSkill(SKILLS[currentBattle.menuIndex]);
  }
}

function executePlayerSkill(skill) {
  const m = currentBattle.monster;
  if (player.mp < skill.mp) {
    battleLog = "Not enough MP!";
    return;
  }

  player.mp -= skill.mp;
  let dmg = 0;

  if (skill.name.includes('Strike')) {
    dmg = Math.max(5, player.level * 10 + 12 - m.def);
    m.currentHp -= dmg;
    sound.playHit();
    battleLog = `You strike ${m.name} for ${dmg} DMG!`;
  } else if (skill.name.includes('Holy Smite')) {
    dmg = Math.max(10, player.level * 14 + 18 - m.def);
    if (m.undead) dmg = Math.floor(dmg * 1.5);
    m.currentHp -= dmg;
    sound.playMagic();
    battleLog = `Radiant Light smites ${m.name} for ${dmg} DMG!`;
  } else if (skill.name.includes('Shield Bash')) {
    dmg = Math.max(4, player.level * 8 + 6 - m.def);
    m.currentHp -= dmg;
    if (Math.random() < 0.7) {
      m.stunned = true;
      battleLog = `Bash deals ${dmg} DMG and STUNS ${m.name}!`;
    } else {
      battleLog = `Bash deals ${dmg} DMG!`;
    }
    sound.playHit();
  } else if (skill.name.includes('Shadow Rend')) {
    dmg = Math.max(6, player.level * 9 + 8 - m.def);
    m.currentHp -= dmg;
    m.bleed = 3;
    sound.playHit();
    battleLog = `Shadow Rend deals ${dmg} DMG and inflicts Bleed!`;
  } else if (skill.name.includes('Siphon Life')) {
    dmg = Math.max(8, player.level * 11 + 10 - m.def);
    m.currentHp -= dmg;
    const heal = Math.floor(dmg * 0.6);
    player.hp = Math.min(player.maxHp, player.hp + heal);
    sound.playMagic();
    battleLog = `Siphoned ${dmg} DMG & restored ${heal} HP!`;
  } else if (skill.name.includes('Guard')) {
    currentBattle.guarding = true;
    player.mp = Math.min(player.maxMp, player.mp + 15);
    sound.playSelect();
    battleLog = `Guarding! +15 MP gained.`;
  } else if (skill.name.includes('Potion')) {
    if (player.potions > 0) {
      player.potions--;
      player.hp = Math.min(player.maxHp, player.hp + 80);
      sound.playHeal();
      battleLog = `Used Potion! Restored 80 HP.`;
    } else {
      battleLog = `No Potions left!`;
      return;
    }
  } else if (skill.name.includes('Flee')) {
    battleLog = `Escaped back to safe ground!`;
    setTimeout(() => { gameState = 'OVERWORLD'; }, 600);
    return;
  }

  // Check Enemy Death
  if (m.currentHp <= 0) {
    sound.playVictory();
    player.exp += m.exp;
    player.gold += m.gold;
    battleLog = `Defeated ${m.name}! +${m.exp} EXP, +${m.gold} Gold!`;
    
    if (m.isBoss) player.quests.bossDefeated = true;

    // Remove monster
    monsters.splice(battleIndex, 1);

    // Level Up Check
    if (player.exp >= player.level * 100) {
      player.level++;
      player.maxHp += 20;
      player.hp = player.maxHp;
      player.maxMp += 10;
      player.mp = player.maxMp;
      battleLog += ` LEVEL UP! Reached Lv.${player.level}!`;
    }

    setTimeout(() => { gameState = 'OVERWORLD'; }, 1200);
    return;
  }

  // Enemy Turn
  currentBattle.turn = 'MONSTER';
  setTimeout(executeMonsterTurn, 800);
}

function executeMonsterTurn() {
  const m = currentBattle.monster;

  // Process Enemy Bleed
  if (m.bleed && m.bleed > 0) {
    m.bleed--;
    m.currentHp -= 10;
    battleLog = `${m.name} suffers 10 Bleed DMG!`;
    if (m.currentHp <= 0) {
      sound.playVictory();
      monsters.splice(battleIndex, 1);
      setTimeout(() => { gameState = 'OVERWORLD'; }, 1000);
      return;
    }
  }

  if (m.stunned) {
    m.stunned = false;
    battleLog = `${m.name} is Stunned and loses their turn!`;
    currentBattle.turn = 'PLAYER';
    return;
  }

  let rawDmg = Math.max(4, m.atk - Math.floor(player.level * 2));
  if (currentBattle.guarding) rawDmg = Math.floor(rawDmg * 0.5);

  player.hp -= rawDmg;
  sound.playHit();
  battleLog = `${m.name} attacks for ${rawDmg} DMG!`;
  currentBattle.guarding = false;

  if (player.hp <= 0) {
    player.hp = player.maxHp;
    currentFloor = 0;
    activeMap = TOWN_MAP;
    player.x = 10; player.y = 13;
    player.px = player.x * TILE_SIZE; player.py = player.y * TILE_SIZE;
    showDialogue(["💀 You were struck down! Awakened at the Town Hearth."]);
    return;
  }

  currentBattle.turn = 'PLAYER';
}

// --- RENDER ENGINE ---
const canvas = document.getElementById('canvas');
const ctx = canvas.getContext('2d');

function render() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);

  if (gameState === 'OVERWORLD' || gameState === 'DIALOGUE' || gameState === 'TRANSITION') {
    renderOverworld();
  } else if (gameState === 'BATTLE') {
    renderBattle();
  }

  if (gameState === 'DIALOGUE') {
    renderDialogueBox();
  }

  // Screen Wipe Transition
  if (gameState === 'TRANSITION') {
    if (transitionState === 'IN') {
      transitionAlpha += 0.08;
      if (transitionAlpha >= 1) {
        transitionAlpha = 1;
        if (pendingAction) { pendingAction(); pendingAction = null; }
        transitionState = 'OUT';
      }
    } else if (transitionState === 'OUT') {
      transitionAlpha -= 0.08;
      if (transitionAlpha <= 0) {
        transitionAlpha = 0;
        gameState = 'OVERWORLD';
      }
    }
    ctx.fillStyle = `rgba(0, 0, 0, ${transitionAlpha})`;
    ctx.fillRect(0, 0, canvas.width, canvas.height);
  }

  requestAnimationFrame(gameLoop);
}

function renderOverworld() {
  // Render Map Tiles
  for (let r = 0; r < MAP_ROWS; r++) {
    for (let c = 0; c < MAP_COLS; c++) {
      const tile = activeMap[r][c];
      const x = c * TILE_SIZE;
      const y = r * TILE_SIZE;

      if (tile === 1) { // Wall
        ctx.fillStyle = currentFloor === 0 ? '#334155' : '#1e1b4b';
        ctx.fillRect(x, y, TILE_SIZE, TILE_SIZE);
        ctx.strokeStyle = '#475569';
        ctx.strokeRect(x, y, TILE_SIZE, TILE_SIZE);
      } else if (tile === 0) { // Floor
        ctx.fillStyle = currentFloor === 0 ? '#1e293b' : '#0f172a';
        ctx.fillRect(x, y, TILE_SIZE, TILE_SIZE);
      } else if (tile === 2) { // Stairs / Gate
        ctx.fillStyle = '#f59e0b';
        ctx.fillRect(x + 2, y + 2, TILE_SIZE - 4, TILE_SIZE - 4);
      } else if (tile === 3) { // Chest
        ctx.fillStyle = '#d97706';
        ctx.fillRect(x + 3, y + 3, TILE_SIZE - 6, TILE_SIZE - 6);
      } else if (tile === 4) { // Hearth
        ctx.fillStyle = '#ef4444';
        ctx.fillRect(x + 2, y + 2, TILE_SIZE - 4, TILE_SIZE - 4);
      } else if (tile === 5) { // Roderick
        ctx.fillStyle = '#3b82f6';
        ctx.fillRect(x + 2, y + 2, TILE_SIZE - 4, TILE_SIZE - 4);
      } else if (tile === 6) { // Maeve
        ctx.fillStyle = '#ec4899';
        ctx.fillRect(x + 2, y + 2, TILE_SIZE - 4, TILE_SIZE - 4);
      } else if (tile === 7) { // Garrick
        ctx.fillStyle = '#eab308';
        ctx.fillRect(x + 2, y + 2, TILE_SIZE - 4, TILE_SIZE - 4);
      }
    }
  }

  // Render Monsters
  monsters.forEach(m => {
    ctx.fillStyle = m.type.color;
    ctx.beginPath();
    ctx.arc(m.px + TILE_SIZE / 2, m.py + TILE_SIZE / 2, 6, 0, Math.PI * 2);
    ctx.fill();
  });

  // Render Player (GBA Retro Sprite style)
  ctx.fillStyle = '#10b981';
  ctx.fillRect(player.px + 2, player.py + 2, TILE_SIZE - 4, TILE_SIZE - 4);
  ctx.fillStyle = '#fef08a'; // Helmet visor
  ctx.fillRect(player.px + 4, player.py + 4, TILE_SIZE - 8, 4);

  // Player Stats Mini Bar
  ctx.fillStyle = 'rgba(15, 23, 42, 0.85)';
  ctx.fillRect(4, 4, 110, 36);
  ctx.fillStyle = '#fff';
  ctx.font = '9px monospace';
  ctx.fillText(`Lv.${player.level} Hero`, 8, 14);
  ctx.fillStyle = '#ef4444';
  ctx.fillText(`HP: ${player.hp}/${player.maxHp}`, 8, 24);
  ctx.fillStyle = '#3b82f6';
  ctx.fillText(`MP: ${player.mp}/${player.maxMp}`, 8, 34);
}

function renderDialogueBox() {
  ctx.fillStyle = '#0f172a';
  ctx.strokeStyle = '#38bdf8';
  ctx.lineWidth = 2;
  ctx.fillRect(10, 170, 300, 60);
  ctx.strokeRect(10, 170, 300, 60);

  ctx.fillStyle = '#fff';
  ctx.font = '11px monospace';
  const line = dialogueQueue[0] || "";
  ctx.fillText(line, 20, 195);
  ctx.fillStyle = '#38bdf8';
  ctx.fillText("Press Z / Space ►", 200, 220);
}

function renderBattle() {
  // Battle Background
  ctx.fillStyle = '#020617';
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  const m = currentBattle.monster;

  // Monster Visual
  ctx.fillStyle = m.color;
  ctx.beginPath();
  ctx.arc(230, 70, m.isBoss ? 28 : 20, 0, Math.PI * 2);
  ctx.fill();

  // Enemy Name & HP
  ctx.fillStyle = '#fff';
  ctx.font = 'bold 12px monospace';
  ctx.fillText(m.name, 180, 25);
  ctx.fillStyle = '#334155';
  ctx.fillRect(180, 30, 100, 8);
  ctx.fillStyle = '#ef4444';
  ctx.fillRect(180, 30, Math.max(0, (m.currentHp / m.hp) * 100), 8);

  // Player Battle Sprite
  ctx.fillStyle = '#10b981';
  ctx.fillRect(50, 90, 24, 24);

  // Battle Message Log
  ctx.fillStyle = '#1e293b';
  ctx.fillRect(10, 125, 300, 22);
  ctx.fillStyle = '#ffd700';
  ctx.font = '10px monospace';
  ctx.fillText(battleLog, 15, 140);

  // Skill Menu Grid
  ctx.fillStyle = '#0f172a';
  ctx.strokeStyle = '#475569';
  ctx.fillRect(10, 152, 300, 82);
  ctx.strokeRect(10, 152, 300, 82);

  SKILLS.forEach((sk, i) => {
    const col = i % 2;
    const row = Math.floor(i / 2);
    const x = 20 + col * 140;
    const y = 168 + row * 16;

    if (i === currentBattle.menuIndex) {
      ctx.fillStyle = '#38bdf8';
      ctx.fillText(`► ${sk.name}`, x - 10, y);
    } else {
      ctx.fillStyle = '#94a3b8';
      ctx.fillText(sk.name, x, y);
    }
  });
}

// --- MAIN LOOP ---
function gameLoop() {
  if (gameState === 'OVERWORLD') {
    updateOverworld();
  }
  render();
}

// Start Game Engine
requestAnimationFrame(gameLoop);
</script>
</body>
</html>
