---
title: "4 台で SIM 決勝の練習をする: make practice-4car と、実際に走らせて見つかった 4 つの罠"
emoji: "🏎️"
type: "tech"
topics: ["自動運転AIチャレンジ", "ROS2", "AWSIM", "Docker", "CycloneDDS"]
published: true
---

Team Hayes（Ajayaditya Lokchandra, Nithisha Venkatesh）です。自動運転AIチャレンジ2026 の SIM 決勝は 4 台同時のレースですが、手元で「別々の提出物 4 つ」を決勝と同じ条件で走らせる方法は、スターターキットにはまだありません。そこで 1 台の PC で最大 4 つの tar.gz をレースさせる `make practice-4car` を作り、PR にしました。

- PR: https://github.com/AutomotiveAIChallenge/aichallenge-racingkart/pull/345
- 関連 issue: https://github.com/AutomotiveAIChallenge/aichallenge-racingkart/issues/348

この記事では使い方と作りに加えて、**実際に 4 台レースを最後まで走らせるまでに踏んだ 4 つの罠**を書きます。どれもユニットテストだけでは見つからず、本物の AWSIM で走らせて初めて出てきたものです。

## 結論

- `make practice-4car SUBMISSIONS="a.tar.gz b.tar.gz c.tar.gz d.tar.gz"` で、SIM 決勝と同じ AWSIM 条件（4 台・6 周・420 秒・sync 開始・ハンディキャップ/ランキング on・NPC なし）のレースが 1 台の PC で走ります。
- 実際に 4 つのビルドで 4 台 × 6 周を完走し、順位・ラップ・ペナルティ・順位の入れ替わりが自動でまとまりました（下の図）。
- 罠は 4 つ: (1) WSL2 で CycloneDDS が動かない、(2) 起動が遅れた車が「準備完了」を受け取れず止まる、(3) Mac で作った tar.gz が弾かれる、(4) スタート指令が黙って失敗する。

