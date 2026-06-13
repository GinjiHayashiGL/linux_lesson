# head3.c 解説

`head` コマンドの拡張実装。`getopt_long` を使ってショートオプション（`-n`）とロングオプション（`--lines`、`--help`）の両方を受け付ける。`-n` 未指定時はデフォルト 10 行を出力する。head.c・head2.c とは異なり、`<getopt.h>` を活用した本格的なオプション解析を導入している。

## 概要

```
./head3 [-n LINES] [FILE ...]
```

---

## インクルードファイル

| ヘッダ | 概要 | 用途 |
|--------|------|------|
| `stdio.h` | 標準入出力ライブラリ | `fopen`、`getc`、`putchar`、`fclose`、`fprintf`、`perror` |
| `stdlib.h` | 標準ユーティリティライブラリ | `atol`、`exit` |
| `getopt.h` | オプション解析ライブラリ。GNU 拡張を含む | `getopt_long`、`struct option` |

---

## 関数

### `main`

```c
int main(int argc, char *argv[])
```

```c
#define _GNU_SOURCE
```
- ファイル先頭で宣言する機能有効化マクロ。`getopt_long` は POSIX 標準には含まれない GNU 拡張であるため、このマクロで GNU 拡張を有効にする必要がある。
- `#include` より前に書かなければならない。

```c
#define DEFAULT_N_LINES 10
```
- `-n` オプションが指定されなかった場合のデフォルト行数。`man head` の仕様（デフォルト 10 行）に合わせた値。

```c
static struct option longopts[] = {
    {"lines", required_argument, NULL, 'n'},
    {"help",  no_argument,       NULL, 'h'},
    {0, 0, 0, 0}
};
```
- `getopt_long` に渡すロングオプションの定義テーブル。`struct option` は `getopt.h` で定義される。
  - フィールド1: ロングオプション名（`--lines`、`--help`）
  - フィールド2: 引数の有無（`required_argument` / `no_argument`）
  - フィールド3: フラグポインタ（`NULL` の場合はフィールド4の値を返す）
  - フィールド4: ショートオプションに対応する文字
- 末尾は `{0, 0, 0, 0}` のセンチネルで終端する。
- `static` にすることでプログラム起動から終了まで静的領域に確保し、`main` 呼び出し前から有効にしている。

```c
int opt;
long nlines = DEFAULT_N_LINES;
```
- `opt` は `getopt_long` の戻り値を受け取るオプション文字。
- `nlines` はデフォルト値で初期化しておくことで、`-n` が指定されなくても 10 行動作を保証する。

```c
while((opt = getopt_long(argc, argv, "n:h", longopts, NULL)) != -1) {
```
- `getopt_long` のシグネチャ: `int getopt_long(int argc, char *const argv[], const char *optstring, const struct option *longopts, int *longindex)`
  - `optstring` の `"n:h"` は `-n`（引数あり）と `-h`（引数なし）を受け付けることを示す。コロン(`:`)が「直前のオプションは引数を必要とする」の意味。
  - `longopts` は上で定義した `longopts[]` を渡す。
  - `longindex` は `NULL`（ロングオプションのインデックスが不要なため）。
  - オプションがなくなると `-1` を返してループ終了。
  - 未知のオプションが与えられた場合は `'?'` を返す。

```c
case 'n':
    nlines = atol(optarg);
    break;
```
- `optarg` は `getopt_long` がセットするグローバル変数で、オプションの引数文字列を指す。
- `atol` のシグネチャ: `long atol(const char *nptr)`。文字列を `long` 型整数に変換する。変換できない場合は `0` を返す（エラー検出は行わない）。

```c
case 'h':
    fprintf(stdout, "Usage: %s [-n LINES] [FILE ...]\n", argv[0]);
    exit(0);
case '?':
    fprintf(stderr, "Usage: %s [-n LINES] [FILE ...]\n", argv[0]);
    exit(1);
```
- `-h` / `--help` は標準出力にヘルプを表示して正常終了。
- `'?'` は未知オプションのエラーケース。標準エラー出力に出力して異常終了。

```c
if (optind == argc) {
    do_head(stdin, nlines);
} else {
```
- `optind` は `getopt_long` が更新するグローバル変数で、オプション解析後の次の引数インデックスを示す。
- `optind == argc` ならすべての引数がオプションであり、ファイル引数が存在しないので標準入力から読む。

```c
for (i = optind; i < argc; i++) {
    FILE *f;
    f = fopen(argv[i], "r");
    if (!f) {
        perror(argv[i]);
        exit(1);
    }
    do_head(f, nlines);
    fclose(f);
}
```
- `optind` から始めることでオプション以外のファイル引数だけを処理する。
- `fopen` のシグネチャ: `FILE *fopen(const char *path, const char *mode)`。失敗時は `NULL` を返す。
- `perror` のシグネチャ: `void perror(const char *s)`。`s` の後に `: errno のメッセージ` を標準エラー出力に出力する。

