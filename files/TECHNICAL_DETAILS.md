# 技術詳細と拡張案

## アルゴリズム詳細

### 1. 暗号学的乱数生成（Rejection Sampling）

#### 問題: 剰余演算による偏り
```javascript
// ❌ 悪い例: 偏りが発生
const bad = (crypto.getRandomValues(new Uint32Array(1))[0] % 6) + 1;
```

**理由**: `2^32` (4,294,967,296) は 6 で割り切れない  
→ 剰余 0-5 の出現確率が微妙に異なる

#### 解決: Rejection Sampling
```javascript
function randint1n(n) {
  const max = 0xFFFFFFFF;                    // 2^32 - 1
  const limit = Math.floor(max / n) * n;    // 6の倍数の最大値
  const buf = new Uint32Array(1);
  
  for (;;) {
    crypto.getRandomValues(buf);
    if (buf[0] < limit) {                   // 偏り範囲を除外
      return (buf[0] % n) + 1;
    }
  }
}
```

**効果**: 完全な一様分布を保証（棄却確率は極小）

---

### 2. 3D回転アルゴリズム

#### 2-1. 出目先行型の数学的基礎

**目標**: 出目 `v` を正面に配置する回転 `(rx, ry, rz)`

**基準姿勢**（ORIENT_3D）:
```
面      | 出目 | 回転
--------|------|------------------
front   | 1    | (0°, 0°, 0°)
bottom  | 2    | (90°, 0°, 0°)
right   | 3    | (0°, -90°, 0°)
left    | 4    | (0°, 90°, 0°)
top     | 5    | (-90°, 0°, 0°)
back    | 6    | (0°, 180°, 0°)
```

**回転数の追加**:
```javascript
const spins = 3 + Math.floor(Math.random() * 3);  // 3-5回転
const rx = baseX + spins * 360;
const ry = baseY + spins * 360;
```

**回転数が結果に影響しない理由**:  
CSS Transform は最終的な姿勢のみを計算（途中経過は補間）

**ランダムノイズの追加** (フリーストップ用):
```javascript
const rx = baseX + spins * 360 + (Math.random() * 20 - 10);
const ry = baseY + spins * 360 + (Math.random() * 20 - 10);
const rz = Math.random() * 180 - 90;
```

#### 2-2. ビタ止めの実装

**transitionend イベント**を利用:
```javascript
cube.addEventListener('transitionend', () => {
  cube.style.transition = 'none';  // アニメーション無効化
  cube.style.transform = `rotateX(${baseX}deg) rotateY(${baseY}deg) rotateZ(0deg)`;
  // ↑ 正確な角度へ瞬時にスナップ
});
```

**注意点**: 
- `transition: none` で視覚的なジャンプを防止
- ユーザーには気づかれない（60fps以下の遅延）

#### 2-3. 演出先行型の課題

演出先行型では、最終的な回転角から出目を「逆算」する必要がある:

```javascript
// 疑似コード（未実装）
function inferValueFromRotation(rx, ry, rz) {
  // 最終的な姿勢 → どの面が正面？
  // → その面の出目を表示
  
  // 問題: rz（Z軸回転）があると判定が複雑化
  // → 完全な実装には quaternion や matrix 分解が必要
}
```

**現在の実装**: 演出先行でも内部的には出目を先に決定  
→ 真の「演出先行」ではなく、視覚的なバリエーション

---

### 3. 2D揺れアニメーション

#### CSS Keyframes
```css
@keyframes shake2d {
  0%   { transform: rotate(0) translate(0, 0); }
  25%  { transform: rotate(6deg) translate(2px, -1px); }
  50%  { transform: rotate(-5deg) translate(-2px, 1px); }
  75%  { transform: rotate(3deg) translate(1px, 2px); }
  100% { transform: rotate(0) translate(0, 0); }
}
```

**特徴**:
- 回転 + 平行移動の組み合わせ
- 物理的なリアリティは低いが軽量
- GPU加速対応（transform使用）

---

