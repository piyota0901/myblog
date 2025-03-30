---
title: "SIGTERMとSIGKILLの違いとシグナル処理の検証"
date: 2025-03-21T15:06:21+09:00
draft: false
description: ""
categories: ["python", "OS"]
---

SIGTERM と SIGKILL の違いについて勉強してみた 📝  
強制終了の`kill`コマンドを実行することでプロセスに送信されるシグナルの種類 ⚡

## そもそもシグナルとは?

---

プロセス間やカーネルからプロセスに対して非同期に信号を送信する仕組みのこと。プロセス間通信の一種。特定のイベントが発生したときに、プロセスに対して割り込みを行うために使用される。  
シグナルハンドラを登録しておけば、シグナル受信時にそのハンドラが呼び出される。そうでない場合は、デフォルトのシグナル処理が実行される。

例えば・・・

- ユーザーが`Ctrl+C`を押すと、`SIGINT`シグナル（キーボードからの割り込みシグナル）がプロセスに送信され、通常はプロセスが終了する。
- 親プロセスが子プロセスの終了を知るために`SIGCHLD`を受け取る。

### シグナルの種類

名前と数値が決まっている。

| 名前    | 数字 | 動作                                 | 意味                                                       |
| ------- | ---- | ------------------------------------ | ---------------------------------------------------------- |
| SIGHUP  | 1    | Term(プロセスの終了)                 | 制御端末の切断、仮想端末の終了                             |
| SIGINT  | 2    | Term(プロセスの終了)                 | キーボードからの割り込みシグナル(`CTRL+C`)                 |
| SIGFPE  | 8    | Core(プロセスの終了とコアダンプ出力) | 不正な浮動小数点演算（ゼロ除算、オーバーフローなど）の発生 |
| SIGKILL | 9    | Term（プロセスの終了）               | 強制終了シグナル                                           |
| SIGTERM | 15   | Term(プロセスの終了)                 | 終了シグナル(`kill`コマンドのデフォルトシグナル)           |
| SIGCHLD | 17   | Ignore(無視)                         | 子プロセスの状態（終了、停止、再開）が変わった             |
| SIGSTOP | 19   | Stop(プロセスの一時停止)             | 一時停止シグナル                                           |
| SIGTSTP | 20   | Stop(プロセスの一時停止)             | 端末からの一時停止シグナル(`CTRL+Z`)                       |

参考リンク

