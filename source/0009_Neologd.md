# MecabでNeologd登録（Windows10）の備忘録
新しいPCで久々にMecabの環境作っているときに、Neologdの登録方法をけっこう忘れていたので備忘録。


※インストールしているMecabはC:\Program Files\MeCab\binのPath通しておく

Neologd辞書をダウンロード。

（ https://github.com/neologd/mecab-ipadic-neologd/tree/master/seed ）

辞書をすべて解凍。（batファイル作成した）

mecab-dict-indexコマンドで、すべてのcsvからdicファイル作る。（batファイル作成した）

C:\Program Files\MeCab\dic内にNEologdフォルダをmkdir。

NEologdフォルダにdicファイルを移動もしくはコピー。

![dicdir](0009_Neologd/dicdir.png)

mecabrcを編集して、すべてのdicファイルを登録。

![mecabrc_fig](0009_Neologd/mecabrc_fig.png)
<br>
<br>
辞書をすべて解凍するbatファイル

```
rem 解凍する辞書のディレクトリに移動
cd C:\Users\<user_name>\mecab-ipadic-neologd\seed
rem 7z.exeをディレクトリ内のすべてのファイルに適用
for %%A in (*.*) do (
        "C:\Program Files\7-Zip\7z.exe" x %%A
)
pause
```
<br>
<br>
csvからdicファイルを作成するbatファイル

```
rem 解凍した辞書のcsvファイルがあるディレクトリに移動
cd C:\Users\<user_name>\mecab-ipadic-neologd\seed
rem mecab-dict-indexコマンドでcsvからdicファイルをNEologdフォルダ内に作成
for %%A in (*.csv) do (
	mecab-dict-index -d "C:\Program Files\MeCab\dic\ipadic" -u "C:\Program Files\MeCab\dic\NEologd\%%A.dic" -f utf8 -t utf8 %%A
)
pause
```
<br>
<br>
mecab-dict-indexのoption

```
-d DIR: システム辞書があるディレクトリ
-u FILE: FILE というユーザファイルを作成
-f charset: CSVの文字コード
-t charset: バイナリ辞書の文字コード
```