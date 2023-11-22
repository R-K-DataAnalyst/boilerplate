# いろいろ可視化の備忘録
普通にヒストグラムを書いた時に、外れ値のせいで横軸のレンジが広くなりすぎて、左隅に1本棒が立っているだけの意味のない可視化になる時が多い。

それを避けるために、クリッピングで数値丸めてからヒストグラム書くと、意味のある可視化になりやすい。
```python
# クリッピングして2クラスのヒストグラム作成
fig=plt.figure(figsize=(18,14))
dim=len(plot_cols)
for i, col in tqdm(enumerate(plot_cols)):
    # クリッピング時のminmax取得
    vmin=df01[col].clip(df01[col].quantile(0.0), df01[col].quantile(0.99)).min()
    vmax=df01[col].clip(df01[col].quantile(0.0), df01[col].quantile(0.99)).max()
    # ヒストグラムのwidth決める
    if vmax>=10000:
        width=round(vmax/50)
    elif round(vmax/20)==0:
        width=1
    else:
        width=round(vmax/20)

    ax = plt.subplot(round(np.ceil(dim/np.sqrt(dim))), round(np.ceil(dim/np.sqrt(dim))), i+1)
    ax2 = ax.twinx()

    VAL = df01[df01[obj_col]==0][col].copy()
    # クリッピング
    if width==1:
        pass
    else:
        VAL = VAL.clip(VAL.quantile(0.0), VAL.quantile(0.99))

    VAL2 = df01[df01[obj_col]==1][col].copy()
    # クリッピング
    if width==1:
        pass
    else:
        VAL2 = VAL2.clip(VAL2.quantile(0.0), VAL2.quantile(0.99))

    # ヒストグラム
    sns.histplot(VAL.to_numpy(), binwidth=width, binrange=(0,max(VAL.max(), VAL2.max())), ax=ax, kde=False, label='0', color ='b', alpha=1.)
    sns.histplot(VAL2.to_numpy(), binwidth=width, binrange=(0,max(VAL.max(), VAL2.max())), ax=ax2, kde=False, label='1', color ='r', alpha=0.5)
    ax.set_title(col+', binwidth:'+str(width))
    ax2.grid(False)
    h1, l1 = ax.get_legend_handles_labels()
    h2, l2 = ax2.get_legend_handles_labels()
    ax.legend(h1+h2, l1+l2, loc='upper right', fontsize=10)
plt.tight_layout()
plt.show()
```
回帰予測の時の、実測値と予測値の散布図作成関数。

