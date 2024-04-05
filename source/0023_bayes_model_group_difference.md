# 階層ベイズモデリングで個人ではなくグループごとの異質性を見たいとき
個体ことの所属グループ配列`group_idx`を定義してランダム効果`r_coef_`をグループ数×説明変数のshapeとしておき、`(coef_+r_coef_)[group_idx,:]`で係数がグループ数分のランダム効果が計算できる。
```python
# モデルの定義
with pm.Model() as model_random2:
    # coords
    model_random2.add_coord('data', values=range(train_df.shape[0]), mutable=True)
    model_random2.add_coord('var', values=purchasecols, mutable=True)
    model_random2.add_coord('group_num', values=np.unique(group_idx_org_enc), mutable=True)

    # 説明変数
    x = pm.MutableData('x', train_df[purchasecols].to_numpy(), dims=('data', 'var'))
    y = pm.MutableData("y", target, dims=('data', ))
    group_idx = pm.MutableData("group_idx", group_idx_org_enc, dims=('data', ))

    # 推論パラメータの超事前分布
    super_mu_keisuu = pm.StudentT('super_mu_keisuu', nu=4, mu=0, sigma=sigma, dims=('var'))  # ランダム係数の平均の超事前分布
    super_mu_seppen = pm.StudentT('super_mu_seppen', nu=4, mu=0, sigma=sigma)  # ランダム切片の平均の超事前分布
    #super_sigma_keisuu = pm.HalfStudentT('super_sigma_keisuu', nu=4, dims=('var'))  # MCMC収束しなかったので設定しない
    #super_sigma_seppen = pm.HalfStudentT('super_sigma_seppen', nu=4)  #  MCMC収束しなかったので設定しない
    
    # 推論パラメータの事前分布
    coef_ = pm.StudentT('coef', nu=4, mu=0, sigma=sigma, dims=('var'))  # 係数の事前分布
    intercept_ = pm.StudentT('intercept', nu=4, mu=0, sigma=sigma)  # 切片の事前分布 
    r_coef_ = pm.StudentT('r_coef_', nu=4, mu=super_mu_keisuu, sigma=sigma, dims=('group_num', 'var'))  # ランダム係数の事前分布  データ×係数の数分の値がある
    r_intercept_ = pm.StudentT('r_intercept_', nu=4, mu=super_mu_seppen, sigma=sigma, dims=('group_num', ))  # ランダム切片の事前分布   データ分の値がある
    
    # linear model
    mu = pm.Deterministic("mu", pm.math.sum((coef_+r_coef_)[group_idx,:]*x, axis=1) + intercept_ + r_intercept_[group_idx], dims=('data', ))
    # # link function
    # link = pm.Deterministic("link", pm.math.exp(mu), dims=('data', ))
    # # likelihood
    # result = pm.Poisson("obs", mu=link, observed=y, dims=('data', ))
    result = pm.Normal('obs', mu=mu, sigma=1, observed=y, dims=('data', ))
```