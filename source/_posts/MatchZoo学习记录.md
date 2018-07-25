---
title: MatchZoo学习记录
date: 2018-06-13 22:35:44
tags:
  - study
categories: 
  - nlp
  - text match
  - 论文笔记
---
MatchZoo是一个text match领域的tookit,由中科院几位学者研究开源工具，发表了[SIGIR短文][2]。首先，我先从Quora相似问题识别任务出发，探索整个MatchZoo的运行流程与机制。（未完待续！）

## 数据处理
MatchZoo中有Quora的样例，在**data/QuoraQP**目录下有着相应的脚本与处理代码。
我们来查看一下最主要的脚本内容：

```bash

!/bin/bash
# 下载Quora训练语料 
# 下载glove词向量840B和6B

# generate the mz-datasets
python prepare_mz_data.py

# generate word embedding
GLOVE='.'
python gen_w2v.py  $GLOVE/glove.840B.300d.txt word_dict.txt embed_glove_d300
python norm_embed.py embed_glove_d300 embed_glove_d300_norm
python gen_w2v.py  $GLOVE/glove.6B.50d.txt word_dict.txt embed_glove_d50
python norm_embed.py embed_glove_d50 embed_glove_d50_norm

# generate idf file
cat word_stats.txt | cut -d ' ' -f 1,4 > embed.idf

# generate data histograms for drmm model
python gen_hist4drmm.py 60

# generate data bin sums for anmm model
python gen_binsum4anmm.py 20 # the default number of bin is 20

echo "Done ..."
```
### 1. prepare_mz_data.py 探究
从上面的shell脚本，我们可以看出，在数据下载好之后，先运行prepare_mz_data.py 。本小节，我们来探究这段代码。（注：该代码是Quora识别任务前期数据处理的核心代码。） [代码地址][1]

 - 从main函数看起，

``` python
    if __name__ == '__main__':
        prepare = Preparation() # 该类是数据准备
        srcdir = './'
        dstdir = './'

        infile = srcdir + 'quora_duplicate_questions.tsv' # 读取语料数据
        #infile = srcdir + 'train.csv'
        corpus, rels = prepare.run_with_one_corpus_for_quora(infile) # 因为这里是一个总体的文件，调用该函数，如果是下载的已经切分好训练集，验证集， 测试集
        prepare.save_corpus(dstdir + 'corpus.txt', corpus) #corpus.txt 记录的是qid -- ques
        rel_train, rel_valid, rel_test = prepare.split_train_valid_test(rels, [0.8, 0.1, 0.1]) # 切分训练语料， 验证集， 测试集， 比例是 8:1:1, 保存！
    
    # Preprocess 是数据处理最主要的类， 该类的参数是各种功能配置，官方的参数配置里有两个配置
    # word_stem_config：是用于是否调用nltk进行词干处理，word_filter_config：词过滤配置，词频小于5扔掉
    preprocessor = Preprocess(word_stem_config={'enable': False}, word_filter_config={'min_freq': 5})
    dids, docs = preprocessor.run(dstdir + 'corpus.txt')
    preprocessor.save_word_dict(dstdir + 'word_dict.txt') # 词典， 形式：词 -- id， 如： 吃饭 12
    preprocessor.save_words_stats(dstdir + 'word_stats.txt') #形式为： 词序， 词频cf, 文档频df, idf，如：12 5 6 2.51 （这里的12对应是吃饭，5是cf, 6是df, 2.51是idf, 这里idf计算有个公式，这里的数值是假的）
```

上面这段代码，最重要的是要追究**dids, docs = preprocessor.run(dstdir + 'corpus.txt')**，下面，我们来看一下相关代码：
```python
    def run(self, file_path): # 这里的file_path是对应上面代码第9行保存的courps文件地址
        print('load...')
        dids, docs = Preprocess.load(file_path) # dids:存储句子id的链表，docs:句子链表

        if self._word_seg_config['enable']: # 分词配置，对每个句子进行分词，英文调用nltk分词，中文用jieba分词
            print('word_seg...')
            docs = Preprocess.word_seg(docs, self._word_seg_config)

        if self._doc_filter_config['enable']: # 限制句子长度, 默认无限制，不满足条件的去除
            print('doc_filter...')
            dids, docs = Preprocess.doc_filter(dids, docs, self._doc_filter_config)

        if self._word_stem_config['enable']: # 官方配置选择了False, 这个是调用nltk的词干处理
            print('word_stem...')
            docs = Preprocess.word_stem(docs)

        if self._word_lower_config['enable']: # 大写转换成小写
            print('word_lower...')
            docs = Preprocess.word_lower(docs)

        # 对句子进行词频cf，文档频率df,idf数据统计
        # 格式：{"词":{"df":num, "idf":num, "cf":num }}
        self._words_stats = Preprocess.cal_words_stat(docs)

        if self._word_filter_config['enable']: # 过滤停用词和设定的词频
            print('word_filter...')
            docs, self._words_useless = Preprocess.word_filter(docs, self._word_filter_config, self._words_stats)

        print('word_index...') # 重新对应 词-id
        docs, self._word_dict = Preprocess.word_index(docs, self._word_index_config)

        return dids, docs
```
暂时先到这里，下次会继续更新，并且补充完整MatchZoo的整体框架图！

[1]: https://github.com/faneshion/MatchZoo/blob/master/data/QuoraQP/prepare_mz_data.py
[2]: https://arxiv.org/pdf/1707.07270.pdf