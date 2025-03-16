---
title: "Python開発環境設定"
date: 2022-11-03T22:00:45+09:00
description: ""
draft: false
subtitle: ""
toc: true
tags: ["poetry", "development", "vscode"]
categories: ["python"]
series: []
---

仕事でもだいたい Python を使うので開発環境メモ。
主に VScode の設定を記載。

## 前提条件

---

以下インストール済み

- pyenv
  - https://github.com/pyenv/pyenv
- poetry
  - https://github.com/python-poetry/poetry
  - `virtualenvs.in-project`は`true`に設定済み
  ```bash
  $ poetry config --list
  cache-dir = "/home/piyota/.cache/pypoetry"
  experimental.new-installer = false
  installer.parallel = true
  virtualenvs.create = true
  virtualenvs.in-project = true
  virtualenvs.path = "{cache-dir}/virtualenvs"  # /home/piyota/.cache/pypoetry/virtualenvs
  ```
- VScode

### VSCode 拡張機能

---

- python
- pylance
- autodocstring
- editorconfig
  - https://editorconfig.org/
  - `setting.json`があるのでオプション

### ライブラリ

---

| ライブラリ名 | 役割               | 備考                                                     |
| ------------ | ------------------ | -------------------------------------------------------- |
| black        | フォーマッター     |                                                          |
| isort        | フォーマッター     | `import`のフォーマッター                                 |
| flake8       | リンター           | https://flake8.pycqa.org/en/latest/user/error-codes.html |
| mypy         | 静的チェッカー     | https://github.com/python/mypy                           |
| pep8-naming  | 命名規則チェッカー | https://github.com/PyCQA/pep8-naming                     |

- `mypy`の静的チェックのイメージ
  ```python
  number = input("What is your favourite number?")
  print("It is", number + 1)  # error: Unsupported operand types for + ("str" and "int")
  ```

### セットアップ

---

- ライブラリインストール

  ```bash
  $ pyenv local 3.10.5
  $ poetry init
  $ poetry shell
  $ poetry add -D black
  $ poetry add -D isort
  $ poetry add -D flake8
  $ poetry add -D mypy
  ```

- `setting.json`
  ```json
  {
    "python.envFile": "${workspaceFolder}/.venv",
    "python.formatting.provider": "black",
    "python.linting.enabled": true,
    "python.linting.pylintEnabled": false, // pylint は使わない
    "python.linting.pycodestyleEnabled": false, // pycodestyleは使わない
    "python.linting.flake8Enabled": true,
    "python.linting.flake8Args": [
      // エラーコード
      // https://pep8.readthedocs.io/en/latest/intro.html#error-codes
      "--max-line-length=110",
      "--ignore=E111, E114, E402, E501"
    ],
    "python.linting.lintOnSave": true,
    "python.linting.mypyEnabled": true,
    "python.languageServer": "Pylance",
    // 型アノテーション設定
    // basic: 型アノテーションがある部分が合っているか検出
    // strict: 型アノテーションがない場合も検出
    "python.analysis.typeCheckingMode": "strict",
    // PEP8 エラーコードチェックシート
    // https://qiita.com/KuruwiC/items/8e12704e338e532eb34a
    "python.formatting.blackArgs": ["--line-length=110"],
    "[python]": {
      "editor.tabSize": 4,
      "editor.formatOnSave": true,
      "editor.codeActionsOnSave": {
        "source.organizeImports": true
      }
    },
    // pytest設定
    "python.testing.pytestArgs": ["test"],
    "python.testing.unittestEnabled": false,
    "python.testing.pytestEnabled": true
  }
  ```
- `launch.json`
  ```json
  {
    // IntelliSense を使用して利用可能な属性を学べます。
    // 既存の属性の説明をホバーして表示します。
    // 詳細情報は次を確認してください: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
      {
        "name": "Python: 現在のファイル",
        "type": "python",
        "request": "launch",
        "program": "${file}",
        "console": "integratedTerminal",
        // 自身のコードのみデバッグするかどうか
        // falseの場合他のライブラリをデバッグ対象に含める
        "justMyCode": true
      }
    ]
  }
  ```
