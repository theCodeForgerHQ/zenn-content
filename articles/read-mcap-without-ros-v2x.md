---
title: "ROS なしで rosbag（mcap）を読む: V2X の位置配列を CDR から手でデコードする"
emoji: "📦"
type: "tech"
topics: ["ros2", "python", "rosbag", "mcap", "aichallenge"]
published: true
---

Team Hayes（Ajayaditya Lokchandra, Nithisha Venkatesh）です。自動運転AIチャレンジ2026 のレースのログ（rosbag, mcap 形式）から、他のカートの位置（V2X）を ROS なしで取り出す方法をまとめます。24 MB のログが 0.55 秒で CSV になります。

英語版は後半にあります。 / English version below.

## なぜ ROS なしで読みたいのか

レースのログからトピックを 1 つ見たいだけでも、普通は ROS 2 と、そのメッセージ型のパッケージが必要です。V2X のメッセージ型（`V2XVehiclePositionArray`）はスターターキット独自のもので、pip では入りません。手元のノート PC で、Docker も ROS も起動せずに見たい、というのが出発点でした。

実際には、mcap ファイルは「メッセージのバイト列を並べた入れ物」で、中身のバイト列は CDR という単純な形式です。形式さえ分かれば、Python の標準ライブラリ（`struct`）で読めます。

## メッセージの形

`/v2x/vehicle_positions` のメッセージは次の構造です。

```
V2XVehiclePositionArray
  std_msgs/Header       header      (int32 sec, uint32 nanosec, string frame_id)
  V2XVehiclePosition[]  vehicles    (uint32 の個数、続いてその数だけ要素)
    std_msgs/Header       header
    string                vehicle_id
    geometry_msgs/Point   position    (float64 x, y, z)
    geometry_msgs/Vector3 covariance  (float64 x, y, z)
```

入っているのは**他のカートの位置だけ**で、向きや速度はありません。速度が必要なら、位置の差分から自分で求めます。

## CDR の 2 つの落とし穴

1. **アラインメント**: 各値は、本体の先頭（最初の 4 バイトのヘッダの後）から数えて、自分のサイズの倍数の位置に置かれます。`float64` なら 8 の倍数です。直前が文字列だと、間に 0〜7 バイトの詰め物が入ります。
2. **文字列**: 先頭の `uint32` は長さですが、**末尾の NUL を含んだ長さ**です。

どちらも、少し間違えると「数字は出るが全部おかしい」状態になります。

## デコーダ（約 30 行）

```python
import struct

class CDR:
    """最小限のリトルエンディアン CDR リーダー（アラインメント付き）"""
    def __init__(self, buf):
        self.b, self.o, self.base = buf, 4, 4   # 最初の 4 バイトはエンコーディングのヘッダ
    def _align(self, n):
        self.o += (-(self.o - self.base)) % n   # 本体の先頭から n の倍数へ
    def u32(self):
        self._align(4); v = struct.unpack_from("<I", self.b, self.o)[0]; self.o += 4; return v
    def i32(self):
        self._align(4); v = struct.unpack_from("<i", self.b, self.o)[0]; self.o += 4; return v
    def f64(self):
        self._align(8); v = struct.unpack_from("<d", self.b, self.o)[0]; self.o += 8; return v
    def string(self):
        n = self.u32()                           # NUL を含む長さ
        s = self.b[self.o:self.o + n - 1].decode(); self.o += n; return s
    def header(self):
        sec, nsec = self.i32(), self.u32(); frame = self.string()
        return sec + nsec * 1e-9, frame

def decode_array(buf):
    c = CDR(buf)
    t_msg, _ = c.header()
    out = []
    for _ in range(c.u32()):
        t, _ = c.header()
        vid = c.string()
        x, y, z = c.f64(), c.f64(), c.f64()
        sx, sy, sz = c.f64(), c.f64(), c.f64()
        out.append((vid, t, x, y, z, sx, sy, sz))
    return t_msg, out
```

mcap の読み出しには `mcap` パッケージ（ROS 不要）を使います。

```python
from mcap.reader import make_reader
with open("race.mcap", "rb") as f:
    for schema, channel, msg in make_reader(f).iter_messages(topics=["/v2x/vehicle_positions"]):
        t_msg, vehicles = decode_array(msg.data)
```

アラインメントの規則を固定するために、自分でメッセージを 1 つ組み立てて（エンコードして）、デコード結果と比べるセルフテストも入れています。

