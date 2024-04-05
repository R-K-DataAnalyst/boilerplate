# 共起ネットワーク作成
ジャカード係数を計算して、共起ネットワーク作成。
```python
%%time
# 共起ネットワークのためのdf作成
def make_jaccard_df(df_jacc, col):
    sentences = df_jacc[col].to_list()
    # 単語組み合わせのリスト作成
    sentence_combinations = [list(itertools.combinations(sentence, 2)) for sentence in tqdm(sentences[:])]
    sentence_combinations = [[tuple(sorted(words)) for words in sentence] for sentence in tqdm(sentence_combinations)]
    
    #  単語の組合せの1次元のリストに変形
    target_combinations = []
    for sentence in tqdm(sentence_combinations):
        target_combinations.extend(sentence)

    ### Jaccard係数を求める ###
    # Jaccard係数 = n(A ∩ B) / n(A ∪ B)
    # 同じ文内にある２つの単語の出現回数
    combi_count = Counter(target_combinations)

    # 単語の組合せと出現回数
    word_associates = []
    for key, value in tqdm(combi_count.items()):
        word_associates.append([key[0], key[1], value])

    word_associates = pd.DataFrame(word_associates, columns=['word1', 'word2', 'intersection_count'])

    # 和集合の計算 n(A ∪ B) = n(A) + n(B) - n(A ∩ B) を利用
    # それぞれの単語の出現回数を計算
    target_words = []
    for word in target_combinations:
        target_words.extend(word)

    word_count = Counter(target_words)
    word_count = [[key, value] for key, value in word_count.items()]
    word_count = pd.DataFrame(word_count, columns=['word', 'count'])

    # 単語の組合せの出現回数のデータにそれぞれの単語の出現回数を結合
    word_associates = pd.merge(word_associates, word_count, left_on='word1', right_on='word', how='left')
    word_associates.drop(columns=['word'], inplace=True)
    word_associates.rename(columns={'count': 'count1'}, inplace=True)
    word_associates = pd.merge(word_associates, word_count, left_on='word2', right_on='word', how='left')
    word_associates.drop(columns=['word'], inplace=True)
    word_associates.rename(columns={'count': 'count2'}, inplace=True)
    word_associates['union_count'] = word_associates['count1'] + word_associates['count2'] - word_associates['intersection_count']
    print('Jaccard係数の算出')
    word_associates['jaccard_coefficient'] = word_associates['intersection_count'] / word_associates['union_count']
    word_associates=word_associates.sort_values('jaccard_coefficient')
    return word_associates

# 共起ネットワークを表示する関数
def plot_network(data, edge_threshold=0., fig_size=(20, 10), file_name=None, dir_path=None, prog='fdp'):

    nodes = list(set(data['node1'].tolist()+data['node2'].tolist()))

    G = nx.Graph()
    #  頂点の追加
    G.add_nodes_from(nodes)

    #  辺の追加
    #  edge_thresholdで枝の重みの下限を定めている
    for i in range(len(data)):
        row_data = data.iloc[i]
        if row_data['value'] > edge_threshold:
            G.add_edge(row_data['node1'], row_data['node2'], weight=row_data['value'])

    # 孤立したnodeを削除
    isolated = [n for n in G.nodes if len([i for i in nx.all_neighbors(G, n)]) == 0]
    for n in isolated:
        G.remove_node(n)

    plt.figure(figsize=fig_size)
    #pos = nx.spring_layout(G, k=0.3)  # k = node間反発係数
    #pos = nx.kamada_kawai_layout(G)
    pos = nx.nx_agraph.graphviz_layout(G, prog=prog)
    
    pr = nx.pagerank(G)
    #pr = nx.degree_centrality(G)
    #pr = nx.eigenvector_centrality(G)
    #pr = nx.closeness_centrality(G)

    # nodeの大きさ
    nx.draw_networkx_nodes(G, pos, node_color=list(pr.values()),
                           cmap=plt.cm.Reds,
                           alpha=0.7,
                           node_size=[60000.*v for v in pr.values()])

    # 日本語ラベル
    nx.draw_networkx_labels(G, pos, font_size=12, font_family='IPAexGothic', font_weight="bold")

    # エッジの太さ調節
    edge_width = [d["weight"] * 300 for (u, v, d) in G.edges(data=True)]
    nx.draw_networkx_edges(G, pos, alpha=0.4, edge_color="darkgrey", width=edge_width)

    plt.axis('off')
    plt.title('data length:{}, edge threshold:{}'.format(round(len(word_associates2)), round(edge_threshold,4)))
    #plt.savefig("png/network_"+name+"_"+okng+".png")
    plt.show()


# 共起ネットワークのためのdf作成
word_associates = make_jaccard_df(df_jacc, 'all_connect_wakachiSentenceList')
display(word_associates)

# 制限つけて共起ネットワーク可視化
n_word_lower = len(df_jacc)*0.1
edge_threshold = word_associates['jaccard_coefficient'].quantile(0.999)#0.01
word_associates2=word_associates.copy()
word_associates2.query('count1 >= @n_word_lower & count2 >= @n_word_lower', inplace=True)
word_associates2.query('jaccard_coefficient >= @edge_threshold', inplace=True)
word_associates2.rename(columns={'word1':'node1', 'word2':'node2', 'jaccard_coefficient':'value'}, inplace=True)
word_associates2 = word_associates2[word_associates2['node1']!=word_associates2['node2']]
display(word_associates2.head())
plot_network(data=word_associates2, edge_threshold=edge_threshold, prog='fdp')  # circo, dot, fdp, neato, nop, nop1, nop2, osage, patchwork, sfdp, twopi

```