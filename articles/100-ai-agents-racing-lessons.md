---
title: "AI エージェント 100 体で自動運転レースを開発して学んだこと: 並列化は網羅の道具で、バグ探しは深さの問題"
emoji: "🤖"
type: "idea"
topics: ["ai", "claudecode", "autoware", "ros2", "aichallenge"]
published: true
---

Team Hayes（Ajayaditya Lokchandra, Nithisha Venkatesh）です。自動運転AIチャレンジ2026 では、開発のかなりの部分を AI コーディングエージェント（Claude Code）に任せました。一度に 100 体を超えるエージェントを並列で動かしたこともあります。その結果、うまくいったことと、高くついた失敗がはっきり分かれたので、数字つきで残しておきます。

英語版は後半にあります。 / English version below.

レースの戦略（どこでどう速く走るか）はこの記事には書きません。書くのは「エージェントをどう使うと成果が出て、どう使うと出ないか」です。

## 102 体のうち、成果を出したのは 12 体

8 月 21 日、信頼性の問題（衝突、停止、スタート、MPC が解けなくなる問題）をまとめて洗い出すために、102 体のエージェントを動かすワークフローを組みました。調べる役、案を出す役、そして出てきた案を検証する役です。

終わってから数えると、次のようになっていました。

- 成果の約 9 割は、約 12 体から出ていた
- 確認できた不具合 8 件は、**すべて 3 体から**見つかっていた。1 体は QP（最適化）の実装を 1 行ずつ読んだ。1 体は自分たちのスタート時のログを読んだ。1 体は他チームの公開走行ログを読んだ
- 検証役のエージェント 90 体は、実行量の 88% を使い、成果は約 1 割だった

