<!doctype html>
<html lang="ja">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>ゴキブリ・サバイバー</title>
  <style>
    * { box-sizing: border-box; }
    body { margin:0; min-height:100vh; display:grid; place-items:center; background:#151a20; color:#f7f3e8; font-family:system-ui, "Yu Gothic UI", sans-serif; overflow:hidden; }
    #game { position:relative; width:min(96vw, 960px); aspect-ratio:16/10; border:3px solid #9d793e; border-radius:12px; overflow:hidden; box-shadow:0 18px 55px #0009; background:#263327; cursor:crosshair; }
    canvas { width:100%; height:100%; display:block; }
    #hud { position:absolute; inset:0; pointer-events:none; padding:14px; font-weight:800; text-shadow:1px 1px 2px #000; font-size:clamp(12px, 1.7vw, 18px); }
    #top { display:flex; justify-content:space-between; gap:10px; }
    #weapons { position:absolute; left:14px; bottom:12px; line-height:1.55; }
    .active { color:#ffd65a; }
    .locked { color:#c9c9c9; opacity:.8; }
    #notice { position:absolute; top:22%; left:50%; transform:translateX(-50%); text-align:center; font-size:clamp(18px, 3vw, 31px); color:#ffe36f; opacity:0; transition:opacity .2s; white-space:nowrap; }
    #help { position:absolute; right:14px; bottom:12px; text-align:right; line-height:1.45; font-size:.85em; opacity:.9; }
    #overlay { position:absolute; inset:0; display:grid; place-items:center; background:#10151dcc; text-align:center; pointer-events:auto; }
    #overlay h1 { margin:0 0 10px; font-size:clamp(28px, 5vw, 52px); color:#ffdc6a; }
    #overlay p { margin:7px; font-size:clamp(14px, 2vw, 19px); }
    button { margin-top:15px; border:0; border-radius:8px; padding:11px 22px; font:inherit; font-weight:800; background:#f0b642; color:#241809; cursor:pointer; }
    button:disabled { opacity:.48; cursor:default; }
    .menu { width:min(90%,720px); }
    .menu-grid { display:grid; grid-template-columns:1fr 1fr; gap:14px; text-align:left; }
    .menu-card { min-height:250px; padding:17px; border:1px solid #9d793e; border-radius:10px; background:#1a2630; }
    .menu-card h2 { margin:0 0 9px; color:#ffdc6a; font-size:1.35em; }
    .menu-card p { font-size:.92em !important; margin:7px 0 !important; }
    .menu-card button { width:100%; margin-top:10px; }
    .coins { margin:0 0 14px !important; color:#ffe17c; font-size:1.1em !important; }
    .dev { background:#9a5cdd; color:#fff; }
    @media (max-width:620px) { .menu-grid { grid-template-columns:1fr; } .menu-card { min-height:0; } }
  </style>
</head>
<body>
  <main id="game" aria-label="ゴキブリ・サバイバー">
    <canvas id="canvas" width="960" height="600"></canvas>
    <section id="hud">
      <div id="top"><span id="hp">❤️ HP: 5 / 5</span><span id="ammo">🔸 弾薬: 60 / 120</span><span id="score">スコア: 0　撃破: 0</span><span id="time">生存: 0秒</span></div>
      <div id="weapons"></div>
      <div id="help">移動: WASD / 矢印キー<br>照準・射撃: マウス＋クリック</div>
      <div id="notice"></div>
    </section>
    <div id="overlay"><div><h1>🪳 ゴキブリ・サバイバー</h1><p>ゴキブリを撃退して生き残れ！</p><p>20体で散弾銃、30体以降は兵士が出現。</p><button id="start">ゲーム開始</button></div></div>
  </main>
<script>
(() => {
  const canvas = document.querySelector('#canvas'), ctx = canvas.getContext('2d');
  const W = canvas.width, H = canvas.height;
  const game = document.querySelector('#game'), overlay = document.querySelector('#overlay');
  const hpEl = document.querySelector('#hp'), ammoEl = document.querySelector('#ammo'), scoreEl = document.querySelector('#score'), timeEl = document.querySelector('#time');
  const weaponsEl = document.querySelector('#weapons'), noticeEl = document.querySelector('#notice');
  const saveKey='gokiburi-survivor-progress-v1';
  let progress={coins:0,spherePacks:0,laserUnlocked:false,developerMode:false};
  try { progress={...progress,...JSON.parse(localStorage.getItem(saveKey)||'{}')}; } catch (_) {}
  if(progress.spherePack && !progress.spherePacks) progress.spherePacks=1;
  const saveProgress=()=>localStorage.setItem(saveKey,JSON.stringify(progress));
  let keys = {}, mouse = {x:W/2,y:H/2,down:false}, running = false, last = 0, startAt = 0, nextSpawn = 0, nextShot = 0, nextDrinkCheck = 0, boostEnds = 0, bombCooldownUntil = 0, sphereSpin = 0;
  let player, enemies, bullets, enemyBullets, laserBeams, particles, floorDust, aoeEffects, lightningWarnings, lightningEffects, ammoDrops, heartDrops, drinkDrops, spheres, bomb, ammo, score, kills, shotgunUnlocked, rifleUnlocked, rifleNotified, gunnerSpawned, soldierNotified, bossNotified, weapon, gameMode, battleBossesDefeated, bossAppeared, bossActive, bossDefeated, noticeTimer;

  function reset() {
    player = {x:W/2,y:H/2,r:17,hp:5,maxHp:5,invincible:0,angle:0}; enemies=[]; bullets=[]; enemyBullets=[]; laserBeams=[]; particles=[]; floorDust=Array.from({length:260},()=>({x:Math.random()*W,y:Math.random()*H,r:Math.random()*2+.35,a:Math.random()*.24+.05})); aoeEffects=[]; lightningWarnings=[]; lightningEffects=[]; ammoDrops=[]; heartDrops=[]; drinkDrops=[]; spheres=[]; bomb=null; ammo=60;
    for(let i=0;i<progress.spherePacks*4;i++) spheres.push({nextDamage:0,x:0,y:0});
    score=0; kills=0; shotgunUnlocked=false; rifleUnlocked=false; rifleNotified=false; gunnerSpawned=false; soldierNotified=false; bossNotified=false; weapon='pistol'; gameMode='tutorial'; battleBossesDefeated=0; bossAppeared=false; bossActive=false; bossDefeated=false; nextSpawn=0; nextShot=0; nextDrinkCheck=0; boostEnds=0; bombCooldownUntil=0; sphereSpin=0;
    updateHud();
  }
  function updateHud() {
    hpEl.textContent=`❤️ HP: ${Math.max(0,player.hp)} / ${player.maxHp}`;
    ammoEl.textContent=`🔸 弾薬: ${ammo===Infinity?'∞':ammo}`;
    scoreEl.textContent=`スコア: ${score}　撃破: ${kills}　コイン: ${progress.coins}`;
    weaponsEl.innerHTML = `<div class="${weapon==='pistol'?'active':''}">[1] 🔫 ピストル ${weapon==='pistol'?'（使用中）':''}</div>`+
      (shotgunUnlocked ? `<div class="${weapon==='shotgun'?'active':''}">[2] 💥 散弾銃 ${weapon==='shotgun'?'（使用中）':''}</div>` : `<div class="locked">[2] 💥 散弾銃（あと ${Math.max(0,20-kills)} 体でアンロック）</div>`) +
      (rifleUnlocked ? `<div class="${weapon==='rifle'?'active':''}">[3] 🔥 アサルトライフル ${weapon==='rifle'?'（使用中）':''}</div>` : `<div class="locked">[3] 🔥 アサルトライフル（銃士を倒すと入手）</div>`) +
      (progress.laserUnlocked ? `<div class="${weapon==='laser'?'active':''}">[4] ⚡ ランページレーザー ${weapon==='laser'?'（使用中）':''}</div>` : '')+
      `<div class="${spheres.length?'active':'locked'}">🔵 スフィア：${spheres.length} 個（兵士撃破で追加）</div>`+
      `<div class="${performance.now()<boostEnds?'active':'locked'}">🥤 ドリンク：${performance.now()<boostEnds?`強化中（${Math.ceil((boostEnds-performance.now())/1000)}秒）`:'1秒ごとに1%で出現'}</div>`+
      `<div class="${bomb?'active':'locked'}">💣 爆弾：${bomb?'設置済み（Qで起爆）':performance.now()<bombCooldownUntil?`再使用まで${Math.ceil((bombCooldownUntil-performance.now())/1000)}秒`:'Eで設置 / Qで起爆'}</div>`;
  }
  function showNotice(text) { noticeEl.textContent=text; noticeEl.style.opacity=1; clearTimeout(noticeTimer); noticeTimer=setTimeout(()=>noticeEl.style.opacity=0,2600); }
  function spawnEnemy(forceGunner=false) {
    const edge = Math.floor(Math.random()*4), p = edge===0?{x:Math.random()*W,y:-30}:edge===1?{x:W+30,y:Math.random()*H}:edge===2?{x:Math.random()*W,y:H+30}:{x:-30,y:Math.random()*H};
    const bossMinion = bossActive;
    const gunner = !bossMinion && (forceGunner || ((performance.now()-startAt)/1000 >= 60 && Math.random() < .05));
    const soldier = !bossMinion && !gunner && kills >= 30 && Math.random() < .05;
    enemies.push({x:p.x,y:p.y,r:gunner?20:soldier?18:14,speed:(gunner?38:soldier?42:55)+Math.min(70,kills*1.4),hp:gunner?20:soldier?10:1,maxHp:gunner?20:soldier?10:1,soldier,gunner,nextFire:performance.now()+800,hit:0});
  }
  function spawnBoss() {
    enemies.push({x:W/2,y:-55,r:42,speed:32,hp:500,maxHp:500,soldier:false,boss:true,nextAoe:performance.now()+2800,nextLightning:performance.now()+3500,nextBossFire:performance.now()+700,hit:0});
    showNotice('👑 ボス出現！ 雑魚ゴキブリは少数だけ出現');
  }
  function shoot() {
    if (!running || performance.now()<nextShot) return;
    const pelletCount=weapon==='shotgun'?6:1, spread=weapon==='shotgun'?.42:0;
    if (ammo < pelletCount)
