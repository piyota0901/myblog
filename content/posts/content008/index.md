---
title: "シンボリックリンクとハードリンク"
date: 2025-04-19T16:18:24+09:00
draft: false
description: ""
categories: ["OS"]
---

## 特徴

### シンボリックリンク

シンボリックリンク（`symbolic link` または `sym link`）は、Linux において特定のファイルやディレクトリを参照するための特殊なファイルのこと。  
Windows におけるショートカット。

- 参照元のファイルやディレクトリが削除されるとリンクが無効になる(`broken link`になる)
- 異なるファイルシステムやパーティション間でも作成可能
- `inode` を消費する
  - `inode`を持つ独立したファイルとして作成される
- パーミッションは、参照元のファイルやディレクトリのパーミッションに従う
  - シンボリックリンクのパーミッションは設定可能だが使用されない（無視される）

## ハードリンク

ハードリンク(`hard link`)は、あるファイルの実体に対して別名（別のパス）を付ける機能。

- 元のファイルと同じ inode 番号を持つ（同じ実体を参照）
- 同じファイルシステム内でのみ作成できる
- 元ファイルを削除しても消えない
- 元ファイルと同じサイズ（コピーではない）
- ディレクトリに対しては作成不可

シンボリックリンクとの違いは、ハードリンクは**ファイルの中身そのものを指している**。

---

## 検証

### シンボリックリンクの動作を検証

Docker コンテナ（Ubuntu24.04）にて検証した

1. シンボリックリンク作成コマンド`ln`のヘルプを確認する

   ```bash
   root@3213642e5f1a:/workspace/prj001# ln --help
   Usage: ln [OPTION]... [-T] TARGET LINK_NAME
   or:  ln [OPTION]... TARGET
   or:  ln [OPTION]... TARGET... DIRECTORY
   or:  ln [OPTION]... -t DIRECTORY TARGET...
   In the 1st form, create a link to TARGET with the name LINK_NAME.
   In the 2nd form, create a link to TARGET in the current directory.
   In the 3rd and 4th forms, create links to each TARGET in DIRECTORY.
   Create hard links by default, symbolic links with --symbolic. # ←デフォルトでは ハードリンクとのこと
    ・・・
   ```

1. シンボリックリンクを作成する

   ```bash
   root@3213642e5f1a:/workspace/prj001# # 元ファイル作成
   root@3213642e5f1a:/workspace/prj001# mkdir -p /home/root && touch /home/root/original.txt
   root@3213642e5f1a:/workspace/prj001# echo "Hello World by original" >> /home/root/original.txt
   root@3213642e5f1a:/workspace/prj001# # シンボリックリンク作成
   root@3213642e5f1a:/workspace/prj001# mkdir -p /workspaces/prj001 && cd /workspaces/prj001
   root@3213642e5f1a:/workspace/prj001# ln --symbolic /home/root/original.txt link.txt
   ```

1. シンボリックリンクを確認する

   ```bash
   root@3213642e5f1a:/workspace/prj001# ll
   total 8
   drwxr-xr-x 2 root root 4096 Apr 19 08:00 ./
   drwxr-xr-x 3 root root 4096 Apr 19 07:36 ../
   lrwxrwxrwx 1 root root   23 Apr 19 08:00 link.txt -> /home/root/original.txt
   ```

   `->`で表示されていることを確認

1. 中身を見てみる

   ```bash
   root@3213642e5f1a:/workspace/prj001# cat link.txt
   Hello World by original
   ```

