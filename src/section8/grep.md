# grep.c 解説

`grep` コマンドの簡易実装。POSIX 拡張正規表現を使い、パターンに一致する行をファイル（または標準入力）から抽出して標準出力に書き出す。

## 概要

```
./grep <パターン> [ファイル名 ...]
```

ファイルを指定しない場合は標準入力を対象にする。

---

## インクルードファイル

| ヘッダ | 概要 | 用途 |
|--------|------|------|
| `stdio.h` | 標準入出力ライブラリ | `fopen`、`fclose`、`fgets`、`fputs`、`perror` |
| `stdlib.h` | 標準ユーティリティライブラリ | `exit`（プログラム終了） |
| `string.h` | 文字列操作ライブラリ | 将来的な拡張のためのインクルード（本実装では直接使用なし） |
| `sys/types.h` | POSIX 型定義 | `regex_t` などの POSIX 型に必要な基盤型を提供する |
| `regex.h` | POSIX 正規表現ライブラリ | `regex_t`、`regcomp`、`regexec`、`regfree`、`regerror` |

---

## 関数

### `main`

```c
int main(int argc, char *argv[])
```

```c
regex_t pat;
int err;
int i;
```
- `regex_t pat` はコンパイル済み正規表現を保持する構造体。`regcomp` が内部を初期化し、`regfree` で解放する。
- `err` は `regcomp` の戻り値（エラーコード）を受け取る。
- `i` はファイル引数ループのカウンタ。

```c
if (argc < 2) {
    fputs("no pattern\n", stderr);
    exit(1);
}
```
- コマンドライン引数が 1 つもない（パターン未指定）場合はエラーを標準エラー出力に出して終了する。
- `fputs(const char *s, FILE *stream)` — 文字列 `s` をストリームに書き出す。`printf` と異なりフォーマット解析のオーバーヘッドがなく、固定メッセージの出力に適している。エラー時は `EOF` を返す。

```c
err = regcomp(&pat, argv[1], REG_EXTENDED | REG_NOSUB | REG_NEWLINE);
```
- `regcomp(regex_t *preg, const char *pattern, int cflags)` — 正規表現文字列をコンパイルして `preg` に格納する。成功時は `0`、失敗時は非ゼロのエラーコードを返す。
- `REG_EXTENDED` — 拡張正規表現（ERE）として解釈する。`+`、`?`、`|` などが使えるようになる。
- `REG_NOSUB` — マッチ位置を記録しない。`regexec` の `pmatch` 引数が不要になり、マッチ有無だけを判定する今回の用途に最適。
- `REG_NEWLINE` — `^` / `$` を行頭・行末にマッチさせ、`.` が改行にマッチしないようにする。行単位の処理に必要。

```c
if (err != 0) {
    char buf[1024];

    regerror(err, &pat, buf, sizeof buf);
    puts(buf);
    exit(1);
}
```
- `regerror(int errcode, const regex_t *preg, char *errbuf, size_t errbuf_size)` — エラーコードを人間が読めるメッセージ文字列に変換して `errbuf` に書き込む。バッファが小さい場合は切り捨てる。
- コンパイル失敗時にエラー内容を表示して終了する。

```c
if (argc == 2) {
    do_grep(&pat, stdin);
} else {
    for (i = 2; i < argc; i++) {
        FILE *f;

        f = fopen(argv[i], "r");
        if (!f) {
            perror(argv[i]);
            exit(1);
        }
        do_grep(&pat, f);
        fclose(f);
    }
}
```
- ファイル引数がなければ `stdin` を対象にする。これは Unix のフィルタ慣習（パイプやリダイレクトで使える）に則った設計。
- ファイルがある場合は `fopen` で順に開いて `do_grep` を呼ぶ。
- `fopen(const char *path, const char *mode)` — ファイルをテキストモード（`"r"`）で開いてストリームを返す。失敗時は `NULL`。
- `fclose(FILE *stream)` — ストリームを閉じ、バッファをフラッシュして OS にファイルディスクリプタを返却する。エラー時は `EOF`。

```c
regfree(&pat);
exit(0);
```
- `regfree(regex_t *preg)` — `regcomp` が確保した内部メモリを解放する。`regex_t` は使い終わったら必ず呼ぶ必要がある。