## 実際のレースで試す

自分たちのローカル環境で走らせた 3 台レース（3 台とも自チームのビルド、各 6 周、ペナルティ 0）のログで試しました。

- ログ 24 MB を 0.55 秒でデコード
- 他の 2 台それぞれ 5,751 サンプル、328.9 秒分
- サンプル間隔の中央値 50 ms（20 Hz）。欠けも含めた平均は 17.5 Hz
- 位置の分散（covariance）は 0.005 で一定

![1 レース分の V2X 位置を ROS なしでデコードしたもの](https://raw.githubusercontent.com/theCodeForgerHQ/zenn-content/main/images/m6/fig1-paths.png)

位置が 20 Hz で取れれば、相手との距離の推移、どこで詰まったか、ペナルティ（速度が 5 km/h に固定される区間）の検出などが、ROS なしでできます。

## ツールとして公開しています

このデコーダは、Team Hayes のツール集 aichallenge-toolkit の `bagkit.v2x` として公開しています（Apache-2.0）。

```bash
git clone https://github.com/theCodeForgerHQ/aichallenge-toolkit && cd aichallenge-toolkit
pip install -e ".[bag]"
python -m bagkit.v2x race.mcap --out-dir out/     # 車ごとの CSV と統計の JSON
python -m bagkit.v2x --selftest                    # アラインメント規則の確認
```

https://github.com/theCodeForgerHQ/aichallenge-toolkit

## 作り方について

デコーダとこの記事の作成には AI コーディングエージェントを使いました。数値は、上のログに対して実際にツールを実行した結果です。セルフテストでエンコードとデコードの往復を確認しています。

---

## English

Team Hayes (Ajayaditya Lokchandra, Nithisha Venkatesh). How to pull the other karts' positions (V2X) out of a JSAE AI Challenge 2026 race log (rosbag, mcap format) without ROS. A 24 MB log becomes CSV in 0.55 seconds.

### Why read it without ROS

Looking at even one topic in a race log normally needs ROS 2 plus the message-type packages. The V2X type (`V2XVehiclePositionArray`) is specific to the starter kit and is not on pip. We wanted to look at it on a laptop without starting Docker or ROS. An mcap file is a container of message byte strings, and those bytes are plain CDR. Once you know the layout, Python's standard `struct` module can read them.

### The message layout

See the structure above: a header, then a `uint32` count and that many vehicles, each with its own header, a `vehicle_id` string, a position (three `float64`) and a covariance (three `float64`). It carries **only the other karts' positions**: no heading and no speed. If you need speed, difference the positions yourself.

### Two CDR traps

1. **Alignment**: each value sits at a multiple of its own size, counted from the start of the body (after the first 4-byte header). A `float64` goes on a multiple of 8, so after a string there can be 0 to 7 bytes of padding.
2. **Strings**: the leading `uint32` is the length **including the trailing NUL**.

Get either slightly wrong and you get numbers that look plausible and are all wrong.

### The decoder (about 30 lines)

The code above is the whole decoder: a small CDR reader that aligns each read relative to the body start, and `decode_array`, which walks the message. The `mcap` package (no ROS needed) iterates the messages. A self-test builds one message by hand, decodes it and compares, so the alignment rules stay pinned.

### On a real race

We ran it on a 3-car race from our own local setup (all three karts are our own builds; 6 laps each, no penalties): the 24 MB log decodes in 0.55 s; each of the other two karts has 5,751 samples over 328.9 s; the median sample interval is 50 ms (20 Hz), and 17.5 Hz on average including gaps; the position covariance is a constant 0.005 (figure above). With positions at 20 Hz you can follow the gap to a rival, see where a kart got stuck, or detect penalties (stretches where speed is capped at 5 km/h), all without ROS.

### Released as a tool

The decoder is `bagkit.v2x` in Team Hayes' aichallenge-toolkit (Apache-2.0): `pip install -e ".[bag]"`, then `python -m bagkit.v2x race.mcap --out-dir out/` for a per-car CSV and a JSON of statistics, and `python -m bagkit.v2x --selftest` to check the alignment rules. https://github.com/theCodeForgerHQ/aichallenge-toolkit

### How this was made

We used AI coding agents to build the decoder and draft this article. The numbers are from actually running the tool on the log above, and the self-test checks the encode/decode round trip.
<!-- deploy attempt 2 -->
