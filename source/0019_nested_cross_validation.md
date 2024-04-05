# ダブルクロスバリデーション（Double Cross Validation, Nested Cross Validation）のコード

```python
# 弱学習器
dtr_friedman_3 = DecisionTreeRegressor(criterion='friedman_mse', max_depth=3)

# NGBoostのパラメータ
FIXED_PARAMS = {"Base": dtr_friedman_3,
                "natural_gradient": True,
                "random_state": 0,
                "Dist": Normal, 
                "verbose": False,
                "Score": LogScore,
               }

# early_stopping_roundsにかけるか否か
adjust = True
# shapを見るかどうか
shap_show = True

# Outer loop
cv_outer = sklearn.model_selection.LeaveOneOut()
# Inner Loop
# early_stopping_roundsにかける場合、Inner Loopでストップをかける
cv_inner=sklearn.model_selection.LeaveOneOut()

# 結果格納のリスト
outer_scores = list()
outer_trues = list()
outer_preds = list()
outer_stds = list()
models = list()
for train_ix, valid_ix in tqdm(cv_outer.split(Xt_outer, yt_outer)):
    # 学習データと検証データ
    Xt, Xv = Xt_outer.iloc[train_ix, :], Xt_outer.iloc[valid_ix, :]
    yt, yv = yt_outer.iloc[train_ix], yt_outer.iloc[valid_ix]

    # GridSearchCVをInner Loopでする場合のコード
    # # inner loop
    # param_grid = {
    #     "n_estimators": [100, 500],
    #     #"col_sample": [0.01, 0.1, 1.0],
    #     # "minibatch_frac": [0.1, 1.0],
    #     "learning_rate": [0.01, 0.05, 0.1],
    #     'tol': [1e-4, 0.01]
    # }
    # ngb = ngboost.NGBRegressor(**FIXED_PARAMS)
    # clf = sklearn.model_selection.GridSearchCV(ngb, param_grid=param_grid, cv=cv_inner, scoring='neg_mean_absolute_error')
    # clf.fit(Xt, yt)
    # best_param = clf.best_params_
    # best_param.update(FIXED_PARAMS)
    # ngb = ngboost.NGBRegressor(**best_param)

    # Inner Loopでearly_stopping_roundsにかけて、良さげなn_estimatorsを設定
    if adjust:
        itrs = []
        for train_ix2, valid_ix2 in (cv_inner.split(Xt, yt)):
            Xt_inner, Xv_inner = Xt.iloc[train_ix2, :], Xt.iloc[valid_ix2, :]
            yt_inner, yv_inner = yt.iloc[train_ix2], yt.iloc[valid_ix2]
            ngb = ngboost.NGBRegressor(n_estimators=500, verbose=False)
            ngb.fit(Xt_inner, yt_inner, X_val=Xv_inner, Y_val=yv_inner, early_stopping_rounds=50)
            itrs.append(ngb.best_val_loss_itr)
        print('n_estimators', int(np.mean(itrs)))
        ngb = ngboost.NGBRegressor(**FIXED_PARAMS, n_estimators=int(np.mean(itrs)))
    # early_stopping_roundsにかけないのでInner Loop使わない
    else:
        ngb = ngboost.NGBRegressor(**FIXED_PARAMS, n_estimators=250)
        
    ngb.fit(Xt, yt)
    # 予測値と予測のばらつき取得
    yhat = ngb.pred_dist(Xv).params['loc']
    ystd = ngb.pred_dist(Xv).params['scale']
    # score
    mae = sklearn.metrics.mean_absolute_error(yv, yhat)
    # リストに格納
    outer_scores.append(mae)
    outer_preds.append(yhat[0])
    outer_stds.append(ystd[0])
    outer_trues.append(yv.to_numpy()[0])
    models.append(ngb)

    # 毎回のOuter Loopの検証結果をShapで表示
    if shap_show:
        X_v_shap = Xv.reset_index(drop=True).copy()
        explainer = shap.TreeExplainer(model=ngb, model_output=0)
        shap_values = explainer.shap_values(X=X_v_shap)
        shap.decision_plot(explainer.expected_value
                           , shap_values, X_v_shap
                           , ignore_warnings = True, highlight=[0])
```