---
marp: true
theme: default
paginate: true
header: 'Mindcraft 処理フロー'
footer: 'agent.jsを中心に'
---

<!-- _class: lead -->
# Mindcraft 処理フロー

## agent.jsを中心とした階層別解説

---

## 📑 目次

1. **第1層**: トップレベルの流れ
2. **第2層**: Agent.start() の内部構造
3. **第3層**: 実行フェーズの詳細
4. **第4層**: 詳細な処理の内部
5. **第5層**: 主要コンポーネント
6. まとめ

---

<!-- _class: lead -->
# 第1層
## トップレベルの流れ

---

## 第1層: 全体構造

```
起動
 ├─ 初期化フェーズ（1回のみ）
 │   └─ Agent.start() を実行
 └─ 実行フェーズ（継続的）
     ├─ メインループ（300ms周期）
     └─ イベント駆動処理（非同期）
```

**2つのフェーズ:**
- ✅ **初期化**: 起動時に1回だけ実行
- 🔄 **実行**: 継続的に動作

---

<!-- _class: lead -->
# 第2層
## Agent.start() の内部構造

---

## 第2層: 初期化の6つのフェーズ

**ファイル:** `src/agent/agent.js:21-109`

```
Agent.start(load_mem, init_message, count_id)
 │
 ├─ [Phase 1] コンポーネント生成 (21-36行目)
 ├─ [Phase 2] メモリ読み込み (39-42行目)
 ├─ [Phase 3] タスク初期化 (43-51行目)
 ├─ [Phase 4] ボット初期化 (53-56行目)
 ├─ [Phase 5] ログインイベント (58-67行目)
 └─ [Phase 6] スポーンイベント (73-108行目)
```

---

## Phase 1: コンポーネント生成 (21-36)

```javascript
this.actions = new ActionManager(this);
this.prompter = new Prompter(this, settings.profile);
this.history = new History(this);
this.coder = new Coder(this);
this.npc = new NPCContoller(this);
this.memory_bank = new MemoryBank();
this.self_prompter = new SelfPrompter(this);
```

**7つの主要コンポーネントを生成**

---

## Phase 2-3: メモリとタスク (39-51)

### Phase 2: メモリ読み込み
```javascript
if (load_mem) {
    save_data = this.history.load();
}
```

### Phase 3: タスク初期化
```javascript
this.task = new Task(this, settings.task, taskStart);
this.blocked_actions = settings.blocked_actions.concat(...);
blacklistCommands(this.blocked_actions);
```

---

## Phase 4: ボット初期化 (53-56)

```javascript
this.bot = initBot(this.name);
initModes(this);
```

**実行内容:**
- Minecraftボット（mineflayer）の作成
- 動作モード（採掘、戦闘など）の初期化

---

## Phase 5: ログインイベント (58-67)

```javascript
this.bot.on('login', () => {
    console.log(this.name, 'logged in!');
    serverProxy.login();

    // スキン設定
    if (this.prompter.profile.skin)
        this.bot.chat(`/skin set URL ...`);
});
```

**サーバーログイン時の処理**

---

## Phase 6: スポーンイベント (73-108)

```javascript
this.bot.once('spawn', async () => {
    // Vision Interpreter生成
    this.vision_interpreter = new VisionInterpreter(this, ...);

    // イベントハンドラー設定
    this._setupEventHandlers(save_data, init_message);

    // メインループ開始 ← ここが重要！
    this.startEvents();

    // タスク初期化
    if (settings.task) {
        this.task.initBotTask();
        this.task.setAgentGoal();
    }
});
```

---

<!-- _class: lead -->
# 第3層
## 実行フェーズの詳細

---

## 第3層: 実行フェーズの構成

### 2つの実行パターン

1. **メインループ** - `startEvents()`
   - 300msごとに定期実行
   - `src/agent/agent.js:395-477`

2. **イベント駆動処理**
   - メッセージ受信時
   - ボットイベント発生時

---

## 3.1 メインループ: startEvents()

**ファイル:** `src/agent/agent.js:395-477`

### イベントハンドラーの登録 (395-456)

```
startEvents()
 ├─ カスタムイベント
 │   ├─ 'time' → sunrise/noon/sunset/midnight
 │   └─ 'health' → ダメージ検出
 │
 ├─ システムイベント
 │   ├─ 'error', 'end', 'death', 'kicked'
 │   ├─ 'messagestr' → 死亡メッセージ処理
 │   └─ 'idle' → アイドル時の復帰処理
 │
 └─ NPCコントローラー初期化
```

