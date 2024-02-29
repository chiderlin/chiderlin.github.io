---
title: privateGPT Local架起來
date: 2024-02-27 20:39:18
tags:
    - 聊天機器人
    - ChatGPT
    - AI
categories: 其他
---

用 macOS 在本機跑 PrivateGPT 紀錄。
可以上傳 PDF file, 讓機器人進行解析。
![結果圖](privategpt.png)

## 正文

```shell
git clone https://github.com/imartinez/privateGPT
cd privateGPT
```

1. 下載 make

    ```shell
    brew install cmake
    ```

2. 下載 pyenv 3.11 (Poetry 需要 3.11)

    ```shell
    pyenv install 3.11
    pyenv local 3.11
    ```

3. Install Poetry ＋ debug

    ```shell
    curl -sSL https://install.python-poetry.org | python3 -
    ```

    這邊起初在安裝的時候遇到問題

    - 1.我的 python 沒有 3.11
    - 2.找不到 Poetry

    解決：

    1. 移除原本 homebrew 下載的 python 並重新下載
    2. homebrew 下載 `pyenv`
    3. 設定環境變數
    4. 透過 `pyenv` 下載 python 3.11.4 版本
    5. 重新下載 Poetry (先 uninstall 再 install)

    ```Shell
    brew remove python
    brew install pyenv

    echo 'export PATH="$HOME/.pyenv/shims:$PATH"' >> ~/.profile
    source ~/.profile

    pyenv install 3.11.4
    pyenv global 3.11.4

    curl -sSL https://install.python-poetry.org | python3 - --uninstall
    curl -sSL https://install.python-poetry.org | python3 -
    ```

4. 下載 Poetry 後，設定 Poetry 環境變數
   這部分在下載 Poetry 時，它有給予指示。
   ![poetry setting](portry-setting.png)

5. 這時的 `which python`
   -> `/Users/chider/.pyenv/shims/python`

6. 進到專案裡，下載 privateGPT 相關啟動套件

    - install

        ```shell
        poetry install --with ui
        poetry install --with local
        poetry run python scripts/setup
        ```

    - OSX GPU support
        ```shell
        CMAKE_ARGS="-DLLAMA_METAL=on" pip install --force-reinstall --no-cache-dir llama-cpp-python
        ```

7. Run 起來～
   `poetry run python -m private_gpt(or make run)`

---

## 補充：

### 把 local Sqllite 換成 local docker 跑 qdrant

1. 調整專案裡的 setting.yml

    ```yml
    qdrant:
    url: http://localhost:6333
    #path: local_data/private_gpt/qdrant
    ```

2. 下載 qdrant docker

    ```shell
    sudo apt install docker.io
    sudo docker pull qdrant/qdrant

    sudo docker run -p 6333:6333
        -v <path-to-project>/latticefi-private-gpt/local_data/storage:qdrant/storage
        -v <path-to-project>/latticefi-private-gpt/local_data/config:qdrant/config
        qdrant/qdrant
    ```

### 建立一個 python 虛擬環境

更方便日後切換環境開發。

1. 建立虛擬環境 -> `virtualenv -p python3.11 <環境名稱>`
2. 進入虛擬環境 -> `source .venv/bin/activate`
3. 在虛擬環境安裝相關套件
    - `pip install --upgrade pip poetry`
    - `poetry install --with local`
    - `poetry run python scripts/setup`

```shell
virtualenv -p python3.11 .venv
source .venv/bin/activate
pip install --upgrade pip poetry
poetry install --with local
poetry run python scripts/setup
```

## 總結

之後要啟動專案，步驟：

1. 啟動 qdrant container
2. 進入專案
3. 進入虛擬環境
4. 啟動程式 make run
   OK!

## 參考來源：

[Installation](https://docs.privategpt.dev/installation)
[python-poetry](https://python-poetry.org/docs/#installing-with-the-official-installer)
[Poetry issue](https://github.com/python-poetry/install.python-poetry.org/issues/71)
