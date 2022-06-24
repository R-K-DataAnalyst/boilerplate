# Annoyを使った近似近傍探索モデルの備忘録
近似近傍探索モデル作成できるパッケージAnnoyの使い方。
<br>
インストールはAnacondaならconda installで一発。
<br>
pip installでWindowsの場合、エラーが出ることがある。
<br>
その時はエラー文を見たらわかるがVisual C++ Build Toolsが必要なのでインストールする。

```python
# Annoyを使った近傍探索モデル作成
# ユークリッド距離で探索するモデルを用意する
# n_componentは説明変数の変数の数を指定
annoy_index = AnnoyIndex(n_component, metric='euclidean')
# モデルを学習させる
# embedding_trainが説明変数
for i, x in (enumerate(embedding_train)):
    annoy_index.add_item(i, x)# 学習
# ビルドする
annoy_index.build(n_trees=5)# k=5
# 保存
annoy_index.save(knndir+'/kneighbors_pca_'+re.sub('[/]+', '.', obj_col)+'.ann')# 保存

# 適用
# 近傍の5つのデータのIndex番号を取得
indices = np.array([annoy_index.get_nns_by_vector(v, 5) for v in embedding_train])# (len(embedding_train), 5)のshapeのarray
```