## パフォーマンス分析

### レンダリングコスト

| 描画方式 | DOM要素数 | Reflow | Repaint | GPU加速 |
|---------|----------|--------|---------|---------|
| 2D (1個) | 7要素 | なし | あり | ○ |
| 3D (1個) | 13要素 | なし | なし | ◎ |

**3D が意外と高速な理由**:
- `transform-style: preserve-3d` でGPU完全オフロード
- RepaintではなくComposite（合成）のみ

### ボトルネック

#### 1. DOM生成（初回・ボード再構築時）
```javascript
// 改善前
for (let i = 0; i < count; i++) {
  const die = createDie3D();
  board.appendChild(die);  // ← Reflow 発生
}

// 改善後
const fragment = document.createDocumentFragment();
for (let i = 0; i < count; i++) {
  const die = createDie3D();
  fragment.appendChild(die);  // Reflow なし
}
board.appendChild(fragment);  // 1回だけ Reflow
```

#### 2. 大量のサイコロ（12個以上）
- transitionend イベントのオーバーヘッド
- 改善案: Promise.all で並列処理

---

## 拡張案の詳細設計

### 拡張1: 多面体サイコロ（D4, D8, D10, D12, D20）

#### データ構造
```javascript
const POLYHEDRA = {
  d4: {
    faces: 4,
    vertices: [...],  // 四面体の頂点座標
    faceNormals: [...],
    pipLayout: {1: [5], 2: [5], 3: [5], 4: [5]}  // 中央1個
  },
  d6: { /* 現在の実装 */ },
  d8: {
    faces: 8,
    // 八面体...
  },
  // ...
};
```

#### 3D モデル生成
- CSS 3D では複雑な多面体は困難
- → **WebGL / Three.js** への移行を推奨

#### UI変更
```html
<select id="diceType">
  <option value="d4">D4 (四面体)</option>
  <option value="d6" selected>D6 (六面体)</option>
  <option value="d8">D8 (八面体)</option>
  <option value="d10">D10 (十面体)</option>
  <option value="d12">D12 (十二面体)</option>
  <option value="d20">D20 (二十面体)</option>
</select>
```

---

### 拡張2: 物理演算エンジン統合

#### Matter.js 統合例
```javascript
// 物理世界の構築
const engine = Matter.Engine.create();
const world = engine.world;

// サイコロを剛体として追加
const dice = Matter.Bodies.rectangle(x, y, size, size, {
  restitution: 0.8,  // 反発係数
  friction: 0.3,
  density: 0.04
});
Matter.World.add(world, dice);

// 投げる力を加える
Matter.Body.applyForce(dice, dice.position, {
  x: (Math.random() - 0.5) * 0.1,
  y: -0.2
});

// レンダリングループ
function animate() {
  Matter.Engine.update(engine, 1000 / 60);
  // DOM要素の位置・回転を同期
  diceElement.style.transform = `
    translate(${dice.position.x}px, ${dice.position.y}px)
    rotate(${dice.angle}rad)
  `;
  requestAnimationFrame(animate);
}
```

**メリット**:
- リアルなバウンス・回転
- 衝突判定（サイコロ同士、壁）

**デメリット**:
- 外部ライブラリ依存
- パフォーマンス要求大

---

### 拡張3: 統計・分析機能

#### 履歴記録
```javascript
const history = {
  rolls: [],  // 各ロールの結果
  timestamp: [],
  
  add(values) {
    this.rolls.push(values);
    this.timestamp.push(Date.now());
  },
  
  getStats() {
    const flat = this.rolls.flat();
    const counts = {};
    for (let i = 1; i <= 6; i++) {
      counts[i] = flat.filter(v => v === i).length;
    }
    return {
      total: flat.length,
      average: flat.reduce((a, b) => a + b, 0) / flat.length,
      distribution: counts
    };
  }
};
```

