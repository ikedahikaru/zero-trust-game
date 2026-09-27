# ZERO / TRUST — 現行仕様 & 実装メモ（引き継ぎ用）

ハッカー×正体隠匿ゲーム。元仕様書 Ver.0.1 をベースに、HTML 1ファイルのプロトタイプとして実装済み。
このドキュメントは「元仕様からの確定事項・追加ルール」と「コードの構成」をまとめたもの。

- 実装ファイル：`index.html`（HTML/CSS/JS 1ファイル、外部依存は Google Fonts のみ）
- 旧公開版：https://claude.ai/artifact/V7C3qhfTvA2rbkr86ynQ1B （Version 5。現在の `index.html` より古い）
- 現状の形態：**ひとりプレイ（あなた＋CPU）** と **オンライン対戦（ホスト方式P2P、人間1〜10人＋CPU）**。

---

## 1. ゲーム概要

- 4〜10人。全員が匿名の `PLAYER A〜J`（割り当ては毎試合ランダム）。
- 各プレイヤーは秘密の **4桁PW（各桁0〜9）** と **役職・所属** を持つ。
- ATTACKで解析 → HACKで確定 → 4桁特定でクラック。チャットでは嘘OK、SHAREはシステム保証。
- 設計の土台：「人間は嘘をつくが、システムが返す解析結果は常に正しい」（偽情報を返すカードは不採用）。

### 人数別構成

| 人数 | A社 | B社 | BLACK | WHITE |
|---|---|---|---|---|
| 4 | 2 | 2 | 0 | 0 |
| 5 | 2 | 2 | 1 | 0 |
| 6 | 2 | 2 | 1 | 1 |
| 7 | 3 | 3 | 1 | 0 |
| 8 | 3 | 3 | 2 | 0 |
| 9 | 3 | 3 | 2 | 1 |
| 10 | 4 | 4 | 2 | 0 |

- 味方が誰かは知らされない（役職はクラックした時だけ判明）。
- A社・B社にはそれぞれ **企業PW（4桁）** がある。

---

## 2. 勝利条件（元仕様の「要調整」を確定させたもの）

用語：**陥落(X社)** ＝ X社の企業PWがクラック済み（誰か1人が企業PWを4桁そろえた） **または** X社構成員が全員クラック済み。

| 陣営 | 勝利条件 |
|---|---|
| A社 | BLACKが全員クラック済み かつ B社が陥落 |
| B社 | BLACKが全員クラック済み かつ A社が陥落 |
| BLACK | A社・B社がともに陥落（BLACK生存中）／ **または最終ラウンドまで生存** |
| WHITE | A社かB社が勝った時点で生存し、勝者側の誰にも **2桁以上** 特定されていなければ便乗勝利 |

- 最終ラウンド = **8 + 参加人数**（4人→12、10人→18）。
- 最終ラウンド終了時にBLACK全滅かつ決着なし → **時間切れ判定**：各社構成員が他陣営に対して特定した桁数＋他社PWの取得桁数の合計が多い方の勝ち。同点は引き分け。
- A社・B社の陥落が同時に成立（BLACK全滅時）→ 引き分け。
- BLACKは複数いる場合チームとして扱う。

---

## 3. ラウンドの流れ

`ACTION → EXECUTE → RESULT → DISCUSSION → HAND → DRAW → 次ラウンド`

1. **ACTION**：全員が秘密裏に「異なる2カテゴリ」から1つずつ選ぶ。
   - ATTACK＋DEFENSE／FIXED＋ATTACK／FIXED＋DEFENSE（FIXEDは HACK・SHARE から1つ）＝ 5パターン。
2. **EXECUTE**（処理順）
   1. DEFENSEの設置系（Information Lock, Stealth Mode, Mirror Guard, Log Eraser）
   2. 全HACKを解決（処理順はランダム）→ アクセスLv更新・クラック判定
   3. 全ATTACKを解決（処理順はランダム）。このラウンドにHACKでクラックされた相手を対象にしたATTACKは「有効な対象がいない」になる（クラックされた側のATTACKは実行される）
   4. DEFENSEの結果系（Attack Detector, Mirror Guard の反射結果, Access Log）
   5. PW変更系（Password Reset, Swap）※そのラウンドにクラックされた人は変更不可
   6. SHARE
   7. アクセスLvによる行動・結果の可視化（Lv1＝行動、Lv2以上＝MONITOR：行動＋結果）
   8. 使用カードを手札から除去 → 勝敗判定
