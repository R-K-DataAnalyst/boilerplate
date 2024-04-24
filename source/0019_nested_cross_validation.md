# ダブルクロスバリデーション（Double Cross Validation, Nested Cross Validation）のコード

NGboost
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

LightGBM
```python
# ハイパーパラメータ探索
search_params = False
if search_params:
    # LightGBMのハイパーパラメーター探索（OptunaのLightGBMTunerCV使用）
    params = {'task': 'train',
              'boosting_type': 'gbdt',
              'objective': 'binary',
              'metric': 'binary_logloss',
              'verbose': -1,
              'random_state': 0,  # 乱数シード
             }
    
    lgb_train = opt_lgb.Dataset(X_train, y_train, weight=sample_ws)  # set sample weights
    skf = sklearn.model_selection.StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
    # LightGBM学習
    tuner_cv = opt_lgb.LightGBMTunerCV(params, lgb_train
                                       , num_boost_round=1000
                                       , folds=skf
                                       , return_cvbooster=True
                                       , optuna_seed=0
                                       , callbacks=[opt_lgb.early_stopping(stopping_rounds=50, verbose=True)])
    
    # 最適なパラメータを探索する
    tuner_cv.run()
    # 最も良かったスコアとパラメータを書き出す
    print(f'Best score: {tuner_cv.best_score}')
    print('Best params:')
    print(tuner_cv.best_params)
    
params = tuner_cv.best_params

# 良さげな`num_boost_round`を探すために、nested cross validation
# early_stopping_roundsにかけるか否か
adjust = False
# shap計算するか否か
shap_culc = True

# Outer loop
cv_outer = sklearn.model_selection.StratifiedKFold(n_splits=5, shuffle=True, random_state=0)
# Inner Loop
# `num_boost_round`探索のため、Inner Loopでearly_stopping_roundsにかける
cv_inner = sklearn.model_selection.StratifiedKFold(n_splits=5, shuffle=True, random_state=0)

# 結果格納のリスト
outer_trues = []
outer_preds = []
models = []
Xt_outer = X_train.copy()
yt_outer = y_train.copy()
shap_exp = []
shap_val0 = []
shap_val1 = []
valids = []

# Outer Loop
for cv_i, (train_ix, valid_ix) in tqdm(enumerate(cv_outer.split(Xt_outer, yt_outer))):
    # 学習データと検証データ
    Xt, Xv = Xt_outer.iloc[train_ix, :], Xt_outer.iloc[valid_ix, :]
    yt, yv = yt_outer.iloc[train_ix], yt_outer.iloc[valid_ix]
    sample_ws_t = sample_ws[train_ix]
    sample_ws_v = sample_ws[valid_ix]
    if adjust:
        itrs = []
        # Inner Loop
        for train_ix2, valid_ix2 in (cv_inner.split(Xt, yt)):
            Xt_inner, Xv_inner = Xt.iloc[train_ix2, :], Xt.iloc[valid_ix2, :]
            yt_inner, yv_inner = yt.iloc[train_ix2], yt.iloc[valid_ix2]
            sample_ws_t_t = sample_ws_t[train_ix2]
            lgb_train = lightgbm.Dataset(Xt_inner, yt_inner, weight=sample_ws_t_t)
            lgb_valid = lightgbm.Dataset(Xv_inner, yv_inner)
            # 最大1000でearly_stoppingをかける
            lgb = lightgbm.train(params, lgb_train, valid_sets=[lgb_valid], num_boost_round=1000
                                 , callbacks=[lightgbm.early_stopping(stopping_rounds=100, verbose=True)  # early_stopping用コールバック関数
                                              , lightgbm.log_evaluation(0)] # コマンドライン出力用コールバック関数)
                                )
            itrs.append(lgb.best_iteration)
        # 良さげな`num_boost_round`はcvの平均とする
        print('n_estimators', int(np.mean(itrs)))
        lgb_train = lightgbm.Dataset(Xt, yt, weight=sample_ws_t)
        lgb = lightgbm.train(params, lgb_train, num_boost_round=int(np.mean(itrs)))
    # early_stopping_roundsにかけないのでInner Loop使わない
    else:
        # Inner Loopにはかけず`num_boost_round`は固定
        lgb_train = lightgbm.Dataset(Xt, yt, weight=sample_ws_t)
        lgb = lightgbm.train(params, lgb_train, num_boost_round=365)

    # 予測値取得
    yhat = lgb.predict(Xv)
    # リストに格納
    outer_preds.append(yhat)  # 予測値
    outer_trues.append(yv.to_numpy())  # 実測値
    models.append(lgb)  # モデル
    valids.append(Xv)  # 検証データ

    # Shap計算
    if shap_culc:
        print('shap value calculation, outer cv {}'.format(cv_i))
        X_valid_shap = Xv.reset_index(drop=True)
        explainer = shap.TreeExplainer(model=lgb)
        shap_values = explainer.shap_values(X=X_valid_shap)
        shap_exp.append(explainer)
        shap_val0.append(shap_values[0])
        shap_val1.append(shap_values[1])
```