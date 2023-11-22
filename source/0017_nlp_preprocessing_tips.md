# 自然言語処理前処理Tips

前処理してMecabやGiNZAで分かち書きするまで
```python
import os
import numpy as np
import pandas as pd
import matplotlib as mpl
import matplotlib as mpl
import matplotlib.pyplot as plt
from matplotlib import cm
import matplotlib.font_manager as fm
import matplotlib.font_manager as font_manager
import matplotlib.dates as mdates
from matplotlib.ticker import FormatStrFormatter
import seaborn as sns
import tqdm as tq
from tqdm import tqdm
import gzip
import glob
import datetime as dt
import shutil
import collections
import functools
import gc
import sys
import time
import pickle
import zipfile
import json

import spacy
from spacy.matcher import Matcher
import MeCab
import re
from collections import OrderedDict
from collections import Counter
import itertools

# 丸囲い数字のコード取得
def get_marumoji(val):
    # ①を文字コードに変換[bytes型]
    maru_date = "①".encode("UTF8")
    # ①を文字コードに変換[int型]
    maru_code = int.from_bytes(maru_date, 'big')
    # 文字コードの変換
    maru_code += val - 1
    # 文字コードを文字に変換して返却
    return maru_code.to_bytes(4, "big").decode("UTF8")[1]

# 小文字化、記号・丸囲い数字除去、全角を半角など前処理関数
def preprocessing(text, loop_tango=None):    
    text = text.lower()  # 小文字化
    text = re.sub('\r\n','',text)  # \r\nをdelete
    #text = re.sub(r'\d+','',text)  # 数字列をdelete
    text = re.sub(r'\b\d{1,3}(,\d{3})*\b','0', text)  # 桁区切り数字を 0 に変換
    text = re.sub(r'\d+','0', text)  # 数字列を0
    text = re.sub(r'改行','',text)  # 改行をdelete
    text = re.sub(r'\u3000','',text)  # \u3000をdelete
    ZEN = ("".join(chr(0xff01 + i) for i in range(94))).replace('－','')  # 全角文字列一覧
    HAN = ("".join(chr(0x21 + i) for i in range(94))).replace('-','')  # 半角文字列一覧
    ALL=re.sub(r'[a-zA-Zａ-ｚＡ-Ｚ\d]+','',ZEN+HAN)  # 特殊記号たち
    symbol1 = '〜'+'、'+'。'+'~'+'*'+'＊'+ALL+'「'+'」'  # 特殊記号たち
    symbol2 = '!"#$%&\'\\\\()*+-,./:;<=>?@[\\]^_`{|}~「」〔〕“”〈〉『』【】＆＊・（）＄＃＠。、？！｀＋￥％〇°＋－─┼⇒→↓←↑'  # ネットで取ってきた記号たち
    symbol = symbol1+symbol2
    symbol = '['+''.join(OrderedDict.fromkeys(symbol))+']'  # 重複削除
    code_regex = re.compile(symbol)
    text = code_regex.sub('', text)  # 記号を消す
    
    maru_int = []
    for i in range(1,100):
        try:
            tmp = get_marumoji(i)
            maru_int.append(tmp)
        except UnicodeDecodeError:
            'UnicodeDecodeError, Skip a processing'
    code_regex = re.compile('['+"".join(i for i in maru_int)+']')
    text = code_regex.sub('', text)  # 丸囲い数字を消す
    
    ZEN = "".join(chr(0xff01 + i) for i in range(94))
    HAN = "".join(chr(0x21 + i) for i in range(94))
    ZEN2HAN = str.maketrans(ZEN, HAN)
    HAN2ZEN = str.maketrans(HAN, ZEN)
    text = text.translate(ZEN2HAN)  # 全角を半角に
    text = ''.join(text.split())
    text = text.strip()
    # 同じ単語が繰り返しつながっている場合1つにする
    if loop_tango:
        for tango in loop_tango:
            text = loop_word(text, tango)
    return text

