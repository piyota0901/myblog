---
title: "プロセスとスレッド"
date: 2025-03-15T22:41:00+09:00
draft: false
categories: ["OS", "CS", "python"]
---

OS において基本的なプロセスとスレッドをぼんやりと理解していたので勉強した。

## 用語

---

![](./foundation.svg)

- アプリケーションは、1 つ以上のプロセスで構成される。
- プロセスを簡単に説明すると、**実行中のプログラム**。
- スレッドは、プロセスの中で実際に動作する処理の単位。OS はスレッド毎に CPU の処理時間（タイムスライス）を割り当てる。

### プロセス

---

プロセスは、プログラムの実行に必要なリソースを提供（＝環境）する。 (OS から割り当てられる)

**管理するリソース一覧**

![](./process.svg)

割り当てられるメモリ領域の詳細は以下

| 領域名                           | 説明                                                                                     |
| -------------------------------- | ---------------------------------------------------------------------------------------- |
| コード領域（テキストセクション） | 実行可能なコード（Python のインタプリタのバイナリ）を管理する領域                        |
| データ領域                       | グローバル変数と静的変数が保存される領域。                                               |
| スタック領域                     | 関数呼び出しに関する情報（パラメータ、リターンアドレスなど）、ローカル変数を管理する領域 |
| ヒープ領域                       | 動的メモリ割り当てのための領域。                                                         |
| レジスタ領域                     | CPU が即座に処理するデータを保存する非常に小さな領域                                     |

各プロセスは、独立したメモリアドレス空間を持っている。共有していない。

## マルチプロセス

---

その名の通り、複数のプロセスで動作するアプリケーションのこと。  
（例: Chrome。タブ毎にプロセスを作成しているためマルチプロセス）

上記の`プロセス`の説明から互いに独立しているため、相互にやり取りする場合には
プロセス間通信（IPC: Inter-Process Communication）にて行う。
Chrome の場合、ログイン情報などがタブが変わっても保持されている場合があり、
それは IPC によってプロセス間の通信が成り立っているためである。

https://developer.chrome.com/blog/inside-browser-part1?hl=ja

## マルチスレッド

---

マルチスレッドプロセス（複数のスレッドを持つプロセス）では、複数のプロセスが同時に動作することができる。
各スレッドは、独自の`スタック領域`と`レジスタ`（と`スレッドローカルストレージ`）を持っている。  
逆に、他の`コード領域`、`データ領域`、`ヒープ領域`などの他のリソースはスレッド間で共有される。  
同じプロセス内のそれぞれのスレッドで同じメモリアドレス空間を共有しているため、スレッド間の通信は IPC と比較すると効率がいい。**ただし、その分他のスレッドに影響を与える可能性がある**。

![](./thread.svg)

`ヒープ領域`の共有のイメージ

動的に作成されたオブジェクト（インスタンスやリスト）がヒープに配置される。  
そのため複数のスレッドからアクセス可能になっている。

## 並行処理と並列処理

---

![](./concurrency_and_parallelism.svg)

**並行処理**

ある時間の間で、複数のタスクを同時に処理することを意味する。  
イメージとしては、 シングルスレッドで、処理 A と処理 B を切り替えながら実行する。物理的には、1 度に 1 つの処理しか行っていないが、高速に切り替えることで、あたかも同時に進んでいるように見える。

スレッドを切り替える際（コンテキストスイッチ）に、スレッドが持つレジスタ情報・キャッシュなどの情報のロードが発生するためのオーバヘッドが発生する。

ただし、スレッドに限った話ではない。プロセスでも同様。  
また、CPU がマルチコアどうかなどハードウェアにも依存する。