![4 台 × 6 周のラップタイム](https://raw.githubusercontent.com/theCodeForgerHQ/zenn-content/main/images/m8/fig3-lap-times.png)

| 順位 | 出走位置 | 周回 | 合計 | ベスト | ペナルティ |
|---|---|---|---|---|---|
| 1 | 1 | 6 | 292.78 s | 47.36 s | 0 |
| 2 | 4 | 6 | 293.98 s | 48.69 s | 0 |
| 3 | 2 | 6 | 299.42 s | 45.71 s | crash 1（10.5 s） |
| 4 | 3 | 6 | 305.58 s | 46.10 s | crash 1（10.0 s） |

## なぜ必要か

スターターキットには `make dev4` があり、AWSIM を 4 台モードで起動できます。ただし 4 台とも同じ workspace（自分の `aichallenge_submit`）のコピーで、周回数や順位の採点もありません。混雑やスタートの練習にはなりますが、「自分の新ビルド vs 旧ビルド」「公開されている他チームの提出物との練習」はできません。

## 使い方

```bash
make practice-4car SUBMISSIONS="submit/a.tar.gz submit/b.tar.gz submit/c.tar.gz submit/d.tar.gz"
```

- 出走位置 N = `ROS_DOMAIN_ID` N = `SUBMISSIONS` の N 番目（SIM 決勝の「出走位置 N のチームは Autoware PC N」と同じ対応）
- `GRID=rotate ROUND=0..3` で、4 回回すと全員が全位置からスタートします
- `HANDICAP=on|off`、`NPC=0..3`、`CLASS=s2r|e2e`、`HEADLESS=1`（S2R のみ）、`PIN=1`（各車を 3 コアに固定、17 CPU 以上）
- 結果は `output/<timestamp>/` に、AWSIM の `result-summary.json` / `dN-result-details.json` と `practice-summary.md`

## 作り

- tar.gz ごとに `output/practice/ws/<sha256>/` に workspace を作ってビルドし、キャッシュします
- 各車は `autoware-slot` というサービスで起動し、`/aichallenge/workspace` だけをその車の workspace に差し替えます
- **既存の環境には手を入れていません。** `docker-compose*.yml` や `run_autoware.bash`、既存の make ターゲットは変更せず、コンテナ設定は `make practice-4car` の実行中だけ読まれる overlay（`aichallenge/practice/compose.practice.yml`）にあります。`.env` の `COMPOSE_FILE`（gpu / sound の指定）はそのまま引き継ぎます
- `practice-final.sh` の AWSIM 引数は `s2r-final.sh` と同じで（`--sound off` のみ違う）、両者がずれないことをテストで確認しています

## 罠 1: WSL2 では CycloneDDS が参加者を作れない

最初の実行では、AWSIM が `WaitStart` のまま、4 台とも `wait until clock received...` のまま止まりました。ROS のトピックが一切届いていません。

原因はスターターキットの `vehicle/cyclonedds.xml` です。CycloneDDS を `lo` に固定していますが、WSL2 では `lo` に `127.0.0.1` のほかに `10.255.255.254/32` も付いていて、CycloneDDS が参加者を作れず、エラーも出ません。AWSIM は自分の DDS を持っているので、AWSIM と Autoware がお互いを見つけられなくなります。

シミュレーション用の PC では、次のように自動選択にすると動きます（実車ではこの変更をしないでください）。

```xml
<NetworkInterface autodetermine="true" priority="default" multicast="default" />
```

## 罠 2: 起動が遅れた車は「準備完了」を受け取れない

DDS を直すと 3 台は走り出しましたが、4 台目だけが制御要求を出さないまま 600 秒経ちました。各車のログの時刻を並べると理由が分かります。

![AWSIM を先に起動すると、最後の車が Grounded を受け取れない](https://raw.githubusercontent.com/theCodeForgerHQ/zenn-content/main/images/m8/fig2-startup-race.png)

`autostart_orchestrator` は `/awsim/state` が `Grounded` / `Ready` / `Start` になるのを待ってから、初期位置と制御モード要求を送ります。AWSIM はこの状態を各車に 1 回だけ送ります。一方 orchestrator の購読は既定の volatile QoS なので、購読がつながる前に送られたメッセージは受け取れません。AWSIM を先に起動すると、最後に起動した車（d4）は Grounded が送られている最中に購読を作り、取りこぼしました。待機にタイムアウトはないので、エラーも出ずに止まり続けます。

`make practice-4car` では、4 台の Autoware を先に起動し、全員の orchestrator が待機状態になってから AWSIM を起動する順番にしました。これで 4 台とも制御要求を出し、完走しました。同じことが SIM 決勝で起動の遅れた車にも起こりうるので、運営に issue で報告しています（#348）。

## 罠 3: Mac で作った tar.gz が弾かれる

提出物の形式チェック（すべてのエントリが `aichallenge_submit/` の下にあるか）で、1 つの tar.gz が弾かれました。中身を見ると `._aichallenge_submit` というエントリがあります。macOS の `tar` が付ける AppleDouble（拡張属性の入れ物）です。

面白いのは、**Mac の `tar -t` ではこのエントリが表示されない**ことです。bsdtar は `._` を拡張属性として扱い、一覧から隠します。Linux の GNU tar では見えます。評価イメージでは害のない余分なファイルとして展開されるだけなので、形式チェックでは `._` のエントリを無視するようにしました（`..` を含むパスのチェックは全エントリに対して行います）。このテストは Mac では再現しないので、Linux で「修正前は失敗、修正後は成功」を確認しています。

## 罠 4: スタート指令が黙って失敗する

スタート指令は `make awsim-request-start` で送っていましたが、出力を捨てていました。実際には `env: 'ros2': No such file or directory` で失敗していました。`autoware-command` コンテナは起動時に `/aichallenge/workspace/install/setup.bash` だけを source するので、練習専用のチェックアウトのように自分の workspace を一度もビルドしていないと、`ros2` 自体が見つかりません。ROS を明示的に source して送り、失敗したときはログに出すようにしました。

## 1 台の PC での注意

- SIM 決勝では各車に専用の PC（i7-8700 / 16 GB）がありますが、ここでは 1 台の PC で AWSIM と 4 台を動かします。タイミングに敏感なコードは挙動が変わることがあります。`PIN=1` を使い、単独の `make eval` とも比べてください
- 順位の入れ替わりは周回ラインでしか数えていません。同じ周の中で抜いて抜き返された場合は数えません

## まとめ

ユニットテストが通っていても、本物の AWSIM で 4 台を走らせると 4 つの罠が出てきました。どれも「エラーが出ずに止まる」タイプで、ログの時刻を並べるまで原因が分かりませんでした。SIM 決勝前に 4 台で練習したいチームの役に立てば嬉しいです。

### AI の利用について

この機能の実装と調査には AI コーディングエージェントを使いました。数値はすべて、実際に 4 台レースを走らせたログ（`result-summary.json`、`autoware.log`、`awsim.log`）と、Linux 上のテスト（19 件）で確認しています。

---

## English

The SIM final of the Autonomous Driving AI Challenge 2026 is a 4-car race, but the starter kit has no way to race four *different* submissions locally under the final's settings. We built `make practice-4car`, which races up to four submission tarballs on one PC, and sent it as a PR.

- PR: https://github.com/AutomotiveAIChallenge/aichallenge-racingkart/pull/345
- Related issue: https://github.com/AutomotiveAIChallenge/aichallenge-racingkart/issues/348

A real 4-car race with four different builds completed 6 laps each (table and lap-time figure above). Getting there exposed four traps that unit tests could not find:

1. **CycloneDDS under WSL2.** `vehicle/cyclonedds.xml` pins CycloneDDS to `lo`; under WSL2, `lo` also carries `10.255.255.254/32`, CycloneDDS cannot create participants and says nothing, and AWSIM (which has its own DDS) and Autoware never find each other: AWSIM stays at `WaitStart` and every car at `wait until clock received`. On a simulation PC, use `autodetermine="true"` (not on the real kart).
2. **A late stack misses the "ready" state.** The autostart orchestrator waits for `/awsim/state` to be `Grounded`, `Ready` or `Start`. AWSIM sends that state once per car, and the orchestrator subscribes with volatile QoS, so a subscription that connects after the message is sent never sees it, and the wait has no timeout. With AWSIM started first, the last car subscribed while Grounded was being sent and waited forever (figure above). `make practice-4car` now starts the Autoware stacks first and AWSIM last. Because the same can happen to a slow car at the SIM final, we reported it (#348).
3. **Tarballs made on a Mac.** macOS `tar` adds an AppleDouble `._aichallenge_submit` entry, which the layout check rejected. `tar -t` on the Mac hides it (bsdtar treats `._` files as extended attributes); GNU tar on Linux shows it. The eval image extracts it harmlessly, so the check now ignores `._` entries (the `..` check still covers every entry). The regression test cannot fail on a Mac, so the before/after was verified on Linux.
4. **A silent start failure.** The start request went through `make awsim-request-start` with its output discarded, and it was failing with `env: 'ros2': No such file or directory`: the `autoware-command` container only sources `/aichallenge/workspace/install/setup.bash`, which a practice-only checkout has never built. ROS is now sourced explicitly and a failure is logged.

Single-PC caveat: in the SIM final each car has its own PC (i7-8700, 16 GB); here AWSIM and four stacks share one machine, so timing-sensitive code can behave differently. Use `PIN=1` and compare with a solo `make eval`.

How this was made: we used AI coding agents for the implementation and the investigation. Every number comes from the logs of the real 4-car race (`result-summary.json`, `autoware.log`, `awsim.log`) and from 19 tests run on Linux.

Team Hayes (Ajayaditya Lokchandra, Nithisha Venkatesh)
