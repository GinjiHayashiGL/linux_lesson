# stat.c 解説

ファイルのメタデータ（inode 情報）を表示する `stat` コマンドの簡易実装。シンボリックリンク自体の情報を取得するために `lstat` を使っている。

## 概要

```
./stat <ファイル名>
```

指定したファイルの種別・パーミッション・デバイス番号・inode 番号・リンク数・所有者・サイズ・タイムスタンプなどを標準出力に出力する。

---

## インクルードファイル

| ヘッダ | 概要 | 用途 |
|--------|------|------|
| `stdio.h` | 標準入出力ライブラリ | `printf`、`fprintf`、`perror` |
| `stdlib.h` | 標準ユーティリティライブラリ | `exit`（プログラム終了） |
| `sys/types.h` | POSIX 型定義 | `mode_t`、`dev_t`、`ino_t` などの基盤型 |
| `sys/stat.h` | ファイル状態 API | `struct stat`、`lstat`、`S_IFMT`、`S_IS*` マクロ |
| `time.h` | 時刻処理ライブラリ | `ctime`（`time_t` を文字列に変換） |

---

## 関数

### `main`

```c
int main(int argc, char *argv[])
```

```c
struct stat st;
```
- `struct stat` はカーネルから取得したファイルのメタデータをすべて格納する構造体。`lstat` が内部を埋める。

```c
if (argc != 2) {
    fprintf(stderr, "wrong argument\n");
    exit(1);
}
```
- 引数がちょうど 1 つでなければエラーを標準エラー出力に出して終了する。
- `fprintf(FILE *stream, const char *format, ...)` — フォーマット文字列を `stream` に書き出す。`stderr` へ出力することでリダイレクト時もエラーメッセージが端末に残る。エラー時は負値を返す。

```c
if (lstat(argv[1], &st) < 0) {
    perror(argv[1]);
    exit(1);
}
```
- `lstat(const char *pathname, struct stat *statbuf)` — `pathname` のメタデータを取得して `statbuf` に格納する。成功時は `0`、失敗時は `-1` を返し `errno` をセットする。
  - `stat` との違い: `pathname` がシンボリックリンクの場合、`stat` はリンク先を辿るが `lstat` はリンク自体の情報を返す。シンボリックリンクの種別やサイズを正しく取得するために `lstat` を選んでいる。
- `perror(const char *s)` — `s` の後ろにコロンを付け、`errno` に対応するシステムエラーメッセージを標準エラー出力に書き出す。

```c
printf("type\t%o (%s)\n", (st.st_mode & S_IFMT), filetype(st.st_mode));
printf("mode\t%o\n", st.st_mode & ~S_IFMT);
```
- `st.st_mode` は種別ビットとパーミッションビットを 1 つの `mode_t` 値に詰め込んでいる。
- `S_IFMT` はファイル種別を表す上位ビットのマスク（値 `0170000`）。`& S_IFMT` で種別ビットだけを取り出す。
- `& ~S_IFMT` は種別ビットを除いたパーミッションビット（`rwxrwxrwx` に相当する下位 12 ビット）を取り出す。
- 両方とも `%o` で 8 進数表示する。Unix のパーミッションは伝統的に 8 進数で表現される。

```c
printf("dev\t%llu\n", (unsigned long long)st.st_dev);
printf("ino\t%lu\n", (unsigned long)st.st_ino);
printf("rdev\t%llu\n", (unsigned long long)st.st_rdev);
printf("nlink\t%lu\n", (unsigned long)st.st_nlink);
printf("uid\t%d\n", st.st_uid);
printf("gid\t%d\n", st.st_gid);
printf("size\t%ld\n", st.st_size);
printf("blksize\t%lu\n", (unsigned long)st.st_blksize);
printf("blocks\t%lu\n", (unsigned long)st.st_blocks);
```
- `st.st_dev` — ファイルが存在するデバイスの ID。
- `st.st_ino` — inode 番号。同一ファイルシステム内でファイルを一意に識別する。
- `st.st_rdev` — デバイスファイルの場合のデバイス ID。通常ファイルでは `0`。
- `st.st_nlink` — ハードリンク数。`unlink` を呼ぶたびに減少し、`0` になるとデータが解放される。
- `st.st_uid` / `st.st_gid` — 所有ユーザー ID / グループ ID。
- `st.st_size` — バイト単位のファイルサイズ。シンボリックリンクの場合はリンク先パスの文字数。
- `st.st_blksize` — 効率的な I/O のために推奨されるブロックサイズ（ファイルシステムが設定）。
- `st.st_blocks` — 実際に割り当てられた 512 バイトブロック数。スパースファイルでは `st_size / 512` より少なくなる。
- `st_dev` と `st_rdev` は `dev_t` 型の幅が環境依存なため `unsigned long long` にキャストして `%llu` で出力し、切り捨てを防いでいる。同様に `st_ino`・`st_nlink`・`st_blksize`・`st_blocks` は `unsigned long` にキャストして `%lu` で出力する。

