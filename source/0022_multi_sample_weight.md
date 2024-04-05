# 多クラス分類でsample weightを計算する関数
2クラスだけでなく多クラスにも対応。
```python
# 重みづけ
def sample_w(y_train):
    '''
    output sample weight (balanced weight)
    y_train:True Train data
    '''
    n_samples=len(y_train)
    n_classes=len(np.unique(y_train))
    bincounts = {i:len(y_train[y_train==i]) for i in sorted(np.unique(y_train))}
    class_ratio_param = {key:n_samples / (n_classes * bincnt) for key, bincnt in bincounts.items()}
    print('class_ratio_param',class_ratio_param)
    sample_weight=np.array([class_ratio_param[r] for r in y_train])
    return sample_weight
```