# shap.dependence_plotでラベルエンコーディングした変数の目的変数への影響を見る
```python
# shap dependence plot
# 各カラムのカテゴリーが目的変数にどう効いているか確認
cat_cols = ['A','B','C','D','E','F','G','H','I']
dim = len(cat_cols)
fig=plt.figure(figsize=(30,30))
for i, cat_col in enumerate(cat_cols):
    ax = plt.subplot(round(np.ceil(dim/np.sqrt(dim))), round(np.ceil(dim/np.sqrt(dim))), i+1)
    shap.dependence_plot(cat_col, shap_value  # ラベルエンコーディング後のdfで得たshap値
                        , X_valid_shap  # ラベルエンコーディング後のdf
                        , display_features=X_valid_shap.replace(le_list)  # ラベルエンコーディング前のdf
                        , show=False, interaction_index='Z', ax=ax)  # 相互作用はZを見る

    ax.set_xlabel('Categories')
    ax.set_ylabel('SHAP Values')
    ax.set_title(cat_col)
    ax.axhline(y=0, c='gray', ls='--')
    plt.rcParams["font.family"] = prop.get_name()
plt.tight_layout()
plt.show()
```