---
title: アロケータの基礎
aliases: [アロケータの基礎, アロケータ, allocator, メモリアロケータ, malloc, スタックとヒープ, ヒープアロケータ, メモリアロケーション]
domain: programming
tags: [memory, allocator, malloc, heap, stack, arena, fragmentation, performance, rust, realtime]
created: 2026-11-13
updated: 2026-11-13
status: growing
prerequisites: [仮想メモリとページ, ポインタとアドレス, アラインメント, キャッシュ階層, システムコール (brk/mmap), C の malloc/free, Rust の所有権と Drop]
related: ["[[Rust のメモリ管理]]"]
sources:
  - man 3 malloc / man 2 brk / man 2 mmap（Linux man-pages）
  - Rust std::alloc（https://doc.rust-lang.org/std/alloc/）と GlobalAlloc
  - glibc MallocInternals（https://sourceware.org/glibc/wiki/MallocInternals）
  - jemalloc（https://jemalloc.net/）、mimalloc（https://github.com/microsoft/mimalloc）
  - Ulrich Drepper, "What Every Programmer Should Know About Memory"
  - Linux kernel docs, admin-guide/mm/concepts（https://www.kernel.org/doc/html/latest/admin-guide/mm/concepts.html）
---

# アロケータの基礎

## ひとことで

アロケータは **「このサイズのメモリをくれ／返す」という1バイト単位の要求を、OS の「ページ単位（ふつう4KiB）」の仕組みに変換する層**。OS からページをまとめて借り、それを切り分けて返し、返ってきた領域を再利用する。速さの正体は、①システムコールを減らす ②空きを捨てずに使い回す ③サイズ別に探す（O(1) に近づける）の3つ。

そして、**メモリの確保は「スタック（アロケータ不要）」と「ヒープ（アロケータ経由）」の2種類しかない**。ここを分けて理解するのが出発点。

## なぜ効くか / どこで使うか

- アロケータは、アプリ開発者が直接いじれる数少ない性能レバー。既定のアロケータは「あらゆる用途でそこそこ」を狙った妥協点なので、特定の使い方では必ず無駄が出る。
- **リアルタイム音声**: オーディオスレッドで `malloc` を呼ぶと、ロック競合やページフォールトで数十µs〜ms のスパイクが出てグリッチになる。だから「起動時に全部確保し、コールバック内では確保しない」が鉄則。DAW・シンセ・プラグインの設計はほぼこれ。
- **ゲーム**: フレーム単位で arena をリセットする（フレーム終了で一括解放）。1フレーム内の細かい free を消す。
- **サーバー**: マルチスレッドで既定アロケータのロックが競合する → `jemalloc` / `mimalloc` / `tcmalloc` に差し替える。
- **組み込み / no_std**: そもそもアロケータが存在しない世界。固定バッファか arena だけ。
- **Rust**: `Box` / `Vec` / `String` / `Rc` は全部 `GlobalAlloc` 経由。所有権が「いつ free するか」を決め、アロケータが「どう確保するか」を決める。役割分担がきれい。

## 前提となる知識

| 知っておくこと | なぜ必要か |
| --- | --- |
| 仮想メモリとページ | アロケータが OS からもらう単位はページ。`mmap` が何を返しているか分からないと「なぜまとめ取りするのか」が腑に落ちない |
| ポインタとアドレス | アロケータはアドレスを返す関数。`ptr` と `len` と `capacity` の3語で語れることが前提 |
| アラインメント | 型には整列要求がある（`u64` は8、`__m256` は32）。アロケータは整列済みのアドレスを返す義務があり、破ると UB |
| スタックの仕組み | 関数フレームと `rsp` の増減。ここを「アロケータ不要で速い領域」と理解できると話が半減する |
| キャッシュ階層と局所性 | 確保のしかたがそのままキャッシュヒット率になる。連続確保は速く、飛び飛びは遅い |
| システムコールのコスト | `brk` / `mmap` は数百ns〜µs オーダー。「1バイトごとに OS に頼む」のがどれだけ馬鹿げているか分かる |
| C の `malloc` / `free` | 手動解放、二重解放、リーク。Rust の自動解放と対比すると理解が早い |
| Rust の所有権と `Drop` | アロケータの「いつ返すか」を言語側が決めている。→ [[Rust のメモリ管理]] |
| スレッドとロック | アロケータの内部状態は共有資源。マルチスレッドで競合する理由 |