3. **RESULT**：自分が得た情報とシステム通知を表示。
4. **DISCUSSION**：匿名チャット（CPUも1〜3件発言、嘘あり）。
5. **HAND**：KEEP / MULLIGAN。
6. **DRAW**：両手札を上限まで補充。

---

## 4. 手札

- ATTACK手札・DEFENSE手札を別管理。**各5枚**（コード定数 `HAND_SIZE`）。
- MULLIGAN：**枚数制限なし**。選んだカードを捨ててから5枚まで補充。
- デッキは無限（カードごとの重み `w` に従ってランダムドロー）。

---

## 5. 情報の種類とアクセスレベル

### 解析情報（ATTACK由来）と特定済み（HACK由来）
- 解析情報：その時点の事実。相手がPWを変えると古くなる。
- **CONFIRMED**（HACKで当てた桁）：永続アクセス権。相手がその桁を変更すると `追跡：PLAYER X 第2桁 8 → 4` のように自動通知され、表示も更新される。

### アクセスLv（相手に対して何桁CONFIRMEDか）

| Lv | 効果 |
|---|---|
| 1 | 相手の毎ラウンドの行動（カード名・パラメータ）が見える。到達時に過去の行動履歴も取得 |
| 2 | **MONITOR（常時）**：相手の行動＋結果（解析情報を含む）を毎ラウンド自動取得。Information Lock で遮断される |
| 3 | 到達時に**相手の保有解析情報を自分の正式情報としてコピー**（MONITORは継続） |
| 4 | **クラック**：相手は脱落（行動不可）。相手の役職と所属企業PWの2桁（未取得のものからランダム）を取得 |

- クラック発生は全体に公開（誰がやったか・役職は非公開）。

### 企業PW
- **企業サーバーへのHACKは廃止**。企業PWは構成員をクラックしたときの断片（未取得の2桁）でのみ手に入る。
- 1人のプレイヤーが4桁そろえた時点で **企業クラック** ＝ その企業は陥落（勝利条件は §2 のとおり。BLACK全滅も必要）。同じ企業の構成員を2人クラックすれば4桁そろう。
- 企業PWの断片はSHAREでは送れない（各プレイヤー個別の所持）。

---

## 6. FIXED アクション

### HACK（最新ルール）
- 対象プレイヤーを選び、**4桁それぞれに `0〜9` または `*` を入力**。
- 数字を入れた桁：正解なら CONFIRMED、不正解なら `第N桁 ≠ d` という解析情報が残る。
- `*`（ワイルドカード）：その桁は特定できないが、失敗扱いにもならない。
- 全桁 `*` は実行不可。CONFIRMED済みの桁は入力対象外。
- **不正アクセス検知**：数字を入れた桁が1つでも外れると、**された側に攻撃者のプレイヤー名と外れた桁が通知される**（`不正アクセス検知：PLAYER B があなたへのHACKに失敗した（第2桁）`）。攻撃側にも「検知された」と表示。
  - 全桁正解・`*` の桁は検知されない（成功だけのHACKは Access Log 以外では気づかれない）。
  - HACKした側が同じラウンドに **Log Eraser** を使っていれば、失敗しても通知されない（HACKした側には「通知を消去した」と表示）。
- 狙い：「多くの桁を賭けて一気に取るか、確信のある桁だけ入れて正体を隠すか」の駆け引き。

### MONITOR（常時・アクション枠なし）
- FIXEDアクションからは削除。対象にLv2以上（2桁以上CONFIRMED）なら毎ラウンド自動で発動する。
- そのラウンドの対象の行動＋結果（解析情報を含む）を取得。Information Lock で遮断される。

### SHARE
- 共有相手を複数選択（上限なし）。
- 同じラウンドのもう一方のアクション（ATTACK or DEFENSE）の結果を、相手の正式情報としてシステムが配信（改ざん不可）。過去の情報は送れない。

