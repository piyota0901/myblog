---
title: "システムコール"
date: 2025-03-29T16:25:54+09:00
draft: false
description: ""
categories: ["OS"]
---

システムコールについて勉強したのでメモ ✍️

## システムコールとは？

---

アプリケーションが OS の提供するリソースを利用するための仕組みのこと。  
ファイル、ネットワーク、メモリなど、OS が管理するリソースを利用する際に、アプリケーションはシステムコールを呼び出す。

**イメージ図**

![](./overview.svg)

**OS が管理するリソースと利用シーン**

| リソース               | 利用シーンの例                                                                         | 使用される主なシステムコール                       |
| ---------------------- | -------------------------------------------------------------------------------------- | -------------------------------------------------- |
| **ファイル**           | ログファイルにデータを書き込む                                                         | `open()`, `write()`, `read()`, `close()`           |
| **ネットワーク**       | Web API に HTTP リクエストを送る                                                       | `socket()`, `connect()`, `send()`, `recv()`        |
| **プロセス・スレッド** | 子プロセスを作成して並列処理を行う（例: Web サーバーでリクエストごとに子プロセス生成） | `fork()`, `execve()`, `wait()`, `clone()`          |
| **メモリ**             | 動的にメモリを確保して大きなデータ構造を扱う                                           | `mmap()`, `brk()`, `sbrk()`                        |
| **デバイス**           | `/dev/ttyUSB0` にデータを書き込んでシリアル通信する                                    | `open()`, `write()`, `ioctl()`                     |
| **時間**               | 一定時間スリープして、定期的に処理を実行する                                           | `nanosleep()`, `gettimeofday()`, `clock_gettime()` |

