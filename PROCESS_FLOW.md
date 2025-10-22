# Mindcraft 処理フロー（階層別）

## 概要
このドキュメントは、`agent.js`を中心に、Mindcraftの処理の流れを階層的にまとめたものです。
処理を4つの階層に分けて、トップレベルから詳細まで段階的に説明します。

---

# 第1層: トップレベルの流れ

```
起動
 ├─ 初期化フェーズ（1回のみ）
 │   └─ Agent.start() を実行
 └─ 実行フェーズ（継続的）
     ├─ メインループ（300ms周期）
     └─ イベント駆動処理（非同期）
```

---

# 第2層: Agent.start() の内部構造

**ファイル: `src/agent/agent.js:21-109`**

## 2.1 初期化の流れ

```
Agent.start(load_mem, init_message, count_id)
 │
 ├─ [Phase 1] コンポーネント生成 (21-36行目)
 │   ├─ ActionManager
 │   ├─ Prompter
 │   ├─ History
 │   ├─ Coder
 │   ├─ NPCController
 │   ├─ MemoryBank
 │   └─ SelfPrompter
 │
 ├─ [Phase 2] メモリ読み込み (39-42行目)
 │   └─ history.load()
 │
 ├─ [Phase 3] タスク初期化 (43-51行目)
 │   └─ new Task()
 │
 ├─ [Phase 4] ボット初期化 (53-56行目)
 │   ├─ initBot()
 │   └─ initModes()
 │
 ├─ [Phase 5] ログインイベント (58-67行目)
 │   └─ bot.on('login', ...)
 │
 └─ [Phase 6] スポーンイベント (73-108行目)
     ├─ VisionInterpreter生成
     ├─ _setupEventHandlers() 呼び出し
     ├─ startEvents() 呼び出し ← メインループ開始
     └─ タスク初期化
```

---

# 第3層: 実行フェーズの詳細

## 3.1 メインループ: startEvents()

**ファイル: `src/agent/agent.js:395-477`**

### 3.1.1 イベントハンドラーの登録 (395-456行目)

```
startEvents()
 │
 ├─ カスタムイベント
 │   ├─ 'time' → sunrise/noon/sunset/midnight
 │   └─ 'health' → ダメージ検出
 │
 ├─ システムイベント
 │   ├─ 'error' → エラーログ
 │   ├─ 'end' → 切断処理
 │   ├─ 'death' → アクション停止
 │   ├─ 'kicked' → キック処理
 │   ├─ 'messagestr' → 死亡メッセージ処理
 │   └─ 'idle' → アイドル時の復帰処理
 │
 └─ NPCコントローラー初期化
     └─ npc.init()
```

### 3.1.2 メインループの実装 (462-474行目)

```javascript
const INTERVAL = 300; // 300ミリ秒周期

setInterval(async () => {
    await this.update(delta);
}, INTERVAL);
```

**300msごとに実行される update() の内容:**

```
update(delta)  ← 479-483行目
 ├─ bot.modes.update()        ← モードシステム更新
 ├─ self_prompter.update()    ← 自律プロンプト更新
 └─ checkTaskDone()           ← タスク完了チェック
```

## 3.2 イベント駆動処理

### 3.2.1 イベントハンドラーのセットアップ

**ファイル: `src/agent/agent.js:111-183` (_setupEventHandlers)**

```
_setupEventHandlers(save_data, init_message)
 │
 ├─ チャットイベント登録 (146-152行目)
 │   ├─ bot.on('whisper', respondFunc)
 │   └─ bot.on('chat', respondFunc)
 │
 ├─ Auto-eat設定 (155-159行目)
 │
 ├─ メモリ復元処理 (161-176行目)
 │   └─ self_prompter.handleLoad()
 │
 └─ 初期メッセージ送信 (177-182行目)
     └─ handleMessage() または openChat()
```

### 3.2.2 メッセージ受信の流れ

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

# 第4層: 詳細な処理の内部

## 4.1 handleMessage() の完全な流れ

**ファイル: `src/agent/agent.js:218-346`**

### 構造:

```
handleMessage(source, message, max_responses)
 │
 ├─ [準備] タスク完了チェック (219行目)
 │   └─ checkTaskDone()
 │
 ├─ [検証] 入力の妥当性チェック (220-223行目)
 │
 ├─ [設定] レスポンス回数の決定 (225-231行目)
 │   └─ max_responses設定
 │
 ├─ [判定] メッセージ送信元の判定 (233-234行目)
 │   ├─ self_prompt判定
 │   └─ from_other_bot判定
 │
 ├─ [コマンド] ユーザーコマンドの処理 (236-254行目)
 │   ├─ containsCommand() でコマンド検出
 │   ├─ commandExists() で存在確認
 │   └─ executeCommand() で実行
 │
 ├─ [翻訳] メッセージの英語翻訳 (260行目)
 │
 ├─ [ログ] 行動ログの追加 (265-273行目)
 │   └─ bot.modes.flushBehaviorLog()
 │
 ├─ [履歴] 履歴への追加 (276-277行目)
 │   └─ history.add()
 │
 └─ [ループ] レスポンスループ (281-343行目)
     └─ for (i=0; i<max_responses; i++)
         ├─ 中断チェック
         ├─ LLMプロンプト実行
         ├─ レスポンス解析
         └─ コマンド実行 or 会話応答
```

