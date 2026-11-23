<!DOCTYPE html>
<html lang="zh-TW">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
<title>霓虹突擊：無限競技場 (NEON STRIKE)</title>
<script src="https://unpkg.com/peerjs@1.5.2/dist/peerjs.min.js"></script>
<style>
  * { box-sizing: border-box; margin: 0; padding: 0; user-select: none; -webkit-user-select: none; touch-action: none; }
  body { background: #070913; color: #e2e8f0; font-family: 'Segoe UI', system-ui, -apple-system, sans-serif; overflow: hidden; width: 100vw; height: 100vh; }
  canvas { display: block; width: 100%; height: 100%; }

  /* UI 層級 */
  .screen { position: fixed; inset: 0; display: flex; flex-direction: column; z-index: 10; pointer-events: none; }
  .interactive { pointer-events: auto; }
  .hidden { display: none !important; }

  /* 大廳樣式 */
  #lobby-screen { background: radial-gradient(circle at center, rgba(7,9,19,0.55) 0%, rgba(7,9,19,0.92) 100%); padding: 16px; justify-content: space-between; overflow-y: auto; }
  .top-bar { display: flex; justify-content: space-between; align-items: center; gap: 12px; flex-wrap: wrap; }
  .title-box h1 { font-size: clamp(1.5rem, 4vw, 2.5rem); font-weight: 900; letter-spacing: 2px; background: linear-gradient(90deg, #00f0ff, #ff007f); -webkit-background-clip: text; -webkit-text-fill-color: transparent; text-shadow: 0 0 20px rgba(0,240,255,0.3); }
  .title-box p { font-size: 0.8rem; color: #94a3b8; }
  .stats-pill { background: rgba(15, 23, 42, 0.85); border: 1px solid rgba(0, 240, 255, 0.3); border-radius: 999px; padding: 6px 14px; font-size: 0.85rem; display: flex; gap: 12px; align-items: center; }
  .stats-pill input { background: transparent; border: none; border-bottom: 1px solid #00f0ff; color: #fff; width: 90px; font-weight: bold; outline: none; text-align: center; }

  .lobby-main { display: grid; grid-template-columns: 1.2fr 1fr; gap: 16px; max-width: 1100px; width: 100%; margin: auto; }
  @media (max-width: 768px) { .lobby-main { grid-template-columns: 1fr; } }

  .panel { background: rgba(15, 23, 42, 0.82); backdrop-filter: blur(10px); border: 1px solid rgba(148, 163, 184, 0.2); border-radius: 16px; padding: 16px; box-shadow: 0 10px 30px rgba(0,0,0,0.5); }
  .panel h2 { font-size: 1rem; color: #00f0ff; margin-bottom: 12px; display: flex; align-items: center; gap: 6px; text-transform: uppercase; letter-spacing: 1px; }

  .tabs { display: flex; gap: 8px; margin-bottom: 14px; }
  .tab-btn { flex: 1; padding: 10px; border-radius: 10px; border: 1px solid rgba(255,255,255,0.15); background: rgba(30,41,59,0.6); color: #cbd5e1; font-weight: bold; cursor: pointer; transition: 0.2s; }
  .tab-btn.active { background: linear-gradient(135deg, #0284c7, #00f0ff); color: #050811; border-color: #00f0ff; box-shadow: 0 0 15px rgba(0,240,255,0.4); }

  .control-group { margin-bottom: 12px; }
  .control-group label { display: block; font-size: 0.8rem; color: #94a3b8; margin-bottom: 6px; }
  .btn-row { display: flex; gap: 6px; flex-wrap: wrap; }
  .opt-btn { padding: 6px 12px; border-radius: 8px; border: 1px solid rgba(255,255,255,0.15); background: rgba(30,41,59,0.8); color: #e2e8f0; font-size: 0.85rem; cursor: pointer; }
  .opt-btn.active { background: #ff007f; border-color: #ff007f; color: #fff; font-weight: bold; }

  .room-box { display: flex; gap: 8px; margin-top: 8px; flex-wrap: wrap; }
  .room-input { flex: 1; min-width: 110px; padding: 10px; border-radius: 10px; border: 1px solid #334155; background: #0f172a; color: #fff; font-size: 1rem; text-align: center; letter-spacing: 2px; font-weight: bold; }
  .action-btn { padding: 10px 16px; border-radius: 10px; border: none; font-weight: bold; cursor: pointer; transition: 0.15s; font-size: 0.9rem; }
  .btn-cyan { background: #00f0ff; color: #050811; }
  .btn-pink { background: #ff007f; color: #fff; }
  .btn-amber { background: #f59e0b; color: #050811; }
  .btn-green { background: linear-gradient(135deg, #10b981, #059669); color: #fff; font-size: 1.1rem; width: 100%; padding: 14px; margin-top: 8px; box-shadow: 0 0 20px rgba(16,185,129,0.4); }
  .action-btn:active { transform: scale(0.96); }

  /* 聊天室 */
  .chat-container { display: flex; flex-direction: column; height: 220px; }
  .chat-messages { flex: 1; overflow-y: auto; background: rgba(7, 9, 19, 0.7); border-radius: 10px; padding: 8px; font-size: 0.85rem; display: flex; flex-direction: column; gap: 4px; margin-bottom: 8px; border: 1px solid rgba(255,255,255,0.08); }
  .chat-msg { line-height: 1.35; word-break: break-all; }
  .chat-msg.sys { color: #38bdf8; font-style: italic; }
  .chat-msg.cheat { color: #fbbf24; font-weight: bold; }
  .chat-input-row { display: flex; gap: 6px; }
  .chat-input-row input { flex: 1; padding: 8px 10px; border-radius: 8px; border: 1px solid #334155; background: #0f172a; color: #fff; font-size: 0.85rem; touch-action: auto; user-select: auto; }

  /* 規則卡 */
  .rules-grid { display: grid; grid-template-columns: repeat(4, 1fr); gap: 8px; max-width: 1100px; width: 100%; margin: 0 auto; }
  @media (max-width: 640px) { .rules-grid { grid-template-columns: repeat(2, 1fr); } }
  .rule-card { background: rgba(15, 23, 42, 0.75); border: 1px solid rgba(255,255,255,0.1); border-radius: 10px; padding: 8px 10px; font-size: 0.75rem; }
  .rule-card strong { color: #00f0ff; display: block; font-size: 0.85rem; margin-bottom: 2px; }

  /* 遊戲內 HUD */
  #hud-screen { justify-content: space-between; padding: 12px; }
  .hud-top { display: flex; justify-content: space-between; align-items: flex-start; gap: 8px; }
  .leaderboard { background: rgba(15, 23, 42, 0.75); border: 1px solid rgba(255,255,255,0.15); border-radius: 10px; padding: 8px 12px; font-size: 0.8rem; min-width: 170px; }
  .lb-row { display: flex; justify-content: space-between; gap: 10px; padding: 2px 0; }
  .lb-row.me { color: #00f0ff; font-weight: bold; }

  .hud-center-info { background: rgba(15, 23, 42, 0.8); border: 1px solid rgba(0,240,255,0.3); border-radius: 999px; padding: 6px 16px; font-weight: bold; font-size: 0.9rem; display: flex; gap: 12px; align-items: center; }
  .hud-actions { display: flex; gap: 6px; flex-wrap: wrap; justify-content: flex-end; }
  .mini-btn { background: rgba(15, 23, 42, 0.85); border: 1px solid rgba(255,255,255,0.25); color: #fff; padding: 6px 10px; border-radius: 8px; font-size: 0.8rem; font-weight: bold; cursor: pointer; }
  .mini-btn.cmd-btn { border-color: #fbbf24; color: #fbbf24; box-shadow: 0 0 10px rgba(251,191,36,0.4); }

  /* 局內聊天懸浮區 */
  .ingame-chat { position: fixed; top: 65px; left: 12px; width: 240px; max-width: 60vw; z-index: 15; }
  .ingame-chat-log { max-height: 95px; overflow-y: auto; background: rgba(7, 9, 19, 0.55); border-radius: 8px; padding: 6px; font-size: 0.75rem; margin-bottom: 4px; pointer-events: auto; }
  .quick-phrases { display: flex; gap: 4px; flex-wrap: wrap; margin-bottom: 4px; }
  .qp-btn { background: rgba(30,41,59,0.8); border: 1px solid rgba(255,255,255,0.2); color: #e2e8f0; font-size: 0.7rem; padding: 3px 6px; border-radius: 6px; cursor: pointer; }

  /* 觀戰列 */
  .spec-bar { position: fixed; bottom: 20px; left: 50%; transform: translateX(-50%); background: rgba(15,23,42,0.9); border: 1px solid #f59e0b; border-radius: 999px; padding: 8px 16px; display: flex; gap: 10px; align-items: center; z-index: 20; }

  /* 管理員作弊面板 */
  #admin-panel { position: fixed; top: 55px; right: 12px; width: 270px; max-height: 80vh; overflow-y: auto; background: rgba(10, 15, 30, 0.94); border: 1px solid #fbbf24; border-radius: 14px; padding: 12px; z-index: 50; box-shadow: 0 0 25px rgba(251,191,36,0.3); }
  #admin-panel h3 { color: #fbbf24; font-size: 0.95rem; margin-bottom: 8px; display: flex; justify-content: space-between; align-items: center; }
  .cheat-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 6px; }
  .cheat-toggle { padding: 7px 6px; border-radius: 8px; border: 1px solid #475569; background: #1e293b; color: #cbd5e1; font-size: 0.75rem; font-weight: bold; cursor: pointer; text-align: center; }
  .cheat-toggle.on { background: #fbbf24; color: #050811; border-color: #fef08a; box-shadow: 0 0 10px rgba(251,191,36,0.5); }

  /* 結算彈窗 */
  #gameover-modal { position: fixed; inset: 0; background: rgba(5, 8, 17, 0.85); display: flex; align-items: center; justify-content: center; z-index: 40; padding: 16px; }
  .modal-card { background: #0f172a; border: 2px solid #00f0ff; border-radius: 18px; padding: 24px; max-width: 400px; width: 100%; text-align: center; }
  .modal-card h2 { font-size: 1.8rem; color: #00f0ff; margin-bottom: 8px; }
  .modal-stats { background: rgba(30,41,59,0.6); border-radius: 12px; padding: 12px; margin: 14px 0; font-size: 0.9rem; display: flex; flex-direction: column; gap: 6px; }
</style>
</head>
<body>

<canvas id="gameCanvas"></canvas>

<!-- 大廳畫面 -->
<div id="lobby-screen" class="screen">
  <div class="top-bar interactive">
    <div class="title-box">
      <h1>NEON STRIKE // 無限競技場</h1>
      <p>三搖桿極速射擊 • 無限射程 • 命中回彈 +0.5</p>
    </div>
    <div class="stats-pill">
      <span>ID: <input type="text" id="player-name-input" value="特工_01" maxlength="10"></span>
      <span>勝場: <strong id="stat-wins" style="color:#10b981">0</strong></span>
      <span>擊殺: <strong id="stat-kills" style="color:#f59e0b">0</strong></span>
    </div>
  </div>

  <div class="lobby-main">
    <!-- 左側：對戰設定 -->
    <div class="panel interactive">
      <h2>⚔️ 競技場配置</h2>
      <div class="tabs">
        <button class="tab-btn active" id="tab-pve" onclick="switchModeTab('pve')">單人 PVE / 大亂鬥</button>
        <button class="tab-btn" id="tab-pvp" onclick="switchModeTab('pvp')">線上連線 PVP</button>
      </div>

      <!-- PVE 設定 -->
      <div id="pve-settings">
        <div class="control-group">
          <label>參戰人數（1v1v1... 多人混戰大亂鬥）</label>
          <div class="btn-row" id="bot-count-btns">
            <button class="opt-btn active" data-val="1" onclick="selectOpt('botCount', 1, this)">1v1 單挑 (2人)</button>
            <button class="opt-btn" data-val="3" onclick="selectOpt('botCount', 3, this)">4人混戰</button>
            <button class="opt-btn" data-val="7" onclick="selectOpt('botCount', 7, this)">8人大亂鬥</button>
            <button class="opt-btn" data-val="15" onclick="selectOpt('botCount', 15, this)">16人超大亂鬥</button>
          </div>
        </div>
        <div class="control-group">
          <label>AI 強度</label>
          <div class="btn-row" id="ai-diff-btns">
            <button class="opt-btn" data-val="easy" onclick="selectOpt('aiDiff', 'easy', this)">新兵</button>
            <button class="opt-btn active" data-val="normal" onclick="selectOpt('aiDiff', 'normal', this)">老手</button>
            <button class="opt-btn" data-val="ace" onclick="selectOpt('aiDiff', 'ace', this)">王牌</button>
          </div>
        </div>
      </div>

      <!-- PVP 連線設定 -->
      <div id="pvp-settings" class="hidden">
        <div class="control-group">
          <label>線上即時連線（支援 2 人至無上限多人同房大亂鬥）</label>
          <div class="room-box">
            <button class="action-btn btn-cyan" onclick="createOnlineRoom()">創建房間</button>
            <input type="text" id="join-room-code" class="room-input" placeholder="4位房號" maxlength="4">
            <button class="action-btn btn-pink" onclick="joinOnlineRoom(false)">加入參戰</button>
            <button class="action-btn btn-amber" onclick="joinOnlineRoom(true)">進入觀戰席</button>
          </div>
          <p id="pvp-status-text" style="font-size:0.8rem; color:#38bdf8; margin-top:8px;">狀態：未連線（輸入4位數房號即可與好友連線或觀戰）</p>
        </div>
      </div>

      <!-- 通用：復活次數設定 -->
      <div class="control-group">
        <label>每人復活次數上限（0 = 一命淘汰制，∞ = 無限死鬥模式）</label>
        <div class="btn-row" id="respawn-btns">
          <button class="opt-btn" onclick="selectOpt('maxRespawns', 0, this)">0 次 (淘汰)</button>
          <button class="opt-btn" onclick="selectOpt('maxRespawns', 1, this)">1 次</button>
          <button class="opt-btn active" onclick="selectOpt('maxRespawns', 3, this)">3 次</button>
          <button class="opt-btn" onclick="selectOpt('maxRespawns', 5, this)">5 次</button>
          <button class="opt-btn" onclick="selectOpt('maxRespawns', 999, this)">∞ 無限死鬥</button>
        </div>
      </div>

      <button class="action-btn btn-green" id="start-game-btn" onclick="startLocalOrHostGame()">🚀 立即進入戰場</button>
    </div>

    <!-- 右側：大廳聊天室 -->
    <div class="panel interactive">
      <h2>💬 戰術頻道 & 聊天室</h2>
      <div class="chat-container">
        <div class="chat-messages" id="lobby-chat-log"></div>
        <div class="chat-input-row">
          <input type="text" id="lobby-chat-input" placeholder="輸入訊息或代碼..." onkeydown="if(event.key==='Enter') sendChat('lobby')">
          <button class="action-btn btn-cyan" onclick="sendChat('lobby')">發送</button>
        </div>
      </div>
    </div>
  </div>

  <!-- 底部規則速覽 -->
  <div class="rules-grid">
    <div class="rule-card"><strong>❤️ 3 滴血 / 1 彈 1 血</strong>拾取隨機綠色急救包回復 1 滴血</div>
    <div class="rule-card"><strong>∞ 無限射程子彈</strong>子彈飛行距離無限，撞牆或出界消失</div>
    <div class="rule-card"><strong>🎯 命中 +0.5 發彈藥</strong>彈匣上限 3 發，神準連射滾雪球</div>
    <div class="rule-card"><strong>⚡ 1秒 5倍速衝刺</strong>冷卻 10 秒，期間可穿透子彈閃避</div>
  </div>
</div>

<!-- 遊戲內 HUD -->
<div id="hud-screen" class="screen hidden">
  <div class="hud-top">
    <div class="leaderboard interactive" id="leaderboard-box"></div>
    <div class="hud-center-info">
      <span id="hud-mode-label">混戰模式</span>
      <span id="hud-alive-label" style="color:#10b981">存活: 2</span>
    </div>
    <div class="hud-actions interactive">
      <button class="mini-btn cmd-btn hidden" id="hud-cmd-btn" onclick="toggleAdminPanel()">[CMD] 指令</button>
      <button class="mini-btn" id="hud-toggle-spec-btn" onclick="toggleSpectateSelf()">切換觀戰</button>
      <button class="mini-btn" onclick="toggleInGameChat()">💬 聊天</button>
      <button class="mini-btn" style="border-color:#ef4444;color:#fca5a5" onclick="returnToLobby()">離開</button>
    </div>
  </div>

  <!-- 局內微縮聊天室 -->
  <div class="ingame-chat" id="ingame-chat-box">
    <div class="ingame-chat-log" id="ingame-chat-log"></div>
    <div class="quick-phrases interactive">
      <button class="qp-btn" onclick="sendQuickChat('好準！')">好準！</button>
      <button class="qp-btn" onclick="sendQuickChat('差一點！')">差一點！</button>
      <button class="qp-btn" onclick="sendQuickChat('666')">666</button>
      <button class="qp-btn" onclick="sendQuickChat('再來一局！')">再來一局！</button>
    </div>
    <div class="chat-input-row interactive hidden" id="ingame-chat-input-wrap">
      <input type="text" id="ingame-chat-input" placeholder="輸入聊天或代碼..." onkeydown="if(event.key==='Enter') sendChat('ingame')">
      <button class="mini-btn btn-cyan" style="color:#000" onclick="sendChat('ingame')">送出</button>
    </div>
  </div>

  <!-- 觀戰控制列 -->
  <div class="spec-bar interactive hidden" id="spectator-bar">
    <span style="color:#f59e0b;font-weight:bold;font-size:0.85rem;">👁️ 觀戰席</span>
    <button class="mini-btn" onclick="cycleSpectateTarget(-1)">◀ 上一位</button>
    <span id="spec-target-name" style="font-size:0.85rem;min-width:80px;text-align:center;">上帝全景</span>
    <button class="mini-btn" onclick="cycleSpectateTarget(1)">下一位 ▶</button>
    <button class="mini-btn btn-amber" onclick="setGlobalOverview()">全景視角</button>
  </div>
</div>

<!-- 管理員作弊面板 (/YEE123456 解鎖) -->
<div id="admin-panel" class="interactive hidden">
  <h3>
    <span>⚡ 管理員指令面板</span>
    <button class="mini-btn" onclick="toggleAdminPanel()">✕</button>
  </h3>
  <div class="cheat-grid">
    <button class="cheat-toggle" id="cht-god" onclick="toggleCheat('god')">🛡️ 無敵金身</button>
    <button class="cheat-toggle" id="cht-infAmmo" onclick="toggleCheat('infAmmo')">🔥 無限彈藥</button>
    <button class="cheat-toggle" id="cht-noCdDash" onclick="toggleCheat('noCdDash')">⚡ 零冷卻衝刺</button>
    <button class="cheat-toggle" id="cht-speed" onclick="toggleCheat('speed')">👟 2倍極速</button>
    <button class="cheat-toggle" id="cht-autoAim" onclick="toggleCheat('autoAim')">🎯 自動瞄準</button>
    <button class="cheat-toggle" id="cht-homing" onclick="toggleCheat('homing')">🌀 子彈轉彎追蹤</button>
    <button class="cheat-toggle" id="cht-wallBullet" onclick="toggleCheat('wallBullet')">🧱 子彈穿牆</button>
    <button class="cheat-toggle" id="cht-wallWalk" onclick="toggleCheat('wallWalk')">👻 人物穿牆</button>
    <button class="cheat-toggle" id="cht-esp" onclick="toggleCheat('esp')">👁️ 全圖透視</button>
    <button class="cheat-toggle" id="cht-giant" onclick="toggleSizeCheat()">🦖 體型切換</button>
    <button class="cheat-toggle" onclick="spawnCheatHeals()">💚 召喚3補包</button>
    <button class="cheat-toggle" onclick="clearAllBullets()">🧹 清空全場彈幕</button>
    <button class="cheat-toggle" style="grid-column: span 2;" onclick="addExtraRespawns()">➕ 增加 +3 次復活機會</button>
  </div>
</div>

<!-- 結算畫面 -->
<div id="gameover-modal" class="hidden">
  <div class="modal-card interactive">
    <h2 id="go-title">勝利！</h2>
    <p id="go-subtitle" style="color:#94a3b8;font-size:0.9rem;">競技場最後生還者</p>
    <div class="modal-stats" id="go-stats"></div>
    <div style="display:flex;gap:10px;justify-content:center;">
      <button class="action-btn btn-cyan" style="flex:1" onclick="startLocalOrHostGame()">再來一局</button>
      <button class="action-btn btn-pink" style="flex:1" onclick="returnToLobby()">返回大廳</button>
    </div>
  </div>
</div>

<script>
/**
 * 霓虹突擊：無限競技場 (NEON STRIKE: INFINITE ARENA)
 * 核心數值規格：
 * - 基礎移速 V = 220 px/s
 * - 血量 = 3 滴，子彈傷害 = 1 滴
 * - 衝刺技能 = 1秒內移動 5秒距離 (5V = 1100px)，冷卻 10 秒，期間可穿透子彈
 * - 子彈射程 = 無限 (撞牆或地圖邊界消失)，速度 = 6.5V
 * - 彈藥上限 = 3 發，1.5秒裝填1發，射中敵人立即 +0.5 發
 */

const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

function resizeCanvas() {
  canvas.width = window.innerWidth;
  canvas.height = window.innerHeight;
}
window.addEventListener('resize', resizeCanvas);
resizeCanvas();

// 物理常數
const BASE_SPEED = 220; // V: 1秒移動距離
const BULLET_SPEED = BASE_SPEED * 6.5; // 子彈速度
const DASH_DURATION = 1.0; // 1秒衝刺
const DASH_DISTANCE = BASE_SPEED * 5; // 5秒移動距離
const DASH_SPEED = DASH_DISTANCE / DASH_DURATION; // 5V
const DASH_COOLDOWN = 10.0; // 10秒冷卻
const MAX_HP = 3;
const MAX_AMMO = 3;
const AMMO_RELOAD_RATE = 1 / 1.5; // 每1.5秒回復1發
const HIT_AMMO_BONUS = 0.5; // 射中增加0.5發

// 遊戲全域狀態
const state = {
  screen: 'lobby', // 'lobby' | 'game'
  modeTab: 'pve', // 'pve' | 'pvp'
  botCount: 1,
  aiDiff: 'normal',
  maxRespawns: 3,
  isOnline: false,
  isHost: true,
  isSpectator: false,
  specTargetId: null, // null 代表上帝全景視角
  roomCode: '',
  adminUnlocked: false,
  cheats: {
    god: false,
    infAmmo: false,
    noCdDash: false,
    speed: false,
    autoAim: false,
    homing: false,
    wallBullet: false,
    wallWalk: false,
    esp: false,
    sizeMode: 0 // 0: 正常, 1: 巨人(2x), 2: 迷你(0.5x)
  },
  stats: {
    wins: parseInt(localStorage.getItem('ns_wins') || '0'),
    kills: parseInt(localStorage.getItem('ns_kills') || '0')
  }
};

document.getElementById('stat-wins').textContent = state.stats.wins;
document.getElementById('stat-kills').textContent = state.stats.kills;

// 地圖與實體
let mapWidth = 2200;
let mapHeight = 1500;
let walls = [];
let players = {};
let bullets = [];
let heals = [];
let healWarnings = [];
let particles = [];
let floatingTexts = [];
let myId = 'player_local';
let healSpawnTimer = 5.0;
let camera = { x: 1100, y: 750, zoom: 1 };
let screenShake = 0;

// 顏色池
const COLORS = ['#00f0ff', '#ff007f', '#10b981', '#f59e0b', '#a855f7', '#3b82f6', '#ef4444', '#14b8a6'];

// 音效合成器 (Web Audio API)
const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
function playTone(freq, type, duration, vol = 0.1) {
  if (audioCtx.state === 'suspended') audioCtx.resume();
  const osc = audioCtx.createOscillator();
  const gain = audioCtx.createGain();
  osc.type = type;
  osc.frequency.setValueAtTime(freq, audioCtx.currentTime);
  osc.frequency.exponentialRampToValueAtTime(freq * 0.4, audioCtx.currentTime + duration);
  gain.gain.setValueAtTime(vol, audioCtx.currentTime);
  gain.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + duration);
  osc.connect(gain);
  gain.connect(audioCtx.destination);
  osc.start();
  osc.stop(audioCtx.currentTime + duration);
}

// 聊天室系統
function appendChat(sender, text, type = 'normal') {
  const logs = [document.getElementById('lobby-chat-log'), document.getElementById('ingame-chat-log')];
  logs.forEach(log => {
    const div = document.createElement('div');
    div.className = `chat-msg ${type}`;
    div.textContent = sender ? `[${sender}]: ${text}` : text;
    log.appendChild(div);
    log.scrollTop = log.scrollHeight;
  });
}

function sendChat(source) {
  const input = document.getElementById(source === 'lobby' ? 'lobby-chat-input' : 'ingame-chat-input');
  const raw = input.value.trim();
  if (!raw) return;
  input.value = '';

  const myName = document.getElementById('player-name-input').value.trim() || '特工';

  // 檢查作弊代碼 /YEE123456
  if (raw === '/YEE123456') {
    const masked = '/#########';
    appendChat(myName, masked, 'cheat');
    if (state.isOnline) broadcastNet({ type: 'chat', sender: myName, text: masked });
    if (!state.adminUnlocked) {
      state.adminUnlocked = true;
      document.getElementById('hud-cmd-btn').classList.remove('hidden');
      appendChat('', '🔓 [系統] 管理員權限已解鎖！點擊右上角 [CMD] 開啟指令面板。', 'cheat');
      playTone(880, 'sine', 0.3, 0.15);
    }
    toggleAdminPanel(true);
    return;
  }

  appendChat(myName, raw, 'normal');
  if (state.isOnline) {
    broadcastNet({ type: 'chat', sender: myName, text: raw });
  }
}

function sendQuickChat(phrase) {
  const myName = document.getElementById('player-name-input').value.trim() || '特工';
  appendChat(myName, phrase, 'normal');
  if (state.isOnline) {
    broadcastNet({ type: 'chat', sender: myName, text: phrase });
  }
}

function toggleInGameChat() {
  const wrap = document.getElementById('ingame-chat-input-wrap');
  wrap.classList.toggle('hidden');
  if (!wrap.classList.contains('hidden')) {
    document.getElementById('ingame-chat-input').focus();
  }
}

// 管理員作弊面板控制
function toggleAdminPanel(forceShow) {
  const panel = document.getElementById('admin-panel');
  if (typeof forceShow === 'boolean') {
    panel.classList.toggle('hidden', !forceShow);
  } else {
    panel.classList.toggle('hidden');
  }
}

function toggleCheat(key) {
  state.cheats[key] = !state.cheats[key];
  const btn = document.getElementById('cht-' + key);
  if (btn) btn.classList.toggle('on', state.cheats[key]);
  const me = players[myId];
  if (me && key === 'god' && state.cheats.god) me.hp = me.maxHp;
  if (me && key === 'infAmmo' && state.cheats.infAmmo) me.ammo = MAX_AMMO;
  if (me && key === 'noCdDash' && state.cheats.noCdDash) me.dashCd = 0;
}

function toggleSizeCheat() {
  state.cheats.sizeMode = (state.cheats.sizeMode + 1) % 3;
  const labels = ['🦖 體型:正常', '🦖 體型:巨人2X', '🐜 體型:迷你0.5X'];
  const btn = document.getElementById('cht-giant');
  btn.textContent = labels[state.cheats.sizeMode];
  btn.classList.toggle('on', state.cheats.sizeMode !== 0);
  const me = players[myId];
  if (me) {
    if (state.cheats.sizeMode === 1) { me.radius = 44; me.maxHp = 6; me.hp = 6; }
    else if (state.cheats.sizeMode === 2) { me.radius = 12; me.maxHp = 3; me.hp = Math.min(me.hp, 3); }
    else { me.radius = 22; me.maxHp = 3; me.hp = Math.min(me.hp, 3); }
  }
}

function spawnCheatHeals() {
  const me = players[myId];
  const bx = me ? me.x : mapWidth / 2;
  const by = me ? me.y : mapHeight / 2;
  for (let i = 0; i < 3; i++) {
    heals.push({
      id: 'h_' + Math.random(),
      x: Math.max(80, Math.min(mapWidth - 80, bx + (Math.random() - 0.5) * 180)),
      y: Math.max(80, Math.min(mapHeight - 80, by + (Math.random() - 0.5) * 180))
    });
  }
  appendChat('', '💚 [指令] 已召喚 3 個回血急救包！', 'cheat');
}

function clearAllBullets() {
  bullets = [];
  appendChat('', '🧹 [指令] 已清空全場子彈！', 'cheat');
}

function addExtraRespawns() {
  const me = players[myId];
  if (me) {
    me.respawnsLeft += 3;
    appendChat('', `➕ [指令] 剩餘復活次數增加至 ${me.respawnsLeft} 次！`, 'cheat');
  }
}

// 大廳 UI 互動
function switchModeTab(tab) {
  state.modeTab = tab;
  document.getElementById('tab-pve').classList.toggle('active', tab === 'pve');
  document.getElementById('tab-pvp').classList.toggle('active', tab === 'pvp');
  document.getElementById('pve-settings').classList.toggle('hidden', tab !== 'pve');
  document.getElementById('pvp-settings').classList.toggle('hidden', tab !== 'pvp');
  document.getElementById('start-game-btn').classList.toggle('hidden', tab === 'pvp' && !state.isOnline);
}

function selectOpt(key, val, btn) {
  state[key] = val;
  Array.from(btn.parentElement.children).forEach(b => b.classList.remove('active'));
  btn.classList.add('active');
}

// PeerJS 線上連線模組
let peer = null;
let connections = []; // Host 儲存所有 Client 連線
let hostConn = null;    // Client 儲存對 Host 的連線

function createOnlineRoom() {
  const code = Math.floor(1000 + Math.random() * 9000).toString();
  state.roomCode = code;
  state.isOnline = true;
  state.isHost = true;
  state.isSpectator = false;
  const peerId = 'neon_strike_room_' + code;

  document.getElementById('pvp-status-text').textContent = `正在建立房間 ${code}...`;
  if (peer) peer.destroy();
  peer = new Peer(peerId);

  peer.on('open', () => {
    document.getElementById('pvp-status-text').innerHTML = `✅ 房間已建立！房號：<strong style="font-size:1.2rem;color:#fbbf24">${code}</strong>（等待玩家加入，也可直接按下方開始）`;
    document.getElementById('start-game-btn').classList.remove('hidden');
    appendChat('', `🌐 已創建線上房間 [${code}]，分享房號給好友即可加入對戰或觀戰！`, 'sys');
  });

  peer.on('connection', conn => {
    connections.push(conn);
    conn.on('data', data => handleNetDataAsHost(conn, data));
    conn.on('close', () => {
      connections = connections.filter(c => c !== conn);
      if (conn.playerId && players[conn.playerId]) {
        appendChat('', `⚠️ ${players[conn.playerId].name} 離開了房間`, 'sys');
        delete players[conn.playerId];
      }
    });
  });

  peer.on('error', err => {
    document.getElementById('pvp-status-text').textContent = `連線錯誤: ${err.type}，請重試`;
  });
}

function joinOnlineRoom(asSpectator) {
  const code = document.getElementById('join-room-code').value.trim();
  if (code.length !== 4) {
    alert('請輸入 4 位數房間代碼！');
    return;
  }
  state.roomCode = code;
  state.isOnline = true;
  state.isHost = false;
  state.isSpectator = asSpectator;

  document.getElementById('pvp-status-text').textContent = `正在連線至房間 ${code}...`;
  if (peer) peer.destroy();
  peer = new Peer();

  peer.on('open', id => {
    myId = id;
    hostConn = peer.connect('neon_strike_room_' + code);
    hostConn.on('open', () => {
      const myName = document.getElementById('player-name-input').value.trim() || '特工';
      hostConn.send({
        type: 'join',
        id: myId,
        name: myName,
        isSpectator: asSpectator
      });
      enterGameUI();
    });
    hostConn.on('data', data => handleNetDataAsClient(data));
    hostConn.on('close', () => {
      alert('與房主斷開連線');
      returnToLobby();
    });
  });

  peer.on('error', () => {
    document.getElementById('pvp-status-text').textContent = `找不到房間 ${code}，請確認房號是否正確`;
  });
}

function broadcastNet(msg) {
  if (!state.isOnline) return;
  if (state.isHost) {
    connections.forEach(c => { if (c.open) c.send(msg); });
  } else if (hostConn && hostConn.open) {
    hostConn.send(msg);
  }
}

function handleNetDataAsHost(conn, data) {
  if (data.type === 'join') {
    conn.playerId = data.id;
    const p = createPlayerEntity(data.id, data.name, false, data.isSpectator);
    players[data.id] = p;
    appendChat('', `👋 ${data.name} ${data.isSpectator ? '進入了觀戰席' : '加入了戰場'}！`, 'sys');
    broadcastNet({ type: 'chat', sender: '', text: `👋 ${data.name} ${data.isSpectator ? '進入了觀戰席' : '加入了戰場'}！` });
    conn.send({ type: 'init_world', walls, mapWidth, mapHeight, maxRespawns: state.maxRespawns });
  } else if (data.type === 'input') {
    const p = players[data.id];
    if (p) {
      p.x = data.x; p.y = data.y; p.angle = data.angle;
      p.hp = data.hp; p.ammo = data.ammo; p.isDashing = data.isDashing;
      p.dead = data.dead; p.isSpectator = data.isSpectator; p.radius = data.radius;
    }
  } else if (data.type === 'shoot') {
    spawnBullet(data.ownerId, data.x, data.y, data.angle, data.homing, data.wallBullet);
    broadcastNet(data);
  } else if (data.type === 'chat') {
    appendChat(data.sender, data.text);
    broadcastNet(data);
  }
}

function handleNetDataAsClient(data) {
  if (data.type === 'init_world') {
    walls = data.walls;
    mapWidth = data.mapWidth;
    mapHeight = data.mapHeight;
    state.maxRespawns = data.maxRespawns;
  } else if (data.type === 'sync_state') {
    // 同步其他玩家與道具
    for (let id in data.players) {
      if (id === myId) {
        // 同步擊殺數與被擊殺判定
        if (players[myId]) {
          players[myId].kills = data.players[myId].kills;
        }
        continue;
      }
      players[id] = data.players[id];
    }
    for (let id in players) {
      if (!data.players[id]) delete players[id];
    }
    heals = data.heals;
    healWarnings = data.healWarnings;
  } else if (data.type === 'shoot') {
    if (data.ownerId !== myId) {
      spawnBullet(data.ownerId, data.x, data.y, data.angle, data.homing, data.wallBullet);
    }
  } else if (data.type === 'hit_confirm') {
    if (data.targetId === myId && players[myId]) {
      damagePlayer(players[myId], data.shooterId);
    }
    if (data.shooterId === myId && players[myId]) {
      players[myId].ammo = Math.min(MAX_AMMO, players[myId].ammo + HIT_AMMO_BONUS);
      spawnFloatingText(players[myId].x, players[myId].y - 35, '+0.5 彈藥', '#00f0ff');
    }
  } else if (data.type === 'chat') {
    appendChat(data.sender, data.text);
  }
}

// 地圖生成器
function buildArenaMap(totalPlayers) {
  const isLarge = totalPlayers > 4;
  mapWidth = isLarge ? 3000 : 2000;
  mapHeight = isLarge ? 2200 : 1400;
  walls = [];

  // 外圍邊界牆
  const t = 40;
  walls.push({ x: 0, y: 0, w: mapWidth, h: t });
  walls.push({ x: 0, y: mapHeight - t, w: mapWidth, h: t });
  walls.push({ x: 0, y: 0, w: t, h: mapHeight });
  walls.push({ x: mapWidth - t, y: 0, w: t, h: mapHeight });

  // 對稱掩體群
  const cols = isLarge ? 4 : 3;
  const rows = isLarge ? 3 : 2;
  const stepX = (mapWidth - 400) / cols;
  const stepY = (mapHeight - 300) / rows;

  for (let r = 0; r < rows; r++) {
    for (let c = 0; c < cols; c++) {
      const cx = 250 + c * stepX + stepX * 0.35;
      const cy = 200 + r * stepY + stepY * 0.35;
      if ((r + c) % 2 === 0) {
        // L型掩體
        walls.push({ x: cx - 70, y: cy - 18, w: 140, h: 36 });
        walls.push({ x: cx - 18, y: cy - 70, w: 36, h: 140 });
      } else {
        // 橫向雙掩體
        walls.push({ x: cx - 85, y: cy - 50, w: 170, h: 32 });
        walls.push({ x: cx - 85, y: cy + 25, w: 170, h: 32 });
      }
    }
  }
}

function getSafeSpawnPoint() {
  for (let attempt = 0; attempt < 40; attempt++) {
    const x = 140 + Math.random() * (mapWidth - 280);
    const y = 140 + Math.random() * (mapHeight - 280);
    if (!circleIntersectsWalls(x, y, 45)) {
      return { x, y };
    }
  }
  return { x: mapWidth / 2, y: mapHeight / 2 };
}

function createPlayerEntity(id, name, isBot, isSpectator = false) {
  const pos = getSafeSpawnPoint();
  const color = COLORS[Object.keys(players).length % COLORS.length];
  return {
    id, name, isBot, isSpectator,
    x: pos.x, y: pos.y,
    vx: 0, vy: 0,
    angle: 0,
    radius: 22,
    color,
    hp: MAX_HP,
    maxHp: MAX_HP,
    ammo: MAX_AMMO,
    dashCd: 0,
    dashTimer: 0,
    dashDirX: 0,
    dashDirY: 0,
    invulnTimer: 2.0, // 出生2秒保護
    dead: false,
    respawnTimer: 0,
    respawnsLeft: state.maxRespawns,
    kills: 0,
    shotsFired: 0,
    shotsHit: 0,
    aiTimer: 0,
    aiMoveAngle: Math.random() * Math.PI * 2
  };
}

// 初始化大廳背景演武
function setupLobbyDemo() {
  state.screen = 'lobby';
  buildArenaMap(4);
  players = {};
  bullets = [];
  heals = [];
  healWarnings = [];
  for (let i = 0; i < 6; i++) {
    const botId = 'demo_bot_' + i;
    players[botId] = createPlayerEntity(botId, '演武AI_' + (i + 1), true, false);
    players[botId].respawnsLeft = 999;
  }
}

// 開始正式對戰
function startLocalOrHostGame() {
  document.getElementById('gameover-modal').classList.add('hidden');
  if (!state.isOnline) {
    myId = 'player_local';
    state.isHost = true;
    state.isSpectator = false;
  }

  const totalCount = state.isOnline ? Object.keys(players).length + 1 : state.botCount + 1;
  buildArenaMap(totalCount);

  const oldPlayers = { ...players };
  players = {};
  bullets = [];
  heals = [];
  healWarnings = [];
  healSpawnTimer = 5.0;

  const myName = document.getElementById('player-name-input').value.trim() || '特工_01';
  players[myId] = createPlayerEntity(myId, myName, false, state.isSpectator);
  if (state.cheats.sizeMode === 1) { players[myId].radius = 44; players[myId].maxHp = 6; players[myId].hp = 6; }
  if (state.cheats.sizeMode === 2) { players[myId].radius = 12; }

  if (state.isOnline && state.isHost) {
    for (let id in oldPlayers) {
      if (id !== myId && !oldPlayers[id].isBot) {
        players[id] = createPlayerEntity(id, oldPlayers[id].name, false, oldPlayers[id].isSpectator);
      }
    }
    broadcastNet({ type: 'init_world', walls, mapWidth, mapHeight, maxRespawns: state.maxRespawns });
  } else if (!state.isOnline) {
    for (let i = 0; i < state.botCount; i++) {
      const botId = 'bot_' + i;
      players[botId] = createPlayerEntity(botId, 'AI戰士_' + (i + 1), true, false);
    }
  }

  enterGameUI();
}

function enterGameUI() {
  state.screen = 'game';
  document.getElementById('lobby-screen').classList.add('hidden');
  document.getElementById('hud-screen').classList.remove('hidden');
  document.getElementById('hud-mode-label').textContent =
    state.maxRespawns >= 999 ? '極限死鬥 (∞復活)' :
    state.maxRespawns === 0 ? '經典淘汰 (0復活)' : `競技場 (${state.maxRespawns}次復活)`;
  updateSpectatorUI();
}

function returnToLobby() {
  document.getElementById('gameover-modal').classList.add('hidden');
  document.getElementById('hud-screen').classList.add('hidden');
  document.getElementById('admin-panel').classList.add('hidden');
  document.getElementById('lobby-screen').classList.remove('hidden');
  state.isOnline = false;
  setupLobbyDemo();
}

// 觀戰系統控制
function toggleSpectateSelf() {
  const me = players[myId];
  if (!me) return;
  me.isSpectator = !me.isSpectator;
  state.isSpectator = me.isSpectator;
  if (!me.isSpectator) {
    const pos = getSafeSpawnPoint();
    me.x = pos.x; me.y = pos.y; me.hp = me.maxHp; me.dead = false; me.invulnTimer = 2.0;
    state.specTargetId = null;
  }
  updateSpectatorUI();
}

function updateSpectatorUI() {
  const me = players[myId];
  const isSpec = !me || me.isSpectator || (me.dead && me.respawnsLeft <= 0);
  document.getElementById('spectator-bar').classList.toggle('hidden', !isSpec);
}

function cycleSpectateTarget(dir) {
  const activeIds = Object.keys(players).filter(id => !players[id].isSpectator && !players[id].dead);
  if (activeIds.length === 0) {
    state.specTargetId = null;
    document.getElementById('spec-target-name').textContent = '上帝全景';
    return;
  }
  let idx = activeIds.indexOf(state.specTargetId);
  idx = (idx + dir + activeIds.length) % activeIds.length;
  state.specTargetId = activeIds[idx];
  document.getElementById('spec-target-name').textContent = players[state.specTargetId].name;
}

function setGlobalOverview() {
  state.specTargetId = null;
  document.getElementById('spec-target-name').textContent = '上帝全景';
}

// ==========================================
// 三搖桿觸控與滑鼠鍵盤控制系統
// 左搖桿：移動
// 右下搖桿：瞄準雷射線，放開射擊
// 右上搖桿：衝刺方向預覽，放開衝刺
// ==========================================
const joysticks = {
  move: { active: false, touchId: null, originX: 0, originY: 0, curX: 0, curY: 0, dx: 0, dy: 0 },
  aim:  { active: false, touchId: null, baseX: 0, baseY: 0, curX: 0, curY: 0, dx: 0, dy: 0, angle: 0, valid: false },
  dash: { active: false, touchId: null, baseX: 0, baseY: 0, curX: 0, curY: 0, dx: 0, dy: 0, angle: 0, valid: false }
};

function getRightJoystickCenters() {
  const w = canvas.width;
  const h = canvas.height;
  return {
    aimX: w - 115,
    aimY: h - 105,
    dashX: w - 95,
    dashY: h - 245,
    radius: 52
  };
}

// 觸控事件
canvas.addEventListener('touchstart', e => {
  if (state.screen !== 'game') return;
  const me = players[myId];
  if (!me || me.isSpectator || me.dead) return;
  e.preventDefault();
  const centers = getRightJoystickCenters();

  for (let i = 0; i < e.changedTouches.length; i++) {
    const t = e.changedTouches[i];
    const tx = t.clientX;
    const ty = t.clientY;

    // 判斷是否按在右上「衝刺技能搖桿」附近
    if (!joysticks.dash.active && Math.hypot(tx - centers.dashX, ty - centers.dashY) <= 68) {
      joysticks.dash.active = true;
      joysticks.dash.touchId = t.identifier;
      joysticks.dash.baseX = centers.dashX;
      joysticks.dash.baseY = centers.dashY;
      joysticks.dash.curX = tx;
      joysticks.dash.curY = ty;
      joysticks.dash.valid = false;
      continue;
    }

    // 判斷是否按在右半邊（瞄準射擊搖桿）
    if (tx >= canvas.width * 0.48 && !joysticks.aim.active) {
      joysticks.aim.active = true;
      joysticks.aim.touchId = t.identifier;
      joysticks.aim.baseX = Math.hypot(tx - centers.aimX, ty - centers.aimY) < 90 ? centers.aimX : tx;
      joysticks.aim.baseY = Math.hypot(tx - centers.aimX, ty - centers.aimY) < 90 ? centers.aimY : ty;
      joysticks.aim.curX = tx;
      joysticks.aim.curY = ty;
      joysticks.aim.valid = false;
      continue;
    }

    // 左半邊（浮動移動搖桿）
    if (tx < canvas.width * 0.48 && !joysticks.move.active) {
      joysticks.move.active = true;
      joysticks.move.touchId = t.identifier;
      joysticks.move.originX = tx;
      joysticks.move.originY = ty;
      joysticks.move.curX = tx;
      joysticks.move.curY = ty;
      joysticks.move.dx = 0;
      joysticks.move.dy = 0;
    }
  }
}, { passive: false });

canvas.addEventListener('touchmove', e => {
  if (state.screen !== 'game') return;
  e.preventDefault();
  const maxR = 52;

  for (let i = 0; i < e.changedTouches.length; i++) {
    const t = e.changedTouches[i];
    if (joysticks.move.active && t.identifier === joysticks.move.touchId) {
      joysticks.move.curX = t.clientX;
      joysticks.move.curY = t.clientY;
      const vx = t.clientX - joysticks.move.originX;
      const vy = t.clientY - joysticks.move.originY;
      const dist = Math.hypot(vx, vy);
      if (dist > 4) {
        const clamped = Math.min(dist, maxR);
        joysticks.move.dx = (vx / dist) * (clamped / maxR);
        joysticks.move.dy = (vy / dist) * (clamped / maxR);
      } else {
        joysticks.move.dx = 0;
        joysticks.move.dy = 0;
      }
    } else if (joysticks.aim.active && t.identifier === joysticks.aim.touchId) {
      joysticks.aim.curX = t.clientX;
      joysticks.aim.curY = t.clientY;
      const vx = t.clientX - joysticks.aim.baseX;
      const vy = t.clientY - joysticks.aim.baseY;
      const dist = Math.hypot(vx, vy);
      joysticks.aim.valid = dist > 12;
      if (dist > 4) joysticks.aim.angle = Math.atan2(vy, vx);
    } else if (joysticks.dash.active && t.identifier === joysticks.dash.touchId) {
      joysticks.dash.curX = t.clientX;
      joysticks.dash.curY = t.clientY;
      const vx = t.clientX - joysticks.dash.baseX;
      const vy = t.clientY - joysticks.dash.baseY;
      const dist = Math.hypot(vx, vy);
      joysticks.dash.valid = dist > 10;
      if (dist > 4) joysticks.dash.angle = Math.atan2(vy, vx);
    }
  }
}, { passive: false });

function handleTouchEndOrCancel(e) {
  const me = players[myId];
  for (let i = 0; i < e.changedTouches.length; i++) {
    const t = e.changedTouches[i];
    if (joysticks.move.active && t.identifier === joysticks.move.touchId) {
      joysticks.move.active = false;
      joysticks.move.dx = 0;
      joysticks.move.dy = 0;
    } else if (joysticks.aim.active && t.identifier === joysticks.aim.touchId) {
      if (me && !me.dead && !me.isSpectator && (joysticks.aim.valid || state.cheats.autoAim)) {
        const shootAngle = getFinalAimAngle(me, joysticks.aim.angle);
        tryPlayerShoot(me, shootAngle);
      }
      joysticks.aim.active = false;
      joysticks.aim.valid = false;
    } else if (joysticks.dash.active && t.identifier === joysticks.dash.touchId) {
      if (me && !me.dead && !me.isSpectator) {
        const dashAng = joysticks.dash.valid ? joysticks.dash.angle :
          (Math.hypot(joysticks.move.dx, joysticks.move.dy) > 0.1 ? Math.atan2(joysticks.move.dy, joysticks.move.dx) : me.angle);
        tryPlayerDash(me, dashAng);
      }
      joysticks.dash.active = false;
      joysticks.dash.valid = false;
    }
  }
}
canvas.addEventListener('touchend', handleTouchEndOrCancel);
canvas.addEventListener('touchcancel', handleTouchEndOrCancel);

// 電腦鍵盤 + 滑鼠支援（方便桌面測試與遊玩）
const keys = {};
let mouseAiming = false;
let mouseScreen = { x: 0, y: 0 };

window.addEventListener('keydown', e => {
  if (document.activeElement && document.activeElement.tagName === 'INPUT') return;
  keys[e.key.toLowerCase()] = true;
  if (e.code === 'Space' && state.screen === 'game') {
    const me = players[myId];
    if (me && !me.dead && !me.isSpectator) {
      const worldMouseX = (mouseScreen.x - canvas.width / 2) / camera.zoom + camera.x;
      const worldMouseY = (mouseScreen.y - canvas.height / 2) / camera.zoom + camera.y;
      const ang = Math.atan2(worldMouseY - me.y, worldMouseX - me.x);
      tryPlayerDash(me, ang);
    }
  }
});
window.addEventListener('keyup', e => { keys[e.key.toLowerCase()] = false; });

canvas.addEventListener('mousedown', e => {
  if (state.screen !== 'game') return;
  const me = players[myId];
  if (!me || me.dead || me.isSpectator) return;
  const centers = getRightJoystickCenters();
  mouseScreen = { x: e.clientX, y: e.clientY };

  // 點擊右上衝刺按鈕
  if (Math.hypot(e.clientX - centers.dashX, e.clientY - centers.dashY) <= 55) {
    tryPlayerDash(me, me.angle);
    return;
  }
  mouseAiming = true;
});

canvas.addEventListener('mousemove', e => {
  mouseScreen = { x: e.clientX, y: e.clientY };
});

canvas.addEventListener('mouseup', () => {
  if (!mouseAiming) return;
  mouseAiming = false;
  const me = players[myId];
  if (me && !me.dead && !me.isSpectator && state.screen === 'game') {
    const worldMouseX = (mouseScreen.x - canvas.width / 2) / camera.zoom + camera.x;
    const worldMouseY = (mouseScreen.y - canvas.height / 2) / camera.zoom + camera.y;
    const rawAng = Math.atan2(worldMouseY - me.y, worldMouseX - me.x);
    tryPlayerShoot(me, getFinalAimAngle(me, rawAng));
  }
});

// 自動瞄準計算
function getFinalAimAngle(shooter, fallbackAngle) {
  if (shooter.id === myId && state.cheats.autoAim) {
    let bestTarget = null;
    let bestDist = Infinity;
    for (let id in players) {
      const p = players[id];
      if (id === shooter.id || p.dead || p.isSpectator) continue;
      const d = Math.hypot(p.x - shooter.x, p.y - shooter.y);
      if (d < bestDist) {
        bestDist = d;
        bestTarget = p;
      }
    }
    if (bestTarget) {
      // 基礎移動預判
      const timeToHit = bestDist / BULLET_SPEED;
      const predX = bestTarget.x + bestTarget.vx * timeToHit * 0.6;
      const predY = bestTarget.y + bestTarget.vy * timeToHit * 0.6;
      return Math.atan2(predY - shooter.y, predX - shooter.x);
    }
  }
  return fallbackAngle;
}

// 射擊與衝刺動作
function tryPlayerShoot(p, angle) {
  const isInfAmmo = (p.id === myId && state.cheats.infAmmo);
  if (!isInfAmmo && p.ammo < 1.0) return;

  if (!isInfAmmo) p.ammo -= 1.0;
  p.angle = angle;
  p.shotsFired++;

  const isHoming = (p.id === myId && state.cheats.homing);
  const isWallBullet = (p.id === myId && state.cheats.wallBullet);

  const muzzleX = p.x + Math.cos(angle) * (p.radius + 8);
  const muzzleY = p.y + Math.sin(angle) * (p.radius + 8);

  spawnBullet(p.id, muzzleX, muzzleY, angle, isHoming, isWallBullet);
  if (p.id === myId) {
    playTone(520, 'triangle', 0.12, 0.08);
    screenShake = 4;
    if (state.isOnline) {
      broadcastNet({ type: 'shoot', ownerId: p.id, x: muzzleX, y: muzzleY, angle, homing: isHoming, wallBullet: isWallBullet });
    }
  }
}

function spawnBullet(ownerId, x, y, angle, homing = false, wallBullet = false) {
  const owner = players[ownerId];
  bullets.push({
    ownerId,
    x, y,
    vx: Math.cos(angle) * BULLET_SPEED,
    vy: Math.sin(angle) * BULLET_SPEED,
    angle,
    color: homing ? '#fbbf24' : (owner ? owner.color : '#00f0ff'),
    homing,
    wallBullet,
    life: 6.0 // 無限射程（飛出地圖邊界或撞牆銷毀）
  });
}

function tryPlayerDash(p, angle) {
  const noCd = (p.id === myId && state.cheats.noCdDash);
  if (!noCd && p.dashCd > 0) return;
  if (p.dashTimer > 0) return;

  p.angle = angle;
  p.dashTimer = DASH_DURATION; // 1秒內衝刺5秒距離
  p.dashCd = noCd ? 0 : DASH_COOLDOWN; // 10秒冷卻
  p.dashDirX = Math.cos(angle);
  p.dashDirY = Math.sin(angle);
  if (p.id === myId) playTone(320, 'sawtooth', 0.25, 0.08);
}

// 碰撞檢測工具
function circleIntersectsWalls(cx, cy, r) {
  for (let w of walls) {
    const closestX = Math.max(w.x, Math.min(cx, w.x + w.w));
    const closestY = Math.max(w.y, Math.min(cy, w.y + w.h));
    if (Math.hypot(cx - closestX, cy - closestY) < r) return true;
  }
  return false;
}

function resolveWallCollision(p) {
  if (p.id === myId && state.cheats.wallWalk) {
    // 人物穿牆仍限制在地圖最外框內
    p.x = Math.max(p.radius, Math.min(mapWidth - p.radius, p.x));
    p.y = Math.max(p.radius, Math.min(mapHeight - p.radius, p.y));
    return;
  }
  for (let w of walls) {
    const closestX = Math.max(w.x, Math.min(p.x, w.x + w.w));
    const closestY = Math.max(w.y, Math.min(p.y, w.y + w.h));
    const dx = p.x - closestX;
    const dy = p.y - closestY;
    const dist = Math.hypot(dx, dy);
    if (dist < p.radius) {
      if (dist > 0) {
        const overlap = p.radius - dist;
        p.x += (dx / dist) * overlap;
        p.y += (dy / dist) * overlap;
      } else {
        p.x += p.radius;
      }
    }
  }
}

// 射線檢測（用於雷射瞄準線落點計算）
function castLaserRay(startX, startY, angle, ignoreWalls = false) {
  const step = 12;
  const dx = Math.cos(angle) * step;
  const dy = Math.sin(angle) * step;
  let cx = startX;
  let cy = startY;
  for (let i = 0; i < 300; i++) {
    cx += dx;
    cy += dy;
    if (cx <= 0 || cx >= mapWidth || cy <= 0 || cy >= mapHeight) {
      return { x: cx, y: cy, hitType: 'wall' };
    }
    if (!ignoreWalls) {
      for (let w of walls) {
        if (cx >= w.x && cx <= w.x + w.w && cy >= w.y && cy <= w.y + w.h) {
          return { x: cx, y: cy, hitType: 'wall' };
        }
      }
    }
    for (let id in players) {
      const p = players[id];
      if (id === myId || p.dead || p.isSpectator) continue;
      if (Math.hypot(cx - p.x, cy - p.y) <= p.radius) {
        return { x: cx, y: cy, hitType: 'enemy' };
      }
    }
  }
  return { x: cx, y: cy, hitType: 'none' };
}

function spawnParticles(x, y, color, count = 12) {
  for (let i = 0; i < count; i++) {
    const ang = Math.random() * Math.PI * 2;
    const spd = 60 + Math.random() * 240;
    particles.push({
      x, y,
      vx: Math.cos(ang) * spd,
      vy: Math.sin(ang) * spd,
      color,
      life: 0.35 + Math.random() * 0.25,
      maxLife: 0.6
    });
  }
}

function spawnFloatingText(x, y, text, color) {
  floatingTexts.push({ x, y, text, color, life: 1.0 });
}

function damagePlayer(target, shooterId) {
  if (target.dead || target.invulnTimer > 0) return;
  if (target.id === myId && state.cheats.god) return;

  target.hp -= 1;
  target.invulnTimer = 0.45; // 短暫受擊無敵避免同幀重疊
  spawnParticles(target.x, target.y, target.color, 16);
  if (target.id === myId) {
    screenShake = 10;
    playTone(160, 'sawtooth', 0.25, 0.15);
  }

  if (target.hp <= 0) {
    target.hp = 0;
    target.dead = true;
    spawnParticles(target.x, target.y, '#ff007f', 30);

    const killer = players[shooterId];
    if (killer) {
      killer.kills++;
      if (killer.id === myId && state.screen === 'game') {
        state.stats.kills++;
        localStorage.setItem('ns_kills', state.stats.kills);
        document.getElementById('stat-kills').textContent = state.stats.kills;
      }
    }

    if (state.screen === 'game') {
      appendChat('', `💀 ${killer ? killer.name : '未知'} 擊殺了 ${target.name}！`, 'sys');
    }

    // 判斷復活次數
    if (target.respawnsLeft > 0) {
      if (target.respawnsLeft < 999) target.respawnsLeft--;
      target.respawnTimer = 2.5;
    } else {
      target.respawnTimer = -1; // 徹底淘汰
      if (target.id === myId && state.screen === 'game') {
        appendChat('', '👁️ 復活次數已耗盡，已切換至觀戰席！', 'sys');
        updateSpectatorUI();
      }
    }
  }
}

// ==========================================
// 主遊戲邏輯更新迴圈
// ==========================================
let lastTime = performance.now();
let netSyncTimer = 0;

function update(dt) {
  if (screenShake > 0) screenShake = Math.max(0, screenShake - dt * 35);

  // 更新玩家與 AI
  for (let id in players) {
    const p = players[id];
    if (p.isSpectator) continue;

    if (p.dead) {
      if (p.respawnTimer > 0) {
        p.respawnTimer -= dt;
        if (p.respawnTimer <= 0) {
          const pos = getSafeSpawnPoint();
          p.x = pos.x; p.y = pos.y;
          p.hp = p.maxHp;
          p.ammo = MAX_AMMO;
          p.dashCd = 0;
          p.dashTimer = 0;
          p.invulnTimer = 2.0; // 重生2秒無敵護盾
          p.dead = false;
          if (p.id === myId) updateSpectatorUI();
        }
      }
      continue;
    }

    // 狀態計時器
    if (p.invulnTimer > 0) p.invulnTimer -= dt;
    if (p.dashCd > 0) p.dashCd = Math.max(0, p.dashCd - dt);

    // 彈藥自然回復
    if (p.id === myId && state.cheats.infAmmo) {
      p.ammo = MAX_AMMO;
    } else if (p.ammo < MAX_AMMO) {
      p.ammo = Math.min(MAX_AMMO, p.ammo + AMMO_RELOAD_RATE * dt);
    }

    // 本地玩家控制
    if (id === myId && state.screen === 'game') {
      let mx = joysticks.move.dx;
      let my = joysticks.move.dy;
      if (keys['w'] || keys['arrowup']) my -= 1;
      if (keys['s'] || keys['arrowdown']) my += 1;
      if (keys['a'] || keys['arrowleft']) mx -= 1;
      if (keys['d'] || keys['arrowright']) mx += 1;
      const mLen = Math.hypot(mx, my);
      if (mLen > 1) { mx /= mLen; my /= mLen; }

      if (p.dashTimer > 0) {
        p.dashTimer -= dt;
        p.vx = p.dashDirX * DASH_SPEED;
        p.vy = p.dashDirY * DASH_SPEED;
        if (Math.random() < 0.6) spawnParticles(p.x, p.y, p.color, 1);
      } else {
        const speedMult = state.cheats.speed ? 2.0 : 1.0;
        p.vx = mx * BASE_SPEED * speedMult;
        p.vy = my * BASE_SPEED * speedMult;
        if (mLen > 0.1 && !joysticks.aim.active && !mouseAiming) {
          p.angle = Math.atan2(my, mx);
        }
      }

      if (joysticks.aim.active) {
        p.angle = getFinalAimAngle(p, joysticks.aim.angle);
      } else if (mouseAiming) {
        const wmx = (mouseScreen.x - canvas.width / 2) / camera.zoom + camera.x;
        const wmy = (mouseScreen.y - canvas.height / 2) / camera.zoom + camera.y;
        p.angle = getFinalAimAngle(p, Math.atan2(wmy - p.y, wmx - p.x));
      }

      p.x += p.vx * dt;
      p.y += p.vy * dt;
      resolveWallCollision(p);
    }
    // AI 控制（房主或單人模式執行）
    else if (p.isBot && state.isHost) {
      updateBotAI(p, dt);
    }
  }

  // 更新子彈（無限射程，直到擊中或撞牆/出界）
  for (let i = bullets.length - 1; i >= 0; i--) {
    const b = bullets[i];
    b.life -= dt;

    // 子彈轉彎追蹤邏輯
    if (b.homing) {
      let nearest = null;
      let minDist = 650;
      for (let id in players) {
        const p = players[id];
        if (id === b.ownerId || p.dead || p.isSpectator) continue;
        const d = Math.hypot(p.x - b.x, p.y - b.y);
        if (d < minDist) { minDist = d; nearest = p; }
      }
      if (nearest) {
        const targetAng = Math.atan2(nearest.y - b.y, nearest.x - b.x);
        let diff = targetAng - b.angle;
        while (diff > Math.PI) diff -= Math.PI * 2;
        while (diff < -Math.PI) diff += Math.PI * 2;
        b.angle += Math.sign(diff) * Math.min(Math.abs(diff), 4.2 * dt);
        b.vx = Math.cos(b.angle) * BULLET_SPEED;
        b.vy = Math.sin(b.angle) * BULLET_SPEED;
      }
    }

    b.x += b.vx * dt;
    b.y += b.vy * dt;

    // 出界銷毀
    if (b.x < 0 || b.x > mapWidth || b.y < 0 || b.y > mapHeight || b.life <= 0) {
      bullets.splice(i, 1);
      continue;
    }

    // 撞牆檢測（若無子彈穿牆）
    if (!b.wallBullet) {
      let hitWall = false;
      for (let w of walls) {
        if (b.x >= w.x && b.x <= w.x + w.w && b.y >= w.y && b.y <= w.y + w.h) {
          hitWall = true;
          break;
        }
      }
      if (hitWall) {
        spawnParticles(b.x, b.y, b.color, 6);
        bullets.splice(i, 1);
        continue;
      }
    }

    // 命中玩家檢測
    let hitPlayer = false;
    for (let id in players) {
      const p = players[id];
      if (id === b.ownerId || p.dead || p.isSpectator) continue;
      // 衝刺期間可穿透子彈！
      if (p.dashTimer > 0) continue;

      if (Math.hypot(b.x - p.x, b.y - p.y) <= p.radius + 6) {
        hitPlayer = true;
        const shooter = players[b.ownerId];
        if (shooter) {
          shooter.shotsHit++;
          // 核心規則：射中增加 0.5 發彈藥！
          shooter.ammo = Math.min(MAX_AMMO, shooter.ammo + HIT_AMMO_BONUS);
          if (shooter.id === myId) {
            playTone(760, 'sine', 0.15, 0.12);
            spawnFloatingText(shooter.x, shooter.y - 38, '+0.5 彈藥', '#00f0ff');
          }
        }

        if (state.isHost) {
          damagePlayer(p, b.ownerId);
          if (state.isOnline) {
            broadcastNet({ type: 'hit_confirm', targetId: p.id, shooterId: b.ownerId });
          }
        }
        break;
      }
    }

    if (hitPlayer) {
      bullets.splice(i, 1);
    }
  }

  // 回血急救包生成與拾取（房主負責生成）
  if (state.isHost) {
    healSpawnTimer -= dt;
    const maxHeals = Math.max(2, Math.floor(Object.keys(players).length / 2));
    if (healSpawnTimer <= 0 && (heals.length + healWarnings.length) < maxHeals) {
      healSpawnTimer = 10.0;
      const pos = getSafeSpawnPoint();
      healWarnings.push({ x: pos.x, y: pos.y, timer: 2.0 }); // 2秒地面預警光圈
    }

    for (let i = healWarnings.length - 1; i >= 0; i--) {
      healWarnings[i].timer -= dt;
      if (healWarnings[i].timer <= 0) {
        heals.push({ id: 'h_' + Math.random(), x: healWarnings[i].x, y: healWarnings[i].y });
        healWarnings.splice(i, 1);
      }
    }
  }

  for (let i = heals.length - 1; i >= 0; i--) {
    const h = heals[i];
    for (let id in players) {
      const p = players[id];
      if (p.dead || p.isSpectator || p.hp >= p.maxHp) continue;
      if (Math.hypot(p.x - h.x, p.y - h.y) <= p.radius + 18) {
        p.hp = Math.min(p.maxHp, p.hp + 1);
        spawnParticles(h.x, h.y, '#10b981', 14);
        if (p.id === myId) {
          playTone(620, 'sine', 0.25, 0.12);
          spawnFloatingText(p.x, p.y - 35, '+1 HP', '#10b981');
        }
        heals.splice(i, 1);
        break;
      }
    }
  }

  // 更新粒子與浮動文字
  for (let i = particles.length - 1; i >= 0; i--) {
    const pt = particles[i];
    pt.x += pt.vx * dt;
    pt.y += pt.vy * dt;
    pt.life -= dt;
    if (pt.life <= 0) particles.splice(i, 1);
  }
  for (let i = floatingTexts.length - 1; i >= 0; i--) {
    const ft = floatingTexts[i];
    ft.y -= 35 * dt;
    ft.life -= dt;
    if (ft.life <= 0) floatingTexts.splice(i, 1);
  }

  // 攝影機跟隨與縮放
  updateCamera(dt);

  // 線上同步廣播 (25Hz)
  if (state.isOnline && state.screen === 'game') {
    netSyncTimer += dt;
    if (netSyncTimer >= 0.04) {
      netSyncTimer = 0;
      const me = players[myId];
      if (state.isHost) {
        broadcastNet({ type: 'sync_state', players, heals, healWarnings });
      } else if (me && hostConn && hostConn.open) {
        hostConn.send({
          type: 'input', id: myId,
          x: me.x, y: me.y, angle: me.angle,
          hp: me.hp, ammo: me.ammo, isDashing: me.dashTimer > 0,
          dead: me.dead, isSpectator: me.isSpectator, radius: me.radius
        });
      }
    }
  }

  // 更新 HUD 與勝負判定
  if (state.screen === 'game') {
    updateHUDAndWinCondition();
  }
}

function updateBotAI(bot, dt) {
  bot.aiTimer -= dt;

  // 尋找最近敵人或回血包
  let nearestEnemy = null;
  let enemyDist = Infinity;
  for (let id in players) {
    const p = players[id];
    if (id === bot.id || p.dead || p.isSpectator) continue;
    const d = Math.hypot(p.x - bot.x, p.y - bot.y);
    if (d < enemyDist) { enemyDist = d; nearestEnemy = p; }
  }

  if (bot.aiTimer <= 0) {
    bot.aiTimer = state.aiDiff === 'easy' ? 0.8 : state.aiDiff === 'ace' ? 0.25 : 0.45;
    if (bot.hp < bot.maxHp && heals.length > 0) {
      bot.aiMoveAngle = Math.atan2(heals[0].y - bot.y, heals[0].x - bot.x);
    } else if (nearestEnemy) {
      const toEnemy = Math.atan2(nearestEnemy.y - bot.y, nearestEnemy.x - bot.x);
      // 保持中距離橫向走位
      bot.aiMoveAngle = enemyDist > 420 ? toEnemy + (Math.random() - 0.5) * 0.6 : toEnemy + Math.PI * 0.5 * (Math.random() > 0.5 ? 1 : -1);
    } else {
      bot.aiMoveAngle += (Math.random() - 0.5) * 2;
    }

    // 射擊決策
    if (nearestEnemy && bot.ammo >= 1.0) {
      const aimAng = Math.atan2(nearestEnemy.y - bot.y, nearestEnemy.x - bot.x);
      const spread = state.aiDiff === 'easy' ? 0.35 : state.aiDiff === 'ace' ? 0.06 : 0.16;
      tryPlayerShoot(bot, aimAng + (Math.random() - 0.5) * spread);
    }

    // 王牌 AI 懂得用衝刺閃躲或突進
    if (state.aiDiff === 'ace' && bot.dashCd <= 0 && nearestEnemy && enemyDist < 500 && Math.random() < 0.3) {
      tryPlayerDash(bot, bot.aiMoveAngle);
    }
  }

  if (bot.dashTimer > 0) {
    bot.dashTimer -= dt;
    bot.vx = bot.dashDirX * DASH_SPEED;
    bot.vy = bot.dashDirY * DASH_SPEED;
  } else {
    const spdFactor = state.aiDiff === 'easy' ? 0.75 : 1.0;
    bot.vx = Math.cos(bot.aiMoveAngle) * BASE_SPEED * spdFactor;
    bot.vy = Math.sin(bot.aiMoveAngle) * BASE_SPEED * spdFactor;
    if (nearestEnemy) {
      bot.angle = Math.atan2(nearestEnemy.y - bot.y, nearestEnemy.x - bot.x);
    }
  }

  bot.x += bot.vx * dt;
  bot.y += bot.vy * dt;
  resolveWallCollision(bot);
}

function updateCamera(dt) {
  if (state.screen === 'lobby') {
    camera.x = mapWidth / 2;
    camera.y = mapHeight / 2;
    camera.zoom = Math.min(canvas.width / mapWidth, canvas.height / mapHeight);
    return;
  }

  const me = players[myId];
  const isSpec = !me || me.isSpectator || (me.dead && me.respawnsLeft <= 0);

  if (isSpec && !state.specTargetId) {
    // 觀戰上帝全景視角
    camera.x += (mapWidth / 2 - camera.x) * 6 * dt;
    camera.y += (mapHeight / 2 - camera.y) * 6 * dt;
    const targetZoom = Math.min(canvas.width / mapWidth, canvas.height / mapHeight) * 0.92;
    camera.zoom += (targetZoom - camera.zoom) * 6 * dt;
  } else {
    const target = isSpec && players[state.specTargetId] ? players[state.specTargetId] : me;
    if (target) {
      camera.x += (target.x - camera.x) * 10 * dt;
      camera.y += (target.y - camera.y) * 10 * dt;
      const desiredZoom = Math.min(1.0, Math.max(0.65, canvas.width / 1250));
      camera.zoom += (desiredZoom - camera.zoom) * 8 * dt;
    }
  }
}

function updateHUDAndWinCondition() {
  const activePlayers = Object.values(players).filter(p => !p.isSpectator);
  const alivePlayers = activePlayers.filter(p => !p.dead || p.respawnsLeft > 0);
  document.getElementById('hud-alive-label').textContent = `存活: ${alivePlayers.length} / ${activePlayers.length}`;

  // 更新排行榜 Top 4
  const sorted = [...activePlayers].sort((a, b) => b.kills - a.kills);
  const lb = document.getElementById('leaderboard-box');
  lb.innerHTML = sorted.slice(0, 4).map((p, idx) => {
    const respStr = p.respawnsLeft >= 999 ? '∞' : `♥×${p.respawnsLeft}`;
    return `<div class="lb-row ${p.id === myId ? 'me' : ''}">
      <span>#${idx + 1} ${p.name}</span>
      <span>${p.kills}殺 (${respStr})</span>
    </div>`;
  }).join('');

  // 勝利結算判定（非無限復活模式且至少2人參戰時，僅剩1人存活即結束）
  if (state.maxRespawns < 999 && activePlayers.length >= 2 && alivePlayers.length <= 1) {
    const winner = alivePlayers[0];
    state.screen = 'gameover';
    const isMeWin = winner && winner.id === myId;
    if (isMeWin) {
      state.stats.wins++;
      localStorage.setItem('ns_wins', state.stats.wins);
      document.getElementById('stat-wins').textContent = state.stats.wins;
    }
    const me = players[myId];
    const acc = me && me.shotsFired > 0 ? Math.round((me.shotsHit / me.shotsFired) * 100) : 0;
    document.getElementById('go-title').textContent = isMeWin ? '🏆 勝利！' : (winner ? `👑 ${winner.name} 獲勝` : '平手');
    document.getElementById('go-stats').innerHTML = `
      <div>本局擊殺：<strong>${me ? me.kills : 0}</strong></div>
      <div>射擊命中率：<strong>${acc}%</strong>（觸發 ${me ? me.shotsHit : 0} 次 +0.5 彈藥）</div>
    `;
    document.getElementById('gameover-modal').classList.remove('hidden');
  }
}

// ==========================================
// 畫面渲染引擎
// ==========================================
function render() {
  ctx.fillStyle = '#070913';
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  ctx.save();
  const shakeX = (Math.random() - 0.5) * screenShake;
  const shakeY = (Math.random() - 0.5) * screenShake;
  ctx.translate(canvas.width / 2 + shakeX, canvas.height / 2 + shakeY);
  ctx.scale(camera.zoom, camera.zoom);
  ctx.translate(-camera.x, -camera.y);

  // 1. 繪製競技場地板網格
  ctx.strokeStyle = 'rgba(0, 240, 255, 0.06)';
  ctx.lineWidth = 1;
  const gridSize = 80;
  for (let x = 0; x <= mapWidth; x += gridSize) {
    ctx.beginPath(); ctx.moveTo(x, 0); ctx.lineTo(x, mapHeight); ctx.stroke();
  }
  for (let y = 0; y <= mapHeight; y += gridSize) {
    ctx.beginPath(); ctx.moveTo(0, y); ctx.lineTo(mapWidth, y); ctx.stroke();
  }

  // 2. 繪製回血道具預警光圈 & 急救包
  healWarnings.forEach(w => {
    ctx.strokeStyle = 'rgba(16, 185, 129, 0.6)';
    ctx.lineWidth = 3;
    ctx.beginPath();
    ctx.arc(w.x, w.y, 26 * (1 - w.timer / 2), 0, Math.PI * 2);
    ctx.stroke();
  });

  heals.forEach(h => {
    ctx.save();
    ctx.translate(h.x, h.y);
    ctx.fillStyle = 'rgba(16, 185, 129, 0.25)';
    ctx.beginPath(); ctx.arc(0, 0, 22, 0, Math.PI * 2); ctx.fill();
    ctx.fillStyle = '#10b981';
    ctx.fillRect(-10, -4, 20, 8);
    ctx.fillRect(-4, -10, 8, 20);
    ctx.restore();
  });

  // 3. 繪製實體掩體牆壁
  walls.forEach(w => {
    ctx.fillStyle = '#1e293b';
    ctx.fillRect(w.x, w.y, w.w, w.h);
    ctx.strokeStyle = '#38bdf8';
    ctx.lineWidth = 2;
    ctx.strokeRect(w.x, w.y, w.w, w.h);
  });

  // 4. 繪製瞄準雷射線 & 衝刺軌跡預覽（僅本地存活玩家操作時顯示）
  const me = players[myId];
  if (me && !me.dead && !me.isSpectator && state.screen === 'game') {
    if (joysticks.aim.active || mouseAiming) {
      const ray = castLaserRay(me.x, me.y, me.angle, state.cheats.wallBullet);
      ctx.save();
      ctx.strokeStyle = me.ammo >= 1.0 || state.cheats.infAmmo ? 'rgba(0, 240, 255, 0.7)' : 'rgba(239, 68, 68, 0.5)';
      ctx.lineWidth = 2.5;
      ctx.setLineDash([10, 8]);
      ctx.beginPath();
      ctx.moveTo(me.x, me.y);
      ctx.lineTo(ray.x, ray.y);
      ctx.stroke();

      // 落點準星光點
      ctx.fillStyle = ray.hitType === 'enemy' ? '#ff007f' : '#00f0ff';
      ctx.beginPath();
      ctx.arc(ray.x, ray.y, 6, 0, Math.PI * 2);
      ctx.fill();
      ctx.restore();
    }

    if (joysticks.dash.active) {
      const dashAng = joysticks.dash.valid ? joysticks.dash.angle : me.angle;
      const endX = me.x + Math.cos(dashAng) * DASH_DISTANCE;
      const endY = me.y + Math.sin(dashAng) * DASH_DISTANCE;
      ctx.save();
      ctx.strokeStyle = me.dashCd <= 0 ? 'rgba(245, 158, 11, 0.75)' : 'rgba(148, 163, 184, 0.35)';
      ctx.lineWidth = 14;
      ctx.lineCap = 'round';
      ctx.beginPath();
      ctx.moveTo(me.x, me.y);
      ctx.lineTo(endX, endY);
      ctx.stroke();
      ctx.restore();
    }
  }

  // 5. 繪製子彈
  bullets.forEach(b => {
    ctx.save();
    ctx.fillStyle = b.color;
    ctx.shadowColor = b.color;
    ctx.shadowBlur = 12;
    ctx.beginPath();
    ctx.arc(b.x, b.y, b.homing ? 7 : 5, 0, Math.PI * 2);
    ctx.fill();
    ctx.restore();
  });

  // 6. 繪製玩家角色與頭頂雙層狀態條
  for (let id in players) {
    const p = players[id];
    if (p.isSpectator) continue;
    if (p.dead) {
      if (p.respawnTimer > 0) {
        ctx.fillStyle = '#94a3b8';
        ctx.font = 'bold 14px sans-serif';
        ctx.textAlign = 'center';
        ctx.fillText(`復活倒數 ${p.respawnTimer.toFixed(1)}s`, p.x, p.y);
      }
      continue;
    }

    ctx.save();
    // 衝刺期間半透明殘影發光
    if (p.dashTimer > 0) {
      ctx.globalAlpha = 0.65;
      ctx.shadowColor = '#f59e0b';
      ctx.shadowBlur = 20;
    }

    // 角色主體
    ctx.fillStyle = p.color;
    ctx.beginPath();
    ctx.arc(p.x, p.y, p.radius, 0, Math.PI * 2);
    ctx.fill();

    // 槍口方向指示器
    ctx.strokeStyle = '#ffffff';
    ctx.lineWidth = 4;
    ctx.beginPath();
    ctx.moveTo(p.x, p.y);
    ctx.lineTo(p.x + Math.cos(p.angle) * (p.radius + 10), p.y + Math.sin(p.angle) * (p.radius + 10));
    ctx.stroke();

    // 無敵保護罩 / 金身特效
    if (p.invulnTimer > 0 || (p.id === myId && state.cheats.god)) {
      ctx.strokeStyle = (p.id === myId && state.cheats.god) ? '#fbbf24' : '#38bdf8';
      ctx.lineWidth = 3;
      ctx.beginPath();
      ctx.arc(p.x, p.y, p.radius + 7, 0, Math.PI * 2);
      ctx.stroke();
    }

    // 全圖透視 ESP 標記（作弊開啟時）
    if (state.cheats.esp && p.id !== myId) {
      ctx.strokeStyle = '#ef4444';
      ctx.lineWidth = 2;
      ctx.strokeRect(p.x - p.radius - 10, p.y - p.radius - 10, (p.radius + 10) * 2, (p.radius + 10) * 2);
    }

    // 頭頂名稱 + 3格血條 + 3格彈藥進度條
    const barW = 54;
    const startX = p.x - barW / 2;
    const baseY = p.y - p.radius - 26;

    ctx.fillStyle = '#e2e8f0';
    ctx.font = 'bold 12px sans-serif';
    ctx.textAlign = 'center';
    ctx.fillText(p.name, p.x, baseY - 4);

    // 血條（每格 1 滴血）
    const hpSegW = (barW - (p.maxHp - 1) * 2) / p.maxHp;
    for (let i = 0; i < p.maxHp; i++) {
      ctx.fillStyle = i < p.hp ? (p.hp === 1 ? '#ef4444' : '#10b981') : 'rgba(255,255,255,0.18)';
      ctx.fillRect(startX + i * (hpSegW + 2), baseY, hpSegW, 6);
    }

    // 彈藥進度條（3發上限，顯示小數 +0.5 進度）
    const ammoY = baseY + 8;
    const ammoSegW = (barW - 4) / MAX_AMMO;
    for (let i = 0; i < MAX_AMMO; i++) {
      ctx.fillStyle = 'rgba(255,255,255,0.15)';
      ctx.fillRect(startX + i * (ammoSegW + 2), ammoY, ammoSegW, 4);
      const fillRatio = Math.max(0, Math.min(1, p.ammo - i));
      if (fillRatio > 0) {
        ctx.fillStyle = fillRatio >= 1 ? '#fbbf24' : 'rgba(0, 240, 255, 0.75)';
        ctx.fillRect(startX + i * (ammoSegW + 2), ammoY, ammoSegW * fillRatio, 4);
      }
    }

    ctx.restore();
  }

  // 7. 粒子與浮動文字
  particles.forEach(pt => {
    ctx.globalAlpha = Math.max(0, pt.life / pt.maxLife);
    ctx.fillStyle = pt.color;
    ctx.beginPath(); ctx.arc(pt.x, pt.y, 4, 0, Math.PI * 2); ctx.fill();
  });
  ctx.globalAlpha = 1;

  floatingTexts.forEach(ft => {
    ctx.globalAlpha = Math.max(0, ft.life);
    ctx.fillStyle = ft.color;
    ctx.font = 'bold 14px sans-serif';
    ctx.textAlign = 'center';
    ctx.fillText(ft.text, ft.x, ft.y);
  });
  ctx.globalAlpha = 1;

  ctx.restore();

  // 8. 繪製三搖桿觸控介面（左移動、右下瞄準射擊、右上衝刺技能）
  if (state.screen === 'game' && me && !me.dead && !me.isSpectator) {
    renderThreeJoysticks(me);
  }
}

function renderThreeJoysticks(me) {
  const c = getRightJoystickCenters();

  // 左側移動搖桿（按住時顯示在觸控起點，平時左下角微弱提示圈）
  const moveBaseX = joysticks.move.active ? joysticks.move.originX : 115;
  const moveBaseY = joysticks.move.active ? joysticks.move.originY : canvas.height - 115;
  const moveKnobX = joysticks.move.active ? moveBaseX + joysticks.move.dx * c.radius : moveBaseX;
  const moveKnobY = joysticks.move.active ? moveBaseY + joysticks.move.dy * c.radius : moveBaseY;
  drawStick(moveBaseX, moveBaseY, moveKnobX, moveKnobY, c.radius, 'rgba(255,255,255,0.25)', '#ffffff', '移動');

  // 右下瞄準射擊搖桿
  const aimBaseX = joysticks.aim.active ? joysticks.aim.baseX : c.aimX;
  const aimBaseY = joysticks.aim.active ? joysticks.aim.baseY : c.aimY;
  let aimKnobX = aimBaseX, aimKnobY = aimBaseY;
  if (joysticks.aim.active) {
    const vx = joysticks.aim.curX - aimBaseX;
    const vy = joysticks.aim.curY - aimBaseY;
    const dist = Math.min(Math.hypot(vx, vy), c.radius);
    const ang = Math.atan2(vy, vx);
    aimKnobX = aimBaseX + Math.cos(ang) * dist;
    aimKnobY = aimBaseY + Math.sin(ang) * dist;
  }
  const canShoot = me.ammo >= 1.0 || state.cheats.infAmmo;
  drawStick(aimBaseX, aimBaseY, aimKnobX, aimKnobY, c.radius,
    canShoot ? 'rgba(0, 240, 255, 0.25)' : 'rgba(239, 68, 68, 0.2)',
    canShoot ? '#00f0ff' : '#ef4444',
    `射擊 (${Math.floor(me.ammo)}/3)`
  );

  // 右上衝刺技能搖桿（附帶 10 秒冷卻圓弧進度條）
  const dashBaseX = c.dashX;
  const dashBaseY = c.dashY;
  let dashKnobX = dashBaseX, dashKnobY = dashBaseY;
  if (joysticks.dash.active) {
    const vx = joysticks.dash.curX - dashBaseX;
    const vy = joysticks.dash.curY - dashBaseY;
    const dist = Math.min(Math.hypot(vx, vy), c.radius * 0.85);
    const ang = Math.atan2(vy, vx);
    dashKnobX = dashBaseX + Math.cos(ang) * dist;
    dashKnobY = dashBaseY + Math.sin(ang) * dist;
  }
  const dashReady = me.dashCd <= 0;
  drawStick(dashBaseX, dashBaseY, dashKnobX, dashKnobY, c.radius * 0.85,
    dashReady ? 'rgba(245, 158, 11, 0.28)' : 'rgba(100, 116, 139, 0.2)',
    dashReady ? '#f59e0b' : '#64748b',
    dashReady ? '⚡衝刺' : `${me.dashCd.toFixed(1)}s`
  );

  // 冷卻圓弧進度環
  if (!dashReady) {
    const ratio = 1 - (me.dashCd / DASH_COOLDOWN);
    ctx.strokeStyle = '#f59e0b';
    ctx.lineWidth = 4;
    ctx.beginPath();
    ctx.arc(dashBaseX, dashBaseY, c.radius * 0.85, -Math.PI / 2, -Math.PI / 2 + Math.PI * 2 * ratio);
    ctx.stroke();
  }
}

function drawStick(bx, by, kx, ky, r, bgCol, fgCol, label) {
  ctx.save();
  ctx.fillStyle = bgCol;
  ctx.strokeStyle = fgCol;
  ctx.lineWidth = 2;
  ctx.beginPath(); ctx.arc(bx, by, r, 0, Math.PI * 2); ctx.fill(); ctx.stroke();

  ctx.fillStyle = fgCol;
  ctx.beginPath(); ctx.arc(kx, ky, r * 0.42, 0, Math.PI * 2); ctx.fill();

  ctx.fillStyle = '#e2e8f0';
  ctx.font = 'bold 11px sans-serif';
  ctx.textAlign = 'center';
  ctx.fillText(label, bx, by + r + 14);
  ctx.restore();
}

// 主動畫迴圈
function gameLoop(now) {
  const dt = Math.min(0.05, (now - lastTime) / 1000);
  lastTime = now;
  update(dt);
  render();
  requestAnimationFrame(gameLoop);
}

setupLobbyDemo();
appendChat('', '歡迎來到《霓虹突擊：無限競技場》！支援單人混戰與線上多人連線。', 'sys');
requestAnimationFrame(gameLoop);
</script>
</body>
</html>