推奨順: 仮想メモリ → スタック/ヒープ → `malloc` の動き → 戦略（bump / free list / size class）→ 自分の用途での計測。

## 本体の解説

### 1. スタックとヒープは「別物」ではなく「別の確保方法」

| | スタック | ヒープ |
| --- | --- | --- |
| 誰が確保するか | コンパイラ（関数プロローグ） | アロケータ（実行時の関数呼び出し） |
| サイズ決定 | コンパイル時に既知 | 実行時に決まる |
| 解放 | 関数を抜けたら自動（`rsp` を戻すだけ） | 明示的（`free` / `Drop`） |
| 速さ | 加算・減算のみ（数ns） | 数十〜数百ns、競合時はµs |
| 上限 | ある（Linux のメインスレッドで既定 8MiB 程度、`ulimit -s`） | 仮想アドレス空間と物理メモリ次第 |
| 用途 | ローカル変数、小さい固定サイズ | 可変長、関数をまたいで生きるデータ |

重要なのは **スタックはアロケータを通らない** こと。`let x = 0u64;` はアロケーションではない。`Box::new(x)` や `Vec::new()` がアロケータを呼ぶ。

```
高いアドレス
+------------------+
| スタック ↓        |  ← rsp が上下するだけ。アロケータ不要
+------------------+
|       （空き）    |
+------------------+
| ヒープ ↑          |  ← アロケータが管理。brk / mmap で伸びる
+------------------+
| .bss / .data     |  ← 静的変数
+------------------+
| .text            |  ← コード
+------------------+
低いアドレス
```

（スレッドごとにスタックは別。Rust の `std::thread` の既定スタックサイズは 2MiB。）

#### 補足: 関数プロローグ＝「スタック確保」の正体

**関数プロローグ**は、関数が呼ばれた直後に実行される「フレームを用意する」定型コード。戻る直前の片付けが**エピローグ**。x86-64 だとだいたいこの3命令。

```asm
push %rbp        # 呼び出し元のフレーム基準を保存
mov  %rsp,%rbp   # この関数のフレーム基準点を作る
sub  $0x20,%rsp  # ローカル変数ぶん rsp を下げる ← これが「スタックの確保」
...
leave            # = mov %rbp,%rsp ; pop %rbp （エピローグ）
ret
```

実際に `gcc -O0` で `long work(long n)` をコンパイルして逆アセンブルするとこうなる（手元で実行して確認した実物）。

```asm
0000000000001139 <work>:
    1139: push   %rbp
    113a: mov    %rsp,%rbp
    113d: mov    %rdi,-0x48(%rbp)   # 引数 n をスタックに置く = ローカル変数
    1141: pxor   %xmm0,%xmm0
    1145: movaps %xmm0,-0x40(%rbp)  # long buf[8] をゼロ埋め
    ...
    1168: pop    %rbp               # エピローグ
    1169: ret                       # 戻りアドレスへジャンプ
```

この例で `sub $..,%rsp` が無いのは、**リーフ関数**（他の関数を呼ばない関数）が red zone（rsp の下 128 バイト）を自由に使えるから。関数を呼ぶ関数なら `sub` が入る。

ここが重要な点:

- スタックの「確保」は **整数の減算1命令**。OS にもアロケータにも問い合わせない。だから数ns。
- フレームは関数を抜けると `rsp` が戻るだけで消える。**解放し忘れが原理的に起きない**。
- 逆に、フレームが積み上がってスタック領域を超えると、ガードページに触れて `SIGSEGV`（スタックオーバーフロー）。だから深い再帰や巨大なローカル配列は死ぬ。

つまり「スタックは速い」の正体は、**プロローグで `rsp` を動かすだけだから**。ヒープが遅いのは、その一命令の代わりにアロケータの関数呼び出し・空き探索・（必要なら）`mmap` が挟まるから。