---

## 7. ATTACKカード

| ID | 名前 | 指定 | 効果 | 重み |
|---|---|---|---|---|
| A01 | Parity Scan | 対象・桁 | 奇数/偶数 | 3 |
| A02 | Random Parity Scan | なし | 未取得の(プレイヤー×桁)からランダム2箇所の奇偶 | 2 |
| A03 | Range Scan | 対象・桁 | 0〜4 / 5〜9 | 3 |
| A04 | Random Range Scan | なし | 未取得箇所ランダム2箇所の範囲 | 2 |
| A05 | Digit Search | 対象・数字 | その数字の個数 | 2 |
| A07 | Checksum | 対象 | 4桁合計 | 2 |
| A08 | Random Checksum | なし | ランダム2人の合計が ≤18 / ≥19 | 1 |
| A09 | Compare | 対象・桁・桁2 | 2桁の大小/同値 | 2 |
| A10 | Cross Compare | 対象・桁・対象2・桁2 | 異なる2人の桁の大小/同値 | 1 |
| A11 | Duplicate Scan | 対象 | 重複数字の有無 | 1 |
| A13 | Position Search | 桁・数字 | 条件に一致する人数 | 1 |

- ATTACKを遮断するのは Mirror Guard（指定相手からのATTACKのみ）。遮断された場合、攻撃者には「防御に遮断された」と表示。
- A10・A13 の結果は解析ソルバーには入らない（テキストのみ）。
- 削除済み：Global Digit Search（A06）, Unique Count（A12）, Frequency Scan（A14）。

## 8. DEFENSEカード

| ID | 名前 | 指定 | 効果 | 重み |
|---|---|---|---|---|
| D03 | Attack Detector | — | 自分をATTACKしたプレイヤーを特定 | 2 |
| D06 | Password Reset | 桁 | 指定桁をランダムな別の数字へ | 3 |
| D10 | Swap | 桁・桁2 | 2桁を交換 | 1 |
| D18 | Information Lock | — | 自分へのMONITORを遮断 | 1 |
| D19 | Stealth Mode | — | ランダムに対象を選ぶATTACK（A02, A04, A08）の抽選から外れる | 2 |
| D20 | Mirror Guard | 対象 | 指定相手からのATTACKを防ぎ、相手が指定していた桁と同じ桁の **相手のPWの数字** を取得（解析情報「第N桁 = d」）。桁指定のないATTACKならランダム1桁 | 1 |
| D21 | Log Eraser | — | 同じラウンドの自分のHACKが失敗しても、相手に通知されない（HACKと組み合わせて使う） | 2 |
| D22 | Access Log | — | 自分をATTACKまたはHACKの対象にしたプレイヤー全員と、その種類（ATTACK / HACK）を確認。成功したHACKも含む | 1 |

- 旧カード（Firewall, Wide Firewall, Counter Trace, Active Counter, Random Reset, Manual Reset, Shuffle, Honeypot, Hack Shield, Hack Detector）は削除済み。
- Mirror Guard の「指定していた桁」：Parity Scan / Range Scan / Compare は1つ目の桁、Cross Compare は自分側の桁、Random Parity / Random Range は抽選された桁。防げるのはATTACKのみ（HACKは防げない）。指定相手（どの相手を警戒しているか）は行動として公開される（Lv1以上の相手から見える）。
- Log Eraser を使っても Access Log には記録される（Access Log は Log Eraser を見破れる）。
- Stealth Mode は Global Digit Search / Position Search（全体集計）には影響しない。

- PW変更の結果テキストには新しい数字を出さない（SHAREやMONITORで漏れないように、「第2桁を変更した」だけ）。

---

## 9. UI（実装済み）