```c
printf("atime\t%s", ctime(&st.st_atime));
printf("mtime\t%s", ctime(&st.st_mtime));
printf("ctime\t%s", ctime(&st.st_ctime));
```
- `ctime(const time_t *timep)` — `time_t` を `"Www Mmm DD HH:MM:SS YYYY\n"` 形式の文字列ポインタに変換する。末尾に `\n` が含まれるため `printf` のフォーマット文字列には `\n` を入れていない。スレッドセーフではないが、本実装はシングルスレッドなので問題ない。
- `st.st_atime` — 最終アクセス時刻（`read` などで更新）。
- `st.st_mtime` — 最終内容変更時刻（`write` などで更新）。
- `st.st_ctime` — 最終状態変更時刻（パーミッション変更や `rename` などで更新）。内容変更時も更新される。

```c
exit(0);
```
- すべての情報の出力に成功したらステータス `0` で終了する。

---

### `filetype`

```c
static char *filetype(mode_t mode)
```
- `static` 修飾子によりこの翻訳単位（ファイル）内限定の関数とし、外部へのシンボル漏れを防ぐ。
- `mode_t mode` は `st.st_mode` をそのまま受け取る。

```c
if(S_ISREG(mode))  return "file";
if(S_ISDIR(mode))  return "directory";
if(S_ISCHR(mode))  return "chardev";
if(S_ISBLK(mode))  return "blockdev";
if(S_ISFIFO(mode)) return "fifo";
if(S_ISLNK(mode))  return "symlink";
if(S_ISSOCK(mode)) return "socket";
return "unknown";
```
- `S_IS*(mode_t m)` は `<sys/stat.h>` が提供するマクロ群で、`m & S_IFMT` の値を特定の種別定数と比較して `1` または `0` を返す。
- 各マクロが対応する種別:

| マクロ | 種別 | `S_IFMT` との比較値 |
|--------|------|---------------------|
| `S_ISREG` | 通常ファイル | `S_IFREG` (`0100000`) |
| `S_ISDIR` | ディレクトリ | `S_IFDIR` (`0040000`) |
| `S_ISCHR` | キャラクタデバイス | `S_IFCHR` (`0020000`) |
| `S_ISBLK` | ブロックデバイス | `S_IFBLK` (`0060000`) |
| `S_ISFIFO` | 名前付きパイプ | `S_IFIFO` (`0010000`) |
| `S_ISLNK` | シンボリックリンク | `S_IFLNK` (`0120000`) |
| `S_ISSOCK` | ソケット | `S_IFSOCK` (`0140000`) |

- どのマクロにも一致しない場合は `"unknown"` を返す。将来的な拡張やカーネルが新種別を追加した場合のフォールバック。

---

## 処理の流れ

```
main
 ├─ 引数チェック（argc != 2）
 │   └─ エラー時: fprintf(stderr, ...) → exit(1)
 ├─ lstat(argv[1], &st)         // inode 情報取得（リンク自体を対象）
 │   └─ 失敗時: perror(argv[1]) → exit(1)
 ├─ printf("type\t%o (%s)\n", st.st_mode & S_IFMT, filetype(st.st_mode))
 │   └─ filetype(mode)          // S_IS* マクロで種別文字列を返す
 ├─ printf("mode\t%o\n", st.st_mode & ~S_IFMT)
 ├─ printf(dev, ino, rdev, nlink, uid, gid, size, blksize, blocks)
 ├─ printf("atime\t%s", ctime(&st.st_atime))
 ├─ printf("mtime\t%s", ctime(&st.st_mtime))
 ├─ printf("ctime\t%s", ctime(&st.st_ctime))
 └─ exit(0)
```

---

## 関数・システムコール一覧

| 関数（またはシステムコール） | シグネチャ | 説明 |
|------------------------------|-----------|------|
| `lstat` | `int lstat(const char *pathname, struct stat *statbuf)` | ファイルのメタデータを取得する。シンボリックリンクはリンク自体の情報を返す。成功時 `0`、失敗時 `-1` |
| `ctime` | `char *ctime(const time_t *timep)` | `time_t` を人間可読な文字列（末尾 `\n` 含む）に変換する。スレッドアンセーフ |
| `S_ISREG` 等 | `int S_IS*(mode_t m)` | `mode_t` のファイル種別ビットを判定するマクロ群。一致で `1`、不一致で `0` |
| `printf` | `int printf(const char *format, ...)` | フォーマット文字列を標準出力に書き出す。エラー時は負値を返す |
| `fprintf` | `int fprintf(FILE *stream, const char *format, ...)` | フォーマット文字列を指定ストリームに書き出す。エラー時は負値を返す |
| `perror` | `void perror(const char *s)` | `s` に続けて `errno` に対応するエラーメッセージを標準エラー出力に出力する |
| `exit` | `void exit(int status)` | プログラムを終了する。`0` で正常終了、非ゼロで異常終了 |
