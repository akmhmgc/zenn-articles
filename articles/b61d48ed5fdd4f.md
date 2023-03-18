---
title: "GPT-3を使ったシェルスクリプトでコミットメッセージ作成を自動化しよう！"
emoji: "🗒"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [git]
published: false
---

## はじめに
コミットメッセージって大切ですよね。とはいえ毎回考えるの面倒じゃないですか？英語で書かなきゃいけない場合とかは考える手間がかかりますよね。この記事では、OpenAIのGPT-3を活用し、たった一つのコマンドでgit diffの出力から適切なコミットメッセージを自動生成する方法をご紹介します。
さらに、git hooksを利用して、コミットメッセージを完全に自動化する方法も解説します。この方法を取り入れることでより効率的に作業に集中できるでしょう。

## 手順

1. まず、GPT-3 APIを利用できるように、Pythonでopenaiパッケージをインストールします。

```sh
$ pip install openai
```

2. 次に、以下のシェルスクリプト`generate_commit_message.sh`とPythonスクリプト`generate_commit_message.py`を作成します。

generate_commit_message.sh:

```generate_commit_message.sh
#!/bin/bash

# GPT-3 APIキーを環境変数に設定
export OPENAI_API_KEY="your_api_key_here"

# git diffの出力をファイルに保存
git diff --staged > diff_output.txt

# Pythonスクリプトを実行してコミットメッセージを生成
python3 generate_commit_message.py

# 一時ファイルを削除
rm diff_output.txt
```

generate_commit_message.py:

```generate_commit_message.py
import openai
import os
import json

# 環境変数からAPIキーを取得
api_key = os.getenv("OPENAI_API_KEY")

# APIキーを設定
openai.api_key = api_key

# git diffの出力を読み込む
with open("diff_output.txt", "r") as file:
    diff_output = file.read()

# GPT-3へのリクエストを作成
prompt = f"Given the following git diff output, suggest an appropriate commit message in English:\n\n{diff_output}\n\nCommit message:"
response = openai.Completion.create(
    engine="text-davinci-003",
    prompt=prompt,
    max_tokens=100,
    n=1,
    stop=None,
    temperature=0.7,
)

# コミットメッセージを取得して表示
commit_message = response.choices[0].text.strip()
print(commit_message)
```

```
Given the following git diff output, suggest an appropriate commit message in English:\n\n{diff_output}\n\nCommit message:
```

の部分がプロンプトになっています。もっと良いメッセージがあるかもしれないのでお好みでどうぞ。
また、今回は適当にtoken数を100としていますがもう少し増やした方が良いかもしれません。

3. シェルスクリプトとPythonスクリプトを同じディレクトリに保存し、シェルスクリプトに実行権限を与えます。

```sh
$ chmod +x generate_commit_message.sh
```

4. これで、git diffの出力からコミットメッセージを自動生成できるようになりました。シェルスクリプトを実行するだけで、GPT-3が生成したコミットメッセージが表示されます。

```sh
$ ./generate_commit_message.sh
```

## さらに楽をしたい場合
更に、git hooksを利用して、コミットメッセージの自動生成を完全に自動化する方法を説明します。

1. まず、.git/hooksディレクトリに移動します。

```sh
cd .git/hooks
```

6. 次に、`prepare-commit-msg`ファイルを作成し、以下の内容を追加します。この内容は、先ほどの`generate_commit_message.sh`と同じです。

```prepare-commit-msg
#!/bin/bash

# GPT-3 APIキーを環境変数に設定
export OPENAI_API_KEY="your_api_key_here"

# git diffの出力をファイルに保存
git diff > diff_output.txt

# Pythonスクリプトを実行してコミットメッセージを生成
COMMIT_MESSAGE=$(python3 /path/to/your/generate_commit_message.py)

# 一時ファイルを削除
rm diff_output.txt

# コミットメッセージを設定
echo "$COMMIT_MESSAGE" > "$1"
```
2. `prepare-commit-msg`ファイルに実行権限を付与します。

```sh
chmod +x prepare-commit-msg
```

3. 先ほどの`generate_commit_message.py`ファイルは、適切な場所（例えば、プロジェクトのルートディレクトリ）に置いておき、`prepare-commit-msg`ファイル内で正しいパスが指定されていることを確認してください。

これで、git commitを実行する度に、自動的にGPT-3がコミットメッセージを生成して設定してくれます。ただし、必要に応じて手動でメッセージを調整することを忘れずに行ってください。また、GPT-3 APIの利用にはコストがかかるため、利用料金に注意してください。

## 最後に
これで、GPT-3を活用したコミットメッセージの自動生成と、git hooksによる完全な自動化ができるようになりました。この方法を取り入れることで手間をかけずに、より効率的に作業に集中できるようになりました！