### 2. OS がくれるのは「ページ」だけ

- x86-64 のページは 4KiB（ヒュージページは 2MiB / 1GiB）。
- `mmap` は「この長さの仮想アドレス領域をくれ」と言うだけ。**物理メモリはまだ割り当てられていない**。初回アクセスでページフォールトが起きて、そこで初めて物理ページが付く（demand paging）。
- `brk` はプロセスのヒープ終端を動かす（連続領域なので断片化に弱い）。`mmap` は独立した領域を任意の場所に取れる（`munmap` で個別に返せる）。
- つまり **アロケータの入口も出口もページ単位**。1バイト欲しいだけでも、OS から見れば4KiB。この差を埋めるのがアロケータの存在理由。
- 逆に言えば、**一度 `mmap` した領域は、`free` しても OS にはすぐ返らない**（アロケータが再利用のために抱え込む）。だから「解放したのに RSS が減らない」は正常な挙動。

### 3. アロケータの仕事は3つだけ

1. **まとめ取り**: ページを確保してプールにする（`mmap` / `brk` は滅多に呼ばない）
2. **切り出し**: 要求サイズ＋アラインメントを満たすアドレスを返す（メタデータをどこかに置く必要がある）
3. **再利用**: `free` された領域を、次の要求に回す

この3つの実装方針の違いが、そのままアロケータの性格になる。

### 4. 代表的な戦略

**bump / arena（バンプアロケータ）**
ポインタを前進させるだけ。`alloc(n)` は `ptr += n; return old_ptr;` に近い。`free` は個別にはできず、**まとめてリセット**する。O(1)、断片化ゼロ、実装数十行。代わりにメモリは戻せない。

- 使う場所: フレーム単位（ゲーム）、リクエスト単位（サーバー）、コンパイル時のAST、音声バッファの一時作業領域。
- 考え方: 「同じ寿命のものをまとめて作って、まとめて捨てる」。個別 `free` が要らない設計にすると爆速になる。

**free list（フリーリスト）**
空きブロックを連結リストで持つ。要求サイズに合うブロックを探して返す（first-fit / best-fit など）。`free` で隣が空いていれば結合（coalesce）する。

- 弱点: 外部断片化（合計は足りているのに連続領域が取れない）、探索コスト。

**size class / segregated fit**
サイズを離散的なクラスに丸める（8, 16, 32, 48, 64 … ）。クラスごとにリストを持つので探索が O(1) に近い。丸めた分の無駄は内部断片化として現れる。glibc の bins、jemalloc の size class がこれ。

**slab**
同じサイズのオブジェクト専用のページ群。確保・解放が速く、同じ型が固まって並ぶのでキャッシュに乗る。カーネルの slab allocator、ユーザー空間では固定サイズプール。

**buddy**
2の冪で分割・結合する。管理が単純で外部断片化は起きにくいが、内部断片化が出る。カーネルの物理ページ管理など。

**スレッド対応（tcmalloc / jemalloc / mimalloc）**
スレッドローカルなキャッシュを持ってロックを回避し、サイズクラス＋ページ単位で OS に返す。マルチスレッドのサーバーで既定 `malloc` より速いことが多い。

### 5. `malloc` / `free` の実態（glibc ptmalloc2 の場合）

細部はバージョン依存なので「だいたいこう」として読む（`> 要確認:` の印を付けた箇所は手元の glibc で確認する）。

- 要求サイズにヘッダを足し、アラインメント（x86-64 では 16 バイト）に丸めてチャンクを作る。ヘッダは 16 バイト程度（サイズ + 前チャンク情報）。
- 一定サイズ（既定 128KiB 程度、動的に調整される）未満は `brk` 側のヒープから、以上は `mmap` で独立領域を取る。→ 大きな確保は `free` で OS に返りやすい。`> 要確認: MMAP_THRESHOLD の既定値`
- `free` はサイズクラス別の bin に戻す。隣接チャンクが空なら結合する。
- スレッドごとに arena を持つことでロック競合を緩和するが、スレッド数が多いと競合も断片化も増える。

### 6. Rust の場合

