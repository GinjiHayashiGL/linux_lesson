# head2.c 解説

`head` コマンドの拡張実装。head.c が標準入力のみを対象としていたのに対し、複数のファイルパスを引数として受け取れるように拡張した版。

## 概要

```
./head2 <行数> [ファイル ファイル ...]
```

ファイルを指定しない場合は標準入力を読み込む。複数ファイルを指定すると、それぞれの先頭 N 行を順番に出力する。

```sh
./head2 5 < test_files/test_file.txt
./head2 3 test_files/test_file.txt
./head2 3 test_files/test_file.txt test_files/test_file.txt
```

---

## インクルードファイル

| ヘッダ | 概要 | 用途 |
|--------|------|------|
| `<stdio.h>` | 標準入出力 | `FILE`, `fopen`, `fclose`, `getc`, `putchar`, `fprintf`, `perror`, `EOF` |
| `<stdlib.h>` | 汎用ユーティリティ | `atol`, `exit` |

---

## 関数

### main

```c
int
main(int argc, char *argv[])
```

```c
if (argc < 2) {
    fprintf(stderr, "Usage: %s n [file file...]\n", argv[0]);
    exit(1);
}
```

- head.c では `argc != 2` で厳密に引数 1 つを要求していたが、ここでは `argc < 2` に緩和している。行数の指定（`argv[1]`）は必須だが、ファイル引数は任意のためこの条件になっている
- `fprintf(FILE *stream, const char *fmt, ...)` — 任意のストリームへフォーマット文字列を出力する。エラーメッセージは `stderr` に分けることでリダイレクト時に画面に表示され続ける。成功時は書き込んだ文字数、エラー時は負値を返す

```c
nlines = atol(argv[1]);
```

- `atol(const char *str)` — 文字列を `long` 整数に変換する。変換できない文字に達した時点で変換を停止し、それまでの値を返す。負数や非数値を渡した場合でも変換を続けるが、`do_head` 側で `nlines <= 0` のガードがあるため安全に扱える
- `main` の冒頭で変換しておき、`do_head` に渡す引数として使い回す設計

```c
if (argc == 2) {
    do_head(stdin, nlines);
} else {
    int i;

    for (i = 2; i < argc; i++) {
        FILE *f;

        f = fopen(argv[i], "r");
        if (!f) {
            perror(argv[i]);
            exit(1);
        }
        do_head(f, nlines);
        fclose(f);
    }
}
```

- `argc == 2` のとき（行数のみ指定）は `stdin` を直接 `do_head` に渡す。head.c と同じ動作
- `argc > 2` のときは `argv[2]` 以降をファイルパスとして扱い、`fopen` で順に開いて `do_head` を呼び出す
- `fopen(const char *path, const char *mode)` — ファイルをテキストモード `"r"` で開き、`FILE *` を返す。失敗時は `NULL` を返す。`NULL` の場合は `perror` でエラーメッセージを出力してプロセスを終了する
- `perror(const char *s)` — `s`（ここではファイル名）に続けて `errno` に対応するシステムエラーメッセージを標準エラー出力に出力する。ファイルが存在しない・権限がないといった原因を利用者に伝えるのに適している
- ループの最後で `fclose(f)` を呼び出し、ファイルディスクリプタを解放する。複数ファイルを処理するため、1ファイルごとに開閉することでリソースリークを防いでいる

```c
exit(0);
```

- 全ファイルの処理が正常に完了したことを示す正常終了

---

### do_head

```c
static void
do_head(FILE *f, long nlines)
{
    int c;

    if (nlines <= 0) return;
    while ((c = getc(f)) != EOF) {
        if (putchar(c) < 0) exit(1);
        if (c == '\n') {
            nlines--;
            if (nlines == 0) return;
        }
    }
}
```

