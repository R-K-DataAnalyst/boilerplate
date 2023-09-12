FROM python:3.10
USER root

RUN apt update
RUN apt -y install locales && \
    localedef -f UTF-8 -i ja_JP ja_JP.UTF-8

ENV LANG ja_JP.UTF-8
ENV LANGUAGE ja_JP:ja
ENV LC_ALL ja_JP.UTF-8
ENV TZ JST-9
ENV TERM xterm

RUN apt install sudo
RUN apt install -y vim less
RUN apt install -y --no-install-recommends graphviz
RUN apt install -y fonts-ipaexfont
# vim install and config
RUN apt install -y vim-gtk3
RUN touch ~/.vimrc
RUN echo 'set clipboard&' >> ~/.vimrc
RUN echo 'set clipboard^=unnamedplus' >> ~/.vimrc
RUN ["/bin/bash", "-c", "source ~/.vimrc"]

# for pymc
RUN apt install libblas-dev -y
# for mecab
RUN apt install mecab -y
RUN apt install libmecab-dev -y
RUN apt install mecab-ipadic-utf8 -y
RUN git clone --depth 1 https://github.com/neologd/mecab-ipadic-neologd.git
RUN mecab-ipadic-neologd/bin/install-mecab-ipadic-neologd -n -y
RUN mv /usr/lib/x86_64-linux-gnu/mecab/dic/mecab-ipadic-neologd /var/lib/mecab/dic
RUN mv /etc/mecabrc /etc/mecabrc_old
RUN echo 'dicdir = /var/lib/mecab/dic/mecab-ipadic-neologd' >> /etc/mecabrc
RUN cp /etc/mecabrc /usr/local/etc/

RUN mkdir -p /root/work
COPY requirements.txt /root/work
WORKDIR /root/work

RUN pip install --upgrade pip
RUN pip install --upgrade setuptools
RUN pip install -r requirements.txt
# jupyter config
RUN jupyter lab --generate-config
RUN echo "c.NotebookApp.ip = '0.0.0.0'" >> /root/.jupyter/jupyter_lab_config.py
RUN echo "c.NotebookApp.port = 8888" >> /root/.jupyter/jupyter_lab_config.py
RUN echo "c.NotebookApp.token = ''" >> /root/.jupyter/jupyter_lab_config.py
RUN echo "c.NotebookApp.allow_root = True" >> /root/.jupyter/jupyter_lab_config.py
RUN echo "c.NotebookApp.open_browser = False" >> /root/.jupyter/jupyter_lab_config.py