---

### `do_grep`

```c
static void do_grep(regex_t *pat, FILE *src)
```
- `static` 修飾子によりこの翻訳単位（ファイル）内限定の関数とし、シンボルの漏れを防ぐ。
- `regex_t *pat` はコンパイル済みパターンへのポインタ。`const` を付けたいところだが、`regexec` のシグネチャが `const regex_t *` を受け取るため、呼び出し側で暗黙変換される。

```c
char buf[4096];
```
- 1 行分のバッファ。4096 バイトはカーネルの典型的なページサイズと同じで、一般的なテキスト行には十分な容量。

```c
while (fgets(buf, sizeof buf, src)) {
```
- `fgets(char *s, int size, FILE *stream)` — ストリームから最大 `size - 1` バイト読み込み、改行または EOF で停止する。改行文字も `s` に含まれる。EOF またはエラー時は `NULL` を返す。
- 行末の `\n` が `buf` に残るため、`fputs` で再出力しても余分な改行が入らない。

```c
    if (regexec(pat, buf, 0, NULL, 0) == 0) {
        fputs(buf, stdout);
    }
```
- `regexec(const regex_t *preg, const char *string, size_t nmatch, regmatch_t pmatch[], int eflags)` — `string` をコンパイル済みパターン `preg` と照合する。マッチすれば `0`、不一致は `REG_NOMATCH` を返す。
- `REG_NOSUB` でコンパイルしているため `nmatch = 0`、`pmatch = NULL` を渡せばよく、マッチ位置の記録コストがかからない。
- マッチした行はそのまま `fputs` で標準出力へ書き出す。

---

## 処理の流れ

```
main
 ├─ 引数チェック（argc < 2）
 ├─ regcomp(argv[1], REG_EXTENDED|REG_NOSUB|REG_NEWLINE)  // パターンのコンパイル
 ├─ [argc == 2] do_grep(&pat, stdin)                       // 標準入力を処理
 │   または
 │   for (各ファイル引数)
 │    ├─ fopen(argv[i], "r")
 │    ├─ do_grep(&pat, f)
 │    └─ fclose(f)
 └─ regfree(&pat)

do_grep(pat, src)
 └─ loop: fgets(buf, sizeof buf, src)
     ├─ regexec(pat, buf, 0, NULL, 0) == 0?
     │   └─ YES: fputs(buf, stdout)
     └─ NO: 次の行へ
```

---

## 関数・システムコール一覧

| 関数 | シグネチャ | 説明 |
|------|-----------|------|
| `regcomp` | `int regcomp(regex_t *preg, const char *pattern, int cflags)` | 正規表現をコンパイルする。成功時 `0`、失敗時は非ゼロのエラーコード |
| `regexec` | `int regexec(const regex_t *preg, const char *string, size_t nmatch, regmatch_t pmatch[], int eflags)` | 文字列とコンパイル済みパターンを照合する。マッチで `0`、不一致で `REG_NOMATCH` |
| `regfree` | `void regfree(regex_t *preg)` | `regcomp` が確保した内部メモリを解放する |
| `regerror` | `size_t regerror(int errcode, const regex_t *preg, char *errbuf, size_t errbuf_size)` | 正規表現エラーコードを人間可読なメッセージに変換する |
| `fopen` | `FILE *fopen(const char *path, const char *mode)` | ファイルを開いてストリームを返す。失敗時は `NULL` |
| `fclose` | `int fclose(FILE *stream)` | ストリームを閉じてバッファをフラッシュする。エラー時は `EOF` |
| `fgets` | `char *fgets(char *s, int size, FILE *stream)` | ストリームから最大 `size-1` バイト、または改行まで読み込む。EOF・エラーで `NULL` |
| `fputs` | `int fputs(const char *s, FILE *stream)` | 文字列をストリームに書き出す。エラー時は `EOF` |
| `perror` | `void perror(const char *s)` | `s` に続けて `errno` に対応するエラーメッセージを標準エラー出力に出力する |
| `exit` | `void exit(int status)` | プログラムを終了する。`0` で正常終了、非ゼロで異常終了 |
