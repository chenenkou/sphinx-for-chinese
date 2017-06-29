# sphinx-for-chinese

一款专注于中文搜索的全文检索软件，在sphinx的基础上添加了中文处理模块并优化了中文搜索效果
sphinx-for-chinese项目起始于2008年12月份，在2009年4月份在Google Code（http://code.google.com/p/sphinx-for-chinese ）上推出第一个公开版本，日后陆续根据原版sphinx的更新第一时间内做出升级，在2011年10月份推出独立官方网站


## 安装

解压 
$ tar -xvf sphinx-for-chinese-2.0.2-dev-r2894.tar.gz
$ cd sphinx-for-chinese-2.0.2-dev-r2894

编译（假设安装到/usr/local/sphinx-for-chinese目录，下文同）
$ ./configure --prefix=/usr/local/sphinx-for-chinese
--prefix 指定安装路径
--with-mysql 编译mysql支持
--with-pgsql 编译pgsql支持
$ make
$ make install

配置中文支持

$ tar -xvf xdict_1.1.tar.gz
$ /usr/local/sphinx-for-chinese/bin/mkdict xdict_1.1.txt xdict #从xdict_1.1.txt生成xdict文件，xdict_1.1.txt文件可以根据需要进行修改
$ cp xdict /usr/local/sphinx-for-chinese/etc/

修改sphinx.conf索引配置文件

在索引配置项中添加以下两项
charset_type = utf-8
chinese_dictionary = /usr/local/sphinx-for-chinese/etc/xdict

至此，完成中文支持配置。


## 一些注意事项

sphinx-for-chinese只支持UTF-8编码，数据源输出数据时请做转换，使用MySQL时一般需要添加"SET NMAES utf8"语句。 使用xmlpipe时，需要注意两点：一个是XML中尽可能使用CDATA标签，以避免特殊字符影响xml解析；另一个是sphinx配置中启用xmlpipe_fixup_utf8=1选项，以尽可能的避免因非 法UTF-8字符串引起解析错误。
若需要检查中文分词支持是否启用，请使用search命令，例子如下：

./search -c ../etc/sphinx.conf 分享身边的精彩
sphinx-for-chinese 2.1.0-dev (r3006)
Copyright (c) 2008-2011, sphinx-search.com

using config file '../etc/sphinx.conf'...
index 'test1': query '分享身边的精彩 ': returned 0 matches of 0 total in 0.000 sec

words:
1. '分享': 6 documents, 7 hits
2. '身边': 26 documents, 38 hits
3. '的': 5344 documents, 178743 hits
4. '精彩': 5 documents, 6 hits

可以看到words中列出了各个中文单词，说明中文分词启用成功。

出现乱码时，请检查数据源的编码是否为UTF-8，程序API中的调用是否为UTF-8，若为命令行测试，请检查终端环境是否为UTF-8。windows的命令行环境为GBK，若在windows的命>令行下进行测试，请注意输入数据的编码。

如果数据源不是MySQL，而是oracle、纯文本或者其他数据源，可以采用xmlpipe的方式进行索引。具体方法是采用容易快速开发的语言，如PHP，Python，Ruby或者Lua（C，C++等 当然也可以）等读取数据源，然后按照既定格式输出XML格式的数据，供sphinx读取。99.9％的情况下sphinx是可以索引任何数据的，不需要额外的低层处理。


## 中文搜索优化

对中文进行全文检索时，一般需要进行中文分词（segmentation）。中文分词的过程，一般叫做tokenize，也就是将一段文本分 成多个token，索引的时候对每个token进行倒排索引（inverted index）。中文与拉丁语系不同，例如英文单词之间用空格做区分，而中文没有明显的单词分隔，这就需要算法对 中文字符串进行分词，而分词的精确度就会影响中文搜索的效果。举个例子，“研究生命起源”如果分成“研究生”“命”“起源”，用“研究”一词是搜索不到的，用“生命”也搜索不到； 如果分成“研究”“生命”“起源”，则用“研究”和“生命”都可以搜索到。同理，如果“上海市”被分成“上海”“市”，用“上海”是搜索不到的。

为了提高搜索效果，一般可以：

提高分词精度。这个一般与分词算法有关，而现在常用的基于词典的分词算法在分词精度上差别不大（不会有数量级的差别）。另外可以对词典进行调整，比如针对医药类网站， 可以在词典中添加医药类词库，针对特殊行业领域进行词典优化，也可以提高分词效果。关于词库，一般可以参考搜狗细胞词库。

采用同义词、近义词处理。这一部分主要是针对“餐厅”“餐馆”或者“上海”“上海市”等同义或者近义处理。现在有些针对索引的分词算法采用多分的处理方法，比如将“上海市”分为 “上海市”“上海”“市”，这样“上海市”“上海”都可以搜索到，但这样会增加token数量，增加索引数据，影响搜索效率，并且单纯靠算法对多分的处理粒度很难控制；另外还有将同>义词词库整合到全文检索里的做法，这样会增加搜索程序的复杂度，不利于升级和微调。这里建议的做法是将同义词、近义词的处理放到搜索外围，即对用户输入的搜索语句进行 处理和转换，利用sphinx的搜索语法进行处理。具体做法，可以整理一份同义词、近义词的词库，利用内存型数据库保存，作成daemon或者web service的接口，对用户的搜索输>入进行预处理，不仅开发成本低，速度快，而且模块化高，容易调整，利于升级。


## 搜索性能优化以及高可用容错集群搭建

当索引数据过大或者访问量过大时，可以：

对索引数据进行分区，这一部分与数据库拆表十分类似。即把数据水平或者垂直分区，并在应用程序里做一些调整，不同的搜索请求，分配到不同的索引。这种做法的思路，还是 尽可能的减少单个索引数据块（index data block）的大小，进而减少每一次请求所需扫描数据的大小，提高响应时间。

合理的更新策略。根据更新频率，采用main+delta的两层处理或者main+today+delta的三层处理，也可以减少更新负担，提高索引数据的更新速度。这一部分通常需要具体问题具 体分析。

采用分布式处理，即将数据水平分区，分布在多台机器上。这一部分可以参考http://sphinxsearch.com/docs/2.0.2/distributed.html。

高可用和容错处理。一个是采用replication的处理方法，即一台机器作为master负责索引更新，不接受外部请求，另外多台机器运行sphinx实例，作为slave接受外部请求。master通过inotify和rsync更新slave上的索引数据，外部请求根据算法分配到多个slave上，实现负载均衡和容错处理。同时，还可以利用HAProxy和VRRP实现高可用性和容错集群的>搭建。另外一个方法，是采用sphinx自带的分布式处理方法，并结合heartbeat或者VRRP实现容错处理。