実測値±rateの割合と実測値±small_thresholdの範囲に網掛けして、範囲内のサンプル数もカウントする。
```python
def obs_pred_plot(zissoku, yosoku# zissoku, yosoku：numpy array
                  , rate=0.1, small_threshold=5, model_name=''):
    # 閾値範囲を示すarray
    zissoku_lower = zissoku*(1-rate)
    zissoku_upper = zissoku*(1+rate)
    zissoku_lower_small = zissoku-small_threshold
    zissoku_upper_small = zissoku+small_threshold

    # 閾値範囲内だけのarrayのIndex
    inrange_ind = np.where(((yosoku>=zissoku_lower)&(yosoku<=zissoku_upper))|((yosoku>=zissoku_lower_small)&(yosoku<=zissoku_upper_small)))[0]
    
    ## 精度指標計算(一般的なものと、こちらで定義したものの2種)
    # 一般的なもの
    r2 = sklearn.metrics.r2_score(zissoku, yosoku) # r2 score
    rmse = np.sqrt(sklearn.metrics.mean_squared_error(zissoku, yosoku)) # rmse
    mape = np.mean(np.abs((yosoku - zissoku) / (zissoku+0.0001))) # mape
    # こちらで定義したもの
    good_ratio = (inrange_ind.shape[0])/(len(yosoku)) # 予測結果が閾値内に入る割合
    print('R2 score: {:.3f}'.format(r2))
    print('RMSE: {:.3f}'.format(rmse))
    print('MAPE: {:.3f}'.format(mape))
    print('Inside threshold ratio: {:.3f}'.format(good_ratio))
    
    # グラフの上限指定
    lim=max(zissoku.max(), yosoku.max())
    # グラフの上限指定
    limmin=min(zissoku.min(), yosoku.min())
    # 理想線引くためのlinear
    linear=np.linspace(limmin,lim,11)
    # figsize
    fig = plt.figure(figsize=(8,8))
    ax=plt.subplot(1,1,1)
    # 散布図（すべて）
    ax.scatter(zissoku, yosoku, alpha=0.5, label='Outside threshold')
    # 散布図（範囲内）
    ax.scatter(zissoku[inrange_ind]
               , yosoku[inrange_ind], edgecolors="k", alpha=1.
               , label='Inside threshold')
    # 範囲を可視化
    ax.fill_between(linear,linear*(1-rate),linear*(1+rate)
                    , facecolor='r',  alpha=0.3
                    , label='Threshold range(ratio):±{:.3f}'.format(rate))
    # 範囲を可視化（実測値が小さいところは±%の範囲が極小なので±実数で判断）
    ax.fill_between(linear, linear-small_threshold, linear+small_threshold
                    , facecolor='g', alpha=0.3
                    , label='Threshold range(small range):±{:.3f}'.format(small_threshold))
    # 理想的な予測結果の線を可視化
    ax.plot(linear,linear, label='Perfecrt predict line', ls='dotted', c='k', lw=3.)
    # 各指標を記す
    ax.annotate("R2 score: {:.3f}".format(r2), xy = (lim*0.5, lim*0.15), size = 12, color = "k")
    ax.annotate('RMSE: {:.3f}'.format(rmse), xy = (lim*0.5, lim*0.11), size = 12, color = "k")
    ax.annotate('MAPE: {:.3f}'.format(mape), xy = (lim*0.5, lim*0.07), size = 12, color = "k")
    ax.annotate('Inside threshold ratio: {:.3f}'.format(good_ratio), xy = (lim*0.5, lim*0.03), size = 12, color = "k")

    ax.legend()
    ax.set_xlim(limmin,lim)
    ax.set_ylim(limmin,lim)
    ax.set_xlabel('observation')
    ax.set_ylabel('prediction')
    ax.set_title(model_name+' Observation-Prediction plot')
    plt.show()
```
ただの棒グラフ作成用関数。
```python
# 棒グラフ作成
def plots_bar(df02, obj_col, key_col, xlabel='', flg=0):
    # 棒グラフにする数値を集計
    tmp1 = df02[df02[obj_col]==flg].groupby([key_col])[['col01']].count().reset_index()
    #display(tmp1)
    tmp2 = df02[df02[obj_col]==flg].groupby([key_col])[['col02']].sum().reset_index()
    #display(tmp2)
    tmp3 = df02[df02[obj_col]==flg].groupby([key_col])[['col03']].nunique().reset_index()
    #display(tmp3)
    tmp = pd.merge(tmp1, tmp2, on=[key_col], how='left')
    tmp = pd.merge(tmp, tmp3, on=[key_col], how='left')

    fig=plt.figure(figsize=(15,15))
    ax=plt.subplot(3,1,1)
    ax.bar(tmp[key_col], tmp['col01'],color='blue',alpha=0.6)
    ax.set_xlabel(xlabel)
    ax.set_ylabel('col01')
    plt.setp(ax.get_xticklabels(), rotation=30, ha='right')

    ax=plt.subplot(3,1,2)
    ax.bar(tmp[key_col], tmp['col02'],color='blue',alpha=0.6)
    ax.set_xlabel(xlabel)
    ax.set_ylabel('col02')
    plt.setp(ax.get_xticklabels(), rotation=30, ha='right')

    ax=plt.subplot(3,1,3)
    ax.bar(tmp[key_col], tmp['col03'],color='blue',alpha=0.6)
    ax.set_xlabel(xlabel)
    ax.set_ylabel('col03')
    plt.setp(ax.get_xticklabels(), rotation=30, ha='right')

    plt.tight_layout()
    plt.show()
```
箱ひげ図。ただし、外れ値が大きい場合、箱ひげがつぶれて見えないので、外れ値は無しにする。