- **タイトル画面**：人数選択（4〜10）、構成表、ルール早見表。
- **レイアウト**：中央にコンソール（自分のPW・役職・勝利条件＋フェーズごとの操作）、TARGETS盤面、ログ（INTEL / CHAT / SYSTEM タブ）。PC幅は3カラム、スマホ幅は1カラム。
- **解析ソルバー**：各相手について、自分の解析情報＋CONFIRMEDから 10000通りを絞り込み、桁ごとの候補数字と残り候補数を表示。情報が矛盾したら「矛盾あり」を表示し、古い情報から除外して再計算。
- **TARGETSクリック指定**
  - プレイヤーカードをクリック → カードの「対象」を指定（Cross Compare は対象→対象2 と自動で進む。SHARE は複数トグル）。
  - 桁マスをクリック → 「対象＋桁」を同時に指定（Compare/Cross Compare は2回クリック）。HACK は桁マスクリックで「最有力の数字 ⇄ `*`」を切り替え。
  - 自分のPWの桁をクリック → DEFENSE の桁指定（Password Reset / Swap）。
  - 指定済みのマスに `ATK1`・`HACK 5`・`DEF1` などのラベルを表示。どこに入るかを盤面上部に表示。コンソールの選択スロットをクリックすると入力先を切り替え。
- **CPU**：狙う相手を持ち続けて攻撃。候補が絞れた桁だけHACK（BLACKはより慎重）。不正アクセスを受けると攻撃者を狙い返し、チャットで告発。チャットでは一定確率で嘘をつく（BLACK 50%、WHITE 25%、他 12%）。
- 自分がクラックされると観戦モード（全員の正体とPWを表示）。ゲーム終了画面で全員の役職・PWを公開。

---

## 10. コード構成（`index.html` 内 `<script>`）

| 区分 | 主な関数・データ |
|---|---|
| データ | `COMP`（人数構成）, `ROLE`, `AC`（ATTACK）, `DC`（DEFENSE）, `FX`（FIXED説明）, `HAND_SIZE`, `ALL`（0000〜9999） |
| 状態 | `S`（ゲーム全体：players, co（企業PW）, sys, chat, round, max, winner, epoch）, `UI`（選択中カード・入力先・タブ等） |
| プレイヤー | `{id, letter, role, human, pw, alive, handA, handD, facts, feed, conf[targetId][4], lvSeen, roleKnown, coConf, history, act, rr, fx, atk, hk, suspect}` |
| 情報 | `mk`（ファクト生成）, `predOf`（候補判定の関数）, `addFact`（同じ種類の情報は新しい方で上書き）, `gain`（フィードに追加＋ファクト登録＋ラウンド結果 `rr` に記録）, `factText` |
| ソルバー | `analyze(viewer, target)` → `{n, per[4][10], conflict, stale}`（epochでキャッシュ） |
| 実行 | `execute`, `resolveAttack`, `resolveHack`, `updateAccess`, `crack`, `crackCompany`, `resolveDefense`, `resolveReset`, `resolveShare`, `resolveVisibility`, `checkWin` |
| CPU | `botAct`, `hackPlan`, `botAttack`, `botDefense`, `botMulligan`, `botLine`（チャット） |
| UI操作 | `pickCard`, `playerFields/activePF/targetPick`（プレイヤー指定）, `posSteps/activePP/digitPick/cellMarks`（桁指定）, `hackGuesses`, `slotError`, `humanAct` |
| 描画 | `render`（全体再描画、innerHTML方式）, `consoleHTML`, `actionHTML`, `boardHTML`, `logsHTML`, `overHTML` など |

- エンジン部分はDOMに依存しない。Node から `<script>` の中身を読み込めば、CPU同士の試合を回して動作とバランスを確認できる（`newGame(n, true)` で全員CPU）。

### 直近のシミュレーション（CPU同士、15試合ずつ。DEFENSE 8種版）

| 人数 | 結果 | 平均ラウンド |
|---|---|---|
| 4 | A 7 / B 7 / 引分 1 | 12.0 |
| 6 | BLACK 9 / B 4 / A 2 | 14.0 |
| 8 | BLACK 14 / B 1 | 16.0 |
| 10 | BLACK 14 / B 1 | 18.0 |

→ CPUは役職を推理しないため、人数が多いとBLACKの逃げ切りが多い。人間が推理すれば変わるが、バランス調整の余地あり。

---

## 10.4 BGM（実装済み）