#### カイ二乗検定
```javascript
function chiSquareTest(observed, expected) {
  let chiSq = 0;
  for (let i = 0; i < observed.length; i++) {
    chiSq += Math.pow(observed[i] - expected[i], 2) / expected[i];
  }
  const df = observed.length - 1;  // 自由度
  const pValue = /* カイ二乗分布表から計算 */;
  return { chiSq, pValue };
}
```

#### UI追加
```html
<div class="stats-panel">
  <h3>統計情報</h3>
  <canvas id="histogram"></canvas>
  <table>
    <tr><th>出目</th><th>1</th><th>2</th><th>3</th><th>4</th><th>5</th><th>6</th></tr>
    <tr><th>回数</th><td id="c1">0</td>...</tr>
    <tr><th>確率</th><td id="p1">0%</td>...</tr>
  </table>
  <div>χ² 値: <span id="chiSq">-</span></div>
</div>
```

---

### 拡張4: サウンド効果

#### Web Audio API 使用
```javascript
const audioCtx = new (window.AudioContext || window.webkitAudioContext)();

function playRollSound() {
  const oscillator = audioCtx.createOscillator();
  const gainNode = audioCtx.createGain();
  
  oscillator.type = 'square';
  oscillator.frequency.setValueAtTime(440, audioCtx.currentTime);
  oscillator.frequency.exponentialRampToValueAtTime(
    110, audioCtx.currentTime + 0.1
  );
  
  gainNode.gain.setValueAtTime(0.3, audioCtx.currentTime);
  gainNode.gain.exponentialRampToValueAtTime(
    0.01, audioCtx.currentTime + 0.3
  );
  
  oscillator.connect(gainNode);
  gainNode.connect(audioCtx.destination);
  
  oscillator.start(audioCtx.currentTime);
  oscillator.stop(audioCtx.currentTime + 0.3);
}

function playLandSound() {
  // 着地音（低めの周波数）
  // ...
}
```

#### プリセット音源
```javascript
const sounds = {
  roll: new Audio('sounds/dice-roll.mp3'),
  land: new Audio('sounds/dice-land.mp3')
};

// ロール開始時
sounds.roll.play();

// 着地時（transitionend）
sounds.land.play();
```

---

### 拡張5: マルチプレイヤー対応

#### WebSocket サーバー（Node.js）
```javascript
const WebSocket = require('ws');
const wss = new WebSocket.Server({ port: 8080 });

const rooms = new Map();  // roomId -> Set<WebSocket>

wss.on('connection', (ws) => {
  ws.on('message', (message) => {
    const data = JSON.parse(message);
    
    switch (data.type) {
      case 'join':
        if (!rooms.has(data.roomId)) {
          rooms.set(data.roomId, new Set());
        }
        rooms.get(data.roomId).add(ws);
        break;
        
      case 'roll':
        // 同じルームの全員に結果を送信
        const room = rooms.get(data.roomId);
        room.forEach(client => {
          if (client.readyState === WebSocket.OPEN) {
            client.send(JSON.stringify({
              type: 'roll-result',
              values: data.values,
              total: data.total,
              player: data.player
            }));
          }
        });
        break;
    }
  });
});
```

#### クライアント側
```javascript
const ws = new WebSocket('ws://localhost:8080');
const roomId = 'room-12345';

ws.onopen = () => {
  ws.send(JSON.stringify({ type: 'join', roomId }));
};

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  if (data.type === 'roll-result') {
    displayOtherPlayerRoll(data.player, data.values, data.total);
  }
};

function rollAndBroadcast() {
  const values = getTargetValues();
  ws.send(JSON.stringify({
    type: 'roll',
    roomId,
    values,
    total: values.reduce((a, b) => a + b, 0),
    player: myName
  }));
  // 自分の画面でもロール
  executeRoll();
}
```

---

### 拡張6: テーマシステム

