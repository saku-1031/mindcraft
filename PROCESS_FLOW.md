# Mindcraft 処理フロー

## 概要
このドキュメントは、Mindcraftの処理の流れを初期化からメインループまで順番にまとめたものです。

---

## 1. 初期化フェーズ（起動時に一度だけ実行）

### 1.1 プログラム起動
**エントリーポイント: `main.js`**

```
main.js (1行目～)
  ↓
コマンドライン引数の解析
  ↓
設定ファイルの読み込み (settings.js)
```

### 1.2 MindServerの初期化
**実行箇所: `src/mindcraft/mindcraft.js` の `Mindcraft.init()`**

```
Mindcraft.init()
  ↓
MindServerインスタンス作成（ポート8080）
  ↓
WebSocketサーバー起動
```

### 1.3 エージェント生成
**実行箇所: `src/mindcraft/mindcraft.js` の `Mindcraft.createAgent()`**

各プロファイルごとに以下が実行されます：

```
createAgent(profile)
  ↓
サブプロセス起動 (init_agent.js)
  ↓
MindServerへの接続
  ↓
Agentインスタンス生成
```

### 1.4 エージェント初期化（6つのフェーズ）
**実行箇所: `src/agent/agent.js` の `Agent.start()`**

```
Phase 1: コンポーネント初期化
  - ActionManager作成 (src/agent/action_manager.js)
  - NPCコントローラー作成 (src/agent/npc/index.js)
  - 各種マネージャーのセットアップ

Phase 2: メモリ初期化
  - 短期記憶の初期化 (src/agent/memory.js)
  - 長期記憶の読み込み
  - スキル記憶の読み込み

Phase 3: タスクシステム初期化
  - コマンドレジストリの登録 (src/agent/commands/)
  - アクションレジストリの登録
  - 動作モードの設定

Phase 4: Minecraftボット初期化
  - mineflayerボットの作成
  - プラグインの読み込み（pathfinder, pvp, collectBlockなど）
  - イベントリスナーの登録

Phase 5: Minecraftサーバーへのログイン
  - 認証処理
  - サーバー接続

Phase 6: スポーン完了
  - ワールドへのスポーン
  - 初期メッセージの送信
  - イベントループの開始
```

---

## 2. メインループフェーズ（継続的に実行）

### 2.1 メインイベントループ
**実行箇所: `src/agent/agent.js:395-477` の `startEvents()`**

**周期: 300ミリ秒ごと**

```javascript
setInterval(() => {
  update()
}, 300)
```

#### ループ内で実行される処理：

```
【300msごとに実行】

1. モード実行 (modes)
   - 各動作モード（採掘、戦闘、探索など）の更新
   - 実行箇所: agent.js内のモード管理

2. セルフプロンプター実行
   - 自律的な意思決定
   - 実行箇所: src/agent/self_prompter.js
   - LLMに現在の状態を渡して次のアクションを決定

3. タスクチェック
   - 実行中のタスクの状態確認
   - タスク完了・失敗の処理
   - 次のタスクの開始

4. 環境監視
   - 周囲のエンティティ検出
   - インベントリの状態確認
   - 体力・空腹度のチェック
```

### 2.2 非同期メッセージ処理（イベント駆動）
**実行箇所: `src/agent/agent.js` の `handleMessage()`**

メッセージを受信した際に実行される4段階のパイプライン：

```
メッセージ受信
  ↓
Phase 1: バリデーション
  - メッセージの妥当性チェック
  - 送信者の確認
  ↓
Phase 2: コマンド検出
  - コマンド形式（!command）の検出
  - コマンドの解析
  ↓
Phase 3: 履歴への追加
  - 会話履歴への保存
  - メモリへの記録
  ↓
Phase 4: LLM処理
  - プロンプト生成 (src/models/prompter.js)
  - LLMへのリクエスト
  - レスポンスの処理
  - アクション実行 or チャット送信
```

### 2.3 アクション実行システム
**実行箇所: `src/agent/action_manager.js`**

```
アクション実行リクエスト
  ↓
ActionManager.executeAction()
  ↓
1. アクション種別の判定
2. 該当アクションの実行
3. タイムアウト管理（最大10分）
4. 成功/失敗の判定
5. 結果の返却
```

### 2.4 ボットイベント処理
**実行箇所: `src/agent/agent.js` のイベントハンドラー群**

Minecraftボットからのイベントに反応：

```
- physicsTick: 物理演算の更新
- chat: チャットメッセージの受信
- entityHurt: エンティティのダメージ
- entityDead: エンティティの死亡
- health: 体力変化
- death: 自身の死亡
- kicked: サーバーからのキック
- error: エラー発生
```

---

## 3. 処理フロー全体図

```
[起動]
  │
  ├─ main.js
  │   └─ Mindcraft.init()
  │       └─ MindServer起動（8080ポート）
  │
  ├─ Mindcraft.createAgent() ×プロファイル数
  │   └─ init_agent.js（サブプロセス）
  │       └─ Agent.start()
  │           ├─ [1] コンポーネント初期化
  │           ├─ [2] メモリ初期化
  │           ├─ [3] タスクシステム初期化
  │           ├─ [4] Botクライアント初期化
  │           ├─ [5] サーバーログイン
  │           └─ [6] スポーン完了
  │
  ↓
[メインループ開始]
  │
  ├─ 【300msごと】startEvents()
  │   └─ update()
  │       ├─ モード実行
  │       ├─ セルフプロンプター実行
  │       └─ タスク状態チェック
  │
  ├─ 【イベント駆動】メッセージ受信
  │   └─ handleMessage()
  │       ├─ バリデーション
  │       ├─ コマンド検出
  │       ├─ 履歴保存
  │       └─ LLM処理 → アクション実行
  │
  └─ 【イベント駆動】Botイベント
      └─ chat, health, death, etc.
```

---

## 4. 主要ファイルと役割

| ファイル | 役割 | 初期化 | ループ |
|---------|------|--------|--------|
| `main.js` | エントリーポイント | ✓ | |
| `src/mindcraft/mindcraft.js` | システム初期化 | ✓ | |
| `src/mindcraft/mindserver.js` | 通信ハブ | ✓ | ✓ |
| `src/agent/agent.js` | エージェントメインロジック | ✓ | ✓ |
| `src/agent/action_manager.js` | アクション実行管理 | ✓ | ✓ |
| `src/agent/self_prompter.js` | 自律意思決定 | | ✓ |
| `src/agent/commands/` | コマンド実装 | ✓ | ✓ |
| `src/models/prompter.js` | LLMプロンプト生成 | | ✓ |
| `src/agent/memory.js` | 記憶管理 | ✓ | ✓ |

---

## 5. タイミングチャート

```
時刻 | 処理内容
-----|----------
T=0ms      | プログラム起動 (main.js)
T=100ms    | MindServer起動
T=200ms    | Agent生成開始
T=500ms    | Minecraftボット接続開始
T=2000ms   | ログイン完了
T=3000ms   | スポーン完了、初期化終了
-----------|-------------------------------------
T=3000ms   | メインループ開始
T=3300ms   | 1回目のupdate()実行
T=3600ms   | 2回目のupdate()実行
T=3900ms   | 3回目のupdate()実行
...        | 300msごとにupdate()が永続実行
```

---

## まとめ

- **初期化**: `main.js` → `Mindcraft.init()` → `Agent.start()`（6フェーズ）
- **メインループ**: `startEvents()` が300msごとに `update()` を実行
- **イベント処理**: メッセージやボットイベントは非同期で処理
- **中心となるファイル**: `src/agent/agent.js`（395-477行目がメインループ）