※[Linux のシステムコール一覧](https://man7.org/linux/man-pages/dir_section_2.html)

## システムコールの目的

---

大きく以下の２点が目的になる

- **OS リソースのインターフェース（抽象化）の提供**

  抽象化されたインターフェースを利用するだけで済むため、アプリケーションは OS の内部実装を知る必要がない。

- **セキュリティと安全性の確保**

## システムコールの詳細

---

システムコールは、アプリケーションが OS のリソースを利用するために呼び出すが、CPU の内部では**ユーザーモードからカーネルモードに切り替える**という重要な操作が行われている。また、CPU はモードに応じてアクセスできるメモリ領域が制限される。

| モード         | アクセス可能な空間          | 主な動作主体                         |
| -------------- | --------------------------- | ------------------------------------ |
| ユーザーモード | ユーザー空間のみ            | アプリケーションなどの一般プログラム |
| カーネルモード | ユーザー空間 + カーネル空間 | OS（カーネル）本体                   |

### ユーザーモードとカーネルモード

通常アプリケーション（プログラム）は`ユーザーモード`という制限されたモードで動作している。このモードでは、OS の重要な機能に直接アクセスすることはできない。
一方で OS（カーネル）は、`カーネルモード`と呼ばれる特権モードで動作しており、すべてのリソースにアクセスすることができる。

システムコールは、このモードの切り替えを行い、アプリケーションが OS の機能の一部を利用できるようにする仕組み。

### ユーザー空間とカーネル空間

OS は仮想メモリを使って、メモリ空間を「ユーザー空間」と「カーネル空間」にあらかじめ分離しており、CPU のモードに応じてアクセスできる範囲を制限している。

- **ユーザー空間**

  アプリケーションが使用できる領域。プログラムのコード、変数、ヒープ、スタックなどが配置される。

- **カーネル空間**

  OS（カーネル）が使用する領域。デバイス制御やプロセス管理などの内部処理が行われる。

この分離により、アプリケーションが OS や他のアプリのメモリに勝手にアクセスすることを防ぎ、セキュリティと安定性が実現されている。

### これまでのまとめ

例えば、アプリケーションがファイルからデータを読み取る際の流れは以下のようになる

1. アプリケーションがシステムコール（`read()`）を呼び出す（ユーザーモード）
1. システムコールによりカーネルモードへ切り替わる
1. カーネル空間でファイルの内容を読み込む
1. 読み込んだデータをユーザー空間のバッファにコピー
1. 処理が完了すると、再びユーザーモードに戻る

![](./usermode_kernelmodel.svg)

カーネル空間からユーザー空間へのコピー処理は、I/O が遅くなる原因の１つ。  
`Zero-copy`と呼ばれるカーネル空間、ユーザー空間のコピーを省略して直接やり取りする技術がある。代表的な方法としては、`mmap()`(カーネルのメモリ空間をユーザー空間にマッピング)、`sendfile()`（ファイル ⇒ ソケットへコピーなしで転送）などがある。

## 実際の環境でシステムコールを確認する

---

`strace`コマンドを利用するため、インストールする。

```bash
$ sudo apt install strace
```

今回の確認では、`uv`は利用しない。直接 python を利用する。理由としては、`uv`経由で Python を実行する場合、`uv`の中で python が別プロセスとして実行され、`strace`だと`uv`プロセスしか追うことができないためである。`-f`オプションで子プロセスも追えるようにできるが大量の出力が表示されてしまうので今回は使わない。

### 単純な演算処理を実行する

演算処理のみであれば、ユーザーモードで実行されるためシステムコールが発生しないことを確認する

<details>
<summary>コード</summary>

```python
def compute():
    """演算処理のみを行う関数
    """
    total = 0
    for i in range(1_000_000):
        total += i
    return total

if __name__ == "__main__":
    compute()
```

</details>

<details>
<summary>実行結果</summary>

```bash
$ strace python3 main.py
execve("/usr/bin/python3", ["python3", "main.py"], 0x7ffd966dffa8 /* 38 vars */) = 0
brk(NULL)                               = 0x17825000
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f5fb96bd000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=19983, ...}) = 0
mmap(NULL, 19983, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f5fb96b8000
close(3)                                = 0
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libm.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 832) = 832
fstat(3, {st_mode=S_IFREG|0644, st_size=952616, ...}) = 0
mmap(NULL, 950296, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f5fb95cf000
mmap(0x7f5fb95df000, 520192, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x10000) = 0x7f5fb95df000
mmap(0x7f5fb965e000, 360448, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x8f000) = 0x7f5fb965e000
mmap(0x7f5fb96b6000, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0xe7000) = 0x7f5fb96b6000
close(3)                                = 0
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libz.so.1", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 832) = 832
fstat(3, {st_mode=S_IFREG|0644, st_size=113000, ...}) = 0
mmap(NULL, 110744, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f5fb95b3000
mmap(0x7f5fb95b5000, 73728, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x2000) = 0x7f5fb95b5000
mmap(0x7f5fb95c7000, 24576, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x14000) = 0x7f5fb95c7000
mmap(0x7f5fb95cd000, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1a000) = 0x7f5fb95cd000
close(3)                                = 0
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libexpat.so.1", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 832) = 832
fstat(3, {st_mode=S_IFREG|0644, st_size=170240, ...}) = 0
mmap(NULL, 172160, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f5fb9588000
mmap(0x7f5fb958c000, 114688, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x4000) = 0x7f5fb958c000
mmap(0x7f5fb95a8000, 32768, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x20000) = 0x7f5fb95a8000
mmap(0x7f5fb95b0000, 12288, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x27000) = 0x7f5fb95b0000
close(3)                                = 0
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\220\243\2\0\0\0\0\0"..., 832) = 832
pread64(3, "\6\0\0\0\4\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0"..., 784, 64) = 784
fstat(3, {st_mode=S_IFREG|0755, st_size=2125328, ...}) = 0
pread64(3, "\6\0\0\0\4\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0"..., 784, 64) = 784
mmap(NULL, 2170256, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f5fb9376000
mmap(0x7f5fb939e000, 1605632, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x28000) = 0x7f5fb939e000
mmap(0x7f5fb9526000, 323584, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1b0000) = 0x7f5fb9526000
mmap(0x7f5fb9575000, 24576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1fe000) = 0x7f5fb9575000
mmap(0x7f5fb957b000, 52624, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7f5fb957b000
close(3)                                = 0
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f5fb9374000
arch_prctl(ARCH_SET_FS, 0x7f5fb9375080) = 0
set_tid_address(0x7f5fb9375350)         = 133068
set_robust_list(0x7f5fb9375360, 24)     = 0
rseq(0x7f5fb93759a0, 0x20, 0, 0x53053053) = 0
mprotect(0x7f5fb9575000, 16384, PROT_READ) = 0
mprotect(0x7f5fb95b0000, 8192, PROT_READ) = 0
mprotect(0x7f5fb95cd000, 4096, PROT_READ) = 0
mprotect(0x7f5fb96b6000, 4096, PROT_READ) = 0
mprotect(0xa27000, 4096, PROT_READ)     = 0
mprotect(0x7f5fb96f5000, 8192, PROT_READ) = 0
prlimit64(0, RLIMIT_STACK, NULL, {rlim_cur=8192*1024, rlim_max=RLIM64_INFINITY}) = 0
munmap(0x7f5fb96b8000, 19983)           = 0
getrandom("\x6b\x31\xf4\xf5\x62\x9d\x66\x1b", 8, GRND_NONBLOCK) = 8
brk(NULL)                               = 0x17825000
brk(0x17846000)                         = 0x17846000
openat(AT_FDCWD, "/usr/lib/locale/locale-archive", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/share/locale/locale.alias", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=2996, ...}) = 0
read(3, "# Locale name alias data base.\n#"..., 4096) = 2996
read(3, "", 4096)                       = 0
close(3)                                = 0
openat(AT_FDCWD, "/usr/lib/locale/C.UTF-8/LC_CTYPE", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/lib/locale/C.utf8/LC_CTYPE", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=360460, ...}) = 0
mmap(NULL, 360460, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f5fb931b000
close(3)                                = 0
openat(AT_FDCWD, "/usr/lib/x86_64-linux-gnu/gconv/gconv-modules.cache", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=27028, ...}) = 0
mmap(NULL, 27028, PROT_READ, MAP_SHARED, 3, 0) = 0x7f5fb9314000
close(3)                                = 0
futex(0x7f5fb957a72c, FUTEX_WAKE_PRIVATE, 2147483647) = 0
getcwd("/home/tatsuro/workspaces/myblog/tmp", 4096) = 36
getrandom("\xcc\xb9\x88\x4a\xc3\x62\x3e\x84\x95\xec\x62\x29\x01\xf4\xbb\x3a\x99\x9a\xa4\xba\x6f\x54\x75\xdd", 24, GRND_NONBLOCK) = 24
gettid()                                = 133068
mmap(NULL, 1048576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f5fb9214000
mmap(NULL, 266240, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f5fb91d3000
mmap(NULL, 135168, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f5fb91b2000
brk(0x17867000)                         = 0x17867000
mmap(NULL, 16384, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f5fb96b9000
brk(0x17888000)                         = 0x17888000
newfstatat(AT_FDCWD, "/home/tatsuro/.bun/bin/python3", 0x7ffed0a05040, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/home/tatsuro/.local/bin/python3", 0x7ffed0a05040, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/home/tatsuro/.volta/bin/python3", 0x7ffed0a05040, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/home/tatsuro/.vscode-server/bin/ddc367ed5c8936efe395cffeec279b04ffd7db78/bin/remote-cli/python3", 0x7ffed0a05040, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/home/tatsuro/.bun/bin/python3", 0x7ffed0a05040, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/home/tatsuro/.local/bin/python3", 0x7ffed0a05040, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/home/tatsuro/.volta/bin/python3", 0x7ffed0a05040, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/home/tatsuro/.volta/bin/python3", 0x7ffed0a05040, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/home/tatsuro/.cargo/bin/python3", 0x7ffed0a05040, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/usr/local/sbin/python3", 0x7ffed0a05040, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/usr/local/bin/python3", 0x7ffed0a05040, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/usr/sbin/python3", 0x7ffed0a05040, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/usr/bin/python3", {st_mode=S_IFREG|0755, st_size=8019136, ...}, 0) = 0
openat(AT_FDCWD, "/usr/pyvenv.cfg", O_RDONLY) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/bin/pyvenv.cfg", O_RDONLY) = -1 ENOENT (No such file or directory)
readlink("/usr/bin/python3", "python3.12", 4096) = 10
readlink("/usr/bin/python3.12", 0x7ffed0a00060, 4096) = -1 EINVAL (Invalid argument)
openat(AT_FDCWD, "/usr/bin/python3._pth", O_RDONLY) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/bin/python3.12._pth", O_RDONLY) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/bin/pybuilddir.txt", O_RDONLY) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/usr/bin/Modules/Setup.local", 0x7ffed0a05040, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/usr/bin/lib/python312.zip", 0x7ffed0a04e00, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/usr/lib/python312.zip", 0x7ffed0a04e60, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/usr/bin/lib/python3.12/os.py", 0x7ffed0a04e60, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/usr/bin/lib/python3.12/os.pyc", 0x7ffed0a04e60, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/usr/lib/python3.12/os.py", {st_mode=S_IFREG|0644, st_size=39786, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/bin/lib/python3.12/lib-dynload", 0x7ffed0a04e60, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/usr/lib/python3.12/lib-dynload", {st_mode=S_IFDIR|0755, st_size=12288, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/lib-dynload", {st_mode=S_IFDIR|0755, st_size=12288, ...}, 0) = 0
openat(AT_FDCWD, "/etc/localtime", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=309, ...}) = 0
fstat(3, {st_mode=S_IFREG|0644, st_size=309, ...}) = 0
brk(0x178a9000)                         = 0x178a9000
read(3, "TZif2\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\4\0\0\0\4\0\0\0\0"..., 4096) = 309
lseek(3, -176, SEEK_CUR)                = 133
read(3, "TZif2\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\4\0\0\0\4\0\0\0\0"..., 4096) = 176
close(3)                                = 0
brk(0x178a8000)                         = 0x178a8000
newfstatat(AT_FDCWD, "/usr/lib/python312.zip", 0x7ffed0a04890, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/usr/lib", {st_mode=S_IFDIR|0755, st_size=4096, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python312.zip", 0x7ffed0a04c10, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/usr/lib/python3.12", {st_mode=S_IFDIR|0755, st_size=20480, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12", {st_mode=S_IFDIR|0755, st_size=20480, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12", {st_mode=S_IFDIR|0755, st_size=20480, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/python3.12", O_RDONLY|O_NONBLOCK|O_CLOEXEC|O_DIRECTORY) = 3
fstat(3, {st_mode=S_IFDIR|0755, st_size=20480, ...}) = 0
getdents64(3, 0x178897b0 /* 202 entries */, 32768) = 6752
getdents64(3, 0x178897b0 /* 0 entries */, 32768) = 0
close(3)                                = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/encodings/__init__.cpython-312-x86_64-linux-gnu.so", 0x7ffed0a04c10, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/usr/lib/python3.12/encodings/__init__.abi3.so", 0x7ffed0a04c10, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/usr/lib/python3.12/encodings/__init__.so", 0x7ffed0a04c10, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/usr/lib/python3.12/encodings/__init__.py", {st_mode=S_IFREG|0644, st_size=5884, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/encodings/__init__.py", {st_mode=S_IFREG|0644, st_size=5884, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/python3.12/encodings/__pycache__/__init__.cpython-312.pyc", O_RDONLY|O_CLOEXEC) = 3
fcntl(3, F_GETFD)                       = 0x1 (flags FD_CLOEXEC)
fstat(3, {st_mode=S_IFREG|0644, st_size=5790, ...}) = 0
ioctl(3, TCGETS, 0x7ffed0a04850)        = -1 ENOTTY (Inappropriate ioctl for device)
lseek(3, 0, SEEK_CUR)                   = 0
lseek(3, 0, SEEK_CUR)                   = 0
fstat(3, {st_mode=S_IFREG|0644, st_size=5790, ...}) = 0
read(3, "\313\r\r\n\0\0\0\0\303(\242g\374\26\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\6\0\0"..., 5791) = 5790
read(3, "", 1)                          = 0
close(3)                                = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/encodings", {st_mode=S_IFDIR|0755, st_size=20480, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/encodings", {st_mode=S_IFDIR|0755, st_size=20480, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/encodings", {st_mode=S_IFDIR|0755, st_size=20480, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/python3.12/encodings", O_RDONLY|O_NONBLOCK|O_CLOEXEC|O_DIRECTORY) = 3
fstat(3, {st_mode=S_IFDIR|0755, st_size=20480, ...}) = 0
getdents64(3, 0x17892d20 /* 125 entries */, 32768) = 4224
getdents64(3, 0x17892d20 /* 0 entries */, 32768) = 0
close(3)                                = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/encodings/aliases.py", {st_mode=S_IFREG|0644, st_size=15677, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/encodings/aliases.py", {st_mode=S_IFREG|0644, st_size=15677, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/python3.12/encodings/__pycache__/aliases.cpython-312.pyc", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=12408, ...}) = 0
ioctl(3, TCGETS, 0x7ffed0a03dd0)        = -1 ENOTTY (Inappropriate ioctl for device)
lseek(3, 0, SEEK_CUR)                   = 0
lseek(3, 0, SEEK_CUR)                   = 0
fstat(3, {st_mode=S_IFREG|0644, st_size=12408, ...}) = 0
read(3, "\313\r\r\n\0\0\0\0\303(\242g==\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\5\0\0"..., 12409) = 12408
read(3, "", 1)                          = 0
close(3)                                = 0
brk(0x178c9000)                         = 0x178c9000
newfstatat(AT_FDCWD, "/usr/lib/python3.12/encodings", {st_mode=S_IFDIR|0755, st_size=20480, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/encodings/utf_8.py", {st_mode=S_IFREG|0644, st_size=1005, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/encodings/utf_8.py", {st_mode=S_IFREG|0644, st_size=1005, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/python3.12/encodings/__pycache__/utf_8.cpython-312.pyc", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=2147, ...}) = 0
ioctl(3, TCGETS, 0x7ffed0a048e0)        = -1 ENOTTY (Inappropriate ioctl for device)
lseek(3, 0, SEEK_CUR)                   = 0
lseek(3, 0, SEEK_CUR)                   = 0
fstat(3, {st_mode=S_IFREG|0644, st_size=2147, ...}) = 0
read(3, "\313\r\r\n\0\0\0\0\303(\242g\355\3\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\5\0\0"..., 2148) = 2147
read(3, "", 1)                          = 0
close(3)                                = 0
rt_sigaction(SIGPIPE, {sa_handler=SIG_IGN, sa_mask=[], sa_flags=SA_RESTORER|SA_ONSTACK, sa_restorer=0x7f5fb93bb330}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGXFSZ, {sa_handler=SIG_IGN, sa_mask=[], sa_flags=SA_RESTORER|SA_ONSTACK, sa_restorer=0x7f5fb93bb330}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGHUP, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGINT, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGQUIT, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGILL, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGTRAP, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGABRT, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGBUS, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGFPE, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGKILL, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGUSR1, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGSEGV, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGUSR2, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGPIPE, NULL, {sa_handler=SIG_IGN, sa_mask=[], sa_flags=SA_RESTORER|SA_ONSTACK, sa_restorer=0x7f5fb93bb330}, 8) = 0
rt_sigaction(SIGALRM, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGTERM, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGSTKFLT, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGCHLD, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGCONT, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGSTOP, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGTSTP, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGTTIN, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGTTOU, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGURG, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGXCPU, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGXFSZ, NULL, {sa_handler=SIG_IGN, sa_mask=[], sa_flags=SA_RESTORER|SA_ONSTACK, sa_restorer=0x7f5fb93bb330}, 8) = 0
rt_sigaction(SIGVTALRM, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGPROF, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGWINCH, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGIO, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGPWR, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGSYS, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGRT_2, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGRT_3, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGRT_4, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGRT_5, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGRT_6, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGRT_7, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGRT_8, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGRT_9, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGRT_10, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGRT_11, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGRT_12, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGRT_13, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGRT_14, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGRT_15, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGRT_16, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGRT_17, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGRT_18, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGRT_19, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGRT_20, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGRT_21, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGRT_22, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGRT_23, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGRT_24, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGRT_25, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGRT_26, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGRT_27, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGRT_28, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGRT_29, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGRT_30, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGRT_31, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGRT_32, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGINT, {sa_handler=0x6e72f0, sa_mask=[], sa_flags=SA_RESTORER|SA_ONSTACK, sa_restorer=0x7f5fb93bb330}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
fstat(0, {st_mode=S_IFCHR|0620, st_rdev=makedev(0x88, 0xf), ...}) = 0
fcntl(0, F_GETFD)                       = 0
fstat(0, {st_mode=S_IFCHR|0620, st_rdev=makedev(0x88, 0xf), ...}) = 0
ioctl(0, TCGETS, {c_iflag=BRKINT|ICRNL|IXON|IXANY|IMAXBEL|IUTF8, c_oflag=NL0|CR0|TAB0|BS0|VT0|FF0|OPOST|ONLCR, c_cflag=B38400|CS8|CREAD|HUPCL, c_lflag=ISIG|ICANON|ECHO|ECHOE|ECHOK|IEXTEN|ECHOCTL|ECHOKE, ...}) = 0
lseek(0, 0, SEEK_CUR)                   = -1 ESPIPE (Illegal seek)
ioctl(0, TCGETS, {c_iflag=BRKINT|ICRNL|IXON|IXANY|IMAXBEL|IUTF8, c_oflag=NL0|CR0|TAB0|BS0|VT0|FF0|OPOST|ONLCR, c_cflag=B38400|CS8|CREAD|HUPCL, c_lflag=ISIG|ICANON|ECHO|ECHOE|ECHOK|IEXTEN|ECHOCTL|ECHOKE, ...}) = 0
fcntl(1, F_GETFD)                       = 0
fstat(1, {st_mode=S_IFCHR|0620, st_rdev=makedev(0x88, 0xf), ...}) = 0
ioctl(1, TCGETS, {c_iflag=BRKINT|ICRNL|IXON|IXANY|IMAXBEL|IUTF8, c_oflag=NL0|CR0|TAB0|BS0|VT0|FF0|OPOST|ONLCR, c_cflag=B38400|CS8|CREAD|HUPCL, c_lflag=ISIG|ICANON|ECHO|ECHOE|ECHOK|IEXTEN|ECHOCTL|ECHOKE, ...}) = 0
lseek(1, 0, SEEK_CUR)                   = -1 ESPIPE (Illegal seek)
ioctl(1, TCGETS, {c_iflag=BRKINT|ICRNL|IXON|IXANY|IMAXBEL|IUTF8, c_oflag=NL0|CR0|TAB0|BS0|VT0|FF0|OPOST|ONLCR, c_cflag=B38400|CS8|CREAD|HUPCL, c_lflag=ISIG|ICANON|ECHO|ECHOE|ECHOK|IEXTEN|ECHOCTL|ECHOKE, ...}) = 0
fcntl(2, F_GETFD)                       = 0
fstat(2, {st_mode=S_IFCHR|0620, st_rdev=makedev(0x88, 0xf), ...}) = 0
ioctl(2, TCGETS, {c_iflag=BRKINT|ICRNL|IXON|IXANY|IMAXBEL|IUTF8, c_oflag=NL0|CR0|TAB0|BS0|VT0|FF0|OPOST|ONLCR, c_cflag=B38400|CS8|CREAD|HUPCL, c_lflag=ISIG|ICANON|ECHO|ECHOE|ECHOK|IEXTEN|ECHOCTL|ECHOKE, ...}) = 0
lseek(2, 0, SEEK_CUR)                   = -1 ESPIPE (Illegal seek)
ioctl(2, TCGETS, {c_iflag=BRKINT|ICRNL|IXON|IXANY|IMAXBEL|IUTF8, c_oflag=NL0|CR0|TAB0|BS0|VT0|FF0|OPOST|ONLCR, c_cflag=B38400|CS8|CREAD|HUPCL, c_lflag=ISIG|ICANON|ECHO|ECHOE|ECHOK|IEXTEN|ECHOCTL|ECHOKE, ...}) = 0
mmap(NULL, 1048576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f5fb90b2000
newfstatat(AT_FDCWD, "/usr/bin/pyvenv.cfg", 0x7ffed0a04850, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/usr/pyvenv.cfg", 0x7ffed0a048b0, 0) = -1 ENOENT (No such file or directory)
geteuid()                               = 1000
getuid()                                = 1000
getegid()                               = 1000
getgid()                                = 1000
newfstatat(AT_FDCWD, "/home/tatsuro/.local/lib/python3.12/site-packages", 0x7ffed0a04a80, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/usr/local/lib/python3.12/dist-packages", {st_mode=S_IFDIR|0755, st_size=4096, ...}, 0) = 0
openat(AT_FDCWD, "/usr/local/lib/python3.12/dist-packages", O_RDONLY|O_NONBLOCK|O_CLOEXEC|O_DIRECTORY) = 3
fstat(3, {st_mode=S_IFDIR|0755, st_size=4096, ...}) = 0
getdents64(3, 0x178b5a60 /* 2 entries */, 32768) = 48
getdents64(3, 0x178b5a60 /* 0 entries */, 32768) = 0
close(3)                                = 0
newfstatat(AT_FDCWD, "/usr/lib/python3/dist-packages", {st_mode=S_IFDIR|0755, st_size=12288, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/python3/dist-packages", O_RDONLY|O_NONBLOCK|O_CLOEXEC|O_DIRECTORY) = 3
fstat(3, {st_mode=S_IFDIR|0755, st_size=12288, ...}) = 0
getdents64(3, 0x178b5a60 /* 155 entries */, 32768) = 6376
getdents64(3, 0x178b5a60 /* 0 entries */, 32768) = 0
close(3)                                = 0
newfstatat(AT_FDCWD, "/usr/lib/python3/dist-packages/distutils-precedence.pth", {st_mode=S_IFREG|0644, st_size=151, ...}, AT_SYMLINK_NOFOLLOW) = 0
openat(AT_FDCWD, "/usr/lib/python3/dist-packages/distutils-precedence.pth", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=151, ...}) = 0
ioctl(3, TCGETS, 0x7ffed0a04720)        = -1 ENOTTY (Inappropriate ioctl for device)
lseek(3, 0, SEEK_CUR)                   = 0
read(3, "import os; var = 'SETUPTOOLS_USE"..., 8192) = 151
newfstatat(AT_FDCWD, "/usr/lib/python3.12", {st_mode=S_IFDIR|0755, st_size=20480, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/lib-dynload", {st_mode=S_IFDIR|0755, st_size=12288, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/lib-dynload", {st_mode=S_IFDIR|0755, st_size=12288, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/lib-dynload", {st_mode=S_IFDIR|0755, st_size=12288, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/python3.12/lib-dynload", O_RDONLY|O_NONBLOCK|O_CLOEXEC|O_DIRECTORY) = 4
fstat(4, {st_mode=S_IFDIR|0755, st_size=12288, ...}) = 0
getdents64(4, 0x178bf190 /* 49 entries */, 32768) = 3104
getdents64(4, 0x178bf190 /* 0 entries */, 32768) = 0
close(4)                                = 0
newfstatat(AT_FDCWD, "/usr/local/lib/python3.12/dist-packages", {st_mode=S_IFDIR|0755, st_size=4096, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/local/lib/python3.12/dist-packages", {st_mode=S_IFDIR|0755, st_size=4096, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/local/lib/python3.12/dist-packages", {st_mode=S_IFDIR|0755, st_size=4096, ...}, 0) = 0
openat(AT_FDCWD, "/usr/local/lib/python3.12/dist-packages", O_RDONLY|O_NONBLOCK|O_CLOEXEC|O_DIRECTORY) = 4
fstat(4, {st_mode=S_IFDIR|0755, st_size=4096, ...}) = 0
getdents64(4, 0x178bf190 /* 2 entries */, 32768) = 48
getdents64(4, 0x178bf190 /* 0 entries */, 32768) = 0
close(4)                                = 0
newfstatat(AT_FDCWD, "/usr/lib/python3/dist-packages", {st_mode=S_IFDIR|0755, st_size=12288, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3/dist-packages", {st_mode=S_IFDIR|0755, st_size=12288, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3/dist-packages", {st_mode=S_IFDIR|0755, st_size=12288, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/python3/dist-packages", O_RDONLY|O_NONBLOCK|O_CLOEXEC|O_DIRECTORY) = 4
fstat(4, {st_mode=S_IFDIR|0755, st_size=12288, ...}) = 0
getdents64(4, 0x178bf190 /* 155 entries */, 32768) = 6376
getdents64(4, 0x178bf190 /* 0 entries */, 32768) = 0
close(4)                                = 0
newfstatat(AT_FDCWD, "/usr/lib/python3/dist-packages/_distutils_hack/__init__.cpython-312-x86_64-linux-gnu.so", 0x7ffed0a04330, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/usr/lib/python3/dist-packages/_distutils_hack/__init__.abi3.so", 0x7ffed0a04330, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/usr/lib/python3/dist-packages/_distutils_hack/__init__.so", 0x7ffed0a04330, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/usr/lib/python3/dist-packages/_distutils_hack/__init__.py", {st_mode=S_IFREG|0644, st_size=6299, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3/dist-packages/_distutils_hack/__init__.py", {st_mode=S_IFREG|0644, st_size=6299, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/python3/dist-packages/_distutils_hack/__pycache__/__init__.cpython-312.pyc", O_RDONLY|O_CLOEXEC) = 4
fstat(4, {st_mode=S_IFREG|0644, st_size=9998, ...}) = 0
ioctl(4, TCGETS, 0x7ffed0a03fd0)        = -1 ENOTTY (Inappropriate ioctl for device)
lseek(4, 0, SEEK_CUR)                   = 0
lseek(4, 0, SEEK_CUR)                   = 0
fstat(4, {st_mode=S_IFREG|0644, st_size=9998, ...}) = 0
read(4, "\313\r\r\n\0\0\0\0\n_\337d\233\30\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\6\0\0"..., 9999) = 9998
read(4, "", 1)                          = 0
close(4)                                = 0
read(3, "", 8192)                       = 0
close(3)                                = 0
newfstatat(AT_FDCWD, "/usr/lib/python3/dist-packages/zope.interface-6.1-nspkg.pth", {st_mode=S_IFREG|0644, st_size=529, ...}, AT_SYMLINK_NOFOLLOW) = 0
openat(AT_FDCWD, "/usr/lib/python3/dist-packages/zope.interface-6.1-nspkg.pth", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=529, ...}) = 0
ioctl(3, TCGETS, 0x7ffed0a04780)        = -1 ENOTTY (Inappropriate ioctl for device)
lseek(3, 0, SEEK_CUR)                   = 0
read(3, "import sys, types, os;has_mfs = "..., 8192) = 529
brk(0x178ea000)                         = 0x178ea000
newfstatat(AT_FDCWD, "/usr/lib/python3.12", {st_mode=S_IFDIR|0755, st_size=20480, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/types.py", {st_mode=S_IFREG|0644, st_size=10993, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/types.py", {st_mode=S_IFREG|0644, st_size=10993, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/python3.12/__pycache__/types.cpython-312.pyc", O_RDONLY|O_CLOEXEC) = 4
fstat(4, {st_mode=S_IFREG|0644, st_size=14939, ...}) = 0
ioctl(4, TCGETS, 0x7ffed0a04100)        = -1 ENOTTY (Inappropriate ioctl for device)
lseek(4, 0, SEEK_CUR)                   = 0
lseek(4, 0, SEEK_CUR)                   = 0
fstat(4, {st_mode=S_IFREG|0644, st_size=14939, ...}) = 0
read(4, "\313\r\r\n\0\0\0\0\303(\242g\361*\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\6\0\0"..., 14940) = 14939
read(4, "", 1)                          = 0
close(4)                                = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12", {st_mode=S_IFDIR|0755, st_size=20480, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/importlib/__init__.cpython-312-x86_64-linux-gnu.so", 0x7ffed0a03f20, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/usr/lib/python3.12/importlib/__init__.abi3.so", 0x7ffed0a03f20, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/usr/lib/python3.12/importlib/__init__.so", 0x7ffed0a03f20, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/usr/lib/python3.12/importlib/__init__.py", {st_mode=S_IFREG|0644, st_size=4774, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/importlib/__init__.py", {st_mode=S_IFREG|0644, st_size=4774, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/python3.12/importlib/__pycache__/__init__.cpython-312.pyc", O_RDONLY|O_CLOEXEC) = 4
fstat(4, {st_mode=S_IFREG|0644, st_size=4565, ...}) = 0
ioctl(4, TCGETS, 0x7ffed0a03bc0)        = -1 ENOTTY (Inappropriate ioctl for device)
lseek(4, 0, SEEK_CUR)                   = 0
lseek(4, 0, SEEK_CUR)                   = 0
fstat(4, {st_mode=S_IFREG|0644, st_size=4565, ...}) = 0
read(4, "\313\r\r\n\0\0\0\0\303(\242g\246\22\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\5\0\0"..., 4566) = 4565
read(4, "", 1)                          = 0
close(4)                                = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12", {st_mode=S_IFDIR|0755, st_size=20480, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/warnings.py", {st_mode=S_IFREG|0644, st_size=21760, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/warnings.py", {st_mode=S_IFREG|0644, st_size=21760, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/python3.12/__pycache__/warnings.cpython-312.pyc", O_RDONLY|O_CLOEXEC) = 4
fstat(4, {st_mode=S_IFREG|0644, st_size=23792, ...}) = 0
ioctl(4, TCGETS, 0x7ffed0a03550)        = -1 ENOTTY (Inappropriate ioctl for device)
lseek(4, 0, SEEK_CUR)                   = 0
lseek(4, 0, SEEK_CUR)                   = 0
fstat(4, {st_mode=S_IFREG|0644, st_size=23792, ...}) = 0
read(4, "\313\r\r\n\0\0\0\0\303(\242g\0U\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\6\0\0"..., 23793) = 23792
read(4, "", 1)                          = 0
close(4)                                = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/importlib", {st_mode=S_IFDIR|0755, st_size=4096, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/importlib", {st_mode=S_IFDIR|0755, st_size=4096, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/importlib", {st_mode=S_IFDIR|0755, st_size=4096, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/python3.12/importlib", O_RDONLY|O_NONBLOCK|O_CLOEXEC|O_DIRECTORY) = 4
fstat(4, {st_mode=S_IFDIR|0755, st_size=4096, ...}) = 0
getdents64(4, 0x178d09c0 /* 14 entries */, 32768) = 456
getdents64(4, 0x178d09c0 /* 0 entries */, 32768) = 0
close(4)                                = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/importlib/_abc.py", {st_mode=S_IFREG|0644, st_size=1354, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/importlib/_abc.py", {st_mode=S_IFREG|0644, st_size=1354, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/python3.12/importlib/__pycache__/_abc.cpython-312.pyc", O_RDONLY|O_CLOEXEC) = 4
fstat(4, {st_mode=S_IFREG|0644, st_size=1643, ...}) = 0
ioctl(4, TCGETS, 0x7ffed0a03a00)        = -1 ENOTTY (Inappropriate ioctl for device)
lseek(4, 0, SEEK_CUR)                   = 0
lseek(4, 0, SEEK_CUR)                   = 0
fstat(4, {st_mode=S_IFREG|0644, st_size=1643, ...}) = 0
read(4, "\313\r\r\n\0\0\0\0\303(\242gJ\5\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\5\0\0"..., 1644) = 1643
read(4, "", 1)                          = 0
close(4)                                = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12", {st_mode=S_IFDIR|0755, st_size=20480, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/threading.py", {st_mode=S_IFREG|0644, st_size=60123, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/threading.py", {st_mode=S_IFREG|0644, st_size=60123, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/python3.12/__pycache__/threading.cpython-312.pyc", O_RDONLY|O_CLOEXEC) = 4
fstat(4, {st_mode=S_IFREG|0644, st_size=65346, ...}) = 0
ioctl(4, TCGETS, 0x7ffed0a03a00)        = -1 ENOTTY (Inappropriate ioctl for device)
lseek(4, 0, SEEK_CUR)                   = 0
lseek(4, 0, SEEK_CUR)                   = 0
fstat(4, {st_mode=S_IFREG|0644, st_size=65346, ...}) = 0
read(4, "\313\r\r\n\0\0\0\0\303(\242g\333\352\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\5\0\0"..., 65347) = 65346
read(4, "", 1)                          = 0
close(4)                                = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12", {st_mode=S_IFDIR|0755, st_size=20480, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/functools.py", {st_mode=S_IFREG|0644, st_size=38126, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/functools.py", {st_mode=S_IFREG|0644, st_size=38126, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/python3.12/__pycache__/functools.cpython-312.pyc", O_RDONLY|O_CLOEXEC) = 4
fstat(4, {st_mode=S_IFREG|0644, st_size=40506, ...}) = 0
ioctl(4, TCGETS, 0x7ffed0a03390)        = -1 ENOTTY (Inappropriate ioctl for device)
lseek(4, 0, SEEK_CUR)                   = 0
lseek(4, 0, SEEK_CUR)                   = 0
fstat(4, {st_mode=S_IFREG|0644, st_size=40506, ...}) = 0
read(4, "\313\r\r\n\0\0\0\0\303(\242g\356\224\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\7\0\0"..., 40507) = 40506
read(4, "", 1)                          = 0
close(4)                                = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12", {st_mode=S_IFDIR|0755, st_size=20480, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/collections/__init__.cpython-312-x86_64-linux-gnu.so", 0x7ffed0a03080, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/usr/lib/python3.12/collections/__init__.abi3.so", 0x7ffed0a03080, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/usr/lib/python3.12/collections/__init__.so", 0x7ffed0a03080, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/usr/lib/python3.12/collections/__init__.py", {st_mode=S_IFREG|0644, st_size=52378, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/collections/__init__.py", {st_mode=S_IFREG|0644, st_size=52378, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/python3.12/collections/__pycache__/__init__.cpython-312.pyc", O_RDONLY|O_CLOEXEC) = 4
fstat(4, {st_mode=S_IFREG|0644, st_size=73089, ...}) = 0
ioctl(4, TCGETS, 0x7ffed0a02d20)        = -1 ENOTTY (Inappropriate ioctl for device)
lseek(4, 0, SEEK_CUR)                   = 0
lseek(4, 0, SEEK_CUR)                   = 0
fstat(4, {st_mode=S_IFREG|0644, st_size=73089, ...}) = 0
brk(0x17918000)                         = 0x17918000
read(4, "\313\r\r\n\0\0\0\0\303(\242g\232\314\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\5\0\0"..., 73090) = 73089
read(4, "", 1)                          = 0
close(4)                                = 0
brk(0x17906000)                         = 0x17906000
newfstatat(AT_FDCWD, "/usr/lib/python3.12", {st_mode=S_IFDIR|0755, st_size=20480, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/keyword.py", {st_mode=S_IFREG|0644, st_size=1073, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/keyword.py", {st_mode=S_IFREG|0644, st_size=1073, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/python3.12/__pycache__/keyword.cpython-312.pyc", O_RDONLY|O_CLOEXEC) = 4
fstat(4, {st_mode=S_IFREG|0644, st_size=1041, ...}) = 0
ioctl(4, TCGETS, 0x7ffed0a026b0)        = -1 ENOTTY (Inappropriate ioctl for device)
lseek(4, 0, SEEK_CUR)                   = 0
lseek(4, 0, SEEK_CUR)                   = 0
fstat(4, {st_mode=S_IFREG|0644, st_size=1041, ...}) = 0
read(4, "\313\r\r\n\0\0\0\0\303(\242g1\4\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\3\0\0"..., 1042) = 1041
read(4, "", 1)                          = 0
close(4)                                = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12", {st_mode=S_IFDIR|0755, st_size=20480, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/operator.py", {st_mode=S_IFREG|0644, st_size=10965, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/operator.py", {st_mode=S_IFREG|0644, st_size=10965, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/python3.12/__pycache__/operator.cpython-312.pyc", O_RDONLY|O_CLOEXEC) = 4
fstat(4, {st_mode=S_IFREG|0644, st_size=17364, ...}) = 0
ioctl(4, TCGETS, 0x7ffed0a026b0)        = -1 ENOTTY (Inappropriate ioctl for device)
lseek(4, 0, SEEK_CUR)                   = 0
lseek(4, 0, SEEK_CUR)                   = 0
fstat(4, {st_mode=S_IFREG|0644, st_size=17364, ...}) = 0
read(4, "\313\r\r\n\0\0\0\0\303(\242g\325*\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\4\0\0"..., 17365) = 17364
read(4, "", 1)                          = 0
close(4)                                = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12", {st_mode=S_IFDIR|0755, st_size=20480, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/reprlib.py", {st_mode=S_IFREG|0644, st_size=6569, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/reprlib.py", {st_mode=S_IFREG|0644, st_size=6569, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/python3.12/__pycache__/reprlib.cpython-312.pyc", O_RDONLY|O_CLOEXEC) = 4
fstat(4, {st_mode=S_IFREG|0644, st_size=9909, ...}) = 0
ioctl(4, TCGETS, 0x7ffed0a026b0)        = -1 ENOTTY (Inappropriate ioctl for device)
lseek(4, 0, SEEK_CUR)                   = 0
lseek(4, 0, SEEK_CUR)                   = 0
fstat(4, {st_mode=S_IFREG|0644, st_size=9909, ...}) = 0
read(4, "\313\r\r\n\0\0\0\0\303(\242g\251\31\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\4\0\0"..., 9910) = 9909
read(4, "", 1)                          = 0
close(4)                                = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12", {st_mode=S_IFDIR|0755, st_size=20480, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/_weakrefset.py", {st_mode=S_IFREG|0644, st_size=5893, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/_weakrefset.py", {st_mode=S_IFREG|0644, st_size=5893, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/python3.12/__pycache__/_weakrefset.cpython-312.pyc", O_RDONLY|O_CLOEXEC) = 4
fstat(4, {st_mode=S_IFREG|0644, st_size=11751, ...}) = 0
ioctl(4, TCGETS, 0x7ffed0a03390)        = -1 ENOTTY (Inappropriate ioctl for device)
lseek(4, 0, SEEK_CUR)                   = 0
lseek(4, 0, SEEK_CUR)                   = 0
fstat(4, {st_mode=S_IFREG|0644, st_size=11751, ...}) = 0
read(4, "\313\r\r\n\0\0\0\0\303(\242g\5\27\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\4\0\0"..., 11752) = 11751
read(4, "", 1)                          = 0
close(4)                                = 0
gettid()                                = 133068
newfstatat(AT_FDCWD, "/usr/lib/python3/dist-packages", {st_mode=S_IFDIR|0755, st_size=12288, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3/dist-packages/zope/__init__.cpython-312-x86_64-linux-gnu.so", 0x7ffed0a04820, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/usr/lib/python3/dist-packages/zope/__init__.abi3.so", 0x7ffed0a04820, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/usr/lib/python3/dist-packages/zope/__init__.so", 0x7ffed0a04820, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/usr/lib/python3/dist-packages/zope/__init__.py", {st_mode=S_IFREG|0644, st_size=56, ...}, 0) = 0
read(3, "", 8192)                       = 0
close(3)                                = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/dist-packages", 0x7ffed0a04ae0, 0) = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/usr/lib/python3.12", {st_mode=S_IFDIR|0755, st_size=20480, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/sitecustomize.py", {st_mode=S_IFREG|0644, st_size=155, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/sitecustomize.py", {st_mode=S_IFREG|0644, st_size=155, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/python3.12/__pycache__/sitecustomize.cpython-312.pyc", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=300, ...}) = 0
ioctl(3, TCGETS, 0x7ffed0a043c0)        = -1 ENOTTY (Inappropriate ioctl for device)
lseek(3, 0, SEEK_CUR)                   = 0
lseek(3, 0, SEEK_CUR)                   = 0
fstat(3, {st_mode=S_IFREG|0644, st_size=300, ...}) = 0
read(3, "\313\r\r\n\0\0\0\0\273$\26f\233\0\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\4\0\0"..., 301) = 300
read(3, "", 1)                          = 0
close(3)                                = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12", {st_mode=S_IFDIR|0755, st_size=20480, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/lib-dynload", {st_mode=S_IFDIR|0755, st_size=12288, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/local/lib/python3.12/dist-packages", {st_mode=S_IFDIR|0755, st_size=4096, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3/dist-packages", {st_mode=S_IFDIR|0755, st_size=12288, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3/dist-packages/apport_python_hook.py", {st_mode=S_IFREG|0644, st_size=8695, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3/dist-packages/apport_python_hook.py", {st_mode=S_IFREG|0644, st_size=8695, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/python3/dist-packages/__pycache__/apport_python_hook.cpython-312.pyc", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=8876, ...}) = 0
ioctl(3, TCGETS, 0x7ffed0a03d50)        = -1 ENOTTY (Inappropriate ioctl for device)
lseek(3, 0, SEEK_CUR)                   = 0
lseek(3, 0, SEEK_CUR)                   = 0
fstat(3, {st_mode=S_IFREG|0644, st_size=8876, ...}) = 0
read(3, "\313\r\r\n\0\0\0\0\247\22!f\367!\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\2\0\0"..., 8877) = 8876
read(3, "", 1)                          = 0
close(3)                                = 0
getcwd("/home/tatsuro/workspaces/myblog/tmp", 1024) = 36
newfstatat(AT_FDCWD, "/usr/lib/python3.12", {st_mode=S_IFDIR|0755, st_size=20480, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3.12/lib-dynload", {st_mode=S_IFDIR|0755, st_size=12288, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/local/lib/python3.12/dist-packages", {st_mode=S_IFDIR|0755, st_size=4096, ...}, 0) = 0
newfstatat(AT_FDCWD, "/usr/lib/python3/dist-packages", {st_mode=S_IFDIR|0755, st_size=12288, ...}, 0) = 0
newfstatat(AT_FDCWD, "/home/tatsuro/workspaces/myblog/tmp/main.py", {st_mode=S_IFREG|0644, st_size=548, ...}, 0) = 0
openat(AT_FDCWD, "/home/tatsuro/workspaces/myblog/tmp/main.py", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=548, ...}) = 0
ioctl(3, TCGETS, 0x7ffed0a04f20)        = -1 ENOTTY (Inappropriate ioctl for device)
lseek(3, 0, SEEK_CUR)                   = 0
lseek(3, 0, SEEK_CUR)                   = 0
lseek(3, -22, SEEK_END)                 = 526
lseek(3, 0, SEEK_CUR)                   = 526
read(3, "e\")\n    # read_file()\n", 4096) = 22
lseek(3, 0, SEEK_END)                   = 548
lseek(3, 0, SEEK_CUR)                   = 548
lseek(3, 0, SEEK_SET)                   = 0
lseek(3, 0, SEEK_CUR)                   = 0
fstat(3, {st_mode=S_IFREG|0644, st_size=548, ...}) = 0
read(3, "def compute():\n    \"\"\"\346\274\224\347\256\227\345\207\246\347"..., 549) = 548
read(3, "", 1)                          = 0
lseek(3, 0, SEEK_SET)                   = 0
close(3)                                = 0
newfstatat(AT_FDCWD, "/home/tatsuro/workspaces/myblog/tmp/main.py", {st_mode=S_IFREG|0644, st_size=548, ...}, 0) = 0
readlink("main.py", 0x7ffed09f46c0, 4096) = -1 EINVAL (Invalid argument)
getcwd("/home/tatsuro/workspaces/myblog/tmp", 1024) = 36
readlink("/home/tatsuro/workspaces/myblog/tmp/main.py", 0x7ffed09f4260, 1023) = -1 EINVAL (Invalid argument)
openat(AT_FDCWD, "/home/tatsuro/workspaces/myblog/tmp/main.py", O_RDONLY) = 3
ioctl(3, FIOCLEX)                       = 0
fstat(3, {st_mode=S_IFREG|0644, st_size=548, ...}) = 0
ioctl(3, TCGETS, 0x7ffed0a05650)        = -1 ENOTTY (Inappropriate ioctl for device)
lseek(3, 0, SEEK_CUR)                   = 0
fstat(3, {st_mode=S_IFREG|0644, st_size=548, ...}) = 0
read(3, "def compute():\n    \"\"\"\346\274\224\347\256\227\345\207\246\347"..., 4096) = 548
lseek(3, 0, SEEK_SET)                   = 0
read(3, "def compute():\n    \"\"\"\346\274\224\347\256\227\345\207\246\347"..., 4096) = 548
read(3, "", 4096)                       = 0
close(3)                                = 0
write(1, "Execute compute()\n", 18Execute compute()
)     = 18
write(1, "Done\n", 5Done
)                   = 5
rt_sigaction(SIGINT, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER|SA_ONSTACK, sa_restorer=0x7f5fb93bb330}, {sa_handler=0x6e72f0, sa_mask=[], sa_flags=SA_RESTORER|SA_ONSTACK, sa_restorer=0x7f5fb93bb330}, 8) = 0
munmap(0x7f5fb96b9000, 16384)           = 0
exit_group(0)                           = ?
+++ exited with 0 +++
```

</details>

Python の場合、演算処理だけにしてもインタプリターの実行準備などで大量のシステムコールが発生してることがわかる。

もう少し見やすくするために、フィルタリングして確認する。  
また、わかりやすいように、`print`(システムコールの`write`)で挟んでシステムコールが呼ばれているのかを確認する。

<details>
<summary>変更後のコード</summary>

```python
def compute():
    """演算処理のみを行う関数
    """
    total = 0
    for i in range(1_000_000):
        total += i
    return total

if __name__ == "__main__":
    print("Execute compute()")
    compute()
    print("Done")
```

</details>

フィルタリングした実行結果

```bash
$ strace -e trace=read,write python3 main.py
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 832) = 832
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 832) = 832
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 832) = 832
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\220\243\2\0\0\0\0\0"..., 832) = 832
read(3, "# Locale name alias data base.\n#"..., 4096) = 2996
read(3, "", 4096)                       = 0
read(3, "TZif2\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\4\0\0\0\4\0\0\0\0"..., 4096) = 309
read(3, "TZif2\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\4\0\0\0\4\0\0\0\0"..., 4096) = 176
read(3, "\313\r\r\n\0\0\0\0\303(\242g\374\26\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\6\0\0"..., 5791) = 5790
read(3, "", 1)                          = 0
read(3, "\313\r\r\n\0\0\0\0\303(\242g==\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\5\0\0"..., 12409) = 12408
read(3, "", 1)                          = 0
read(3, "\313\r\r\n\0\0\0\0\303(\242g\355\3\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\5\0\0"..., 2148) = 2147
read(3, "", 1)                          = 0
read(3, "import os; var = 'SETUPTOOLS_USE"..., 8192) = 151
read(4, "\313\r\r\n\0\0\0\0\n_\337d\233\30\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\6\0\0"..., 9999) = 9998
read(4, "", 1)                          = 0
read(3, "", 8192)                       = 0
read(3, "import sys, types, os;has_mfs = "..., 8192) = 529
read(4, "\313\r\r\n\0\0\0\0\303(\242g\361*\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\6\0\0"..., 14940) = 14939
read(4, "", 1)                          = 0
read(4, "\313\r\r\n\0\0\0\0\303(\242g\246\22\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\5\0\0"..., 4566) = 4565
read(4, "", 1)                          = 0
read(4, "\313\r\r\n\0\0\0\0\303(\242g\0U\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\6\0\0"..., 23793) = 23792
read(4, "", 1)                          = 0
read(4, "\313\r\r\n\0\0\0\0\303(\242gJ\5\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\5\0\0"..., 1644) = 1643
read(4, "", 1)                          = 0
read(4, "\313\r\r\n\0\0\0\0\303(\242g\333\352\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\5\0\0"..., 65347) = 65346
read(4, "", 1)                          = 0
read(4, "\313\r\r\n\0\0\0\0\303(\242g\356\224\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\7\0\0"..., 40507) = 40506
read(4, "", 1)                          = 0
read(4, "\313\r\r\n\0\0\0\0\303(\242g\232\314\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\5\0\0"..., 73090) = 73089
read(4, "", 1)                          = 0
read(4, "\313\r\r\n\0\0\0\0\303(\242g1\4\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\3\0\0"..., 1042) = 1041
read(4, "", 1)                          = 0
read(4, "\313\r\r\n\0\0\0\0\303(\242g\325*\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\4\0\0"..., 17365) = 17364
read(4, "", 1)                          = 0
read(4, "\313\r\r\n\0\0\0\0\303(\242g\251\31\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\4\0\0"..., 9910) = 9909
read(4, "", 1)                          = 0
read(4, "\313\r\r\n\0\0\0\0\303(\242g\5\27\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\4\0\0"..., 11752) = 11751
read(4, "", 1)                          = 0
read(3, "", 8192)                       = 0
read(3, "\313\r\r\n\0\0\0\0\273$\26f\233\0\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\4\0\0"..., 301) = 300
read(3, "", 1)                          = 0
read(3, "\313\r\r\n\0\0\0\0\247\22!f\367!\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\2\0\0"..., 8877) = 8876
read(3, "", 1)                          = 0
read(3, "e\")\n    # read_file()\n", 4096) = 22
read(3, "def compute():\n    \"\"\"\346\274\224\347\256\227\345\207\246\347"..., 549) = 548
read(3, "", 1)                          = 0
read(3, "def compute():\n    \"\"\"\346\274\224\347\256\227\345\207\246\347"..., 4096) = 548
read(3, "def compute():\n    \"\"\"\346\274\224\347\256\227\345\207\246\347"..., 4096) = 548
read(3, "", 4096)                       = 0
write(1, "Execute compute()\n", 18Execute compute()
)     = 18
write(1, "Done\n", 5Done
)                   = 5
+++ exited with 0 +++
```

注目する部分は、最後の以下の部分。

```bash
write(1, "Execute compute()\n", 18Execute compute()
)     = 18
write(1, "Done\n", 5Done
)                   = 5
```

`write(1,・・・)`の`1`は、ファイルディスクリプタであり、標準出力(`1`)になる。`print`関数は、標準出力に出力するため`1`となっており、その内容が`Execute compute()`と`Done`となっていることが確認できる。

また、`Execute compute()`と`Done`の間にある`compute()`については、何も表示されていないこと(=システムコールが呼ばれていない)が確認できる。これは、`compute()`が演算処理のみで、ユーザーモードの動作のみで完了できるためである。学習した通りの結果となっている。

### ファイル操作の処理を実行する

ファイルの読み込み・書き込みの操作のあるプログラムを実行して、システムコールが呼ばれることを確認する。

<details>
<summary>コード</summary>

```python
def read_and_write():
    """ファイルを読み込む関数
    """
    with open("test.txt", "r") as f:
        _ = f.readlines()

    with open("test.txt", "w") as f:
        f.write("Hello, World!\n")


if __name__ == "__main__":
    print("Execute read_and_write()")
    read_and_write()
    print("Done")
```

</details>

実行結果

```bash
$ strace -e trace=write,read python3 main.py
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 832) = 832
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 832) = 832
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 832) = 832
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\220\243\2\0\0\0\0\0"..., 832) = 832
read(3, "# Locale name alias data base.\n#"..., 4096) = 2996
read(3, "", 4096)                       = 0
read(3, "TZif2\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\4\0\0\0\4\0\0\0\0"..., 4096) = 309
read(3, "TZif2\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\4\0\0\0\4\0\0\0\0"..., 4096) = 176
read(3, "\313\r\r\n\0\0\0\0\303(\242g\374\26\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\6\0\0"..., 5791) = 5790
read(3, "", 1)                          = 0
read(3, "\313\r\r\n\0\0\0\0\303(\242g==\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\5\0\0"..., 12409) = 12408
read(3, "", 1)                          = 0
read(3, "\313\r\r\n\0\0\0\0\303(\242g\355\3\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\5\0\0"..., 2148) = 2147
read(3, "", 1)                          = 0
read(3, "import os; var = 'SETUPTOOLS_USE"..., 8192) = 151
read(4, "\313\r\r\n\0\0\0\0\n_\337d\233\30\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\6\0\0"..., 9999) = 9998
read(4, "", 1)                          = 0
read(3, "", 8192)                       = 0
read(3, "import sys, types, os;has_mfs = "..., 8192) = 529
read(4, "\313\r\r\n\0\0\0\0\303(\242g\361*\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\6\0\0"..., 14940) = 14939
read(4, "", 1)                          = 0
read(4, "\313\r\r\n\0\0\0\0\303(\242g\246\22\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\5\0\0"..., 4566) = 4565
read(4, "", 1)                          = 0
read(4, "\313\r\r\n\0\0\0\0\303(\242g\0U\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\6\0\0"..., 23793) = 23792
read(4, "", 1)                          = 0
read(4, "\313\r\r\n\0\0\0\0\303(\242gJ\5\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\5\0\0"..., 1644) = 1643
read(4, "", 1)                          = 0
read(4, "\313\r\r\n\0\0\0\0\303(\242g\333\352\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\5\0\0"..., 65347) = 65346
read(4, "", 1)                          = 0
read(4, "\313\r\r\n\0\0\0\0\303(\242g\356\224\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\7\0\0"..., 40507) = 40506
read(4, "", 1)                          = 0
read(4, "\313\r\r\n\0\0\0\0\303(\242g\232\314\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\5\0\0"..., 73090) = 73089
read(4, "", 1)                          = 0
read(4, "\313\r\r\n\0\0\0\0\303(\242g1\4\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\3\0\0"..., 1042) = 1041
read(4, "", 1)                          = 0
read(4, "\313\r\r\n\0\0\0\0\303(\242g\325*\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\4\0\0"..., 17365) = 17364
read(4, "", 1)                          = 0
read(4, "\313\r\r\n\0\0\0\0\303(\242g\251\31\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\4\0\0"..., 9910) = 9909
read(4, "", 1)                          = 0
read(4, "\313\r\r\n\0\0\0\0\303(\242g\5\27\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\4\0\0"..., 11752) = 11751
read(4, "", 1)                          = 0
read(3, "", 8192)                       = 0
read(3, "\313\r\r\n\0\0\0\0\273$\26f\233\0\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\4\0\0"..., 301) = 300
read(3, "", 1)                          = 0
read(3, "\313\r\r\n\0\0\0\0\247\22!f\367!\0\0\343\0\0\0\0\0\0\0\0\0\0\0\0\2\0\0"..., 8877) = 8876
read(3, "", 1)                          = 0
read(3, "e\")\n    # read_file()\n", 4096) = 22
read(3, "# def compute():\n#     \"\"\"\346\274\224\347\256\227"..., 549) = 548
read(3, "", 1)                          = 0
read(3, "# def compute():\n#     \"\"\"\346\274\224\347\256\227"..., 4096) = 548
read(3, "# def compute():\n#     \"\"\"\346\274\224\347\256\227"..., 4096) = 548
read(3, "", 4096)                       = 0
write(1, "Execute read_and_write()\n", 25Execute read_and_write()
) = 25
read(3, "Hello, World!\n", 8192)        = 14
read(3, "", 8192)                       = 0
write(3, "Hello, World!\n", 14)         = 14
write(1, "Done\n", 5Done
)                   = 5
+++ exited with 0 +++
```

注目するのは以下の部分

```bash
write(1, "Execute read_and_write()\n", 25Execute read_and_write()
) = 25
read(3, "Hello, World!\n", 8192)        = 14
read(3, "", 8192)                       = 0
write(3, "Hello, World!\n", 14)         = 14
write(1, "Done\n", 5Done
)                   = 5
```

`Execute compute()`と`Done`の間で、`read`と`write`が発生していることが確認できる。  
また、そのファイルディスクリプタが`3`となっており、ファイルであることが確認できる。

1 回目の`read(3, "Hello, World!\n", 8192)        = 14`は、最大 8192 バイト（バッファサイズ）読み込もうとしたら、実際に 14 バイト読み込めたということを意味している。2 回目の`read(3, "", 8192)                       = 0`は、`EOF`(=読み取るデータがなかった)を意味する。このように、python の`readline()`（または`read`）を使うと最低２回はシステムコール`read()`が呼ばれる。最後に必ず読み込むデータがないこと(`EOF`)の確認が行われる。

**バッファサイズ**

`8192`は、Python 側で設定しているデフォルトのバッファサイズ。  
`io`モジュールで確認できる。

```bash
$ python3
Python 3.12.3 (main, Feb  4 2025, 14:48:35) [GCC 13.3.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import io
>>> io.DEFAULT_BUFFER_SIZE
8192
```

ちなみに、Python の`open(・・・, buffering=4)`ではバッファサイズは変わらない。  
あくまで Python 内部でのバッファサイズを変えているだけになる。