1. inode の増減を確認する

   ```bash
   root@3213642e5f1a:/workspace/prj001# # 現時点のinode数を確認する
   root@3213642e5f1a:/workspace/prj001# df -i .
   Filesystem       Inodes  IUsed    IFree IUse% Mounted on
   overlay        67108864 752119 66356745    2% /
   root@3213642e5f1a:/workspace/prj001# # シンボリックリンクを新たに作成する
   root@3213642e5f1a:/workspace/prj001# ln --symbolic /home/root/original.txt link2.txt
   root@3213642e5f1a:/workspace/prj001# ll
   total 8
   drwxr-xr-x 2 root root 4096 Apr 19 08:14 ./
   drwxr-xr-x 3 root root 4096 Apr 19 07:36 ../
   lrwxrwxrwx 1 root root   23 Apr 19 08:00 link.txt -> /home/root/original.txt
   lrwxrwxrwx 1 root root   23 Apr 19 08:14 link2.txt -> /home/root/original.txt
   root@3213642e5f1a:/workspace/prj001# df -i .
   Filesystem       Inodes  IUsed    IFree IUse% Mounted on
   overlay        67108864 752128 66356736    2% /
   ```

   `IUsed`を確認すると`752119` -> `752128`に増加している。つまり、シンボリックリンク作成で inode が消費されたことを確認。

   ```bash
   root@3213642e5f1a:/workspace/prj001# # シンボリックリンクを削除する
   root@3213642e5f1a:/workspace/prj001# rm link2.txt
   root@3213642e5f1a:/workspace/prj001# df -i .
   Filesystem       Inodes  IUsed    IFree IUse% Mounted on
   overlay        67108864 752130 66356734    2% /
   ```

   シンボリックリンクを削除したが、`IUSed`が逆に増加した。
   `overlay`ファイルシステムが影響している。削除操作を記録するための新しい inode が生成されるため、トータルで inode が増加する。（削除マーカー（whiteout ファイル）が作成される）
   ※このような挙動は、Docker の `overlay` ファイルシステムにおける「削除の記録＝ whiteout ファイル」が原因。通常の `ext4` 等とは異なる挙動をする点に注意。

1. WSL で試してみる

   ```bash
   $ # ファイルシステムの確認
   $ df -T .
   Filesystem     Type     1K-blocks      Used Available Use% Mounted on
   ・・・
   /dev/sdc       ext4    1055762868  37923448 964135948   4% / # ← ext4であることを確認
   ・・・
   $
   $ touch original.txt # 元ファイル作成
   $ df -i .
   Filesystem       Inodes  IUsed    IFree IUse% Mounted on
   /dev/sdc       67108864 752141 66356727    2% /
   $ ln -s original.txt link.txt
   $ df -i .
   Filesystem       Inodes  IUsed    IFree IUse% Mounted on
   /dev/sdc       67108864 752142 66356723    2% /
   $ rm link.txt
   $ df -i .
   Filesystem       Inodes  IUsed    IFree IUse% Mounted on
   /dev/sdc       67108864 752141 66356723    2% / # 1つ減った
   $ # 数秒待ってもう一度確認
   $ df -i .
   Filesystem       Inodes  IUsed    IFree IUse% Mounted on
   /dev/sdc       67108864 752144 66356720    2% / # 3つ増えた？？？
   $ # 数秒待ってもう一度確認
   $ df -i .
   Filesystem       Inodes  IUsed    IFree IUse% Mounted on
   /dev/sdc       67108864 752145 66356720    2% / # もう1つ増えた？？？
   ```

   **仮想化技術**を使用している場合は、単純な増減ではなさそうと推測