- `bgm/title.mp3`：タイトル・ロビー・試合終了画面。`bgm/action.mp3`：試合中（ACTION〜HAND）。どちらもループ再生、音量0.45。
- `render()` のたびに `bgmUpdate()` を呼び、今の画面に合う曲を流す。自動再生の制限があるため、最初の pointerdown / keydown で再生を試みる。
- 右下の固定ボタン（`#bgm-btn`、`#app` の外に置く）で ON / OFF。設定は localStorage `zt-bgm` に保存する。
- Suno（有料プラン）で作成。

## 10.5 オンライン対戦（実装済み）

- 方式：**ホスト方式P2P**。WebRTC（PeerJS 1.5.4 を jsDelivr から、オンライン開始時にだけ読み込む）。シグナリングにはPeerJSの公開サーバーを使う。静的ホスティング（GitHub Pages）だけで動く。
- ホスト＝「部屋を作る」を押した人のブラウザ。ゲームの状態 `S` をすべて持ち、処理を行う。ルームコードは5文字（PeerJSのID `zerotrust-v1-<コード>`）。URLの `#room=CODE` を開くと、コードが入力済みになる。
- **ひとりプレイも同じ処理経路**：接続なしのホストとして動く（`submit` → `hostHandle` → `hostAdvance`）。
- 進行：各フェーズで、必要な人間プレイヤー全員の入力がそろったら次へ進む。
  - ACTION：生存している人間全員の行動（`S.acts`）。生存者がいなければ、観戦者全員の「ラウンドを実行」
  - RESULT / DISCUSSION：生存している人間全員の「次へ」（`S.ready`）。生存者がいなければ全員
  - HAND：生存している人間全員のMULLIGAN（`S.mulls`）
  - 最終ラウンドの「最終結果を見る」は各自の画面だけで切り替える（`UI.over`）
- 通信メッセージ
  - クライアント→ホスト：`hello{token}` / `act{act,ph,r}` / `ready{ph,r}` / `mull{a,d,ph,r}` / `chat{text}`
  - ホスト→クライアント：`lobby{count,n}` / `state{me,S}` / `bye{why}`
  - `ph`・`r`（フェーズ・ラウンド）が現在と違う入力は捨てる（遅れて届いた入力の誤処理を防ぐ）
- **情報の秘匿**（`viewFor(id)`）：本人以外のプレイヤーについては、次の情報しか送らない。
  - letter / alive
  - 本人がクラックして判明した役職
  - 本人がCONFIRMED済みの桁だけのPW
  
  他人の手札・解析情報・フィード・人間かCPUか・行動内容（`S.acts`）は送らない。企業PWは陥落するまで送らない。本人がクラックされた後と試合終了後は、全情報を送る（ひとりプレイの観戦モードと同じ）。
- 入力の検証（`cleanAct` / `cleanMull`）：手札にないカード、範囲外の値、組み合わせの誤りはホスト側で除外する。不正な行動はCPUの行動に置き換える。
- 切断と再接続：クライアントの端末ごとにトークン（localStorage `zt-token`）を持つ。切断中の席はCPUが代行し（誰が人間かを知られないよう、全体への通知はしない）、同じトークンで入り直すと元の席に戻る。試合中に新しい人は参加できない。
- ホストがページを閉じると試合は終了する（閉じる前に確認ダイアログを出す）。
- 制限：ホストのブラウザは全情報を持つ。TURNサーバーがないため、対称型NATの回線同士では接続できないことがある。

## 11. 未実装・今後の検討事項

- オンライン対戦の強化：ホストの不正を防ぐ専用サーバー、TURNサーバー（接続できない回線への対応）、ホスト切断時の引き継ぎ、行動の制限時間。
- BLACKの勝ちやすさの調整（最終ラウンド数、BLACKの勝利条件、CPUの推理力）。
- ホワイトハッカーの「解析されていない」条件（現在は2桁以上特定で失格）の妥当性。
- A03/A04 の差し替え候補、監視系ATTACK（A15以降）の再設計。
- DEFENSEカードの追加・調整（現在8種）。
- HACKで検知された時、外れた桁まで相手に伝えるかどうか（現在は伝える）。
- CPUのチャット応答はテンプレートのみ（人間の発言内容は解釈しない）。
- 状態の保存（リロードでゲームは消える）。
