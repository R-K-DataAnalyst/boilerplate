# OptunaのLightGBMTunerCV備忘録
意外にLGBMのOptuna最適化をしたことがないことに気づいたのでコードを書いた。

不均衡データに対して実施したので、損失関数に重みづけしているようなかコードになっている。
```python
# 損失関数に重みづけ
def sample_w(y_train, multip=1):
    '''
    output sample weight (balanced weight)
    y_train:True Train data
    multip:重み調整
    '''
    n_samples=len(y_train)
    n_classes=len(y_train.unique())
    bincount0=len(y_train[y_train==0])
    bincount1=len(y_train[y_train>0])
    class_ratio0=n_samples / (n_classes * bincount0)
    class_ratio1=n_samples / (n_classes * bincount1)
    class_ratio1 = class_ratio1*multip
    class_ratio_param=[class_ratio0,class_ratio1]
    print('class_ratio_param',class_ratio_param)
    w0=class_ratio0 #weight associated to 0's
    w1=class_ratio1 #weight associated to 1's
    sample_weight=np.array([w0 if r==0 else w1 for r in y_train])
    return sample_weight

# 損失関数に重みづけするLGBMモデル作成
# OptunaのLightGBMTunerCV使用
# sklearnのAPIは使わない
def lgb_weight(X_train, y_train, multip=1):#, X_valid, y_valid, X_test):
    params = {'task': 'train',
              'boosting_type': 'gbdt',
              'objective': 'binary',
              'metric': 'binary_logloss',
              'verbose': -1,
              'random_state': 0,  # 乱数シード
             }

    w_train=sample_w(y_train, multip=multip)
    #w_valid=sample_w(y_valid, multip=multip)
    lgb_train = opt_lgb.Dataset(X_train, y_train, weight=w_train)
    #lgb_valid = opt_lgb.Dataset(X_valid, y_valid, reference=lgb_train, weight=w_valid)
    skf = StratifiedKFold(n_splits=5, shuffle=True, random_state=0)
    # LightGBM学習
    #lgb_results = {}
    # CV使用Ver
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
    
    # CV使用しないVer
    #model = opt_lgb.train(params
    #                      , lgb_train
    #                      , num_boost_round=1000
    #                      , valid_sets=[lgb_train, lgb_valid]
    #                      , callbacks=[opt_lgb.early_stopping(stopping_rounds=50
    #                                                          , verbose=True)# early_stopping用コールバック関数
    #                                   #, opt_lgb.log_evaluation(10)
    #                                   , opt_lgb.record_evaluation(lgb_results)]
    #                     )
    
    #y_pred = model.predict(X_test, num_iteration=model.best_iteration)
    #y_pred_proba=model.predict_proba(X_test)[:, 1]
    #return model, y_pred
    return tuner_cv

# LightGBMTunerCVでモデル作成時、n_splits数だけモデルができる
# すべてのモデルの結果の平均をとる関数
def cv_model_output(models, X_test):
    preds = []
    for mdl in models:
        pred = mdl.predict(X_test)
        preds.append(pred)
    pred = np.mean(np.array(preds), axis=0)
    return pred
```