![成果の 9 割は約 12 体から、検証エージェントは実行量の 88% / Value came from a few agents; verify agents used most of the run](https://raw.githubusercontent.com/theCodeForgerHQ/zenn-content/main/images/m10/fig1-value.png)

## 検証エージェントは不具合を見つけられない

検証役は「前の段階で誰かが出した候補」しか見ません。つまり**ふるい**であって、**探索**ではありません。候補に入っていない不具合を、検証役が見つけることは構造的にありません。

検証を増やせば見落としが減る、という直感は間違いでした。見落としは候補を出す段階で起きていて、検証を何倍にしても戻ってきません。

![候補に入らなかった不具合は、検証では戻らない / Defects never proposed are never recovered by verification](https://raw.githubusercontent.com/theCodeForgerHQ/zenn-content/main/images/m10/m10-filter.png)

## 並列化は「網羅」の道具、バグ探しは「深さ」の問題

うまくいった並列化は、どれも**広くて浅い**仕事でした。

- 数百のリポジトリから特定のパターンを探す
- 96 本のログをパースする
- 50 か所の呼び出し元を確認する

うまくいかなかったのは、**狭くて深い**仕事を並列にしたときです。「なぜこの関数で車が止まるのか」は、失敗のしかたを頭に置いたまま、その 1 本のコードパスを読む 1 人の読み手の仕事です。100 人に分けても速くなりません。

目に見える規模（エージェントの数）が努力の証拠のように感じられて、つい増やしたくなります。これは自分たちで気をつけるべき失敗のパターンでした。

## 高くついた 4 つの失敗

**1. 失敗しているコードを読む前に、パラメータを回した。** ある距離の計算で、単位のラベルが間違っていました（中身は約 18 m なのに「0.5 秒」として扱われていた）。この症状のまわりで 4 日間パラメータ調整を続けましたが、原因の関数は 1 回も通して読まれていませんでした。読めば数分で分かる種類の不具合でした。

**2. 計測器を疑わなかった。** 3 台レースのログに `d3 laps: []`（3 台目の周回記録が空）と何日も出ていました。これは、3 台での計測がすべて「止まった車」を相手にしていたという意味でした。比較の基準が変なときは、それ自体が一番大事な情報です。

**3. 試行回数を確かめなかった。** 同じビルドを同じスタート位置で走らせた 2 回で、ペナルティが 0 回と 5 回でした。1 回ずつの比較では設定の良し悪しを区別できません。比べる前に、同じ条件でのばらつきを測る必要があります。

**4. 「測りやすいもの」を最適化した。** 1 台だけの走行は速くて数字がきれいですが、衝突は起きません。聞かれていたのは「レースでの信頼性」なのに、きれいな数字の出る 1 台走行に時間を使いがちでした。

## 今のやり方

- 並列化する前に、失敗しているコードパスを 1 行ずつ読む
- エージェントの数は仕事の形で決める: 大量の資料を探すなら探し方ごとに 2 から 5 体、自分たちのコードの不具合を探すなら 1 から 3 体に「経路全体を読め」と指示する、検証は間違えると高くつく候補だけに最大 3 体
- 15 体を超えそうなら、1 体増やすごとに「前の 1 体が見つけられないものを何を見つけるのか」を書く。書けなければ増やさない
- 「試して駄目だったこと」の一覧を残す。これがないと、毎回同じ案を試して回ることになる

![仕事の形でエージェントの数を決める / Size the agents by the shape of the work](https://raw.githubusercontent.com/theCodeForgerHQ/zenn-content/main/images/m10/m10-sizing.png)

## OSS 貢献でも同じだった

大会のリポジトリへの貢献（修正 PR、翻訳、Issue）にもエージェントを使いました。ここでも同じことが起きました。

- 修正の候補は、1 つずつコードを読んで「修正前に失敗し、修正後に通るテスト」を作れたものだけを出した。数えると、最初の候補のうち 4 件はすでに上流で直っていて、読み直して取り下げた
- PR を出した後、自動レビュー（Copilot と Codex）が本当の問題を 11 件指摘した。人の目もエージェントの目も、1 回では足りない
- エージェントの「確認しました」をそのまま信じない。「全角ダッシュは入っていません」と報告したファイルに入っていたことがあり、最後は自分たちで grep した
- 共有の作業フォルダを、あるエージェントが丸ごと消した。以後、エージェントごとに別のフォルダを使っている

## まとめ

エージェントを増やすのは、**調べる範囲を広げたいとき**です。**不具合を見つけたいとき**は、1 本のコードパスを最後まで読む少数の読み手を置き、計測器とばらつきを先に確かめます。

## 作り方について

この記事で使った数字は、ワークフローの実行記録と、そのとき書いた振り返りのメモから取りました。文章の下書きにも AI を使い、数字は記録と照合しています。

---

## English

Team Hayes (Ajayaditya Lokchandra, Nithisha Venkatesh). In the JSAE AI Challenge 2026 we handed a large share of development to AI coding agents (Claude Code), sometimes running more than 100 in parallel. What worked and what cost us were clearly different, so here they are with numbers. This article is about how to use agents, not about racing strategy.

### 12 of 102 agents did the work

On 21 August we ran a 102-agent workflow to find reliability problems (collisions, stalls, the start, the MPC failing to solve): agents that investigated, agents that proposed changes, and agents that verified the proposals. Counting afterwards: about 12 agents produced about 90% of the value. All 8 confirmed defects came from **3 agents**: one read the QP (optimisation) code line by line, one read our own start logs, and one read the field's public race logs. The 90 verify agents used 88% of the run and produced about 10% of the value.

### Verify agents cannot find defects

A verify agent only sees candidates an earlier stage proposed. It is a **filter, not a search**, so by construction it never finds a defect nobody proposed. The intuition that more verification means fewer misses was wrong: the misses happen at the proposal stage, and multiplying verifiers does not bring them back.

### Fan-out is for coverage; bug-finding is depth

The parallelism that worked was always **wide and shallow**: search hundreds of repositories for a pattern, parse 96 logs, check 50 call sites. What failed was parallelising **narrow and deep** work. "Why does this function stop the car?" is a job for one reader who reads that one code path while holding the failure in mind; splitting it across 100 agents does not make it faster. Visible scale feels like effort, and reaching for it is a failure mode to watch for.

### Four expensive mistakes

1. **Tuning before reading.** A distance calculation had a mislabelled unit (about 18 m of distance treated as "0.5 seconds"). We spent four days tuning parameters around its symptom; the function was never read end to end, and reading it would have taken minutes.
2. **Not checking the instrument.** Our 3-car logs said `d3 laps: []` (no laps for the third car) for days. It meant every 3-car measurement was against a parked car. When the baseline behaves oddly, that is the most important signal.
3. **Not checking n.** The same build in the same grid slot got 0 penalties in one run and 5 in another. Single runs cannot separate configurations; measure run-to-run spread first.
4. **Optimising the measurable thing.** Solo runs are fast and give clean numbers, but a solo run cannot contain a collision. The question was race reliability, yet the clean numbers kept pulling our time towards solo runs.

### How we work now

- Read the failing code path line by line before any fan-out.
- Size by the shape of the work: 2-5 agents for sweeping a large corpus (one per search method), 1-3 agents told to read the whole path for defects in our own code, and at most 3 verifiers, only on candidates that are expensive to get wrong.
- Above about 15 agents, write down what each extra agent will find that the previous one will not. If there is no answer, do not add it.
- Keep a list of refuted ideas, or every session re-tries the same things.

### The same in open-source work

We also used agents for our contributions to the challenge repositories (fix PRs, translations, issues). Each fix was only sent if someone read the code and wrote a test that fails before the fix and passes after; four early candidates turned out to be fixed upstream already and were dropped. After the PRs went up, automated reviewers (Copilot and Codex) still found 11 real problems, so one pass, human or agent, is not enough. Do not trust an agent's "checked": one reported a file free of em-dashes that contained them, so we grep ourselves at the end. One agent deleted a whole shared working folder; each agent now gets its own.

### Summary

Add agents when you need to **cover more ground**. When you need to **find a defect**, put a few readers on one code path end to end, and check the instrument and the run-to-run spread first.

### How this was made

The numbers come from the workflow's run records and the notes we wrote at the time. The draft was written with AI help, and the numbers were checked against those records.

---

## Team Hayes の自動運転AIチャレンジ2026 シリーズ / series

- [最近傍 waypoint の argmin が 1.5 m で逆向きの区間に飛ぶ / argmin snaps to the opposite leg](https://qiita.com/TeamHayes/items/2bd248ad00a8011406f4)
- [v_max を下げたら速くなった / Lowering v_max made it faster](https://qiita.com/TeamHayes/items/934870c19786e3e6b3fe)
- [quaternion の z を yaw だと思っていませんか / quaternion z is not yaw](https://qiita.com/TeamHayes/items/f004f15b7c0fe1f4b512)
- [make eval が止まる・提出物が入れ替わる: スターターキットの罠と PR / starter-kit traps and fixes](https://zenn.dev/ajayaditya/articles/aichallenge-starter-kit-traps-2026)
- [AI エージェント 100 体で学んだこと / Lessons from 100 AI agents](https://zenn.dev/ajayaditya/articles/100-ai-agents-racing-lessons)
- [日本語が読めなくても参加できるように: ドキュメントとツールを英語対応 / Making the challenge readable without Japanese](https://qiita.com/TeamHayes/items/7aaf06afe8f39d8f6dfd)
- ツール集 / toolkit: [aichallenge-toolkit](https://github.com/theCodeForgerHQ/aichallenge-toolkit)
