---
title: "ブロッキングI/OとノンブロッキングI/O"
date: 2025-04-03T21:25:54+09:00
draft: false
description: ""
categories: ["OS"]
---

ブロッキング I/O とノンブロッキング I/O について勉強したので記録 ✍️

## ブロッキング I/O とノンブロッキング I/O とは・・・？

---

簡単に言うと**I/O 処理が発生した場合にプログラムが処理を待つか/待たないかの違い**。

**ブロッキング I/O**

I/O 操作が完了するまでプログラムが待機する（＝これを`ブロック`と呼ぶ）  
例えば、ネットワーク通信でデータ受信を待っているとき、データを受信するまで処理が止まる。

**ノンブロッキング I/O**

I/O 操作の完了を待たずに処理が進む。  
例えば、ネットワーク通信でデータを受信していない場合は、「まだ読み取れない」旨の結果が返る。

ノンブロッキング I/O は、イベントループと組み合わせて非同期処理として実装することで、複数の I/O を効率よく並行処理できる。

## イベントループとは・・・？

---

単一スレッドで非同期処理を実現するための仕組み。  
イベントループには以下のメリットがある。

- **スレッドのオーバヘッドを抑制できる**

  複数のスレッドを立てずに、多数の I/O やタイマーを扱える

- **ロック競合の回避**

  単一スレッドで動作するため、共有リソースへの排他制御がシンプル

- **高いスループットと低レイテンシ**

  ブロッキングを最小限にし、処理待ちの時間を他のタスクに振り分けることで効率的に処理が行える

イベントループ自体は、**何かのイベントが発生したら、それに対応する処理を順番に実行する仕組み**になる。  
ネットワーク通信といった I/O に限らず、定期実行やユーザー操作など、イベント駆動の処理全般を扱うための仕組みでもある。

### イベントループの基本構造

1. **イベントの登録**

   「イベントに対応する処理」をあらかじめ登録する

   - ソケットが読み込み可能となったらこの関数を呼ぶ
   - 一定時間後にこの処理を実行する
   - ボタンが押されたたらこの関数を実行する

1. **イベントの監視**

   登録されたイベントが実際に発生するのを**ノンブロッキング**で待つ  
   この時、OS の機能（例:`select` / `poll` / `epoll`）を使用して「今、反応すべきイベントがあるか？」を監視している

1. **イベントの実行**

   何らかのイベントが発生すると、それに対応する処理（コールバック関数など）を呼び出す  
   この時、イベントループは「処理が終わるまで他のことをしない」のではなく、**処理が終わった後にすぐ次のイベントが来ていないか確認する**というサイクルを繰り返す

1. **次のイベントへ**

   イベントの処理が終わると、監視フェーズに戻り、次のイベントを待つ  
   この流れを繰り返していく（ループ）

**ノンブロッキング I/O の場合**

ノンブロッキング I/O の場合は、**I/O の準備ができたタイミングで処理を実行するよう、イベントループにあらかじめ処理（コールバックなど）を登録しておく**。

- ネットワーク通信によるデータ受信を「今すぐにできなければすぐ戻る（ブロックしない）」というモードで実行する

- 読み書きできるようになるまで、イベントループが OS の監視機能（`select` / `poll` / `epoll` など）を使って待ち受け、準備が整ったタイミングでコールバックなどの処理を呼び出す

## ファイル操作における注意点

---

ファイル操作について、OS（特に Unix 系）では`select`や`epoll`での監視に対応していないため、  
ノンブロッキング I/O ＋ イベントループによる非同期処理には向いていない。

- 通常のファイルは「いつでも読み書き可能なもの」として扱われるため、準備状態を待つ意味がない。

- `select`や`epoll`に渡すと「常に準備完了」と見なされ非同期に**待ち続ける**ことができない。

ファイル操作における「ノンブロッキング I/O」は、技術的には存在するが、実用上意味を持たないケースが多く、イベントループでは扱えない。

Linux では、`O_DIRECT`や`io_uring`によって非同期ファイル IO が可能。  
ただし、低レベル API の知識やクロスプラットフォームでは使いにくいなどの課題がある。

**言語ごとのファイル操作に関する非同期処理方法**

以下は、言語ごとのファイル I/O における非同期処理の方法をまとめた表です。

