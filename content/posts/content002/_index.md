---
title: "pip installの動作"
date: "2023-11-19T15:57:01+09:00"
draft: false
tags: ["pip"]
categories: ["python"]
---

`pip`は「つかえればいいや！」で過ごしていたが、今回困りそうなことがあったので深掘りしてみた。

## 前提条件

---

- Ubuntu 22.04LTS
- 開発用ユーザー(taro: sudo 有) 開発サーバー
- アプリ用ユーザー(app) 本番サーバー
- 言語は Python

## 困ったこと

---

アプリを動作させる app ユーザーが使用できるライブラリのパスと  
開発用ユーザーが使用できるライブラリのパスが異なっていた。  
両方とも`pip`でインストールしたのになぜ・・・が深堀の背景。

- 開発サーバー：taro

  ```bash
  $ pip show requests
  ・・・
  Location: /usr/local/lib/python3.9/site-packages <- システムにインストールされている
  ・・・
  ```

- 本番サーバー：app
  ```
  $ pip show requests
  ・・・
  Location: /home/app/.local/lib/python3.9/site-packages <- ユーザーのホームディレクトリにインストールされている
  ・・・
  ```

## 結論

---

- 仮想環境(`venv`)を利用するほうが良い
- `pip`のデフォルトのインストール先はシステム
  - `/usr/local/lib/python3.9/site-packages`
- デフォルトのインストール先にパーミッションがない場合、ユーザーのホームディレクトリにインストール
  - `/home/app/.local/lib/python3.9/site-packages`
  - インストール時に`Defaulting to user installation because normal site-packages is not writeable`というメッセージが出力される
  - `pip`の仕様。どのバージョンからかは不明
- `sudo pip install <package名>`とすると root 権限となりシステムにインストール
- システムに一度インストールされたライブラリをユーザーのホームディレクトリにインストールし直したい
  - `--target /home/app/.local/`と指定する
  - `--target`オプションがないと、システムに既にインストールされているためスキップされる

## 調べた内容

---

### 開発用ユーザー taro で普通にインストール

アプリ用ユーザー app と同様にホームディレクトリにインストールされる

```bash
taro@db85db87e1dd:~$ pip install requests
Defaulting to user installation because normal site-packages is not writeable
Collecting requests
・・・
taro@db85db87e1dd:~$ pip show requests
・・・
Location: /home/taro/.local/lib/python3.9/site-packages
・・・
```

### 開発用ユーザーで sudo でインストール

システムにインストールされる

```bash
taro@db85db87e1dd:~$ pip uninstall requests
[sudo] password for taro:                   ← Defaulting to user installation・・・は表示されない
Collecting requests
Using cached requests-2.31.0-py3-none-any.whl (62 kB)
・・・
Installing collected packages: requests
Successfully installed requests-2.31.0
taro@db85db87e1dd:~$
taro@db85db87e1dd:~$
taro@db85db87e1dd:~$ pip show requests
・・・
Location: /usr/local/lib/python3.9/site-packages ← システムにインストールされた
Requires: certifi, charset-normalizer, idna, urllib3
Required-by:
```

### システムにインストールされた状態でユーザーインストールを試みる

ユーザーインストールはスキップされる

```bash
taro@db85db87e1dd:~$ pip install requests --user
Requirement already satisfied: requests in /usr/local/lib/python3.9/site-packages (2.31.0)
Requirement already satisfied: charset-normalizer<4,>=2 in /usr/local/lib/python3.9/site-packages (from requests) (3.3.2)
Requirement already satisfied: urllib3<3,>=1.21.1 in /usr/local/lib/python3.9/site-packages (from requests) (2.1.0)
Requirement already satisfied: certifi>=2017.4.17 in /usr/local/lib/python3.9/site-packages (from requests) (2023.11.17)
Requirement already satisfied: idna<4,>=2.5 in /usr/local/lib/python3.9/site-packages (from requests) (3.4)
WARNING: You are using pip version 22.0.4; however, version 23.3.1 is available.
You should consider upgrading via the '/usr/local/bin/python -m pip install --upgrade pip' command.
```

### `--target`オプションでユーザーインストール先を指定してみる

インストールされる + ユーザーインストールのパスが表示される

