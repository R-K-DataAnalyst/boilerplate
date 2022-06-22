# dockerの備忘録
## dockerでpython環境つくるためのTips
### 参考サイト
DockerでPython実行環境を作ってみる
<br>
https://qiita.com/jhorikawa_err/items/fb9c03c0982c29c5b6d5
<br>
<br>
docker-composeでvolumesを設定する
<br>
https://zenn.dev/ajapa/articles/1369a3c0e8085d
<br>
<br>
Docker Image Python
<br>
https://hub.docker.com/_/python
<br>
<br>
Linux 用 Windows サブシステムで Visual Studio Code の使用を開始する
<br>
https://docs.microsoft.com/ja-jp/windows/wsl/tutorials/wsl-vscode
<br>
<br>
DockerとDocker ComposeでPythonの実行環境を作成する
<br>
https://zuma-lab.com/posts/docker-python-settings
<br>
<br>
Dockerfileで日本語ロケールを設定する方法。およびロケールエラーの回避方法。
<br>
https://qiita.com/YuukiMiyoshi/items/f389ea366060537b5cd9
<br>
<br>
WSL2でLinux～Windows間のファイルのやり取り
<br>
https://bootcamp.fjord.jp/articles/29
<br>
<br>
Dockerコンテナの作成、起動〜停止まで
<br>
https://qiita.com/kooohei/items/0e788a2ce8c30f9dba53
<br>
<br>
Dockerイメージとコンテナの削除方法
<br>
https://qiita.com/tifa2chan/items/e9aa408244687a63a0ae
<br>
<br>
docker,docker-compose設定でデフォルト作業用ディレクトリを設定
<br>
https://webplus8.com/dockerfile-docker-compose-yml-working-directory/
<br>
<br>
docker-compose.ymlの書き方について解説してみた
<br>
https://qiita.com/yuta-ushijima/items/d3d98177e1b28f736f04
<br>
<br>
コンテナに入りたい？それ docker exec でできるよ
<br>
https://qiita.com/yosisa/items/a5670e4da3ff22e9411a

## コマンド
#docker imageの検索
<br>
sudo docker search centos
<br>
<br>
dockerデーモンステータス確認
<br>
sudo service docker status
<br>
<br>
dockerデーモン起動
<br>
sudo service docker start
<br>
<br>
イメージを一覧表示
<br>
sudo docker images
<br>
<br>
イメージの一覧を出力
<br>
sudo docker image ls
<br>
<br>
コンテナーを一覧表示
<br>
sudo docker container ls
<br>
<br>
レジストリからイメージやリポジトリを取得
<br>
sudo docker pull centos:7
<br>
<br>
コンテナを一覧表示
<br>
sudo docker ps
<br>
<br>
コンテナのプロセスを他のプロセスツリーからは隔離（分離）して実行
<br>
コンテナのプロセスに対してttyを割り当てるために、よく -i -t は -it と書かれる
<br>
sudo docker run -it centos
<br>
<br>
自分のターミナルの標準入力、標準出力、標準エラーにアタッチする
<br>
コマンドを自分のターミナル上で直接実行しているかのように、インタラクティブ（双方向）に制御できる
<br>
sudo docker attach <id>
<br>
<br>
1つまたは複数のコンテナを削除
<br>
sudo docker rm <id>
<br>
<br>
docker-compose：yaml形式の設定ファイルで複数コンテナを実行を一括で管理できる
<br>
コンテナを作成し、開始
<br>
up,downでサービスの起動停止
<br>
バックグラウンド実行なら「-d」を付けてup
<br>
docker compose up --build：サービスの構築もしくは再構築
<br>
sudo docker compose up -d --build
<br>
<br>
1つまたは複数のイメージを削除
<br>
sudo docker rmi <id>
<br>
<br>
停止しているdockerコンテナを起動
<br>
-i, --interactive：インタラクティブモードとなるため、継続してコマンド操作などが可能
<br>
sudo docker start <id>
<br>
<br>
1つまたは複数の実行中コンテナを停止
<br>
sudo docker stop <id>
<br>
<br>
実行中のコンテナ内でコマンドを実行
<br>
-it：標準入力を開き続ける＋疑似TTYを割り当て
<br>
bash：対話形式のbashシェルを起動してコマンドの入力を受け付ける状態
<br>
sudo docker exec -it <container-name> bash