| 形態                                             | CPU コア数   | 並行 or 並列                  | 特徴・オーバーヘッド                                                                                                                                                            | イメージ図                                                                                                          |
| ------------------------------------------------ | ------------ | ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| **シングルプロセス + シングルスレッド**          | **1 コア**   | **なし (単独実行)**           | - 1 つの処理のみ、切り替え不要。<br>- 最も軽量だが同時処理は不可。                                                                                                              | [CPU1] -> [Thread(唯一)]                                                                                            |
| **シングルプロセス + シングルスレッド**          | **複数コア** | **1 コア分のみ活用**          | - スレッド 1 つなので 1 コアしか使えず、他コアは遊ぶ。<br>- 並行も並列も実質起こらない。                                                                                        | [CPU1] -> [Thread(唯一)]<br>[CPU2] -> (idle)                                                                        |
| **シングルプロセス + マルチスレッド**            | **1 コア**   | **並行 (Concurrency)**        | - スレッド間を高速切り替えして疑似的並行。<br>- コンテキストスイッチオーバーヘッド。<br>- 同一メモリ空間でデータ競合に注意（ロック等）。                                        | [CPU1] -> [T1 -> T2 -> T1 ...]<br>(タイムスライスで切り替え)                                                        |
| **シングルプロセス + マルチスレッド**            | **複数コア** | **並列 (Parallelism) 可能**   | - 複数コアなら本当の同時実行が可能。<br>- スレッド間でメモリ共有するため競合注意。<br>- 通信は高速（同じプロセス空間）。                                                        | [CPU1] -> [Thread1]<br>[CPU2] -> [Thread2]<br>(同時に動く)                                                          |
| **マルチプロセス (各プロセス シングルスレッド)** | **1 コア**   | **並行 (Concurrency)**        | - プロセスを切り替えながら並行処理。<br>- プロセス間はメモリ分離（安全性高い）。<br>- コンテキストスイッチはスレッド切り替えより重め。                                          | [CPU1] -> [P1] -> [P2] -> [P1] ...<br>(プロセスごとに切り替え)                                                      |
| **マルチプロセス (各プロセス シングルスレッド)** | **複数コア** | **並列 (Parallelism) 可能**   | - 複数コアがあれば物理的に同時実行可。<br>- プロセス間通信(IPC)のコスト、リソース分離で安全性高い。                                                                             | [CPU1] -> [Process1]<br>[CPU2] -> [Process2]<br>(同時に走る)                                                        |
| **マルチプロセス + マルチスレッド**              | **1 コア**   | **並行 (Concurrency)**        | - 各プロセスが複数スレッドを持つが、1 コアしかないので本当の並列にはならず、OS がプロセス＆スレッド両方を高速切り替え。<br>- コンテキストスイッチ頻度が高く、オーバーヘッド大。 | [CPU1] -> [ProcA-T1] -> [ProcA-T2] -> [ProcB-T1] -> ...<br>(プロセス切り替え + スレッド切り替え)                    |
| **マルチプロセス + マルチスレッド**              | **複数コア** | **並列 (Parallelism) 最大化** | - 各プロセスが複数スレッドを持ち、それぞれが複数コアを活用可能。<br>- 理想的には最も多くのタスクを同時実行できる一方、プロセス間通信やスレッド管理のオーバーヘッドが高い。      | [CPU1] -> [ProcA-T1], [ProcA-T2]<br>[CPU2] -> [ProcB-T1], [ProcB-T2]<br>(同時並列 + マルチプロセス・マルチスレッド) |

### 具体例

| カテゴリ       | 技術                                                    | 特徴                                                                                                                                               | 例                                                                                      |
| -------------- | ------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| **並行処理**   | **Node.js (Event Loop)**                                | - シングルスレッド + イベントループで**非同期 I/O**を並行実行<br>- I/O バウンドに強く、ブロッキングが起きにくい                                    | `fs.readFile()` などの非同期 I/O、`http`サーバー                                        |
|                | **Python `asyncio`**                                    | - **シングルスレッド + イベントループ**<br>- `async`/`await` で I/O 待ちを切り替えて効率化<br>- GIL の制約により CPU バウンドには向かない          | `asyncio.create_task()`、`await asyncio.sleep()`                                        |
| **並列処理**   | **Python `multiprocessing`**                            | - **複数プロセス**を立ち上げるため、GIL を回避<br>- CPU バウンドタスクを真の並列で高速化<br>- プロセス間通信(IPC)が必要                            | `multiprocessing.Pool()`, `pool.map()`, CPU バウンド計算                                |
|                | **Python `concurrent.futures` (ProcessPoolExecutor)**   | - `multiprocessing`同様に**複数プロセス**で並列化<br>- シンプルな API (`submit()`, `map()`)<br>- 同時に複数プロセスが実行され CPU コアを活用できる | `ProcessPoolExecutor` で `executor.submit(cpu_bound_task)`                              |
|                | **Go 言語のゴルーチン + ランタイムスケジューラ** (参考) | - Go 言語は軽量スレッド(ゴルーチン)を多数生成し、ランタイムが複数コアを活用<br>- CPU バウンドも並列化しやすい<br>- メモリ安全性・GC あり           | `go func() {...}` で並列タスク<br>`GOMAXPROCS` でコア数設定                             |
| **補足的技術** | **Python `concurrent.futures` (ThreadPoolExecutor)**    | - **マルチスレッド**だが GIL の制約で CPU バウンドは並列化できない<br>- I/O 待ちが多いタスクを複数スレッドで並行処理する場面で有効                 | `ThreadPoolExecutor` で `executor.submit(io_bound_task)`                                |
|                | **Python `run_in_executor()` / `to_thread()`**          | - `asyncio` のイベントループ上から、別スレッドや別プロセスへブロッキング処理をオフロード<br>- シングルスレッドの制約を補うために使用               | `await loop.run_in_executor(None, blocking_func)`<br>`asyncio.to_thread(blocking_func)` |