### 4.1.1 レスポンスループの詳細 (281-343行目)

```
for (i=0; i<max_responses; i++)
 │
 ├─ [チェック] 中断判定 (282行目)
 │   └─ checkInterrupt()
 │
 ├─ [取得] 履歴取得 (283行目)
 │   └─ history.getHistory()
 │
 ├─ [LLM] プロンプト実行 (284行目)
 │   └─ prompter.promptConvo(history)
 │       ↓
 │       LLMがレスポンスを生成
 │
 ├─ [解析] レスポンス内容の確認 (288-291行目)
 │   └─ 空レスポンスならループ終了
 │
 ├─ [検出] コマンド検出 (293行目)
 │   └─ containsCommand(res)
 │
 └─ [分岐] コマンドの有無で分岐
     │
     ├─ [コマンドあり] (295-335行目)
     │   ├─ truncCommandMessage() でトリミング
     │   ├─ 履歴に追加
     │   ├─ commandExists() で存在確認
     │   ├─ 中断チェック
     │   ├─ self_prompter.handleUserPromptedCmd()
     │   ├─ routeResponse() で応答表示
     │   ├─ executeCommand() でコマンド実行
     │   └─ 実行結果を履歴に追加
     │
     └─ [会話応答] (336-340行目)
         ├─ 履歴に追加
         ├─ routeResponse() で応答
         └─ ループ終了
```

## 4.2 routeResponse() の処理

**ファイル: `src/agent/agent.js:348-366`**

```
routeResponse(to_player, message)
 │
 ├─ shut_upチェック (349行目)
 │
 ├─ 送信先の判定 (350-355行目)
 │   └─ system/selfの場合、last_senderにリダイレクト
 │
 └─ 送信方法の分岐
     │
     ├─ [他エージェント宛て] (357-360行目)
     │   └─ convoManager.sendToBot()
     │
     └─ [オープンチャット] (361-365行目)
         └─ openChat()
```

## 4.3 openChat() の処理

**ファイル: `src/agent/agent.js:368-393`**

```
openChat(message)
 │
 ├─ [翻訳] コマンド部分を除いて翻訳 (369-378行目)
 │   ├─ containsCommand() でコマンド検出
 │   ├─ コマンド前の部分のみ翻訳
 │   └─ handleTranslation()
 │
 ├─ [整形] 改行をスペースに変換 (379行目)
 │
 └─ [送信] 送信方法の分岐 (381-392行目)
     │
     ├─ [only_chat_with設定あり] (381-384行目)
     │   └─ bot.whisper() で個別送信
     │
     └─ [通常送信] (386-392行目)
         ├─ speak() で音声出力
         ├─ bot.chat() でゲーム内送信
         └─ sendOutputToServer() でサーバー送信
```

## 4.4 update() の詳細

**ファイル: `src/agent/agent.js:479-483`**

```
update(delta)
 │
 ├─ [モード] bot.modes.update()
 │   └─ 各モード（採掘、戦闘など）の状態更新
 │
 ├─ [自律] self_prompter.update(delta)
 │   └─ 自律的な行動の判断と実行
 │
 └─ [タスク] checkTaskDone()
     └─ タスク完了判定と終了処理
```

---

# 第5層: 主要コンポーネントの詳細

## 5.1 ActionManager

**ファイル: `src/agent/action_manager.js`**

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

## 5.2 SelfPrompter

**ファイル: `src/agent/self_prompter.js`**

```
SelfPrompter
 ├─ update(delta)
 │   └─ 一定間隔で自律的にプロンプト実行
 │
 ├─ shouldInterrupt()
 ├─ handleUserPromptedCmd()
 └─ stop()
```

## 5.3 History

**ファイル: `src/agent/history.js`**

```
History
 ├─ add(source, message)
 ├─ getHistory()
 ├─ save()
 └─ load()
```

## 5.4 ConvoManager

**ファイル: `src/agent/conversation.js`**

```
ConvoManager
 ├─ sendToBot(agent_name, message)
 ├─ receiveFromBot(agent_name, msg_package)
 ├─ inConversation(agent_name)
 └─ endAllConversations()
```

---

# 階層別実行タイミング

## 第1層: 起動〜初期化完了
```
T=0ms      起動
T=3000ms   初期化完了、メインループ開始
```

## 第2層: メインループ開始後
```
T=3000ms   startEvents() 実行
T=3300ms   1回目の update()
T=3600ms   2回目の update()
...        以降300msごと
```

## 第3層: イベント駆動（非同期）
```
随時       chat/whisperイベント
           → respondFunc()
           → handleMessage()
```

## 第4層: handleMessage内部
```
handleMessage() 実行中:
 - プロンプト生成: 数百ms〜数秒
 - LLM応答待機: 1〜10秒程度
 - コマンド実行: 数ms〜数分
```

---

# まとめ: 階層構造

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

**中心ファイル**: `src/agent/agent.js`
- 初期化: 21-109行目
- イベントセットアップ: 111-183行目
- メッセージ処理: 218-346行目
- メインループ: 395-477行目
