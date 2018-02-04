# ranking-specre

## 目的

**spectre(ある複雑な量を単純な成分に分け、ある特定の量の大小によって分布を示したもの。)**  

Intel CPUなどに内在する投機実行のspectreとは用法が違いますのでご注意ください。  

ランキングアルゴリズムは日々進化しています。Googleのサーチエンジンは[200以上の特徴量を用いたり](https://backlinko.com/google-ranking-factors)色々しています。  

これらはGoogleでないと手に入らない特徴量も多数存在しており、容易に、ユーザが最適化できるものではなかったりします  

わかりやすいものでは、ドメイン以内にコンテンツが十分に存在し、それがある程度参照されるものであれば、以前やったようにWelqさんのようにコンテンツの内容の是非によらず、ランクアップしてしまうような意図しない問題もございました。  


## お題
Rankingエンジンのランクの傾向を、検索クエリ結果から、ランキンされたサイトの自然言語的特徴から、獲得しようという意図です。  

これを行うにあたって２つの制約があります。  
- 1. ランキングエンジンのリバースエンジニアリングのような行為は認められるのか
- 2. すべての特徴量を用意することはできないので、自然言語的な特徴量で近似することはできないか

1. に関して述べると、この行為は、例えばGoogleやBingなどのサーチエンジンの競合を作るという意図がない、広域に解釈すれば、ビジネスでなければよいと言えそうである。自己研究に基づくのでこの項目に関しては問題がない。  

2. では類似度が文章の長さなど、Googleは一般公開している特徴量を200を超える量を示しており、現実的には前処理が多すぎる事になり、これらをカバレッジすることはできない。なので、独自に設定した特徴量で今回は行うので、問題はない。

Microsft Bingのランキングエンジンのクエリ(neologdに内在している辞書を利用)することによって、膨大なダイバシティの検索結果が得られます。  
検索結果のランキングを見ることにより、同等のランキングエンジンが作成可能であると期待できます  


## ランキングアルゴリズム一覧
- lambdarank
- lambdamart

## 読むべき論文
- [From RankNet to LambdaRank to LambdaMART: An Overview](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/MSR-TR-2010-82.pdf)

## よく見る評価指標
- [NDCG](https://en.wikipedia.org/wiki/Discounted_cumulative_gain)

## 目的関数
- [PairWise(xgb)](https://github.com/dmlc/xgboost/tree/master/demo/rank)

## ranksvmフォーマット
資料が全くなく、調査して理解するまで結構かかりました。  
ranksvmフォーマットはgroup fileというのが別途必要になっている
<p align="center">
  <img width="550px" src="https://user-images.githubusercontent.com/4949982/35681621-45722bd0-07a1-11e8-9008-b10c1fdd3082.png">
</p>

## bingのデータクローン
- neologdで一般単語をとりだす
- bingでクエリを作成して、ひたすら大量に集める
- 1位=4, ２位=3, 3位=2, ４位=1で、単一ページでないドメイントップのサイトをランキングする  
- 言語処理的にtitle, meta, bodyの自然言語でランキングしてみる

(本当はこれ+DeepLearningでやってもいい)

## オペレーション

### OP1. ランククエリ生成
neologdなどからnoun（名詞）を取り出して、それを検索クエリ群にする  
```python
import glob
import pickle                                                                                                                                                                                                     import json                                                                                                                                                                                        nouns = []
for name in glob.glob('mecab-ipadic-neologd/build/*/*.csv'):
   f = open(name)
   for line in f:
     ents = line.strip().split(',')
     if '名詞' not in ents:
       continue
     term = ents[0]
     nouns.append(term)
open('nouns.json', 'w').write( json.dumps(nouns, indent=2, ensure_ascii=False) )
```

#### OP2. bingをスクレイピング
```console
$ python3 scrape.py
```

#### OP3. フルドメインが入っているリンクをパース
```console
$ python3 scan_pair.py
```

#### OP4. bingの結果からフルドメインをスクレイピング
```console
$ python3 pair_scrape.py
```
#### OP5. フルドメインのHTMLをパース
```console
$ python3 10-parse-htmls.py
```
#### OP6. 分かち書きして特徴量化
```console
$ python3 20-make-vector.py
```
#### OP7. 疎行列で表現してranksvm形式のファイルを出力 
```console
$ python3 30-index.py
```

## 学習
OP7を実行すると学習可能なファイル群が出力されます  
(xgboostのバイナリがlibcなどの互換がなくて実行できない場合は、xgboostを再コンパイルしてください)  
```console
$ cd rank
$ ./xgb mq_train.conf 
```
map(mean average precision)の略で、平均精度です。pairwiseで評価すると、mapでの評価になります。他の関数のndcgなどはうまく動作しません。なぜ？  

1000roundでの精度はこの程度です  
```console
[20:36:05] src/tree/updater_prune.cc:74: tree pruning end, 1 roots, 936 extra nodes, 1252 pruned nodes, max_depth=36                                                                                 
[20:36:05] [999]        test-map:0.721291        
```

学習が完了すると、**rank.model**というファイルが出力されます  

## 予想
**rank.model**をもちいて未知のクエリに対してランキングすることができます  
```console
$ ./xgb mq_predict.conf 
[20:51:20] 144775x162216 matrix with 3899318 entries loaded from test.data
[20:51:20] start prediction...
[20:51:49] writing prediction to pred.txt
```
pred.txtの中にランキングされたファイルが記されています。  

単純なSVMなどを利用すると、簡単に配信システムで計算できますが、勾配ブーストのようなランキングはどうなんですかね。決定木なのでC++のファイルなどに変換させるのが良いと思います  





