# clippingしてhistplot, 箱ひげ, 円グラフなど可視化の備忘録
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
ただの棒グラフ作成用。
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
箱ひげ図作成。ただし、外れ値が大きい場合、箱ひげがつぶれて見えないので、外れ値は無しにする。

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