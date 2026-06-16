# symlink.c 解説

シンボリックリンクを作成する `ln -s` 相当のコマンド実装。

## 概要

```
./symlink <リンク先パス> <リンク名>
```

`argv[1]` を指すシンボリックリンクを `argv[2]` という名前で作成する。

---

## インクルードファイル

| ヘッダ | 概要 | 用途 |
|--------|------|------|
| `stdio.h` | 標準入出力ライブラリ | `fprintf`、`perror` |
| `stdlib.h` | 標準ユーティリティライブラリ | `exit`（プログラム終了） |
| `unistd.h` | POSIX API | `symlink`（シンボリックリンク作成システムコール） |

---

## 関数

### `main`

```c
int main(int argc, char *argv[])
```

```c
if (argc != 3) {
    fprintf(stderr, "%s: wrong number of arguments\n", argv[0]);
    exit(1);
}
```
- 引数がちょうど 2 つ（リンク先・リンク名）でなければエラーを標準エラー出力に出して終了する。
- `fprintf(FILE *stream, const char *format, ...)` — フォーマット文字列を `stream` に書き出す。`stderr` に出力することでリダイレクト時にエラーメッセージが端末に残る。エラー時は負値を返す。
- `argv[0]` をプレフィックスにすることで、どのコマンドが失敗したかをユーザーが把握しやすくなる。

```c
if (symlink(argv[1], argv[2]) < 0) {
    perror(argv[1]);
    exit(1);
}
```
- `symlink(const char *target, const char *linkpath)` — `linkpath` という名前のシンボリックリンクを作成し、その内容に `target` を格納する。成功時は `0`、失敗時は `-1` を返し `errno` をセットする。
  - `target` は実在しなくても作成できる（ダングリングリンク）。
  - `linkpath` が既に存在する場合は失敗する（`errno = EEXIST`）。
- 失敗時は `perror` でエラーメッセージを出力して終了する。
- `perror(const char *s)` — `s` の後ろにコロンを付け、`errno` に対応するシステムエラーメッセージを標準エラー出力に書き出す。エラー原因を人間可読な形で伝えるために使う。
- エラー判定に `< 0` を使うのは、`symlink` の失敗戻り値が `-1` であることへの明示的な意図。

```c
exit(0);
```
- シンボリックリンクの作成に成功したらステータス `0` で終了する。

---

## 処理の流れ

```
main
 ├─ 引数チェック（argc != 3）
 │   └─ エラー時: fprintf(stderr, ...) → exit(1)
 ├─ symlink(argv[1], argv[2])   // シンボリックリンク作成
 │   └─ 失敗時: perror(argv[1]) → exit(1)
 └─ exit(0)
```

---

## 関数・システムコール一覧

| 関数（またはシステムコール） | シグネチャ | 説明 |
|------------------------------|-----------|------|
| `symlink` | `int symlink(const char *target, const char *linkpath)` | `linkpath` を名前とするシンボリックリンクを作成し、`target` を参照先にする。成功時 `0`、失敗時 `-1` |
| `fprintf` | `int fprintf(FILE *stream, const char *format, ...)` | フォーマット文字列を指定ストリームに書き出す。エラー時は負値を返す |
| `perror` | `void perror(const char *s)` | `s` に続けて `errno` に対応するエラーメッセージを標準エラー出力に出力する |
| `exit` | `void exit(int status)` | プログラムを終了する。`0` で正常終了、非ゼロで異常終了 |