| 言語                       | ファイルの非同期処理方法                                               |
| -------------------------- | ---------------------------------------------------------------------- |
| **Python**                 | `aiofiles`：スレッドプールで裏で実行し、`await` で非同期的に扱う       |
| **Node.js**                | `fs.readFile`：libuv が内部でスレッドに投げて非同期化                  |
| **Rust**                   | `tokio::fs`：スレッドでブロッキング I/O をオフロードして非同期的に動作 |
| **JavaScript（ブラウザ）** | File API は基本ブロッキング。`FileReader` など一部は非同期対応         |

[aiofiles](https://github.com/Tinche/aiofiles#:~:text=aiofiles%20helps%20with%20this%20by%20introducing%20asynchronous%20versions%20of%20files%20that%20support%20delegating%20operations%20to%20a%20separate%20thread%20pool.) は、内部でスレッドプールを利用して、ブロッキングなファイル操作をバックグラウンドで実行する。  
このため、表面的には `await` による非同期処理のように見えますが、実際には **裏でマルチスレッドが動いてる**。

たとえば `fp.read()` は本質的には同期的だが、それを別スレッドで実行し、処理が終わるとコルーチンが再開されるため、メインスレッドの処理を止めることなくファイルを読み込める。

## Python による非同期処理の検証

---

このセクションでは、`requests`（同期）と `httpx.AsyncClient`（非同期）を用いて、  
**ブロッキング I/O** と **ノンブロッキング I/O + イベントループによる非同期処理** の違いを、実際の Web API 呼び出しで比較・検証する。

**requests と httpx.AsyncClient の比較表**

| 項目               | requests                       | httpx.AsyncClient                     |
| ------------------ | ------------------------------ | ------------------------------------- |
| 処理方式           | 同期（ブロッキング I/O）       | 非同期（ノンブロッキング I/O）        |
| イベントループ対応 | ❌ 対応していない              | ✅ asyncio 上で動作                   |
| 並行処理           | スレッドやプロセスで対応が必要 | `asyncio.gather()` 等で簡単に並行実行 |
| コードの簡単さ     | とても簡単（初心者向け）       | 非同期構文（async/await）が必要       |
| 適しているケース   | 単純なスクリプト、同期処理     | 多数のリクエストを並行で扱いたい場面  |

※Python では `async def` で定義された非同期関数を「コルーチン」と呼びます。

### requests vs httpx.AsyncClient

[JSONPlaceholder](https://jsonplaceholder.typicode.com/)を使用する。  
5 リクエストを同期処理/非同期処理して、実行時間の違いを確認する。

`httpx.AsyncClient`による非同期処理のほうが速い想定。

**requests**

<details>
    <summary>main.py</summary>

    ```python
    import time
    import requests


    def fetch_todos_sync():
        """同期処理でのAPIリクエスト"""
        start = time.perf_counter()
        for i in range(1, 6):
            response = requests.get(f"https://jsonplaceholder.typicode.com/todos/{i}")
            print(f"Todo {i}: {response.json()}")
        end = time.perf_counter()
        print(f"同期処理の実行時間: {end - start:.2f}秒")


    if __name__ == "__main__":
        fetch_todos_sync()
    ```

</details>

実行結果

```bash
$ uv run python main.py
Todo 1: {'userId': 1, 'id': 1, 'title': 'delectus aut autem', 'completed': False}
Todo 2: {'userId': 1, 'id': 2, 'title': 'quis ut nam facilis et officia qui', 'completed': False}
Todo 3: {'userId': 1, 'id': 3, 'title': 'fugiat veniam minus', 'completed': False}
Todo 4: {'userId': 1, 'id': 4, 'title': 'et porro tempora', 'completed': True}
Todo 5: {'userId': 1, 'id': 5, 'title': 'laboriosam mollitia et enim quasi adipisci quia provident illum', 'completed': False}
同期処理の実行時間: 1.77秒
```

おおよそ１リクエスト`1.77 / 5 = 0.354`[sec]ほど掛かる。  
非同期処理では、並行処理するため、5 リクエスト全体で 0.35~0.5[sec]程になる予想。

**httpx.AsyncClient**

<details>
    <summary>main.py</summary>

    ```python
    import time

    import asyncio
    import httpx


    async def fetch_todos_async():
        start = time.time()
        async with httpx.AsyncClient() as client:
            tasks = [
                client.get(f"https://jsonplaceholder.typicode.com/todos/{i}")
                for i in range(1, 6)
            ]
            responses = await asyncio.gather(*tasks)
            for i, response in enumerate(responses, start=1):
                print(f"Post {i}: {response.json()}")
        duration = time.time() - start
        print(f"[非同期] 所要時間: {duration:.2f}秒")


    if __name__ == "__main__":
        asyncio.run(fetch_todos_async())
    ```
    実行可能な awaitable（コルーチン）で、`asyncio.gather()`に渡す

</details>

実行結果

```bash
$ uv run python main.py
Post 1: {'userId': 1, 'id': 1, 'title': 'delectus aut autem', 'completed': False}
Post 2: {'userId': 1, 'id': 2, 'title': 'quis ut nam facilis et officia qui', 'completed': False}
Post 3: {'userId': 1, 'id': 3, 'title': 'fugiat veniam minus', 'completed': False}
Post 4: {'userId': 1, 'id': 4, 'title': 'et porro tempora', 'completed': True}
Post 5: {'userId': 1, 'id': 5, 'title': 'laboriosam mollitia et enim quasi adipisci quia provident illum', 'completed': False}
[非同期] 所要時間: 0.49秒
```

約**3.6 倍**高速になったことを確認。並行実行されるため、待ち時間が重ならずに済む。

---

### `aiofiles`の非同期処理

`aiofiles`による非同期処理の効果を確認する

**テストファイルの作成**

100MB のファイルを 5 つ作成する

```bash
$ mkdir -p data
$ for i in {1..5}; do
  base64 /dev/urandom | head -c 100000000 > data/file$i.txt
done
```

**通常のファイル読み込み(同期)**

<details>
    <summary>main.py</summary>

    ```python
    import time


    def read_all_files_sync():
        filenames = [f"data/file{i}.txt" for i in range(1, 6)]
        start = time.time()
        for fname in filenames:
            with open(fname, "r") as f:
                content = f.read()
        duration = time.time() - start
        print(f"[同期処理] 所要時間: {duration:.2f}秒")


    if __name__ == "__main__":
        read_all_files_sync()
    ```

</details>

実行結果

```bash
$ uv run python main.py
[同期処理] 所要時間: 0.56秒
```

**`aiofiles`のファイル読み込み(非同期)**

厳密には非同期ではなくマルチスレッド

<details>
    <summary>main.py</summary>

    ```python
    import time

    import aiofiles
    import asyncio


    async def read_all_files():
        start = time.time()

        filenames = [f"data/file{i}.txt" for i in range(1, 6)]

        tasks = [aiofiles.open(filename, mode="r") for filename in filenames]

        # 各ファイルに対して非同期読み込み処理を組み立て
        async def read(f):
            async with f as file:
                content = await file.read()

        await asyncio.gather(*[read(f) for f in tasks])

        duration = time.time() - start
        print(f"[aiofiles - 非同期] 所要時間: {duration:.2f}秒")


    if __name__ == "__main__":
        asyncio.run(read_all_files())

    ```

</details>

実行結果

```bash
$ uv run python main.py
[aiofiles - 非同期] 所要時間: 0.53秒
```

速さについては、通常のファイル読み込みとほとんど変わらない。
スレッドを使って非同期的に見せかけているのみなので、速くはならない。

**では何のために`aiofiles`を使うのか**

ネットワーク通信などが非同期で進んでいる時に、同期的なファイル操作を入れてしまうと、イベントループが止まってしまってパフォーマンスが落ちる。  
だから`aiofiles`を使って、ファイル操作も非同期風にしておくことで、アプリ全体のスループットが落ちないようにしている。

## まとめ

非同期処理フレームワーク（例：FastAPI など）では、イベントループをブロックしないことが重要です。同期的な処理（ファイル I/O やデータベースアクセスなど）をそのまま実行すると、イベントループが止まり、他のリクエストや処理に影響を与えてしまう。

そのため、ファイル操作には aiofiles、データベースアクセスには asyncpg や databases ライブラリなど、非同期対応のライブラリを使うことが望ましい。

これにより、非同期アプリケーション全体のスループットと応答性を維持することができる