- `static` 修飾子により、この関数のリンケージをファイルスコープに限定している。外部から直接呼び出す想定がなく、誤った依存を防ぐための設計
- `nlines <= 0` を先頭でチェックすることで、0 行・負数を引数に渡しても安全に即時リターンする
- `getc(FILE *stream)` — ストリームから 1 文字読み込み、`unsigned char` を `int` として返す。EOF またはエラー時は `EOF`（通常 `-1`）を返す。`fgetc` と機能は同じだが、`getc` はマクロとして実装されることがあり、副作用のある式を引数に渡してはならない点に注意する。読み込みとループ継続判定を `while` の条件式に埋め込むことで 1 行にまとめている
- `putchar(int c)` — 標準出力へ 1 文字書き出す。内部的には `putc(c, stdout)` と等価。成功時は書き込んだ文字を `int` で返し、エラー時は `EOF`（負値）を返す。パイプ切断など出力先が閉じられた場合にも `exit(1)` で即座に終了する
- `'\n'` を検出するたびに `nlines` をデクリメントし、0 になったら `return` する。行カウントをシーク操作に頼らず文字カウントで実現しているため、シーク不可なストリーム（パイプ、`stdin`）でも正しく動作する
- head.c の `do_head` と実装は完全に同一。`main` 側でファイルか stdin かを切り替えているため、この関数自体は変更不要

---

## 処理の流れ

```
main()
  │
  ├─ 引数チェック (argc < 2)
  │      └─ 失敗 → Usage を stderr に出力して exit(1)
  │
  ├─ nlines = atol(argv[1])
  │
  ├─ argc == 2?
  │      └─ Yes → do_head(stdin, nlines)
  │
  └─ argc > 2 → for (i = 2; i < argc; i++)
         ├─ fopen(argv[i], "r")
         │      └─ 失敗 → perror + exit(1)
         ├─ do_head(f, nlines)
         │      ├─ nlines <= 0 → 即 return
         │      └─ ループ: getc(f) != EOF
         │             ├─ putchar(c) → 出力エラーなら exit(1)
         │             └─ c == '\n'?
         │                    ├─ Yes → nlines--
         │                    │          └─ nlines == 0 → return
         │                    └─ No  → 次の文字へ
         └─ fclose(f)
```

---

## 関数・システムコール一覧

| 関数（またはシステムコール） | シグネチャ | 説明 |
|-----------------------------|-----------|------|
| `atol` | `long atol(const char *str)` | 文字列を `long` 整数に変換する |
| `fopen` | `FILE *fopen(const char *path, const char *mode)` | ファイルを開いてストリームを返す。失敗時は `NULL` |
| `fclose` | `int fclose(FILE *stream)` | ストリームを閉じてバッファをフラッシュする。エラーで `EOF` |
| `getc` | `int getc(FILE *stream)` | ストリームから 1 文字読み込む。EOF またはエラー時は `EOF` |
| `putchar` | `int putchar(int c)` | 標準出力へ 1 文字書き出す。エラー時は負値を返す |
| `fprintf` | `int fprintf(FILE *stream, const char *fmt, ...)` | フォーマット文字列をストリームへ出力する |
| `perror` | `void perror(const char *s)` | `s` に続けて `errno` のエラーメッセージを標準エラー出力に出力する |
| `exit` | `void exit(int status)` | プロセスを終了する |

---

## head.c との比較

| 観点 | head.c | head2.c |
|------|--------|---------|
| 入力ソース | 標準入力のみ | 標準入力 または 複数ファイル |
| 引数の形式 | `./head n`（引数は行数のみ） | `./head2 n [file ...]`（ファイルは任意） |
| 引数チェック | `argc != 2`（行数は必須かつ 1 つのみ） | `argc < 2`（行数は必須、ファイルは 0 個以上） |
| ファイルオープン | 不要（stdin を直接渡す） | `fopen` / `fclose` でループ処理 |
| エラー報告 | `fprintf(stderr, ...)` で使い方のみ | `perror` でファイル単位のエラーも表示 |
| `do_head` の実装 | 変更なし | 変更なし（共通ロジックを再利用） |

head.c が標準入力専用だったのに対し、head2.c は `main` のみを拡張することで複数ファイルの処理を追加している。`do_head` の行カウントロジックは変更せずに再利用できており、ファイルと `stdin` の切り替えを呼び出し側（`main`）で吸収した設計になっている。