---

## メインループの実装 (462-474)

```javascript
const INTERVAL = 300; // 300ミリ秒周期

setTimeout(async () => {
    while (true) {
        let start = Date.now();
        await this.update(start - last);
        let remaining = INTERVAL - (Date.now() - start);
        if (remaining > 0) {
            await new Promise(resolve => setTimeout(resolve, remaining));
        }
        last = start;
    }
}, INTERVAL);
```

**🔄 300msごとに update() を実行**

---

## update() の内容 (479-483)

```javascript
async update(delta) {
    await this.bot.modes.update();      // モード更新
    this.self_prompter.update(delta);   // 自律プロンプト
    await this.checkTaskDone();         // タスク完了チェック
}
```

**3つの処理を毎回実行:**
1. 動作モードの更新（採掘、戦闘など）
2. 自律的な意思決定
3. タスク完了判定

---

## 3.2 イベントハンドラーのセットアップ

**ファイル:** `src/agent/agent.js:111-183`

```
_setupEventHandlers(save_data, init_message)
 │
 ├─ チャットイベント登録 (146-152)
 │   ├─ bot.on('whisper', respondFunc)
 │   └─ bot.on('chat', respondFunc)
 │
 ├─ Auto-eat設定 (155-159)
 │
 ├─ メモリ復元処理 (161-176)
 │
 └─ 初期メッセージ送信 (177-182)
```

---

## メッセージ受信の流れ

```
respondFunc(username, message)  ← 121-142行目
 │
 ├─ バリデーション
 │   ├─ 空メッセージチェック
 │   ├─ 自分自身からのメッセージを無視
 │   ├─ only_chat_withリストチェック
 │   └─ ignore_messagesチェック
 │
 ├─ 翻訳処理
 │   └─ handleEnglishTranslation()
 │
 └─ メッセージ処理
     └─ handleMessage() 呼び出し
```

---

<!-- _class: lead -->
# 第4層
## 詳細な処理の内部

---

## 4.1 handleMessage() の全体構造

**ファイル:** `src/agent/agent.js:218-346`

```
handleMessage(source, message, max_responses)
 │
 ├─ [準備] タスク完了チェック
 ├─ [検証] 入力の妥当性チェック
 ├─ [設定] レスポンス回数の決定
 ├─ [判定] メッセージ送信元の判定
 ├─ [コマンド] ユーザーコマンドの処理
 ├─ [翻訳] メッセージの英語翻訳
 ├─ [ログ] 行動ログの追加
 ├─ [履歴] 履歴への追加
 └─ [ループ] レスポンスループ
```

---

## レスポンスループの詳細 (281-343)

```
for (i=0; i<max_responses; i++)
 │
 ├─ [チェック] 中断判定
 ├─ [取得] 履歴取得
 ├─ [LLM] プロンプト実行 ← LLMがレスポンス生成
 ├─ [解析] レスポンス内容の確認
 ├─ [検出] コマンド検出
 └─ [分岐] コマンドの有無で分岐
     ├─ [コマンドあり] → executeCommand()
     └─ [会話応答] → routeResponse()
```

---

## LLMプロンプト実行の詳細

```javascript
// 履歴取得
let history = this.history.getHistory();

// LLMにプロンプト送信
let res = await this.prompter.promptConvo(history);

// レスポンス確認
if (res.trim().length === 0) {
    break; // 空レスポンスならループ終了
}

// コマンド検出
let command_name = containsCommand(res);
```

**⏱️ 処理時間: 1〜10秒程度**

---

## コマンド実行の流れ (295-335)

```
[コマンドあり]
 ├─ truncCommandMessage() でトリミング
 ├─ 履歴に追加
 ├─ commandExists() で存在確認
 ├─ 中断チェック
 ├─ self_prompter.handleUserPromptedCmd()
 ├─ routeResponse() で応答表示
 ├─ executeCommand() でコマンド実行 ← 実際の処理
 └─ 実行結果を履歴に追加
```

---

## 会話応答の流れ (336-340)

```
[会話応答]
 ├─ 履歴に追加
 ├─ routeResponse() で応答
 └─ ループ終了
```

**コマンドがない場合は1回の応答でループ終了**

---

## 4.2 routeResponse() の処理

**ファイル:** `src/agent/agent.js:348-366`