```bash
taro@db85db87e1dd:~$ pip install requests --target /home/taro/.local/lib/python3.9/site-packages/ # site-packagesまで指定必須
Collecting requests
Using cached requests-2.31.0-py3-none-any.whl (62 kB)
Collecting charset-normalizer<4,>=2
Downloading charset_normalizer-3.3.2-cp39-cp39-manylinux_2_17_x86_64.manylinux2014_x86_64.whl (142 kB)
    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 142.3/142.3 KB 1.5 MB/s eta 0:00:00
Collecting urllib3<3,>=1.21.1
Downloading urllib3-2.1.0-py3-none-any.whl (104 kB)
    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 104.6/104.6 KB 6.6 MB/s eta 0:00:00
Collecting idna<4,>=2.5
Downloading idna-3.4-py3-none-any.whl (61 kB)
    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 61.5/61.5 KB 4.1 MB/s eta 0:00:00
Collecting certifi>=2017.4.17
Downloading certifi-2023.11.17-py3-none-any.whl (162 kB)
    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 162.5/162.5 KB 7.7 MB/s eta 0:00:00
Installing collected packages: urllib3, idna, charset-normalizer, certifi, requests
Successfully installed certifi-2023.11.17 charset-normalizer-3.3.2 idna-3.4 requests-2.31.0 urllib3-2.1.0
taro@db85db87e1dd:~$
taro@db85db87e1dd:~$
taro@db85db87e1dd:~$ pip show requests
taro@db85db87e1dd:~$ pip show requests
Name: requests
・・・
Location: /home/taro/.local/lib/python3.9/site-packages
・・・
```

## 読み込むライブラリの優先度

---

### 優先度の整理

`pip show requests`を実行したとき、システムにインストールされているにも関わらず  
ユーザーのホームディレクトリにインストールされた方が表示された  
⇒ システムよりユーザーインストールが優先されていることがわかる

### 優先度の確認

ライブラリはインポートシステムの優先度に従っている。

優先度の確認方法  
リストの上から優先順位が高い

```bash
taro@db85db87e1dd:~$ python -c "import sys;import pprint;pprint.pprint(sys.path,indent=4)"
[   '',
    '/usr/local/lib/python39.zip',
    '/usr/local/lib/python3.9',
    '/usr/local/lib/python3.9/lib-dynload',
    '/home/taro/.local/lib/python3.9/site-packages',
    '/usr/local/lib/python3.9/site-packages']
```

優先度は上から順に高い。カレントディレクトリが高いためカレントディレクトリと同階層のモジュールが import できる。

| パス                                            | 説明                 |
| ----------------------------------------------- | -------------------- |
| ''                                              | カレントディレクトリ |
| '/usr/local/lib/python39.zip'                   | 標準ライブラリ       |
| '/usr/local/lib/python3.9'                      | 標準ライブラリ       |
| '/usr/local/lib/python3.9/lib-dynload'          | 標準ライブラリ       |
| '/home/taro/.local/lib/python3.9/site-packages' | ユーザーインストール |
| '/usr/local/lib/python3.9/site-packages'        | システムインストール |

#### 補足 ①

PYTHONPATH を設定した場合はカレントディレクトリの次の優先度となる

```bash
taro@db85db87e1dd:~$ PYTHONPATH=/home/hoge/ python -c "import sys;import pprint;pprint.pprint(sys.path,indent=4)"
[   '',
    '/home/hoge', # ← 2番目に優先される
    '/usr/local/lib/python39.zip',
    '/usr/local/lib/python3.9',
    '/usr/local/lib/python3.9/lib-dynload',
    '/home/taro/.local/lib/python3.9/site-packages',
    '/usr/local/lib/python3.9/site-packages']
```

### 補足 ②

仮想環境(`venv`)の場合は、システムやユーザーインストールは含まれない  
ライブラリについては完全に独立した環境ということがわかる。

```bash
(.venv) taro@db85db87e1dd:~$ python -c "import sys;import pprint;pprint.pprint(sys.path,indent=4)"
[   '',
    '/usr/local/lib/python39.zip',
    '/usr/local/lib/python3.9',
    '/usr/local/lib/python3.9/lib-dynload',
    '/home/taro/.venv/lib/python3.9/site-packages']
# 以下が含まれていないことがわかる
# '/home/taro/.local/lib/python3.9/site-packages': ユーザーインストール
# '/usr/local/lib/python3.9/site-packages': システムインストール
```