# 同じ単語が繰り返しつながっている場合1つにする
def loop_word(text, tango):
    code_regex = re.compile(r'('+tango+')+')
    text = code_regex.sub(tango, text)  # 記号を消す
    return text

# Mecabで分かち書きする関数
def wakachi_m(text, select_word_type=['名詞','形容詞','動詞']):
    # MeCabの準備
    tagger = MeCab.Tagger()
    tagger.parse('')
    node = tagger.parseToNode(text)
    word_list = []
    while node:
        word_type = node.feature.split(',')[0]
        if word_type in select_word_type:  # 指定した品詞のみ処理
            if word_type == '名詞':
                word_list.append(node.surface)
            else:
                word_list.append(node.feature.split(",")[6])
        else:
            pass
        node = node.next

    # リストを文字列に変換
    #word_chain = ' '.join(word_list)
    if len(word_list)==0:
        'no list!'
    return word_list#, word_chain  # リストでまとめた結果とすべてつなげた文字列を返す

# GiNZA(Sudachi)で分かち書きする関数
def wakachi_g(nlp, text, select_word_type=['NOUN', 'VERB', 'ADJ']):
    # ja_ginzaの準備
    doc = nlp(text)
    word_list = []
    for node in doc:
        if node.pos_ in select_word_type:  # 指定した品詞のみ処理
            word_list.append(node.lemma_.lower())
    # リストを文字列に変換
    #word_chain = ' '.join(word_list)
    if len(word_list)==0:
        'no list!'
    return word_list#, word_chain  # リストでまとめた結果とすべてつなげた文字列を返す

# dfでまとめた文章をすべて分かち書き処理する関数 GiNZA(Sudachi)
def makeWakachiDf(nlp, sentenceDf, col, stop_words, select_word_type=['NOUN', 'VERB', 'ADJ']):
    word_list_noun_list = []
    word_chain_noun_list = []
    for i, text in tqdm(enumerate(sentenceDf[col])):  # 指定した列にある文章を1行ずつ分かち書き処理
        word_list_noun = wakachi_g(nlp, text, select_word_type=select_word_type)
        word_list_noun = ['0' if w.isnumeric() else w for w in word_list_noun]  # ローマ数字や漢数字を0に
        word_list_noun = [w for w in word_list_noun if w not in stop_words]  # ストップワードも設定
        if len(word_list_noun)==0:  # 分かち書きとストップワード除去で何もなかった場合何もない結果を追加
            word_list_noun_list.append([])
            #word_chain_noun_list.append('')
            continue
        word_list_noun_list.append(word_list_noun)  #分かち書きとストップワード除去の結果を追加
        #word_chain_noun_list.append(word_chain_noun)  #分かち書きとストップワード除去の結果を追加
    word_chain_noun_list = [' '.join(w) for w in [word_list for word_list in word_list_noun_list]]
    return word_list_noun_list, word_chain_noun_list  # リストでまとめた結果とすべてつなげた文字列をリストにまとめて返す

# dfでまとめた文章をすべて分かち書き処理する関数  Mecab
def makeWakachiDf_m(sentenceDf, col, stop_words, select_word_type=['名詞','形容詞','動詞']):
    word_list_noun_list = []
    word_chain_noun_list = []
    for i, text in tqdm(enumerate(sentenceDf[col])):  # 指定した列にある文章を1行ずつ分かち書き処理
        word_list_noun = wakachi_m(text, select_word_type=select_word_type)
        word_list_noun = ['0' if w.isnumeric() else w for w in word_list_noun]  # ローマ数字や漢数字を0に
        word_list_noun = [w for w in word_list_noun if w not in stop_words]  # ストップワードも設定
        if len(word_list_noun)==0:  # 分かち書きとストップワード除去で何もなかった場合何もない結果を追加
            word_list_noun_list.append([])
            #word_chain_noun_list.append('')
            continue
        word_list_noun_list.append(word_list_noun)  #分かち書きとストップワード除去の結果を追加
        #word_chain_noun_list.append(word_chain_noun)  #分かち書きとストップワード除去の結果を追加
    word_chain_noun_list = [' '.join(w) for w in [word_list for word_list in word_list_noun_list]]
    return word_list_noun_list, word_chain_noun_list  # リストでまとめた結果とすべてつなげた文字列をリストにまとめて返す