その代わり、stripplotも同時にplotして外れ値まだ可視化する。
```python
# 箱ひげとstripplot作成
def plots_box_strip(df02, df_obj, obj_col, col, key_col, xlabel=''):
    tmp1 = df02.copy()
    tmp1 = tmp1.groupby([key_col, col])[['col02']].sum().reset_index()
    tmp1 = pd.merge(tmp1.reset_index(), df_obj, on=[key_col], how='left')
    tmp1[obj_col].fillna(0, inplace=True)
    
    tmp2 = df02.copy()
    tmp2 = tmp2.groupby([key_col,col])[['col01']].count().reset_index()
    
    tmp3 = pd.merge(tmp1, tmp2, on=[key_col, col], how='left')

    fig=plt.figure(figsize=(25,20))
    ax=plt.subplot(2,1,1)
    ax2=ax.twinx()
    # 箱ひげ
    sns.boxplot(x=col, y='col01', data=tmp3, hue=obj_col, ax=ax, boxprops=dict(alpha=.5), dodge=True, sym="")
    #sns.violinplot(x=col, y='col02', data=tmp3, hue=obj_col, ax=ax, jitter=True, dodge=True, alpha=0.5)#, boxprops=dict(alpha=.5)
    # stripplot
    sns.stripplot(x=col, y='col01', data=tmp3, hue=obj_col, jitter=True, dodge=True, ax=ax2, alpha=0.3, marker='o')
    ax2.grid(False)
    ax.set_ylim(None, tmp3['col01'].max()/10)
    ax.set_xlabel(xlabel)
    plt.setp(ax.get_xticklabels(), rotation=30, ha='right')
    
    ax=plt.subplot(2,1,2)
    ax2=ax.twinx()
    sns.boxplot(x=col, y='col02', data=tmp3, hue=obj_col, ax=ax, boxprops=dict(alpha=.5), dodge=True, sym="")
    #sns.violinplot(x=col, y='col02', data=tmp3, hue=obj_col, ax=ax, jitter=True, dodge=True, alpha=0.5)#, boxprops=dict(alpha=.5)
    sns.stripplot(x=col, y='col02', data=tmp3, hue=obj_col, jitter=True, dodge=True, ax=ax2, alpha=0.3, marker='o')
    ax2.grid(False)
    ax.set_ylim(None, tmp3['col02'].max()/10)
    ax.set_xlabel(xlabel)
    plt.setp(ax.get_xticklabels(), rotation=30, ha='right')

    plt.tight_layout()
    plt.show()
```
ただの円グラフ作成関数。
```python
# 円グラフ作成関数
def pct_abs(pct, raw_data):
    absolute = int(np.sum(raw_data)*(pct/100.))
    return '{:d}\n({:.0f}%)'.format(absolute, pct) if pct > 5 else ''

def plot_chart(y_km):
    km_label=pd.DataFrame(y_km).rename(columns={0:'cluster'})
    km_label['val']=1
    km_label=km_label.groupby('cluster')[['val']].count().reset_index()
    fig=plt.figure(figsize=(5,5))
    ax=plt.subplot(1,1,1)
    ax.pie(km_label['val'],labels=km_label['cluster'], autopct=lambda p: pct_abs(p, km_label['val']))#, autopct="%1.1f%%")
    ax.axis('equal')
    ax.set_title('Cluster Chart (ALL Records:{})'.format(km_label['val'].sum()),fontsize=14)
    plt.show()
```
(予測 - 実測) ÷ 実測の計算。計算後はsns.scatterplot()で可視化したり、表として出力したりご自由に。
```python
# (予測 - 実測) ÷ 実測
period_ = list(np.round(np.linspace(-0.5,0.5,11), 3))
dfresults = pd.read_csv("<FILE NAME>")  # 実測と予測のdf
dfresults.columns=['true', 'pred']
dfresults['diff'] = dfresults['pred'] - dfresults['true']  # 実測予測の差分
dfresults['ratio'] = dfresults['diff'] / dfresults['true']  # 実測予測の差分を実測値で割る
s_cut, bins = pd.cut(dfresults['ratio'], period_, right=False, retbins=True)  # 区切る
labels=bins[:-1]  # 区切る
s_cut = pd.cut(dfresults['ratio'], period_, right=False, labels=labels)  # 区切る
dfresults['period']=s_cut.values  # 区切った区間を追加
dfresults['period']=dfresults['period'].astype(str)
dfresults['period']=dfresults['period'].astype(float)  # float型へ
dfresults.loc[(dfresults['ratio']>=period_[-1])&(~(np.isinf(dfresults['ratio']))), 'period'] = period_[-1]+0.01
dfresults.loc[(dfresults['ratio']<period_[0])&(~(np.isinf(dfresults['ratio']))), 'period'] = period_[0]-0.01
dfresults.loc[(np.isinf(dfresults['ratio'])), 'period'] = 9999
#dfresults.loc[(np.isposinf(dfresults['ratio'])), 'period'] = 9999
#dfresults.loc[(np.isneginf(dfresults['ratio'])), 'period'] = -9999

dfresultsPlot = dfresults.copy()
rep = {i:str(n+1).zfill(2)+'_[{}, {})_DiffRatio'.format(round(i,3), round(i+0.1,3)) for n, i in enumerate(labels)}
rep[period_[-1]+0.01] = str(len(labels)+1).zfill(2)+'_[{}, )_DiffRatio'.format(period_[-1])
rep[period_[0]-0.01] = str(0).zfill(2)+'_[ , {})_DiffRatio'.format(period_[0])
rep[9999] = 'inf'
dfresultsPlot = dfresultsPlot.replace({'period':rep})
```