- [IT medita | Linux の「シグナル」って何だろう？](https://atmarkit.itmedia.co.jp/ait/articles/1708/04/news015.html)

## SIGTERM と SIGKILL

---

### SIGTERM

**SIG**nal and **TERM**inate の略。`SIGTERM`シグナルを受信したプロセスは、無視することもできるためソフトキルとも呼ばれる。  
親プロセス、子プロセスに情報を送信する時間を取得して、子プロセスは`init`によって処理される。

プロセスを止めるための丁寧な方法。

### SIGKILL

`SIGKILL`は、プロセスを強制的に即時終了させるために使用される。`SIGKILL`シグナルを受信したプロセスは、ブロックしたり、無視したりすることができない。  
プロセスを強制的に終了させる野蛮な方法になるため、最終手段として使用するべき。（例えば、応答しないプロセスなどがある場合）  
親プロセスに情報を送信する時間がないため、**ゾンビプロセス(孤児プロセス)**が作成されることがある。

参考リンク

- [SIGTERM vs SIGKILL: What's the Difference?](https://linuxhandbook.com/sigterm-vs-sigkill/)

## ゾンビプロセス/孤児プロセス

| 名前           | 状態                                                                 |
| -------------- | -------------------------------------------------------------------- |
| ゾンビプロセス | 終了したが親が `wait()` を呼んでおらず、プロセステーブルに残っている |
| 孤児プロセス   | 親が異常終了し、`init` などに引き取られて生き続けている              |

---

### ゾンビプロセス

ゾンビプロセスは、プロセスが終了したもののプロセステーブルにまだ残っている状態。  
子プロセスは、基本的に一時的にゾンビプロセスなる。ただし、親プロセスが`wait()`を呼ばないままだと、ゾンビのまま残ってしまう。  
例えば・・・コンテナンテナの場合、`PID1`が`init`ではないため、`PID1`のプロセスが`wait()`を呼ばないままだとゾンビのままになる。

![](./zombie_process.svg)

### 孤児プロセス

孤児プロセスは、親プロセスが異常終了してしまい`wait()`が呼ばれないまま残ってしまった子プロセスのこと。  
最終的に`init`に引き取られる。上記の`SIGKILL`のゾンビプロセスは、この孤児プロセスを指している。

![](./orphan_process.svg)

参考リンク

- [「分かりそう」で「分からない」でも「分かった」気になれる IT 用語辞典 | 孤児プロセス](https://wa3.i-3-i.info/word14951.html)
  ```plaintext
  親プロセスに育児放棄された（親が先に終了しちゃった）プロセスを指して「ゾンビプロセス」と表現していることも多々あります。
  私も基本的には（孤児プロセスも）「ゾンビプロセス」と表現します。
  ほとんどの場合、それで意図が通じるからです。
  ```

## Python によるシグナル処理の検証

---

Python では、標準の`signal`モジュールを使用することで、シグナルをハンドリングできる。  
メインスレッドにのみ、シグナルハンドラは登録される。  
_[signal --- 非同期イベントにハンドラーを設定する](https://docs.python.org/ja/3.12/library/signal.html#module-signal)_

Python の場合、`Ctrl+C`が入力されると`KeyboardInterrupt`（例外）に変換される。

### `KeyboardInterrupt`と SIGINT のハンドリング

どちらでも`SIGINT`をハンドリングできるが、作成するアプリケーションによって`signal`でハンドリングするか、`try-except`でハンドリングするかを考えて実装する必要がある。

- `singal`でハンドリングする場合

  - バックグラウンドプロセス（デーモン・サービス）
    - Linux のサービス（systemd で起動するようなアプリ）
    - Web サーバー、キュー処理、常駐監視スクリプトなど
  - つまり、**ユーザーと対話しないアプリケーション**

- `try-except`でハンドリングする場合

  - 手元で実行する対話的なアプリケーション
  - 一時的なスクリプト
  - つまり、**手動操作前提のアプリケーション**

### シグナルのハンドリング検証

子プロセス（単純に sleep し続けるプログラム）を作成するスクリプト。

```python
import signal
import subprocess
import time
import sys
import os

child = None

def handle_signal(signum, frame):
    print(f"\n[Main] Received signal {signum}. Cleaning up...")
    if child and child.poll() is None:
        print(f"[Main] Terminating child process (pid={child.pid})")
        child.terminate()
        try:
            child.wait(timeout=5)
        except subprocess.TimeoutExpired:
            print(f"[Main] Child didn't exit, killing it")
            child.kill()
    print("[Main] Done. Exiting.")
    sys.exit(0)

# シグナルハンドラ登録
signal.signal(signal.SIGINT, handle_signal)
signal.signal(signal.SIGTERM, handle_signal)

# 自分のPIDを表示
print(f"[Main] PID: {os.getpid()}")
print("[Main] Starting child process...")

# 子プロセス起動
child = subprocess.Popen([sys.executable, "-u", "-c", """
import time
import os
import sys

print(f"[Child] PID: {os.getpid()} | Parent PID: {os.getppid()}")

try:
    while True:
        time.sleep(1)
except KeyboardInterrupt:
    print("[Child] KeyboardInterrupt received.")
finally:
    print("[Child] Cleanup before exit.")
    sys.exit(0)
"""])

# メインループ
try:
    while True:
        time.sleep(0.5)
        if child.poll() is not None:
            print("[Main] Child exited.")
            break
except KeyboardInterrupt:
    handle_signal(signal.SIGINT, None)
```

**親プロセスに`SIGTERM`を送った場合**

以下のことを確認する

- 子プロセスを停止させてクリーンアップすることを確認する
- `SIGTERM`の猶予を与える振る舞いを確認する

検証

1. `main.py`を実行する

   ```bash
   $ uv run python main.py
   [Main] PID: 58433
   [Main] Starting child process...
   [Child] PID: 58434 | Parent PID: 58433

   ```

1. 別のターミナルを開いて、`SIGTERM`を送信する

   - 別ターミナル
     ```bash
     $ kill 58433
     ```
   - スクリプトのターミナル
     ```bash
     [Main] Received signal 15. Cleaning up...
     [Main] Terminating child process (pid=58434)
     [Main] Done. Exiting.
     ```

メインプロセスに`SIGTERM(=15)`が送られて、子プロセスが正しく停止されたことを確認。

**親プロセスに`SIGKILL`を送った場合**

以下のことを確認する。

- 子プロセスを停止させる処理が実行されないことを確認する
- `SIGKILL`の即時強制という振る舞いを確認する

検証

1. `main.py`を実行する

   ```bash
   $ uv run python main.py
   [Main] PID: 59870
   [Main] Starting child process...
   [Child] PID: 59871 | Parent PID: 59870
   ```

1. 別のターミナルを開いて、`SIGKILL`を送信する

   ```bash
   $ kill -9 59870
   ```

1. 子プロセスを確認する

   ```bash
   $ ps -o pid,ppid,stat,command -p 59871
   PID    PPID STAT COMMAND
   59871     625 S    /home/tatsuro/workspaces/learn-sigkill-sigterm/.ve
   (learn-sigkill-sigterm)
   ```

1. 親プロセス(`PPID`: 625)を確認する

   ```bash
   ps 625

   PID TTY      STAT   TIME COMMAND
   625 ?        S      0:00 /init
   (learn-sigkill-sigterm)
   ```

   **init**に引き取られていることを確認

1. 子プロセスを kill する

   ```bash
   $ kill -9 59871
   ```

親プロセスに`SIGKILL`を送信すると、子プロセスを停止しないまま即時終了とんったことを確認。子プロセスは孤児プロセスになり、`init`に引き取られることも確認。

**`init`なのに PID=625 となるのはなぜか**

簡単に言うと`systemd(PID=1)`の配下に`/init`があるため。

```bash
$ pstree -p
systemd(1)─┬─agetty(220)
           ├─agetty(223)
           ├─containerd(209)─┬─{containerd}(234)
           │                 ├─{containerd}(237)
            ・・・
           │
           │
           ├─init-systemd(Ub(2)─┬─SessionLeader(624)───Relay(626)(625)+++
           │                    ├─SessionLeader(939)───Relay(946)(940)+++
           │                    ├─SessionLeader(55675)───Relay(55677)(+
           │                    ├─init(6)───{init}(7)
           │                    ├─login(601)───zsh(697)
           │                    └─{init-systemd(Ub}(8)
           ├─ollama(185)─┬─{ollama}(233)
           │             ├─{ollama}(235)
           │             ├─・・・
           │             └─{ollama}(300)
            ・・・
           │               └─{rsyslogd}(245)
           ├─systemd(681)───(sd-pam)(682)
           ├─systemd-journal(54)
           ├─systemd-logind(190)
           ├─systemd-resolve(160)
           ├─systemd-timesyn(161)───{systemd-timesyn}(168)
           ├─systemd-udevd(99)
           ├─unattended-upgr(230)───{unattended-upgr}(299)
           └─wsl-pro-service(201)─┬─{wsl-pro-service}(246)
                                  ├─{wsl-pro-service}(247)
                                  ├─{wsl-pro-service}(248)
                                  ├─{wsl-pro-service}(249)
                                  ├─{wsl-pro-service}(250)
                                  ├─{wsl-pro-service}(270)
                                  ├─{wsl-pro-service}(271)
                                  ├─{wsl-pro-service}(57371)
                                  ├─{wsl-pro-service}(57372)
                                  ├─{wsl-pro-service}(57373)
                                  ├─{wsl-pro-service}(57374)
                                  └─{wsl-pro-service}(57375)
```

`systemd`の配下に`init-systemd`が存在していることが確認できる。

### 検証のまとめ

- `SIGTERM`：クリーンアップ処理が行われ、子プロセスも安全に終了できる。
- `SIGKILL`：親プロセスが即時終了し、子プロセスは孤児として残る。
- WSL 環境では、`/init` は `systemd` 配下のプロセスであり、孤児プロセスの親として表示されることがある。
