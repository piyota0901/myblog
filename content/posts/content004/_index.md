---
title: "PythonのGILの確認"
date: "2025-03-16T19:48:23+09:00"
draft: false
categories: ["python"]
---

Python の GIL を確認してみた。

[使用したプログラムはこちら 📝](https://github.com/piyota0901/learn-gil-and-async)

## 検証環境

- WSL2
  ```bash
  $ cat /etc/os-release
  PRETTY_NAME="Ubuntu 24.04.1 LTS"
  NAME="Ubuntu"
  VERSION_ID="24.04"
  VERSION="24.04.1 LTS (Noble Numbat)"
  VERSION_CODENAME=noble
  ID=ubuntu
  ID_LIKE=debian
  HOME_URL="https://www.ubuntu.com/"
  SUPPORT_URL="https://help.ubuntu.com/"
  BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
  PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
  UBUNTU_CODENAME=noble
  LOGO=ubuntu-logo
  ```
- Python3.12.7

## 検証 ①

---

ベースライン、マルチスレッド、マルチプロセスによる処理を比較することで確認する。  
CPU バウンドな処理とするため計算処理としている。
予想としては以下。

- 速い順に`マルチプロセス`、`ベースライン`、`マルチスレッド`
- マルチプロセスの処理時間は、ベースライン ÷ タスク数に近いはず
- マルチスレッドは、スレッドを切り替える（コンテキストスイッチ）コスト、GIL の取得・解放コストが掛かるため
  ベースラインより遅くなる

#### プログラム

```python
import time
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor


def cpu_bound_task(n: int) -> int:
    """単純な計算を繰り返してCPUを使うタスク"""
    total = 0
    for _ in range(n):
        total += 1
    return total


def run_threads(n: int, repeat: int):
    """スレッドを使ってCPUバウンドな処理を実行"""
    with ThreadPoolExecutor(max_workers=n) as executor:
        results = list(executor.map(cpu_bound_task, [repeat] * n))
    return results


def run_processes(n: int, repeat: int):
    """プロセスを使ってCPUバウンドな処理を実行"""
    with ProcessPoolExecutor(max_workers=n) as executor:
        results = list(executor.map(cpu_bound_task, [repeat] * n))
    return results


if __name__ == "__main__":
    num_tasks = 4  # 並列に実行する数（CPUコア数と同じくらいが良い）
    repeat = 10**7  # 負荷をかけるためのループ回数

    print("Baseline (single-threaded)...")
    start = time.perf_counter()
    for _ in range(num_tasks):
        cpu_bound_task(repeat)
    print("Baseline time: {:.4f} sec".format(time.perf_counter() - start))

    print("-" * 100)

    print("Starting CPU-bound tasks with threads...")
    start = time.perf_counter()
    run_threads(num_tasks, repeat)
    print("Threads time: {:.4f} sec".format(time.perf_counter() - start))

    print("-" * 100)

    print("Starting CPU-bound tasks with processes...")
    start = time.perf_counter()
    run_processes(num_tasks, repeat)
    print("Processes time: {:.4f} sec".format(time.perf_counter() - start))
```

#### 実行結果

```bash
$ rye run python src/01_gil.py
Baseline (single-threaded)...
Baseline time: 2.1103 sec
----------------------------------------------------------------------------------------------------
Starting CPU-bound tasks with threads...
Threads time: 2.7011 sec
----------------------------------------------------------------------------------------------------
Starting CPU-bound tasks with processes...
Processes time: 0.7536 sec
```

おおよそ、想定の結果。  
マルチプロセスが ベースライン ÷ タスク数より若干遅いのは、**マルチプロセス作成**によるオーバヘッドが原因と思われる。  
また、今回は計算のみだが、データ受け渡しが発生する場合 IPC の通信もオーバヘッドになる。

## 検証 ②

---

I/O バウンドな処理を想定した場合に、マルチスレッドが一番速くなるのか検証する。  
（I/O の処理であれば、マルチスレッドは GIL の影響を受けないため）

予想としては以下。

- 速い順に**`マルチスレッド`**、`マルチプロセス`、`ベースライン`
- マルチプロセス/スレッドの処理時間は、ベースライン ÷ タスク数に近いはず
- マルチスレッドは、マルチプロセスに比べてオーバヘッドが小さいため速いはず

#### プログラム

```python
import os
import time
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor


FILE_COUNT = 30  # 処理するファイル数
FILE_SIZE = 1024 * 1024  # 1MBのファイルを作成


def file_io_task(index):
    """ファイルにデータを書き込み、読み込む処理"""
    filename = f"test_file_{index}.txt"

    # 書き込み
    with open(filename, "wb") as f:
        f.write(os.urandom(FILE_SIZE))  # 1MBのランダムデータを書き込む

    # 読み込み
    with open(filename, "rb") as f:
        f.read()

    os.remove(filename)  # 後始末で削除


def run_baseline():
    """シングルスレッドで実行（Baseline）"""
    for i in range(FILE_COUNT):
        file_io_task(i)


def run_threads():
    """スレッドプールを使ってファイルI/Oを並列実行"""
    with ThreadPoolExecutor(max_workers=FILE_COUNT) as executor:
        executor.map(file_io_task, range(FILE_COUNT))


def run_processes():
    """プロセスプールを使ってファイルI/Oを並列実行"""
    with ProcessPoolExecutor(max_workers=FILE_COUNT) as executor:
        executor.map(file_io_task, range(FILE_COUNT))


if __name__ == "__main__":
    # Baseline (シングルスレッド)
    print("Starting file I/O tasks (Baseline - Single Threaded)...")
    start = time.perf_counter()
    run_baseline()
    print("Baseline time: {:.4f} sec".format(time.perf_counter() - start))

    print("-" * 100)

    # マルチスレッド
    print("Starting file I/O tasks with threads...")
    start = time.perf_counter()
    run_threads()
    print("Threads time: {:.4f} sec".format(time.perf_counter() - start))

    print("-" * 100)

    # マルチプロセス
    print("Starting file I/O tasks with processes...")
    start = time.perf_counter()
    run_processes()
    print("Processes time: {:.4f} sec".format(time.perf_counter() - start))
```

#### 実行結果

```bash
$ rye run python 02_gil.py
Starting file I/O tasks (Baseline - Single Threaded)...
Baseline time: 0.1719 sec
----------------------------------------------------------------------------------------------------
Starting file I/O tasks with threads...
Threads time: 0.0570 sec
----------------------------------------------------------------------------------------------------
Starting file I/O tasks with processes...
Processes time: 0.1234 sec
```

- 順番は、想定通りの結果
- `マルチスレッド`の処理時間が、`ベースライン`/`ファイル数`にならない理由としては、OS の I/O 最適化が原因と思われる
  現在のファイルサイズ (FILE_SIZE = 1MB) では、OS のキャッシュ（Page Cache）に乗ってしまう可能性が高い。(ChatGPT より)
- I/O の処理であれば、仕様通り GIL の影響を受けない

## 検証 ③

---

OS の I/O 最適化を受けないように、`100MB`にして実行してみた。

#### プログラム

```python
FILE_COUNT = 30  # 処理するファイル数
FILE_SIZE = 100 * 1024 * 1024  # 100MBのファイルを作成
```

#### 実行結果

```bash
$ rye run python 02_gil.py
Starting file I/O tasks (Baseline - Single Threaded)...
Baseline time: 14.4427 sec
----------------------------------------------------------------------------------------------------
Starting file I/O tasks with threads...
Threads time: 29.5991 sec
----------------------------------------------------------------------------------------------------
Starting file I/O tasks with processes...
Processes time: 49.8219 sec
```

逆に、マルチスレッド、マルチプロセスが極端に遅くなってしまった。

ディスク I/O のボトルネックになっている可能性

- スレッドやプロセスを増やしても、ディスクの読み書き速度は変わらないため、
  同時に多くのスレッド・プロセスがアクセスすると競合が発生する。
- OS の I/O スケジューラが頻繁にコンテキストスイッチを発生させ、最適化が難しくなる

**10MB に変更して検証**

```python
FILE_SIZE = 10 * 1024 * 1024  # 10MBのファイルを作成
```

```bash
$ rye run python 02_gil.py
Starting file I/O tasks (Baseline - Single Threaded)...
Baseline time: 1.3699 sec
----------------------------------------------------------------------------------------------------
Starting file I/O tasks with threads...
Threads time: 0.3219 sec
----------------------------------------------------------------------------------------------------
Starting file I/O tasks with processes...
Processes time: 0.4788 sec
```

速くなったことから、おそらくディスク I/O がボトルネックになっていたことが考えられる。

## 検証 ④

---

非同期処理による効率化を確認する。I/O としてディスク I/O だけでなくネットワークも含めた。  
[JSONPlaceholder](https://blog.a-1.dev/post/2019-02-05-jsonplacehoder/)という API サーバーを利用。

予想としては以下。

- 速い順に`マルチスレッド`、`シングルスレッド(非同期処理)`、`シングルスレッド(同期処理)`
- 今回は I/O のみの処理のため、GIL の影響を受けないのでマルチスレッドが有効に働く想定。
- `シングルスレッド(同期処理)`では、１つずつ処理するため、待ち時間を有効活用する`シングルスレッド(非同期処理)`より遅くなる想定。

#### プログラム

```python
import os
import time
import asyncio
import aiofiles
import httpx
from concurrent.futures import ThreadPoolExecutor

FILE_COUNT = 10  # 処理するファイル数
FILE_SIZE = 1024 * 1024  # 1MB のファイルを作成
API_URL = "https://jsonplaceholder.typicode.com/todos/"  # ダミーAPI

# ---- シングルスレッド（直列処理） ----
def sync_io_task(index):
    """同期的にファイルI/OとAPIリクエストを実行"""
    filename = f"sync_test_file_{index}.txt"

    # ファイル書き込み
    with open(filename, "wb") as f:
        f.write(os.urandom(FILE_SIZE))

    # ファイル読み込み
    with open(filename, "rb") as f:
        f.read()

    os.remove(filename)

    # APIリクエスト（同期処理）
    response = httpx.get(f"{API_URL}{index}")
    return response.status_code

def run_sync():
    """シングルスレッドで処理"""
    for i in range(FILE_COUNT):
        sync_io_task(i)


# ---- マルチスレッド（ThreadPoolExecutor） ----
def run_threads():
    """スレッドを使ってファイルI/OとAPIリクエストを並列処理"""
    with ThreadPoolExecutor(max_workers=FILE_COUNT) as executor:
        results = list(executor.map(sync_io_task, range(FILE_COUNT)))
    return results


# ---- 非同期処理（asyncio + aiofiles + httpx） ----
async def async_io_task(index):
    """非同期でファイルI/OとAPIリクエストを実行"""
    filename = f"async_test_file_{index}.txt"

    # 非同期ファイル書き込み
    async with aiofiles.open(filename, "wb") as f:
        await f.write(os.urandom(FILE_SIZE))

    # 非同期ファイル読み込み
    async with aiofiles.open(filename, "rb") as f:
        await f.read()

    os.remove(filename)

    # 非同期APIリクエスト
    async with httpx.AsyncClient() as client:
        response = await client.get(f"{API_URL}{index}")

    return response.status_code

async def run_async():
    """asyncioを使って非同期で処理"""
    tasks = [async_io_task(i) for i in range(FILE_COUNT)]
    results = await asyncio.gather(*tasks)
    return results


# ---- 実行テスト ----
if __name__ == "__main__":
    print("Starting synchronous tasks (Single Thread)...")
    start = time.perf_counter()
    run_sync()
    print("Sync time: {:.4f} sec".format(time.perf_counter() - start))

    print("-" * 100)

    print("Starting file I/O tasks with threads...")
    start = time.perf_counter()
    run_threads()
    print("Threads time: {:.4f} sec".format(time.perf_counter() - start))

    print("-" * 100)

    print("Starting file I/O tasks with asyncio...")
    start = time.perf_counter()
    asyncio.run(run_async())
    print("Async time: {:.4f} sec".format(time.perf_counter() - start))
```

#### 実験結果

```bash
$ rye run python 03_async.py
Starting synchronous tasks (Single Thread)...
Sync time: 7.4968 sec
----------------------------------------------------------------------------------------------------
Starting file I/O tasks with threads...
Threads time: 1.0663 sec
----------------------------------------------------------------------------------------------------
Starting file I/O tasks with asyncio...
Async time: 1.4586 sec
```

結果としては、予想通り。I/O バウンドな処理では GIL の影響を受けないため、マルチスレッドが一番高速。  
非同期処理は、ディスクの書き込みの待ち時間を有効活用していると考えられる。
今回の処理では、マルチスレッドとシングルスレッド(非同期処理)で、0.5sec ほどの差になっていることから非同期処理が効率的に処理していることがわかる。

## 検証 ⑤

CPU バウンドな処理に対して、`シングルスレッド（同期処理）`、`マルチスレッド`、`マルチプロセス`、`シングルスレッド(非同期処理)`(←`New`)を比較。  
非同期処理は、CPU バウンドな処理に対しては効果がないことを確認する。

予想としては以下。

- 速い順に`マルチプロセス`、`シングルスレッド（同期処理）`or`シングルスレッド(非同期処理)`、`マルチスレッド`
- 非同期処理は、CPU バウンドな処理に対してあまり効果がないため、同期処理と同程度と予想。

#### プログラム

```python
import time
import asyncio
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor


def cpu_bound_task(n: int) -> int:
    """CPU負荷の高いタスク"""
    total = 0
    for _ in range(n):
        total += 1
    return total


def run_threads(n: int, repeat: int):
    """スレッドを使ってCPUバウンドな処理を実行"""
    with ThreadPoolExecutor(max_workers=n) as executor:
        results = list(executor.map(cpu_bound_task, [repeat] * n))
    return results


def run_processes(n: int, repeat: int):
    """プロセスを使ってCPUバウンドな処理を実行"""
    with ProcessPoolExecutor(max_workers=n) as executor:
        results = list(executor.map(cpu_bound_task, [repeat] * n))
    return results


async def async_cpu_task(n: int):
    """非同期でCPUバウンドな処理を実行（GILの影響で並列化されない）"""
    return cpu_bound_task(n)  # 通常の関数をそのまま呼ぶ


async def run_async(n: int, repeat: int):
    """asyncio で CPU バウンドタスクを実行（並列化されないことを確認）"""
    tasks = [async_cpu_task(repeat) for _ in range(n)]
    results = await asyncio.gather(*tasks)
    return results


if __name__ == "__main__":
    num_tasks = 4  # 並列に実行する数（CPUコア数と同じくらいが良い）
    repeat = 10**7  # 負荷をかけるためのループ回数

    print("Baseline (single-threaded)...")
    start = time.perf_counter()
    for _ in range(num_tasks):
        cpu_bound_task(repeat)
    print("Baseline time: {:.4f} sec".format(time.perf_counter() - start))

    print("-" * 100)

    print("Starting CPU-bound tasks with threads...")
    start = time.perf_counter()
    run_threads(num_tasks, repeat)
    print("Threads time: {:.4f} sec".format(time.perf_counter() - start))

    print("-" * 100)

    print("Starting CPU-bound tasks with processes...")
    start = time.perf_counter()
    run_processes(num_tasks, repeat)
    print("Processes time: {:.4f} sec".format(time.perf_counter() - start))

    print("-" * 100)

    print("Starting CPU-bound tasks with asyncio (Expected to be slow)...")
    start = time.perf_counter()
    asyncio.run(run_async(num_tasks, repeat))
    print("Async time: {:.4f} sec".format(time.perf_counter() - start))
```

#### 実験結果

```bash
rye run python 04_async_cpu_bound.py
Baseline (single-threaded)...
Baseline time: 2.0004 sec
----------------------------------------------------------------------------------------------------
Starting CPU-bound tasks with threads...
Threads time: 3.5352 sec
----------------------------------------------------------------------------------------------------
Starting CPU-bound tasks with processes...
Processes time: 0.7314 sec
----------------------------------------------------------------------------------------------------
Starting CPU-bound tasks with asyncio (Expected to be slow)...
Async time: 1.7862 sec
```

順番は概ね予想通り。ただ、非同期処理が同期処理より 0.2sec ほど速い。  
`asyncio`のイベントループがタスクを適切にスケジューリングしたため`for`より効率的だった可能性。  
例えば、純粋なシングルスレッドの for ループでは 完全に直列 なのに対し、asyncio のイベントループでは Python の内部スケジューラが細かく切り替えられる。