1. パーミッションを確認してみる

   ```bash
   root@3213642e5f1a:/workspace/prj001# # rootユーザーのみ読み取り・書き込み可能に設定
   root@3213642e5f1a:/home/root# chmod 600 original.txt
   root@3213642e5f1a:/home/root# ll
   total 12
   drwxr-xr-x 2 root root 4096 Apr 19 07:35 ./
   drwxr-xr-x 1 root root 4096 Apr 19 08:42 ../
   -rw------- 1 root root   24 Apr 19 07:36 original.txt
   root@3213642e5f1a:/workspace/prj001#
   root@3213642e5f1a:/workspace/prj001#
   root@3213642e5f1a:/workspace/prj001# useradd -m -s /bin/bash bob
   root@3213642e5f1a:/workspace/prj001# su - bob
   bob@3213642e5f1a:~$
   bob@3213642e5f1a:~$ ln -s /home/root/original.txt link.txt
   bob@3213642e5f1a:~$ ll
   total 20
   drwxr-x--- 2 bob  bob  4096 Apr 19 08:43 ./
   drwxr-xr-x 1 root root 4096 Apr 19 08:42 ../
   -rw-r--r-- 1 bob  bob   220 Mar 31  2024 .bash_logout
   -rw-r--r-- 1 bob  bob  3771 Mar 31  2024 .bashrc
   -rw-r--r-- 1 bob  bob   807 Mar 31  2024 .profile
   lrwxrwxrwx 1 bob  bob    23 Apr 19 08:43 link.txt -> /home/root/original.txt # ← リンク自体の権限は全員読み取り・書き込み可能
   bob@3213642e5f1a:~$ cat link.txt
   cat: link.txt: Permission denied
   bob@3213642e5f1a:~$ echo "Hello world by Bob" >> link.txt
   -bash: link.txt: Permission denied
   bob@3213642e5f1a:~$ exit
   root@3213642e5f1a:/home/root# chmod 604 original.txt
   root@3213642e5f1a:/home/root#
   root@3213642e5f1a:/home/root# # 読み取り権限のみ設定してみる
   root@3213642e5f1a:/home/root# chmod 604 original.txt
   total 12
   drwxr-xr-x 2 root root 4096 Apr 19 07:35 ./
   drwxr-xr-x 1 root root 4096 Apr 19 08:42 ../
   -rw----r-- 1 root root   24 Apr 19 07:36 original.txt
   root@3213642e5f1a:/home/root# su - bob
   bob@3213642e5f1a:~$
   bob@3213642e5f1a:~$ cat link.txt
   Hello World by original # 読み取りできたことを確認
   ```

1. 元ファイルを削除した場合の動作を確認

   ```bash
   root@3213642e5f1a:/workspace/prj001# rm /home/root/original.txt
   root@3213642e5f1a:/workspace/prj001# cat hlink.txt
   Hello World by original
   root@3213642e5f1a:/workspace/prj001# cat linx.txt
   cat: linx.txt: No such file or directory
   root@3213642e5f1a:/workspace/prj001#
   ```

### ハードリンクの動作を検証

1. ハードリンクを作成する

   ```bash
   root@3213642e5f1a:/workspace/prj001# ln /home/root/original.txt hlink.txt
   root@3213642e5f1a:/workspace/prj001# ll
   total 12
   drwxr-xr-x 2 root root 4096 Apr 19 08:52 ./
   drwxr-xr-x 3 root root 4096 Apr 19 07:36 ../
   -rw----r-- 2 root root   24 Apr 19 07:36 hlink.txt
   lrwxrwxrwx 1 root root   23 Apr 19 08:00 linx.txt -> /home/root/original.txt
   ```

1. 同じ inode 番号か確認する

   ```bash
   root@3213642e5f1a:/workspace/prj001#
   root@3213642e5f1a:/workspace/prj001# # 同じinode番号になるのか確認
   root@3213642e5f1a:/workspace/prj001# ls -li /home/root/original.txt
   748906 -rw----r-- 2 root root 24 Apr 19 07:36 /home/root/original.txt
   root@3213642e5f1a:/workspace/prj001# ls -li hlink.txt
   748906 -rw----r-- 2 root root 24 Apr 19 07:36 hlink.txt
   ```

   同じ`748906`になっていることを確認

1. 元ファイルを削除しても消えないのか確認する

   ```bash
   root@3213642e5f1a:/workspace/prj001# df -T
   Filesystem     Type     1K-blocks     Used Available Use% Mounted on
   overlay        overlay 1055762868 37971692 964087704   4% /
   tmpfs          tmpfs        65536        0     65536   0% /dev
   tmpfs          tmpfs      4046860        0   4046860   0% /sys/fs/cgroup
   shm            tmpfs        65536        0     65536   0% /dev/shm
   /dev/sdc       ext4    1055762868 37971692 964087704   4% /etc/hosts
   tmpfs          tmpfs      4046860        0   4046860   0% /proc/acpi
   tmpfs          tmpfs      4046860        0   4046860   0% /sys/firmware
   root@3213642e5f1a:/workspace/prj001# cd /dev/
   root@3213642e5f1a:/workspace/prj001# rm /home/root/original.txt
   root@3213642e5f1a:/workspace/prj001# cat hlink.txt
   Hello World by original
   root@3213642e5f1a:/workspace/prj001#
   ```