```rust
// これらはすべて GlobalAlloc を経由する（= OS のページを直接触らない）
let b: Box<u64> = Box::new(0);        // ヒープに 8 バイト
let v: Vec<u32> = Vec::with_capacity(16); // 64 バイトを一度に確保
let s: String = String::with_capacity(32);

// アロケートしない
let x = 0u64;              // スタック
let e: Vec<u8> = Vec::new(); // 容量0。push するまで確保しない
```

- `GlobalAlloc` トレイト: `alloc` / `dealloc` / `realloc`（既定は System = `malloc`/`free` 相当）。
- `Vec` は足りなくなると**容量を倍々**にして `realloc`（または確保＋コピー）。だから `push` を繰り返すと再確保とコピーが起きる。`with_capacity` / `reserve` で消せる。
- Rust 1.32 以降、既定アロケータはシステムのもの（それ以前は jemalloc 同梱）。
- `#[global_allocator]` で差し替え可能（`jemallocator` / `mimalloc` クレートなど）。
- `no_std` + `alloc` を使う場合、アロケータ実装を自分で提供する必要がある。
- 所有権が「いつ `dealloc` するか」を静的に決める。だからアロケータ側は「正しいタイミングで1回だけ呼ばれる」前提で最適化できる。→ [[Rust のメモリ管理]]

### 7. 性能に効く勘所

- **回数 × サイズ**: アロケーションは1回あたり数十ns。ループ内で何万回も呼ぶと効いてくる。`format!`、`collect()`、`clone()` の連発はここを叩く。
- **局所性**: 連続して確保したものはキャッシュに乗る。ポインタで繋いだノードを1個ずつ確保すると、ほぼ確実にキャッシュミスする。`Vec` が `LinkedList` より速いのは主にこれ。
- **false sharing**: 別スレッドが同じ 64 バイトのキャッシュラインを触ると、キャッシュが往復して激遅になる。アラインメントで分離する。
- **解放タイミング**: アロケータは OS にすぐ返さない。長時間動くプロセスでは「解放済みだが抱えている」メモリが増えうる（`malloc_trim` などで返せる）。
- **計測が先**: 勘でアロケータを替えない。heaptrack / massif / dhat / jemalloc の profiling で「回数」「サイズ分布」「ホットスポット」を見てから決める。

## 具体例・コード

### バンプアロケータを 40 行で書く

「まとめて確保して、まとめて捨てる」だけで、`free` の複雑さが全部消える。

```rust
/// 単調増加のみのアリーナ。reset() で全部まとめて回収する。
struct Bump {
    buf: Vec<u8>,
    offset: usize,
}

impl Bump {
    fn new(capacity: usize) -> Self {
        Self { buf: Vec::with_capacity(capacity), offset: 0 }
    }

    /// align バイト境界に揃えて size バイト切り出す。足りなければ None。
    fn alloc(&mut self, size: usize, align: usize) -> Option<&mut [u8]> {
        let start = (self.offset + align - 1) & !(align - 1);
        let end = start.checked_add(size)?;
        if end > self.buf.capacity() {
            return None;
        }
        if self.buf.len() < end {
            self.buf.resize(end, 0); // 容量内なので再確保は起きない
        }
        self.offset = end;
        Some(&mut self.buf[start..end])
    }

    fn reset(&mut self) {
        self.offset = 0;   // ここだけ。個別 free は無い
    }
}

fn main() {
    let mut arena = Bump::new(1024);
    let a = arena.alloc(8, 8).unwrap();
    a.copy_from_slice(b"abcdefgh");
    let b = arena.alloc(4, 4).unwrap();
    b.copy_from_slice(b"wxyz");
    println!("ok"); // 使った分は 12 バイト、あとはポインタ1個進めただけ
    arena.reset();  // 一括回収
}
```

ポイント: **`Vec<u8>` の容量を先に確保しておくので、途中で `malloc` は呼ばれない**。これがリアルタイム用途で効く形。`bumpalo` クレートはこれを本気で作り込んだもの。

### 音声コールバックで確保しない

