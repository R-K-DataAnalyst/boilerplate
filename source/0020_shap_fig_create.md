# Shapで可視化画像の調整をする
`show=False`を設定して`fig = plt.gcf()`を使う
```python
# サマリプロット
shap.summary_plot(shap_values, df_shap, show=False)
fig = plt.gcf()
fig.set_size_inches(20, 12)
fig.subplots_adjust(left=0.8, right=0.9)
ax = plt.gca()
ax.set_title('summary plot', fontsize=14)
plt.tight_layout()
plt.show()

# decision_plot
shap.decision_plot(explainer.expected_value
                   , shap_values[:], df_shap
                   , ignore_warnings = True, feature_display_range=slice(-1, -16, -1)
                   , highlight=len(shap_values[:])-1, show=False)#, link='logit', xlim=[125,180], highlight=len(shap_values[20:])-1)
# 可視化
fig = plt.gcf()
fig.set_size_inches(20, 12)
fig.subplots_adjust(left=0.8, right=0.9)
ax = plt.gca()
ax.set_xlabel('Model Output Value', fontsize=12)
ax.set_ylabel('Important Columns', fontsize=12)
plt.yticks(fontsize=12)
plt.tight_layout()
plt.show()
```