1. 別ファイルシステムで作成できないことを確認する

   ```bash
   root@3213642e5f1a:/dev/tmp# ln /workspace/prj001/hlink.txt
   root@3213642e5f1a:/dev# mkdir /dev/tmp
   root@3213642e5f1a:/dev# cd tmp/
   root@3213642e5f1a:/dev/tmp# ln /workspace/prj001/hlink.txt
   ln: failed to create hard link './hlink.txt' => '/workspace/prj001/hlink.txt': Invalid cross-device link
   ```

   補足として**同じファイルシステム**とは、`ext4`などが揃っていればよいということではなく、  
   **同じマウントポイント内にあるファイル、つまり同じ 1 つのファイルシステム領域内にあるファイル同士**を指す。（つまり、ディスクが違えば（マウントが別なら）、たとえ両方 ext4 でもハードリンクは不可）

1. ディレクトリに対して作成できないことを確認する

   ```bash
   root@3213642e5f1a:/dev/tmp# ln /home/root hlink2.txt
   ln: /home/root: hard link not allowed for directory
   ```

---

## 実用例

**シンボリックリンク**

- Python のバージョンの切り替え

  ```bash
  # 実体のバージョンごとのパス
  /usr/bin/python3.8
  /usr/bin/python3.11

  # /usr/bin/python3 を symlink にして切り替え可能にする
  ln -sf /usr/bin/python3.11 /usr/bin/python3
  ```

- 設定ファイルの切り替え

  ```bash
  # 複数の設定ファイルを用意
  config.dev.json
  config.prod.json

  # 実行時は常に config.json を読み込むようにする
  ln -s config.dev.json config.json
  ```

  環境によってリンク先を変えるだけで、アプリは同じパスから設定を読み取れる。

- Windows のショートカット的な使い方

  ```bash
  ln -s /mnt/data/shared /home/user/shared
  ```

  自分のホームディレクトリから、別のマウントポイントにあるディレクトリへ簡単にアクセスできる。

**ハードリンク**

- 誤削除防止・重要ファイルの安全対策

  ```bash
  ln important.txt .important_backup
  ```

- 重複を避けたファイルバックアップ（差分なし）

  ```bash
  ln /data/project/file1.txt /backup/project/file1.txt
  ```

  コピーとは違い、ストレージを節約しながらバックアップ的に使える。
  ただし、「本当に独立したバックアップ」にはならない（中身は連動する）。

## その他

**コンテナのファイルシステムの確認**

```bash
root@3213642e5f1a:/workspace/prj001# df -T
Filesystem     Type     1K-blocks     Used Available Use% Mounted on
overlay        overlay 1055762868 37923056 964136340   4% /
tmpfs          tmpfs        65536        0     65536   0% /dev
tmpfs          tmpfs      4046860        0   4046860   0% /sys/fs/cgroup
shm            tmpfs        65536        0     65536   0% /dev/shm
/dev/sdc       ext4    1055762868 37923056 964136340   4% /etc/hosts
tmpfs          tmpfs      4046860        0   4046860   0% /proc/acpi
tmpfs          tmpfs      4046860        0   4046860   0% /sys/firmware
```

Docker コンテナでは、通常の`ext4`や`xfs`のような単一のファイルシステムではなく、  
`overlay` or `overlay2`と呼ばれるレイヤー構造のファイルシステムが使われている。

overlay ファイルシステムの構造

- lower layer（読み取り専用）
  ベースとなるイメージ層。Docker イメージとして pull した内容。

- upper layer（読み書き可能）
  コンテナ実行中に作成・変更されるファイルがここに書き込まれる。

- merged layer
  コンテナから見える仮想的なファイルシステム。ユーザーからは透過的に見える。
