# mlflowの備忘録
mlflowにpickleとして保存したモデルをLoadする方法。
<br>
experiment_nameをもとにexperiment_idを取得して、取得したexperiment_idのrun_ind番目のrun_idを取得して、取得したrun_idに保存しているpickleファイルをロードする。
```python
# mlflowからpickleとして保存したモデルをLoad
def load_model_from_mlflow(obj_col, mlflow_dir='mlruns', run_ind=0, mlflow_experiment_name=''):
    # get experiments
    mlflow.set_experiment(mlflow_experiment_name+':'+obj_col)# 実験の名前
    tracking = mlflow.tracking.MlflowClient()
    # experiment_nameからexperiments_id取得
    experiments_id = tracking.get_experiment_by_name([m for m in [n.name for n in tracking.list_experiments()] if obj_col in m and mlflow_experiment_name in m][0]).experiment_id
    run_id = tracking.list_run_infos(experiments_id, run_view_type=ViewType.ACTIVE_ONLY, order_by=["attribute.start_time DESC"])[run_ind].run_id# run_ind番目のrun_idを取得
    modeldir = mlflow_dir+'/'+experiments_id+'/'+run_id+'/'+'artifacts'+'/'+obj_col# 保存しているartifactsのディレクトリpath
    # モデルのLoad
    with open(modeldir+'/model.pkl', 'rb') as web:
        model = pickle.load(web)
    return model
```

mlflowの既存のexperimentのrunに記録を追記する方法。
```python
### mlflowに記録を追記 ###
mlflow.set_experiment(mlflow_experiment_name)# 実験の名前
tracking = mlflow.tracking.MlflowClient()
# experiment_nameからexperiments_id取得
experiments_id = tracking.get_experiment_by_name([m for m in [n.name for n in tracking.list_experiments()] if obj_col in m and mlflow_experiment_name in m][0]).experiment_id
run_id = tracking.list_run_infos(experiments_id, run_view_type=ViewType.ACTIVE_ONLY, order_by=["attribute.start_time DESC"])[run_ind].run_id# 最新のrun_idを取得
# 取得したrun_idでrunをStartして記録する
with mlflow.start_run(run_id=run_id):
    mlflow.log_artifact('new_artifact.png', artifact_path='new_artifact')
```