---

### `do_head`

```c
static void do_head(FILE *f, long nlines)
```
- `static` にすることで他のファイルからこの関数を参照できなくする（内部実装の隠蔽）。
- head.c・head2.c と同一の実装。コア処理を関数として分離することで `main` の責務（オプション解析）とコア処理（行数制限付き出力）を分けている。

```c
int c;

if (nlines <= 0) return;
```
- `int c` は `getc` の戻り値を受け取る変数。`EOF`（`-1`）と通常バイト（0〜255）を区別するため `char` ではなく `int` を使う。
- `nlines <= 0` の早期 `return` で、0 行以下が指定された場合に何も出力しない。

```c
while ((c = getc(f)) != EOF) {
    if (putchar(c) < 0) exit(1);
    if (c == '\n') {
        nlines--;
        if (nlines == 0) return;
    }
}
```
- `getc` のシグネチャ: `int getc(FILE *stream)`。ストリームから 1 バイト読み込む。EOF またはエラー時は `EOF` を返す。`fgetc` のマクロ版で速度優先の実装。
- `putchar` のシグネチャ: `int putchar(int c)`。1 バイトを標準出力に書き出す。エラー時は `EOF`（負の値）を返す。
- 改行文字を検出するたびに `nlines` をデクリメントし、0 になったら `return`（指定行数の出力完了）。

---

## 処理の流れ

```
main
 ├─ getopt_long() ループ        // -n / --lines / -h / --help の解析
 │   ├─ case 'n': nlines = atol(optarg)
 │   ├─ case 'h': ヘルプ表示 → exit(0)
 │   └─ case '?': エラー表示  → exit(1)
 ├─ [ファイル引数なし]
 │   └─ do_head(stdin, nlines) // 標準入力から読む
 └─ [ファイル引数あり]
     └─ for (i = optind; i < argc; i++)
         ├─ fopen(argv[i], "r")
         ├─ do_head(f, nlines)
         └─ fclose(f)

do_head(f, nlines)
 └─ while getc(f) != EOF
     ├─ putchar(c)
     └─ c == '\n' → nlines--; if 0 → return
```

---

## 関数・システムコール一覧

| 関数 | シグネチャ | 説明 |
|------|-----------|------|
| `getopt_long` | `int getopt_long(int argc, char *const argv[], const char *optstring, const struct option *longopts, int *longindex)` | ショート・ロングオプションを解析する。オプションがなくなると `-1` を返す |
| `atol` | `long atol(const char *nptr)` | 文字列を `long` 型整数に変換する。変換不能時は `0` |
| `fopen` | `FILE *fopen(const char *path, const char *mode)` | ファイルを開いてストリームを返す。失敗時は `NULL` |
| `getc` | `int getc(FILE *stream)` | ストリームから 1 バイト読み込む。EOF またはエラー時は `EOF` |
| `putchar` | `int putchar(int c)` | 標準出力に 1 バイト書き出す。エラー時は `EOF` |
| `fclose` | `int fclose(FILE *stream)` | ストリームを閉じてバッファをフラッシュする。エラー時は `EOF` |
| `perror` | `void perror(const char *s)` | `errno` に対応するエラーメッセージを標準エラー出力に出力する |
| `fprintf` | `int fprintf(FILE *stream, const char *format, ...)` | 書式付きで指定ストリームに出力する |
| `exit` | `void exit(int status)` | プログラムを終了する。`0` で正常終了、`1` で異常終了 |

---

## head.c・head2.c との比較

| 観点 | head.c | head2.c | head3.c |
|------|--------|---------|---------|
| 行数指定 | 位置引数 `argv[1]`（必須） | 位置引数 `argv[1]`（必須） | `-n` / `--lines` オプション（省略可、デフォルト 10） |
| デフォルト行数 | なし（指定必須） | なし（指定必須） | 10 行 |
| ヘルプオプション | なし | なし | `-h` / `--help` |
| 入力ソース | 標準入力のみ | 標準入力 or 複数ファイル | 標準入力 or 複数ファイル |
| オプション解析 | なし（位置引数） | なし（位置引数） | `getopt_long` |
| 追加ヘッダ | なし | なし | `<getopt.h>` + `_GNU_SOURCE` |

head.c・head2.c では行数を位置引数で渡す必要があったが、head3.c では `getopt_long` の導入により POSIX 標準の `head -n` 構文に準拠したインターフェースを実現している。`-n` 省略時のデフォルト 10 行も、本物の `head` コマンドと同等の動作になっている。