#### CSS変数ベース
```css
:root {
  --bg-color: #f0f2f5;
  --card-bg: #ffffff;
  --primary-color: #2563eb;
  --text-color: #111111;
  --die-bg: #ffffff;
  --die-border: #d0d4da;
  --pip-color: #111111;
}

[data-theme="dark"] {
  --bg-color: #1a1a1a;
  --card-bg: #2d2d2d;
  --primary-color: #60a5fa;
  --text-color: #f0f0f0;
  --die-bg: #3a3a3a;
  --die-border: #555555;
  --pip-color: #f0f0f0;
}

[data-theme="casino"] {
  --bg-color: #0a5f38;
  --card-bg: #0d7a48;
  --primary-color: #fbbf24;
  --die-bg: #dc2626;
  --pip-color: #ffffff;
}
```

#### テーマ切り替え
```javascript
function setTheme(theme) {
  document.documentElement.setAttribute('data-theme', theme);
  localStorage.setItem('theme', theme);
}

// 初期化時に復元
const savedTheme = localStorage.getItem('theme') || 'light';
setTheme(savedTheme);
```

---

## アクセシビリティ強化

### 現在の対応
- ✅ `aria-live="polite"` で結果読み上げ
- ✅ `aria-label` でサイコロ状態説明
- ✅ キーボードショートカット（Space）

### 追加推奨
```html
<!-- ボタンにより詳細な説明 -->
<button id="rollBtn" 
        aria-label="サイコロを振る。現在の設定: 3個、3D、ビタ止め"
        aria-describedby="roll-help">
  振る
</button>
<div id="roll-help" hidden>
  スペースキーでも実行できます
</div>

<!-- ロール中の状態 -->
<button aria-busy="true" disabled>
  ロール中...
</button>

<!-- 設定変更の通知 -->
<div role="status" aria-live="polite">
  描画方式を3Dに変更しました
</div>

<!-- フォーカス管理 -->
<script>
rollBtn.addEventListener('click', () => {
  executeRoll();
  // ロール完了後、結果にフォーカス
  setTimeout(() => {
    resultsEl.focus();
  }, state.duration * 1000 + 100);
});
</script>
```

---

## セキュリティ考慮事項

### XSS対策
```javascript
// ❌ 悪い例
resultsEl.innerHTML = `出目: ${userInput}`;

// ✅ 良い例
resultsEl.textContent = `出目: ${userInput}`;
// または
resultsEl.innerHTML = DOMPurify.sanitize(userInput);
```

### CSP (Content Security Policy)
```html
<meta http-equiv="Content-Security-Policy" 
      content="default-src 'self'; 
               style-src 'self' 'unsafe-inline'; 
               script-src 'self';">
```

---

## デプロイとホスティング

### 静的サイトホスティング
推奨サービス:
1. **GitHub Pages**: 無料、簡単
2. **Netlify**: CI/CD統合、プレビュー
3. **Vercel**: 高速CDN、Edge Functions
4. **Cloudflare Pages**: DDoS保護

### PWA化
```javascript
// service-worker.js
const CACHE_NAME = 'dice-v1';
const urlsToCache = [
  '/',
  '/unified-dice.html',
  '/styles.css',
  '/script.js'
];

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then(cache => cache.addAll(urlsToCache))
  );
});

self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request)
      .then(response => response || fetch(event.request))
  );
});
```

```json
// manifest.json
{
  "name": "サイコロシミュレーター",
  "short_name": "Dice",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#f0f2f5",
  "theme_color": "#2563eb",
  "icons": [
    {
      "src": "/icon-192.png",
      "sizes": "192x192",
      "type": "image/png"
    }
  ]
}
```

---

## まとめ

### 現在の統合の強み
- 🎯 明確な軸での分類（描画・アルゴリズム・演出）
- 🔧 柔軟な組み合わせ
- 📊 包括的な機能セット
- 🚀 拡張性の高い設計

### 次のステップ
1. **短期**: バグ修正、UIブラッシュアップ
2. **中期**: 統計機能、テーマシステム
3. **長期**: WebGL移行、物理演算、マルチプレイヤー

この統合により、単なる「サイコロアプリ」から「サイコロシミュレーションプラットフォーム」への進化が可能になります。