```
routeResponse(to_player, message)
 │
 ├─ shut_upチェック
 ├─ 送信先の判定
 │   └─ system/selfの場合、last_senderにリダイレクト
 └─ 送信方法の分岐
     ├─ [他エージェント宛て] → convoManager.sendToBot()
     └─ [オープンチャット] → openChat()
```

---

## 4.3 openChat() の処理

**ファイル:** `src/agent/agent.js:368-393`

```
openChat(message)
 │
 ├─ [翻訳] コマンド部分を除いて翻訳
 │   └─ handleTranslation()
 │
 ├─ [整形] 改行をスペースに変換
 │
 └─ [送信] 送信方法の分岐
     ├─ [only_chat_with設定あり] → bot.whisper()
     └─ [通常送信]
         ├─ speak() で音声出力
         ├─ bot.chat() でゲーム内送信
         └─ sendOutputToServer()
```

---

<!-- _class: lead -->
# 第5層
## 主要コンポーネントの詳細

---

## 5.1 ActionManager

**ファイル:** `src/agent/action_manager.js`

```
ActionManager
 ├─ executeAction(command)
 │   ├─ タイムアウト設定（最大10分）
 │   ├─ アクション実行
 │   └─ 結果返却
 │
 ├─ cancelResume()
 ├─ resumeAction()
 └─ stop()
```

**役割:** アクションの実行管理とタイムアウト制御

---

## 5.2 SelfPrompter

**ファイル:** `src/agent/self_prompter.js`

```
SelfPrompter
 ├─ update(delta)
 │   └─ 一定間隔で自律的にプロンプト実行
 │
 ├─ shouldInterrupt()
 ├─ handleUserPromptedCmd()
 └─ stop()
```

**役割:** エージェントの自律的な意思決定

---

## 5.3 History

**ファイル:** `src/agent/history.js`

```
History
 ├─ add(source, message)
 ├─ getHistory()
 ├─ save()
 └─ load()
```

**役割:** 会話履歴の管理と永続化

---

## 5.4 ConvoManager

**ファイル:** `src/agent/conversation.js`

```
ConvoManager
 ├─ sendToBot(agent_name, message)
 ├─ receiveFromBot(agent_name, msg_package)
 ├─ inConversation(agent_name)
 └─ endAllConversations()
```

**役割:** 複数エージェント間の会話管理

---

<!-- _class: lead -->
# 階層別実行タイミング

---

## タイミングチャート

### 第1層: 起動〜初期化完了
```
T=0ms      起動
T=3000ms   初期化完了、メインループ開始
```

### 第2層: メインループ開始後
```
T=3000ms   startEvents() 実行
T=3300ms   1回目の update()
T=3600ms   2回目の update()
...        以降300msごと
```

---

## タイミングチャート (続き)

### 第3層: イベント駆動（非同期）
```
随時       chat/whisperイベント
           → respondFunc()
           → handleMessage()
```

### 第4層: handleMessage内部
```
handleMessage() 実行中:
 - プロンプト生成: 数百ms〜数秒
 - LLM応答待機: 1〜10秒程度
 - コマンド実行: 数ms〜数分
```

---

<!-- _class: lead -->
# まとめ

---

## 階層構造の全体像

```
第1層: 起動 → 初期化 → 実行
         ↓
第2層: Agent.start() → startEvents() + update()
         ↓
第3層: イベントハンドラー + メインループ（300ms）
         ↓
第4層: handleMessage() → レスポンスループ → コマンド実行
         ↓
第5層: 各コンポーネント（ActionManager, SelfPrompter等）
```

---

## 中心ファイル: agent.js

**`src/agent/agent.js` の重要な部分:**

| 行番号 | 処理内容 |
|--------|----------|
| 21-109 | 初期化（Agent.start） |
| 111-183 | イベントセットアップ |
| 218-346 | メッセージ処理 |
| 395-477 | メインループ |

**このファイルがMindcraftの中核！**

---

## 主要な処理パターン

### 🔄 定期実行（300ms）
- `update()` → modes, self_prompter, タスクチェック

### 📨 イベント駆動
- `handleMessage()` → LLM → コマンド実行 or 会話

### ⚡ 一度だけ
- `Agent.start()` → 6つのフェーズで初期化

---

## 重要なポイント

1. **初期化は6つのフェーズ**で段階的に実行
2. **メインループは300ms周期**で常時動作
3. **メッセージ処理は非同期**でイベント駆動
4. **LLMプロンプトがループ内**で複数回実行可能
5. **agent.jsが全ての中心**

---

<!-- _class: lead -->
# ありがとうございました

## 質問・フィードバックをお待ちしています
