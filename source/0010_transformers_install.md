# pytorch, transformersのインストール備忘録

" https://pytorch.org/ "で、自分の環境にあったインストール方法がわかる。

![pytorch_install](0010_transformers_install/pytorch_install.png)

また、日本語でtransformersを使う場合、fugashiやipadicが必要。

なので、pytorch, transformersをインストールするときは以下を実行。

```python
pip install torch torchvision torchaudio
pip install transformers
pip install fugashi
pip install ipadic
```