
# gitへの登録
## ローカルリポジトリをリモートにPush
ローカルで作業したファイルをgitで管理したい場合は、ローカルからリモートにgitの登録をしてPushする

ローカルのリポジトリで、gitの設定を行う
```
cd ~~/[YOUR_REPO]
git init
git add .
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/[YOUR_ACCOUNT_NAME]/[YOUR_REPO].git
git push -u origin main
```

## リモートリポジトリをローカルにクローン
リモートでリポジトリを作成直後に、ローカルで作用したい場合は、リモートからローカルにgitをクローンする

リモートのフォルダをcloneしたい階層に移動して、`git clone`する。
コマンド自体は、gitのWeb側にデフォルトで準備されている。
