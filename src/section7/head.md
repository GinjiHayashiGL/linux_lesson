# head.c 解説

標準入力から先頭 N 行を出力する `head` コマンドの自作実装。

## 概要

```
./head <行数>
./head <行数> < <ファイル>
```

標準入力を読み込み、指定した行数分だけ先頭から出力して終了する。  
シェルのリダイレクトでファイルを標準入力に渡すこともできる。

```sh
./head 5 < test_files/test_file.txt
```

## インクルードファイル

| ヘッダ | 概要 | 用途 |
|--------|------|------|
| `<stdio.h>` | 標準入出力 | `FILE`, `getc`, `putchar`, `fprintf`, `EOF` |
| `<stdlib.h>` | 汎用ユーティリティ | `atol`, `exit` |

## 関数

### main

```c
int
main(int argc, char *argv[])
{
    if (argc != 2) {
        fprintf(stderr, "Usage: %s n\n", argv[0]);
        exit(1);
    }
    do_head(stdin, atol(argv[1]));
    exit(0);
}
```

- 引数が 1 つ（コマンド名を除く）でなければ使い方を標準エラーに出力して終了する
- `fprintf(FILE *stream, const char *fmt, ...)` — フォーマット文字列を任意のストリームへ出力する。`printf` と異なり出力先を指定できるため、エラーメッセージを `stderr` に分けて書き出せる。成功時は書き込んだ文字数、エラー時は負値を返す
- `exit(int status)` — 標準入出力バッファをフラッシュしてプロセスを終了する。`status` が `0` なら正常終了、非 `0` なら異常終了としてシェルに通知される。`return` ではなく `exit` を使うことで、呼び出し元の `main` に戻らず確実に終了する
- `atol(const char *str)` — 文字列を `long` 整数に変換する。変換できない文字が現れた時点で変換を停止し、それまでの値を返す。`argv[1]` が数値文字列であることを前提に使用している
- `stdin` をそのまま渡すことで、シェルのリダイレクト（`./head 5 < file.txt`）やパイプと自然に連携できる

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

- `static` 修飾子により、この関数のリンケージをファイルスコープに限定している。外部から直接呼び出す想定がないため、誤った依存を防ぐ設計
- `nlines <= 0` を先頭でチェックすることで、0 行や負数を引数として渡しても安全に即時リターンする
- `getc(FILE *stream)` — ストリームから 1 文字読み込み、`unsigned char` を `int` として返す。ストリームの終端に達すると `EOF`（通常 `-1`）を返す。`fgetc` と機能は同じだが、`getc` はマクロとして実装されることがあり、引数に副作用のある式を渡してはならない。ここでは `c = getc(f)` の代入を `while` の条件式に埋め込むことで、読み込みとループ継続判定を 1 行にまとめている
- `putchar(int c)` — 標準出力へ 1 文字書き出す。内部的には `putc(c, stdout)` と等価。成功時は書き込んだ文字を `int` で返し、エラー時は `EOF`（負値）を返す。戻り値が負の場合は即 `exit(1)` することで、パイプ切断など出力先が閉じられた状況に対しても堅牢に動作する
- `'\n'` を検出するたびに `nlines` をデクリメントし、0 になったら `return` する。行カウントをファイルポインタや seek に頼らず文字カウントで実現しているため、シーク不可なストリーム（パイプ、`stdin`）でも動作する

## 処理の流れ

```
main()
  │
  ├─ 引数チェック (argc != 2)
  │      └─ 失敗 → Usage を stderr に出力して exit(1)
  │
  └─ do_head(stdin, nlines)
         │
         ├─ nlines <= 0 → 即 return
         │
         └─ ループ: getc(f) != EOF
                │
                ├─ putchar(c) → 出力エラーなら exit(1)
                │
                └─ c == '\n'?
                       ├─ Yes → nlines--
                       │          └─ nlines == 0 → return
                       └─ No  → 次の文字へ
```

## 関数・システムコール一覧

| 関数（またはシステムコール） | シグネチャ | 説明 |
|-----------------------------|-----------|------|
| `atol` | `long atol(const char *str)` | 文字列を `long` 整数に変換する |
| `getc` | `int getc(FILE *stream)` | ストリームから 1 文字読み込む。EOF または文字を返す |
| `putchar` | `int putchar(int c)` | 標準出力へ 1 文字書き出す。エラー時は負値を返す |
| `fprintf` | `int fprintf(FILE *stream, const char *fmt, ...)` | フォーマット文字列をストリームへ出力する |
| `exit` | `void exit(int status)` | プロセスを終了する |
