# poetryの備忘録
Anaconda有償化で社内使用NGの場合、conda仮想環境も使えない。
<br>
conda以外いろいろなアプローチがあるけど、pyenv+poetryという選択もある。
## pyenv+poetry+pipで仮想環境つくるためのTips
### 参考URL
Poetryをサクッと使い始めてみる
<br>
https://qiita.com/ksato9700/items/b893cf1db83605898d8a
<br>
<br>
poetryでパッケージ・仮想環境を管理
<br>
https://rinoguchi.net/2020/06/poetry.html
<br>
<br>
【Python】Poetry始めてみた & Pipenv から poetry へ移行した所感
<br>
https://qiita.com/ragnar1904/items/0e5b8382757ccad9a56c
<br>
<br>
pyenvでのPython仮想環境の作り方まとめ
<br>
https://qiita.com/ysdyt/items/5008e607343b940b3480
<br>
<br>
ゼロから学ぶ Python poetry
<br>
https://rinatz.github.io/python-book/ch04-06-poetry/
<br>
<br>
【pyenv+poetry】Pythonで簡単に仮想環境を作成しよう
<br>
https://qiita.com/ku_a_i/items/167cb61a60c27031ebce
<br>
<br>
## コマンド
#pyenvで使用する使いたいpythonのバージョンを使えるようにしておく
<br>
pyenv global 3.7.7
<br>
<br>
#" poetry new <XXXX> "コマンドで作ったXXXXディレクトリに入る
<br>
cd XXXX
<br>
<br>
#仮想世界に入る
<br>
poetry shell
<br>
<br>
#パッケージ追加
<br>
poetry add <package-name>