## 非同期処理

---

入出力（I/O）待ちなどでブロックすることなく、待機中に他のタスクの処理を進める仕組み。1 つのスレッド（プロセス）内で複数のタスクを並行処理するイメージ。

Python の`asyncio`ライブラリは、シングルスレッドのイベントループによって非同期処理を実現している。

**コルーチン**

コルーチンは、途中で処理を一時停止・再開できる特別な関数。  
Python の場合、ジェネレータ(`yeild`)によって実現されている。

## GIL

---

Global Interpreter Lock の略で、Python に導入されているロック機構。  
Python の C 言語で実装された部分が、スレッドセーフではないため安全性の核を目的として実行する際に排他的に 1 スレッドしか実行できないように制限をかける仕組みが GIL。
これにより、Python では**同時に 1 スレッドしか実行できない**という制約がある。（GIL を獲得したスレッドのみしか実行できない）  
つまり、マルチスレッドで CPU 処理を伴う処理は並列化できないということ。結果として、CPU バウンドな処理に対してマルチスレッドで性能を上げることは難しい。

ただし、（GIL の影響を受けない）I/O バウンドな処理に対してマルチスレッドは有効。

**Python では、GIL があるので、CPU バウンドの場合は、まずはマルチプロセスを検討することが良い**

**バウンド(bound)とは？**

「バウンド (bound)」は、「～に制約されている」「～に縛られている」「～に律速されている（依存している）」という意味になる。

- **CPU バウンド（CPU-bound）**

  - 性能が CPU 処理能力に縛られている状態
  - 処理を速くするには、CPU の処理能力がボトルネックになっている。

- **I/O バウンド（I/O-bound）**
  - 処理性能を制限しているのが I/O（入出力）待ち時間。
  - CPU の性能が余っていても、入出力（DB、ファイル、ネットワーク）の速度に制限される。

## Web アプリケーション

---

NGINX などのリバースプロキシを利用している場合、NGIX 側がマルチプロセスでも、Web アプリケーション側がシングルプロセス・シングルスレッド・同期処理だとパフォーマンス向上はできない。
Web アプリケーション側でも、マルチスレッド・マルチプロセスといった並列処理を入れる、並行処理のための非同期処理を入れるなどが必要。

### WSGI（ウィズギー）/ASGI（アズギー）

Python の場合は、Web サーバー（NGINX、Apache など）と WSGI アプリケーション（Web Server Gateway Interface）Web アプリケーションが明確に分離されている。そのため、パフォーマンス向上を考える場合は、それぞれの設定を検討する。

**WSGI**

同期処理を前提とした仕様。Flask, Django などのフレームワークで使用されている。WSGI サーバーとしては、Gunicorn、uWSGI などがある。Gunicorn（WSGI サーバー）では、複数のワーカー（プロセス）を設定して、同時に複数のリクエストを処理できるようにしている。プロセスマネージャーとして動作。

{{< alert >}}
Flask の`flask run`で動く開発用の簡易サーバーは、`Werkzeug`が提供している。あくまで付属的なサーバー機能のため本番環境などでは、Gunicorn/uWSGI などが採用される。
{{< /alert >}}

**ASGI**

WSGI との互換性を持ち、非同期処理をサポートしたサーバー。ASGI サーバーとしては、NGINX Unit, Uvicorn などが挙げられる。ASGI に準拠した Web アプリケーションフレームワークとしては、FastAPI や Quart が挙げられる。

Gunicorn のワーカープロセスとして Uvicorn を動かす構成とする場合もある。[FastAPI | Gunicorn による Uvicorn のワーカー・プロセスの管理](https://fastapi.tiangolo.com/ja/deployment/server-workers/#gunicornuvicorn)を参照。

また、Uvicorn 自体にも複数のワーカープロセスを起動して実行するオプションもある。ただし、Gunicorn よりも機能は制限されている。[FastAPI | Uvicorn とワーカー](https://fastapi.tiangolo.com/ja/deployment/server-workers/#uvicorn)を参照。

**コンテナでのデプロイ**

コンテナでデプロイする場合は、Gunicorn を使用したマルチプロセスとするかは要件次第。1 プロセス=1 コンテナに基づくのであれば、コンテナごとに Uviron のプロセスは 1 つにする（=複数コンテナを立ち上げて、ワーカーを増やす）。複数コンテナの管理は、k8s や compose などの役割になるのでシンプルになる。[FastAPI | 1 コンテナにつき 1 プロセス](https://fastapi.tiangolo.com/ja/deployment/docker/#11)を参照。

https://learn.microsoft.com/ja-jp/windows/win32/procthread/processes-and-threads
