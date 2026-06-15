# ls.c 解説

ディレクトリの内容（エントリ名）を一行ずつ標準出力へ表示する自作 `ls` コマンド。

## 概要

```
./ls <ディレクトリ> [ディレクトリ ...]
```

---

## インクルードファイル

| ヘッダ | 概要 | 用途 |
|--------|------|------|
| `stdio.h` | 標準入出力ライブラリ | `printf`、`fprintf`、`perror` |
| `stdlib.h` | 標準ユーティリティライブラリ | `exit`（プログラム終了） |
| `sys/types.h` | POSIX 型定義 | `DIR` 型などに必要な基盤型を提供する |
| `dirent.h` | ディレクトリ操作ライブラリ | `DIR`、`struct dirent`、`opendir`、`readdir`、`closedir` |

---

## 関数

### `main`

```c
int main(int argc, char *argv[])
```

```c
int i;
```
- ファイル引数のループカウンタ。

```c
if (argc < 2) {
    fprintf(stderr, "%s: no arguments\n", argv[0]);
    exit(1);
}
```
- ディレクトリ引数が一つもない場合、エラーメッセージを標準エラー出力に書き出して終了する。
- `fprintf(FILE *stream, const char *format, ...)` — 指定ストリームにフォーマット文字列を書き出す。`stderr` に向けることでリダイレクト時にもエラーがターミナルへ届く。エラー時は負値を返す。
- `argv[0]` を使ってコマンド名を動的に埋め込むことで、実行ファイル名が変わっても一貫したメッセージが出る。

```c
for (i = 1; i < argc; i++) {
    do_ls(argv[i]);
}
```
- `argv[1]` 以降を順に `do_ls` へ渡す。複数ディレクトリを一度に指定できる。

```c
exit(0);
```
- `exit(int status)` — プログラムを終了する。`0` で正常終了、非ゼロで異常終了。`return 0` と異なり `atexit` ハンドラも呼ばれる。

---

### `do_ls`

```c
static void do_ls(char *path)
```
- `static` 修飾子によりこの翻訳単位（ファイル）内限定の関数とし、外部へのシンボル漏れを防ぐ。

```c
DIR *d;
struct dirent *ent;
```
- `DIR *d` — ディレクトリストリームへのポインタ。`FILE *` のディレクトリ版に相当する。
- `struct dirent *ent` — `readdir` が返すエントリ情報を保持する構造体。`d_name` メンバにエントリ名が格納される。

```c
d = opendir(path);
if (!d) {
    perror(path);
    exit(1);
}
```
- `opendir(const char *name)` — 指定パスのディレクトリストリームを開き `DIR *` を返す。失敗時は `NULL` を返し `errno` にエラーコードを設定する。`fopen` のディレクトリ版。
- 失敗した場合は `perror(path)` でパス名とエラー内容を標準エラー出力に表示して終了する。
- `perror(const char *s)` — `s` に続けて `errno` に対応するエラーメッセージを標準エラー出力に出力する。シグネチャ: `void perror(const char *s)`。

```c
while (ent = readdir(d)) {
    printf("%s\n", ent->d_name);
}
```
- `readdir(DIR *dirp)` — ディレクトリストリームから次のエントリを読み取り `struct dirent *` を返す。エントリがなくなると `NULL` を返す。シグネチャ: `struct dirent *readdir(DIR *dirp)`。
- ループが `NULL` を受け取った時点で全エントリの列挙が完了する。
- `ent->d_name` にエントリ名（ファイル名・サブディレクトリ名）が入っている。`.` や `..` も含め、ディレクトリ内の全エントリが返される点に注意。
- `printf("%s\n", ent->d_name)` でエントリ名を改行付きで標準出力へ書き出す。

```c
closedir(d);
```
- `closedir(DIR *dirp)` — ディレクトリストリームを閉じてリソースを解放する。`fclose` のディレクトリ版。シグネチャ: `int closedir(DIR *dirp)`。エラー時は `-1` を返す。
- 開いたストリームは必ず閉じる。閉じ忘れるとファイルディスクリプタが枯渇する。

---

## 処理の流れ

```
main(argc, argv)
 ├─ 引数チェック（argc < 2 → fprintf + exit(1)）
 └─ for i = 1..argc-1
     └─ do_ls(argv[i])
         ├─ opendir(path)          // ディレクトリストリームを開く
         │   └─ 失敗 → perror + exit(1)
         ├─ while readdir(d) != NULL
         │   └─ printf(d_name)    // エントリ名を1行出力
         └─ closedir(d)            // ストリームを閉じる
```

---

## 関数・システムコール一覧

| 関数（またはシステムコール） | シグネチャ | 説明 |
|------------------------------|-----------|------|
| `opendir` | `DIR *opendir(const char *name)` | ディレクトリストリームを開く。失敗時は `NULL` |
| `readdir` | `struct dirent *readdir(DIR *dirp)` | 次のディレクトリエントリを返す。末尾で `NULL` |
| `closedir` | `int closedir(DIR *dirp)` | ディレクトリストリームを閉じてリソースを解放する。エラー時は `-1` |
| `perror` | `void perror(const char *s)` | `errno` に対応するエラーメッセージを標準エラー出力に出力する |
| `fprintf` | `int fprintf(FILE *stream, const char *format, ...)` | 指定ストリームへフォーマット出力する。エラー時は負値 |
| `printf` | `int printf(const char *format, ...)` | 標準出力へフォーマット出力する。エラー時は負値 |
| `exit` | `void exit(int status)` | プログラムを終了する。`0` で正常終了、非ゼロで異常終了 |