# 処理まとめ GiNZA(Sudachi)
def make_nlp_df(df_org, col, stop_words, morpho='ja_ginza', select_word_type=['NOUN', 'VERB', 'ADJ'], loop_tango=None):
    df = df_org[[col]].copy()
    # 前処理した文章をdfの新たな列に入れる
    df[col+'_preprocessingSentence'] = [preprocessing(text, loop_tango=loop_tango) for text in df[col].to_list()]
    df = df[df[col+'_preprocessingSentence'].map(lambda x:len(x)>0)]  # 何もない行は消す
    # dfでまとめた文章をすべて分かち書き処理
    print(len(df))
    nlp = spacy.load(morpho)
    word_list_noun_list, word_chain_noun_list = makeWakachiDf(nlp, df, col+'_preprocessingSentence'
                                                              , stop_words, select_word_type=select_word_type)
    # 分かち書き処理の結果をdfに追加
    df[col+'_wakachiSentenceList'] = word_list_noun_list
    df[col+'_wakachiSentenceChain'] = word_chain_noun_list
    # 分かち書き処理の結果、空のリストの行を削除
    df = df[df[col+'_wakachiSentenceList'].map(lambda x:len(x)>0)].reset_index(drop=True).copy()
    return df

# 処理まとめ Mecab
def make_nlp_df_m(df_org, col, stop_words, select_word_type=['名詞','形容詞','動詞'], loop_tango=None):
    df = df_org[[col]].copy()
    # 前処理した文章をdfの新たな列に入れる
    df[col+'_preprocessingSentence'] = [preprocessing(text, loop_tango=loop_tango) for text in df[col].to_list()]
    df = df[df[col+'_preprocessingSentence'].map(lambda x:len(x)>0)]  # 何もない行は消す
    # dfでまとめた文章をすべて分かち書き処理
    print(len(df))
    word_list_noun_list, word_chain_noun_list = makeWakachiDf_m(df, col+'_preprocessingSentence'
                                                                , stop_words, select_word_type=select_word_type)
    # 分かち書き処理の結果をdfに追加
    df[col+'_wakachiSentenceList'] = word_list_noun_list
    df[col+'_wakachiSentenceChain'] = word_chain_noun_list
    # 分かち書き処理の結果、空のリストの行を削除
    df = df[df[col+'_wakachiSentenceList'].map(lambda x:len(x)>0)].reset_index(drop=True).copy()
    return df
```

分かち書きする前の段階で除外したい単語を抜く方法
```python
# df['col']にある"AAAA"などの単語を除外する
replacements = {'AAAA':'','BBBB':'','CCCC':'','DDDD':'','EEEE':'','FFFF':''}
df['col'] = [re.sub('({})'.format('|'.join(map(re.escape, replacements.keys()))), lambda m: replacements[m.group()], intext) for intext in df['col'].to_list()]
```

複合名詞作成。出現頻度カウント。
```python
nlp = spacy.load('ja_ginza')
matcher = Matcher(nlp.vocab)
patterns = [[{'POS':'NOUN'}] * n for n in [2,3,4]]
for pattern in patterns:
    name=str(len(pattern))
    matcher.add(name,[pattern])

texts = df['col'].to_list()
# 複合名詞作成
nps = []
for text in tqdm(texts):
    doc = nlp(text)
    nps += [doc[begin:end].text for m_id, begin, end in (matcher(doc))]

# 出現頻度カウント
counter = Counter(nps)
CompoundNouns = pd.DataFrame([[wd,cnt] for wd,cnt in counter.most_common(100)], columns=['word','count'])
CompoundNouns.head(60)
```