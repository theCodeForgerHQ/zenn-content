---
title: "make eval が止まる・提出物が入れ替わる: 自動運転AIチャレンジのスターターキットで踏んだ罠と直した PR"
emoji: "🧰"
type: "tech"
topics: ["autoware", "ros2", "docker", "bash", "aichallenge"]
published: true
---

Team Hayes（Ajayaditya Lokchandra, Nithisha Venkatesh）です。自動運転AIチャレンジ2026 のスターターキット（[aichallenge-racingkart](https://github.com/AutomotiveAIChallenge/aichallenge-racingkart)）を使う中で、「エラーが出ないのに結果がおかしい」タイプの不具合に何度か当たりました。見つけたものは再現テストを付けて PR にしています。同じところで時間を失う人が減るように、症状・原因・直し方を 1 つずつまとめます。

英語版は後半にあります。 / English version below.

どれも共通しているのは、**失敗しているのに成功と表示される**ことです。終了コードを捨てている、待ち時間に上限がない、古いファイルを黙って使う、の 3 パターンでした。

![3 つのパターンと PR: 終了コードを捨てる、上限のない待機、古いファイルを黙って使う / Three patterns and their PRs](https://raw.githubusercontent.com/theCodeForgerHQ/zenn-content/main/images/m3/m3-patterns.png)

| # | 症状 | PR | 状態 |
|---|---|---|---|
| 1 | `doctor` が存在しないイメージを「ある」と言う | [#276](https://github.com/AutomotiveAIChallenge/aichallenge-racingkart/pull/276) | マージ済み |
| 2 | `make eval` が永遠に待つ | [#317](https://github.com/AutomotiveAIChallenge/aichallenge-racingkart/pull/317) | クローズ（1 コマンドには手厚すぎるという運営の判断） |
| 3 | topic_check が FAIL なのに PASS と集計する | [#319](https://github.com/AutomotiveAIChallenge/aichallenge-racingkart/pull/319) | レビュー待ち |
| 4 | `make eval` が古い提出物を黙って評価する | [#320](https://github.com/AutomotiveAIChallenge/aichallenge-racingkart/pull/320) | レビュー待ち |
| 5 | `make download` が前回の残りの tar を展開する | [#316](https://github.com/AutomotiveAIChallenge/aichallenge-racingkart/pull/316) | レビュー待ち |
| 6 | `setup.bash bootstrap` が途中で止まって見える | [#318](https://github.com/AutomotiveAIChallenge/aichallenge-racingkart/pull/318) | レビュー待ち |
| 7 | `path_constraints_provider` が起動直後に必ず落ちる | [#315](https://github.com/AutomotiveAIChallenge/aichallenge-racingkart/pull/315) | レビュー待ち |

![修正前後: make eval の待機、make download の展開、topic_check の終了コード / Before and after for #317, #316 and #319](https://raw.githubusercontent.com/theCodeForgerHQ/zenn-content/main/images/m3/fig2-before-after.png)

## 1. `doctor` が存在しないイメージを「ある」と言う（#276, マージ済み）

`setup.bash` の docker ヘルパー 4 つが、docker を実行したあと無条件に `return 0` していました（8 箇所）。

```bash
docker_run_no_prompt() {
    if docker_as_user_ok; then
        docker "$@"
        return 0        # docker の終了コードを捨てている
    fi
```

その結果、イメージが 1 つもない環境でも `doctor` は次のように表示していました。

```
✅ image exists: aichallenge-2025-dev
✅ base image exists: ghcr.io/automotiveaichallenge/autoware-universe:humble-latest
```

新しい参加者は、いちばん時間のかかる `./setup.bash pull image` と `./docker_build.sh dev` を飛ばしてよいと判断し、ずっと後で分かりにくい失敗に出会います。イメージ取得のリトライも、失敗を成功と受け取るので一度も再試行しません。docker の本当の終了コードを返すように直し、同日にマージされました。

## 2. `make eval` が永遠に待つ（#317, クローズ）

評価コンテナが起動直後に落ちると（`install/` が空、AWSIM が未配置、launch の失敗など）、`make eval` は `awsim-request-start` の中で次の行を出したまま止まります。

```
Waiting for at least 1 matching subscription(s)...
```

ROS 2 Humble の `ros2 topic pub -1` は「購読者が 1 つ現れるまで、上限なしで待つ」という意味だからです。購読するはずのコンテナはもう存在しないので、終わりません。

PR では待ち時間に上限（`AWSIM_START_TIMEOUT`, 既定 300 秒, 0 で従来どおり）を付け、失敗したら評価コンテナの状態とログの最後の 30 行を表示して終了します。起動直後に exit 127 する評価イメージで試すと、

- 修正前: `Waiting...` を 71 行出し、75 秒でこちらが強制終了するまで待ち続けた
- 修正後: 22 秒で `run_evaluation.bash: No such file or directory` という本当の原因を表示して終了

## 3. topic_check が FAIL なのに PASS と集計する（#319）

`aichallenge/utils/topic_check.sh` の 2 つのチェックが、次の形で終了コードを取っていました。

```bash
result=$(check) || true
rc=$?          # ここの $? は常に true の 0
```

`|| true` の後の `$?` は必ず 0 です。そのため詳細行には FAIL と出るのに、最後の SUMMARY は PASS になり、スクリプトも 0 で終了します。`rc=0; result=$(check) || rc=$?` に直しました。ブレーキ中で AWSIM のトピックが残っている状態を偽の `ros2` で作って試すと、

- 修正前: exit=0, SUMMARY は PASS / PASS
- 修正後: exit=5, SUMMARY は FAIL / FAIL（スクリプトが本来意図していた終了コード）

## 4. `make eval` が古い提出物を黙って評価する（#320）

`./docker_build.sh eval` は `submit/aichallenge_submit.tar.gz` をそのまま評価イメージに入れます。最後に `./create_submit_file.bash` を実行した後でコードを変えていると、`make eval` は古いコードを採点し、何も言いません。「直したはずなのに結果が変わらない」の典型的な原因です。

PR では、提出ディレクトリの中に tar より新しいファイルがあれば 3 行の警告を出します（ビルドは続けます）。`AIC_STRICT_SUBMIT=1` を付けるとエラーにできます。tar の作成後に `config.yaml` を編集して試すと、修正前は警告なし、修正後は警告が出て、tar を作り直すと警告は消えました。

## 5. `make download` が前回の残りの tar を展開する（#316）

`vehicle/download_submission.sh` は展開する tar を `find vehicle/download -name "*.tar.gz" | head -1` で選んでいました。前回の実行が展開前に止まっていると、その tar が残っていて、ディレクトリ内で先に見つかった方が展開されます。それが**別のチームの提出物**であっても、スクリプトは `Extraction completed successfully!` と表示します。

残りの tar 1 つと新しい tar 1 つを、ファイル名の並び順 3 通りで試しました。

- 修正前: 3 回中 2 回、残りの方を展開
- 修正後: 3 回中 3 回、要求した提出物を展開

ダウンロード前に古い tar を消すだけの 6 行の変更です。

## 6. `setup.bash bootstrap` が途中で止まって見える（#318）

docker グループの設定に `newgrp docker` が追加されていました。`newgrp` は新しい対話シェルを起動するだけで、実行中のスクリプトのグループは変えません。そのため端末で `./setup.bash bootstrap` を実行すると、グループ設定の直後にプロンプトが出て止まったように見えます。実際の関数を `ubuntu:22.04` で pty 付きで実行すると、修正前は入れ子のシェルで止まって timeout（exit 124）、修正後は再ログインの案内を出して exit 0 で終わりました。

## 7. `path_constraints_provider` が起動直後に必ず落ちる（#315）

仕様書の「さらに高度な回避」で案内されているノードです。`MPC.__init__` に引数が 2 つ追加されたとき、呼び出し側が位置引数 `True, True` のまま残っていました。

```
TypeError: MPC.__init__() missing 2 required positional arguments: 'use_obstacle_avoidance' and 'use_path_constraints_topic'
```

キーワード引数で渡すように直し、同梱の設定と地図で実際にノードを組み立てるテストを付けました。

## まとめ

- シェルスクリプトでは、`|| true` と `return 0` が終了コードを消していないかを最初に見る
- 「待つ」処理には上限と、失敗したときに原因を表示する経路を付ける
- 生成物（tar, イメージ）を使う処理は、それが最新かどうかを確かめる

## 作り方について

調査とパッチ作成には AI コーディングエージェントを使いました。数値はすべて、修正前に失敗し修正後に通る再現テストで確認しています。各 PR に手順と出力を載せています。

---

## English

Team Hayes (Ajayaditya Lokchandra, Nithisha Venkatesh). While using the JSAE AI Challenge 2026 starter kit ([aichallenge-racingkart](https://github.com/AutomotiveAIChallenge/aichallenge-racingkart)) we kept hitting bugs of one kind: **it fails, but reports success**. Each one below has a PR with a reproduction test. The table above lists them; #276 is merged, #317 was closed by the maintainer (judged too much care for a single command), and the rest are awaiting review. The causes fall into three patterns: an exit code thrown away, a wait with no upper bound, and a stale file used without a word.

**1. `doctor` says images exist when they don't (#276, merged).** The four docker helpers in `setup.bash` ran docker and then `return 0` unconditionally (8 sites). On a host with no images, `doctor` still printed `✅ image exists`, so new participants skipped the two slowest setup steps, and the image-pull retry loop never retried. The helpers now return docker's real exit code.

**2. `make eval` waits forever (#317).** If the evaluation container dies at start-up, `ros2 topic pub -1` in `awsim-request-start` waits with no bound for a subscriber that no longer exists, printing `Waiting for at least 1 matching subscription(s)...`. The PR adds `AWSIM_START_TIMEOUT` (default 300 s; 0 keeps the old behaviour) and prints the container status and its last 30 log lines on failure. With an eval image that exits 127 at start-up: before, 71 "Waiting" lines until we killed it at 75 s; after, it stops in 22 s and shows the real cause, `run_evaluation.bash: No such file or directory`.

**3. topic_check reports PASS on failure (#319).** Two checks read `rc=$?` right after `result=$(check) || true`, so `$?` is always the 0 of `true`: the detail line says FAIL, the SUMMARY says PASS, and the script exits 0. Fixed with `rc=0; result=$(check) || rc=$?`. With a fake `ros2` (kart braking, an AWSIM topic present): before exit 0 and PASS/PASS; after exit 5 and FAIL/FAIL.

**4. `make eval` silently scores old code (#320).** `./docker_build.sh eval` bakes `submit/aichallenge_submit.tar.gz` in as is. Edit the code after the last `./create_submit_file.bash` and you are scoring the old version, with no message. The PR prints a three-line warning when the submit tree is newer than the tarball; `AIC_STRICT_SUBMIT=1` makes it an error.

**5. `make download` can unpack a leftover tarball (#316).** `download_submission.sh` picked the tarball with `find ... | head -1`. If an earlier run stopped before extracting, its tarball is still there, and whichever comes first in directory order is extracted, possibly another team's submission, while the script prints `Extraction completed successfully!`. With one leftover and one new tarball in three name orders: before, the leftover was extracted 2 times out of 3; after, the requested one 3 out of 3.

**6. `setup.bash bootstrap` appears to hang (#318).** `newgrp docker` starts a new interactive shell and never changes the running script's group, so bootstrap seems to stop at a prompt. Running the real function in `ubuntu:22.04` under a pty: before, stuck in the nested shell (timeout, exit 124); after, it prints the re-login hint and exits 0.

**7. `path_constraints_provider` always crashes at start-up (#315).** Two parameters were added to `MPC.__init__` while the caller kept its positional `True, True`, giving `TypeError: ... missing 2 required positional arguments`. The flags are now passed by keyword, with a test that builds the real node from the shipped config and map.

**Takeaways.** In shell scripts, look first for `|| true` and `return 0` swallowing exit codes. Give every wait an upper bound and a path that prints the cause. Before using a generated artefact (a tarball, an image), check that it is current.

**How this was made.** We used AI coding agents for the investigation and the patches. Every number here comes from a reproduction that fails before the fix and passes after it; each PR lists the steps and output.

---

## Team Hayes の自動運転AIチャレンジ2026 シリーズ / series

- [最近傍 waypoint の argmin が 1.5 m で逆向きの区間に飛ぶ / argmin snaps to the opposite leg](https://qiita.com/TeamHayes/items/2bd248ad00a8011406f4)
- [v_max を下げたら速くなった / Lowering v_max made it faster](https://qiita.com/TeamHayes/items/934870c19786e3e6b3fe)
- [quaternion の z を yaw だと思っていませんか / quaternion z is not yaw](https://qiita.com/TeamHayes/items/f004f15b7c0fe1f4b512)
- [make eval が止まる・提出物が入れ替わる: スターターキットの罠と PR / starter-kit traps and fixes](https://zenn.dev/ajayaditya/articles/aichallenge-starter-kit-traps-2026)
- [AI エージェント 100 体で学んだこと / Lessons from 100 AI agents](https://zenn.dev/ajayaditya/articles/100-ai-agents-racing-lessons)
- [日本語が読めなくても参加できるように: ドキュメントとツールを英語対応 / Making the challenge readable without Japanese](https://qiita.com/TeamHayes/items/7aaf06afe8f39d8f6dfd)
- ツール集 / toolkit: [aichallenge-toolkit](https://github.com/theCodeForgerHQ/aichallenge-toolkit)