```rust
// 起動時（RT スレッド外）に確保しておく
let mut scratch: Vec<f32> = Vec::with_capacity(MAX_BLOCK * 2);

// コールバック内（RT スレッド）: push しない、format! しない、clone しない
fn process(input: &[f32], out: &mut [f32], scratch: &mut [f32]) {
    for (o, i) in out.iter_mut().zip(input) {
        *o = *i * 0.5;
    }
    // scratch はスライスとして受け取り、長さは固定。確保なし
}
```

### `Vec` の伸び方を見る

```rust
let mut v: Vec<u32> = Vec::new();
println!("{}", v.capacity()); // 0（確保していない）
for i in 0..10 {
    v.push(i);
    println!("len={} cap={}", v.len(), v.capacity());
}
// 実測（rustc 1.100）: cap は 4,4,4,4,8,8,8,8,16,16 と伸びる
//   → 最初の push で4要素ぶん確保、以降は倍々。cap=0 から始まるので
//     「倍々」の前に「初回4」が挟まる点に注意
// with_capacity(10) なら最初から 10。再確保もコピーもゼロ
```

## つまずきポイント

- **「スタックのほうが速い」は条件つき**: 確保そのものは速いが、大きな配列をスタックに置くとコピーやスタックオーバーフローで負ける。数十MB をローカル変数で持つのは事故のもと。
- **`free` しても OS に返らない**: RSS が減らないのはリークとは限らない。アロケータが抱えているだけのことが多い。
- **リークと断片化は別問題**: リークは「解放し忘れ」、断片化は「解放したのに使えない」。後者は合計メモリが足りていても OOM になる。
- **`malloc(0)` の挙動は実装依存**: `NULL` かユニークなポインタ。`free` してよい、という点だけは共通。
- **`realloc` は移動しうる**: 古いポインタは無効になる。Rust の `Vec` で「中身への参照を持ったまま `push`」が `E0502` になるのは、まさにこの事故を防ぐため。
- **アラインメント違反は UB**: x86 では動いてしまうが、ARM や SIMD 命令で落ちる。「動いたから正しい」が通用しない代表例。
- **`Vec` の倍々確保でピークメモリが 3 倍**: 古い領域＋新しい領域＋コピー中の一時。`shrink_to_fit` や `with_capacity` で制御する。
- **スレッド終了後もキャッシュが残る**: スレッドローカルなアロケータは、スレッドが消えてもメモリを抱え続けることがある。
- **RT スレッドでの `Drop`**: `Box` や `String` を捨てると `free` が走る。コールバック内で `drop` が起きる設計は避ける（別スレッドに渡して捨てる）。
- **計測せずにアロケータを差し替える**: たいてい遅い原因はアロケータではなくアルゴリズムか局所性。

## 疑問・TODO

- [ ] 音声処理なら `jemalloc` / `mimalloc` / システム、実際にどれが有利か計測する（アロケーション回数と最大レイテンシで比較）
- [ ] ヒュージページ（2MiB）を音声バッファに使うと TLB ミスが減るか
- [ ] `allocator_api`（Allocator トレイト）の安定化状況を追う
- [ ] 自前 arena をコールバックに渡す設計を、実際に小さいシンセで書いてみる
- [ ] NUMA 環境での配置（`numa_alloc_local`）は実際どれくらい効くか
- [ ] `malloc_trim` / `MADV_DONTNEED` で OS に返す挙動を実測

## 関連ノート

- [[Rust のメモリ管理]] — 所有権・借用・`Drop` が「いつ解放するか」を決める部分

## 出典

- man 3 malloc、man 2 brk、man 2 mmap（Linux man-pages）
- Rust `std::alloc` / `GlobalAlloc`: https://doc.rust-lang.org/std/alloc/
- glibc MallocInternals: https://sourceware.org/glibc/wiki/MallocInternals
- jemalloc: https://jemalloc.net/ ／ mimalloc: https://github.com/microsoft/mimalloc
- Linux kernel docs（メモリ管理の概念）: https://www.kernel.org/doc/html/latest/admin-guide/mm/concepts.html
- Ulrich Drepper, "What Every Programmer Should Know About Memory": https://people.freebsd.org/~lstewart/articles/cpumemory.pdf
