# 2クラス分類評価備忘録
2クラス分類の表評価アプローチ。

混同行列で分類精度を見られる。

Calibration Curveで分類の予測確率の精度を見られる。

PR curveでPR-AUCを計算して不均衡データの精度指標としたりするなど。
```python
# 混同行列
def print_cmx(y_true, y_pred):
    '''create confusion matrix
    y_true:True data
    y_pred:Pred data
    '''
    labels = sorted(list(set(y_true)))
    cmx_data = confusion_matrix(y_true, y_pred, labels=labels)
    
    df_cmx = pd.DataFrame(cmx_data, index=labels, columns=labels)

    plt.figure(figsize = (6,6))
    sns.heatmap(df_cmx, annot=True, fmt='d', cmap='coolwarm', annot_kws={'fontsize':20},alpha=0.8)
    plt.xlabel('pred', fontsize=10)
    plt.ylabel('real', fontsize=10)
    plt.show()

# 予測確率－実測確率曲線を返す=Calibration Curve
def calib_curve(y_tests, y_pred_probas, xlim=[-0.05,1.05], ylim=[-0.05,1.05]):
    # 実測値入れる
    proba_check=pd.DataFrame(y_tests,columns=['real'])
    # 予測確率値入れる
    proba_check['pred']=y_pred_probas
    # 予測確率値10%刻み
    s_cut, bins = pd.cut(proba_check['pred'], list(np.linspace(0,1,11)), right=False, retbins=True)
    labels=bins[:-1]
    s_cut = pd.cut(proba_check['pred'], list(np.linspace(0,1,11)), right=False, labels=labels)
    proba_check['period']=s_cut.values
    # 予測確率値10%ごとの実際の確率とレコード数集計
    proba_check = pd.merge(proba_check.groupby(['period'])[['real']].mean().reset_index().rename(columns={'real':'real_ratio'})\
                            , proba_check.groupby(['period'])[['real']].count().reset_index().rename(columns={'real':'record_cnt'})\
                            , on=['period'], how='left')
    proba_check['period']=proba_check['period'].astype(str)
    proba_check['period']=proba_check['period'].astype(float)
    fig=plt.figure(figsize=(10,6))
    ax1 = plt.subplot(1,1,1)
    ax2=ax1.twinx()
    ax2.bar(proba_check['period'].values, proba_check['record_cnt'].values, color='gray', label="record_cnt", width=0.05, alpha=0.5)
    ax1.plot(proba_check['period'].values, proba_check['real_ratio'].values, color=sns.color_palette()[0],marker='+', label="real_ratio")
    ax1.plot(proba_check['period'].values, proba_check['period'].values, color=sns.color_palette()[2], label="ideal_line")
    handler1, label1 = ax1.get_legend_handles_labels()
    handler2, label2 = ax2.get_legend_handles_labels()
    ax1.legend(handler1 + handler2, label1 + label2, loc='center right')
    ax1.set_xlim(xlim[0],xlim[1])
    ax1.set_ylim(ylim[0],ylim[1])
    ax1.set_xlabel('period')
    ax1.set_ylabel('real_ratio %')
    ax2.set_ylabel('record_cnt')
    ax2.grid(False)
    plt.show()
    display(proba_check)
    
# rank表
# 予測確率値を昇順に10等分するCalibration Curve
def rank_curve(y_tests, y_pred_probas):
    # 実測値入れる
    proba_check=pd.DataFrame(y_tests,columns=['real'])
    # 予測確率値入れる
    proba_check['pred']=y_pred_probas
    # 予測確率値10等分
    s_cut, bins = pd.qcut(proba_check['pred'], 10, retbins=True)
    label_lower=pd.Series(list(np.round(np.sort(bins[:-1]), 4).astype('str')))
    label_upper=pd.Series(list(np.round(np.sort(bins[1:]), 4).astype('str')))
    labels=label_lower+'-'+label_upper
    labels=pd.DataFrame(labels)
    labels['No']=range(len(labels))
    labels['No']=labels['No'].astype(str).str.zfill(2)
    labels['period']=labels['No']+'_'+labels[0]
    labels=labels['period'].values
    
    bin_max=bins.max()
    s_cut = pd.qcut(proba_check['pred'], 10, labels=labels)
    proba_check['period']=s_cut.values
    proba_check['period']=proba_check['period'].astype(str)
    proba_check.loc[proba_check['pred']>bin_max, 'period']=str(np.round(bin_max, 4))+'_'+str(len(labels)).zfill(2)
    proba_check.loc[proba_check['pred']=='nan', 'period']='-99'
    # 予測確率値10等分範囲ごとの実際の確率とレコード数集計
    proba_check = pd.merge(proba_check.groupby(['period'])[['real']].mean().reset_index().rename(columns={'real':'real_ratio'})
                           , proba_check.groupby(['period'])[['real']].count().reset_index().rename(columns={'real':'record_cnt'})
                           , on=['period'], how='left')

    fig=plt.figure(figsize=(10,6))
    ax1 = plt.subplot(1,1,1)
    ax2=ax1.twinx()
    ax2.bar(proba_check['period'].values, proba_check['record_cnt'].values, color='gray', label="record_cnt", width=0.5, alpha=0.5)
    ax1.plot(proba_check['period'].values, proba_check['real_ratio'].values, color=sns.color_palette()[0],marker='+', label="real_ratio")
    handler1, label1 = ax1.get_legend_handles_labels()
    handler2, label2 = ax2.get_legend_handles_labels()
    ax1.legend(handler1 + handler2, label1 + label2, loc='center right')
    #ax1.set_xlim(-0.05,1.05)
    #ax1.set_ylim(-0.05,1.05)
    ax1.set_xlabel('period')
    ax1.set_ylabel('real_ratio %')
    ax2.set_ylabel('record_cnt')
    ax2.grid(False)
    plt.setp(ax1.get_xticklabels(), rotation=60, ha='right')
    plt.show()
    display(proba_check)
    
# 予測確率－実測確率曲線を返す=Calibration Curve
# 任意の上限指定し上限÷10%刻み（上限30%だったら3%刻みで30%までのCalibration Curve）
def calib_curve_part(y_tests, y_pred_probas, part=0.3, xlim=[-0.05,1.05], ylim=[-0.05,1.05]):
    # 実測値入れる
    proba_check=pd.DataFrame(y_tests,columns=['real'])
    # 予測確率値入れる
    proba_check['pred']=y_pred_probas
    # 任意の上限指定
    proba_check = proba_check[proba_check['pred']<=part]
    s_cut, bins = pd.cut(proba_check['pred'], list(np.linspace(0,part,11)), right=False, retbins=True)
    labels=bins[:-1]
    s_cut = pd.cut(proba_check['pred'], list(np.linspace(0,part,11)), right=False, labels=labels)
    proba_check['period']=s_cut.values
    proba_check = pd.merge(proba_check.groupby(['period'])[['real']].mean().reset_index().rename(columns={'real':'real_ratio'})\
                            , proba_check.groupby(['period'])[['real']].count().reset_index().rename(columns={'real':'record_cnt'})\
                            , on=['period'], how='left')
    proba_check['period']=proba_check['period'].astype(str)
    proba_check['period']=proba_check['period'].astype(float)
    fig=plt.figure(figsize=(10,6))
    ax1 = plt.subplot(1,1,1)
    ax2=ax1.twinx()
    ax2.bar(proba_check['period'].values, proba_check['record_cnt'].values, color='gray', label="record_cnt", width=0.02, alpha=0.5)
    ax1.plot(proba_check['period'].values, proba_check['real_ratio'].values, color=sns.color_palette()[0],marker='+', label="real_ratio")
    ax1.plot(proba_check['period'].values, proba_check['period'].values, color=sns.color_palette()[2], label="ideal_line")
    handler1, label1 = ax1.get_legend_handles_labels()
    handler2, label2 = ax2.get_legend_handles_labels()
    ax1.legend(handler1 + handler2, label1 + label2, loc='center right')
    ax1.set_xlim(xlim[0],xlim[1])
    ax1.set_ylim(ylim[0],ylim[1])
    ax1.set_xlabel('period')
    ax1.set_ylabel('real_ratio %')
    ax2.set_ylabel('record_cnt')
    ax2.grid(False)
    plt.show()
    display(proba_check)


# 予測確率の分布を出す（width%刻み）
# 実測値入れる
df_res = pd.DataFrame(y_test)
# 予測確率値入れる
df_res['pred'] = pred
width=0.1 #10%刻み
dim=1
ax = plt.subplot(round(np.ceil(dim/np.sqrt(dim))), round(np.ceil(dim/np.sqrt(dim))), 1)
ax2 = ax.twinx()
VAL = df_res[df_res[obj_col]==0]['pred'].copy()
sns.histplot(VAL.to_numpy(), binwidth=width, binrange=(0, (width*10)), ax=ax, kde=False, label='record_cnt0', color ='b', alpha=1.)
VAL2 = df_res[df_res[obj_col]==1]['pred'].copy()
sns.histplot(VAL2.to_numpy(), binwidth=width, binrange=(0, (width*10)), ax=ax2, kde=False, label='record_cnt1', color ='r', alpha=0.5)
ax.set_title('binwidth:'+str(width))
ax2.grid(False)
h1, l1 = ax.get_legend_handles_labels()
h2, l2 = ax2.get_legend_handles_labels()
ax.legend(h1+h2, l1+l2, loc='upper right', fontsize=10)
plt.tight_layout()
plt.show()


# 今までの評価指標とかPR curveをまとめて出力
def plot_metric(pred, y_test:pd.Series):
    pred01 = np.copy(np.round(pred))# 予測確率値
    print_cmx(y_test.to_numpy(), pred01)# 混同行列
    calib_curve(y_test.to_numpy(), pred, xlim=[-0.05,1.05], ylim=[-0.05,1.05])# Calibration Curve
    calib_curve(y_test.to_numpy(), pred, xlim=[-0.05,1.05], ylim=[-0.05,0.5])# Calibration Curve拡大
    rank_curve(y_test.to_numpy(), pred)# Rank表
    calib_curve_part(y_test.to_numpy(), pred, part=0.3, xlim=[-0.05,0.31], ylim=[-0.001,0.04])# Calibration Curve3%刻み

    # PR curve
    precision, recall, thresholds = precision_recall_curve(y_test, pred)
    auc = sklearn.metrics.auc(recall, precision)
    plt.scatter(recall, precision, label='PR curve (area = %.4f)'%auc)
    plt.legend()
    plt.title('PR curve')
    plt.xlabel('Recall')
    plt.ylabel('Precision')
    plt.grid(True)
    plt.show()

    plt.scatter(thresholds, recall[1:], label='recall')
    plt.scatter(thresholds, precision[1:], label='precision')
    plt.legend()
    plt.xlabel('threshold')
    plt.ylabel('recall&Precision')
    plt.grid(True)
    plt.show()
```