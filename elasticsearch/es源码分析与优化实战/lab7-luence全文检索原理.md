Lucene 是一个高效的，基于Java 的全文检索库。

说起查找，我们首先想起的就是顺序查找，比如我们有10个文档，要查找含有lucene单词，我们会依次去遍历所有的文档进行查找，直到找到含有这个单词的文档。 这就是一种是顺序扫描法。

小数据量下这种还是很方便，但是大数据量下就很麻烦。

对于大数据下的结构数据，我们可以创建索引（数据库中采用B+树），那么非结构数据进行结构化，然后创建索引就是我们的快速查找的思路。

##### 全文检索大体分两个过程，索引创建 (Indexing) 和搜索索引 (Search) 。

###### 索引创建：将现实世界中所有的结构化和非结构化数据提取信息，创建索引的过程。

###### 搜索索引：就是得到用户的查询请求，搜索创建的索引，然后返回结果的过程。

### 创建索引的流程

##### 1. 将要索引的文档/数据库/字符串导入到lucene中



```dart
//创建文档1
Document document = new Document();
//向文档中添加域
document.add(new TextField(FIELD,"Students should be allowed to go out with their friends, but not allowed to drink beer.", Field.Store.YES));

//创建文档2
Document document1 = new Document();
//向文档中添加域
document1.add(new TextField(FIELD,"My friend Jerry went to school to see his students but found them drunk which is not allowed.", Field.Store.YES));

//将文档添加到本地索引中
indexWriter.addDocument(document);
indexWriter.addDocument(document1);
```

##### 2. 将原文档传给分词器(Tokenizer)

当然这里传给分词器的操作在addDocument后会自行调用，因为这里选择的是TextField，默认是告诉lucene要进行分词，如果为StoredField则不会进行分词操作。

对文档的分解操作都是IndexChain的工作。



```dart
 // 构建索引
//注意这里一定要进行配置分词器，默认的分词器是按照英文设置的，这里采用的版本号默认是8.0.0
IndexWriterConfig config = new IndexWriterConfig(new StandardAnalyzer());
IndexWriter indexWriter = new IndexWriter(dirLucene,config);

......

//将文档添加到本地索引中
indexWriter.addDocument(document);
indexWriter.addDocument(document1);
```

分词器(Tokenizer)会做以下几件事情( 此过程称为Tokenize) ：

- 将文档分成一个一个单独的单词。
- 去除标点符号。
- 去除停词(Stop word) 。

停词(Stop word)就是一种语言中最普通的一些单词，由于没有特别的意义，因而大多数情况下不能成为搜索的关键词，因而创建索引时，这种词会被去掉而减少索引的大小。英语中停词(Stop word)如：“the”,“a”，“this”等。

###### 经过分词(Tokenizer) 后得到的结果称为词元(Token) 。

在我们的例子中，便得到以下词元(Token)：

“Students”，“allowed”，“go”，“their”，“friends”，“allowed”，“drink”，“beer”，“My”，“friend”，“Jerry”，“went”，“school”，“see”，“his”，“students”，“found”，“them”，“drunk”，“allowed”。

注：这里的代码细节是由IndexChain 完成的，IndexChain的起点是 DocFieldProcessor,它会分别调用 DocInverter类(倒排信息处理)和 TowStoredFieldsConsumer类(正向信息处理)。

##### 3. 将得到的词元(Token)传给语言处理组件(Linguistic Processor)。

语言处理组件(linguistic processor)主要是对得到的词元(Token)做一些同语言相关的处理。

对于英语，语言处理组件(Linguistic Processor) 一般做以下几点：

1. 变为小写(Lowercase) 。
2. 将单词缩减为词根形式，如“cars ”到“car ”等。这种操作称为：stemming 。
3. 将单词转变为词根形式，如“drove ”到“drive ”等。这种操作称为：lemmatization 。

Stemming 和 lemmatization的异同：



```csharp
相同之处：Stemming和lemmatization都要使词汇成为词根形式。
两者的方式不同：
    Stemming采用的是“缩减”的方式：“cars”到“car”，“driving”到“drive”。
    Lemmatization采用的是“转变”的方式：“drove”到“drove”，“driving”到“drive”。 
两者的算法不同：
    Stemming主要是采取某种固定的算法来做这种缩减，如去除“s”，去除“ing”加“e”，将“ational”变为“ate”，将“tional”变为“tion”。
    Lemmatization主要是采用保存某种字典的方式做这种转变。比如字典中有“driving”到“drive”，“drove”到“drive”，“am, is, are”到“be”的映射，做转变时，只要查字典就可以了。 
Stemming和lemmatization不是互斥关系，是有交集的，有的词利用这两种方式都能达到相同的转换。 
```

语言处理组件(linguistic processor)的结果称为词(Term) 。

在我们的例子中，经过语言处理，得到的词(Term)如下：

“student”，“allow”，“go”，“their”，“friend”，“allow”，“drink”，“beer”，“my”，“friend”，“jerry”，“go”，“school”，“see”，“his”，“student”，“find”，“them”，“drink”，“allow”。

也正是因为有语言处理的步骤，才能使搜索drove，而drive也能被搜索出来。

##### 4. 将得到的词(Term)传给索引组件(Indexer)。

利用得到的词(Term)创建一个字典。

| Term    | Document ID |
| ------- | ----------- |
| student | 1           |
| allow   | 1           |
| go      | 1           |
| their   | 1           |
| friend  | 1           |
| allow   | 1           |
| drink   | 1           |
| beer    | 1           |
| my      | 2           |
| friend  | 2           |
| jerry   | 2           |
| go      | 2           |
| school  | 2           |
| see     | 2           |
| his     | 2           |
| student | 2           |
| find    | 2           |
| them    | 2           |
| drink   | 2           |
| allow   | 2           |

然后将字典进行排序，相同的字典词进行合并，如go 存在两个。构建成链表

![img](https:////upload-images.jianshu.io/upload_images/14019352-afba0b34f1cabfdc.png?imageMogr2/auto-orient/strip|imageView2/2/w/489/format/webp)

lucene1.png

在词表中还保存了其他的定义信息：

Document Frequency 即文档频次，表示总共有多少文件包含此词(Term)。

Frequency 即词频率，表示此文件中包含了几个此词(Term)。

所以对词(Term) “allow”来讲，总共有两篇文档包含此词(Term)，从而词(Term)后面的文档链表总共有两项，第一项表示包含“allow”的第一篇文档，即1号文档，此文档中，“allow”出现了2次，第二项表示包含“allow”的第二个文档，是2号文档，此文档中，“allow”出。

最终倒排索引构成完毕，我们发现搜索“drive”，“driving”，“drove”，“driven”也能够被搜到。因为在我们的索引中，“driving”，“drove”，“driven”都会经过语言处理而变成“drive”，在搜索时，如果您输入“driving”，输入的查询语句同样经过我们这里的一到三步，从而变为查询“drive”，从而可以搜索到想要的文档。

其次是，Lucene 的 IndexWriter 是线程安全的,即它支持多线程索引。每
 个 IndexChain 都有一个独立的索引内存空间，每一个indexChain是互不干扰的各负责各自部分，索引效率很高。

###### 其次是，在写入内存阶段, Lucene 通过 IndexChain 把 document 分解并把相关信息存储到内存中,等到满足 flush 条件(内存容量或者文档个数积累到临界值),就通过 IndexChain 把内存中的数据flush 到硬盘。

##### 5.lucene索引数据存储本地

在我们进行 Field.Store.YES 或者 StoredField 则会将索引存储在本地。

Lucene的索引结构是有层次结构的，主要分以下几个层次：



```bash
索引(Index)：
    一个目录一个索引，在Lucene中一个索引是放在一个文件夹中的。
    如左图，同一文件夹中的所有的文件构成一个Lucene索引。
段(Segment)：
    一个索引可以包含多个段，段与段之间是独立的，添加新文档可以生成新的段，不同的段可以合并。 
    在建立索引的时候对性能影响最大的地方就是在将索引写入文件的时候, 所以在具体应用的时候就需要对此加以控制，段(Segment) 就是实现这种控制的。稍后详细描述段(Segment) 的控制策略。
    如上图，具有相同前缀文件的属同一个段，图中共两个段 "_0" 和 "_1"。
    segments.gen和segments_5是段的元数据文件，也即它们保存了段的属性信息。
文档(Document)：
    文档是我们建索引的基本单位，不同的文档是保存在不同的段中的，一个段可以包含多篇文档。
    新添加的文档是单独保存在一个新生成的段中，随着段的合并，不同的文档合并到同一个段中。
域(Field)：
    一篇文档包含不同类型的信息，可以分开索引，比如标题，时间，正文，作者等，都可以保存在不同的域里。
    不同域的索引方式可以不同。
词(Term)：
    词是索引的最小单位，是经过词法分析和语言处理后的字符串。
```

Lucene的索引结构中，即保存了正向信息，也保存了反向信息。

所谓正向信息：



```css
按层次保存了从索引，一直到词的包含关系：索引(Index) –> 段(segment) –> 文档(Document) –> 域(Field) –> 词(Term)
也即此索引包含了那些段，每个段包含了那些文档，每个文档包含了那些域，每个域包含了那些词。
既然是层次结构，则每个层次都保存了本层次的信息以及下一层次的元信息，也即属性信息，比如一本介绍中国地理的书，应该首先介绍中国地理的概况，以及中国包含多少个省，每个省介绍本省的基本概况及包含多少个市，每个市介绍本市的基本概况及包含多少个县，每个县具体介绍每个县的具体情况。
如上图，包含正向信息的文件有：
    segments_N保存了此索引包含多少个段，每个段包含多少篇文档。
    XXX.fnm保存了此段包含了多少个域，每个域的名称及索引方式。
    XXX.fdx，XXX.fdt保存了此段包含的所有文档，每篇文档包含了多少域，每个域保存了那些信息。
    XXX.tvx，XXX.tvd，XXX.tvf保存了此段包含多少文档，每篇文档包含了多少域，每个域包含了多少词，每个词的字符串，位置等信息。
```

所谓反向信息：



```css
保存了词典到倒排表的映射：词(Term) –> 文档(Document)
如上图，包含反向信息的文件有：
    XXX.tis，XXX.tii保存了词典(Term Dictionary)，也即此段包含的所有的词按字典顺序的排序。
    XXX.frq保存了倒排表，也即包含每个词的文档ID列表。
    XXX.prx保存了倒排表中每个词在包含此词的文档中的位置。
```

段(Segment) 的控制策略

在建立索引的时候对性能影响最大的地方就是在将索引写入文件的时候, 所以在具体应用的时候就需要对此加以控制:

Lucene默认情况是每加入10份文档(Document)就从内存往index文件写入并生成一个段(Segment) ,然后每10个段(Segment)就合并成一个段(Segment). 这些控制的变量如下：

| IndexWriter属性 | 默认值         | 描述                                      |
| --------------- | -------------- | ----------------------------------------- |
| MergeFactory    | 10             | 控制segment合并的频率和大小               |
| MaxMergeDocs    | Int32.MaxValue | 限制每个segment中包含的文档数             |
| MinMergeDocs    | 10             | 当内存中的文档达到多少的时候再写入segment |

MaxMergeDocs用于控制一个segment文件中最多包含的Document数.比如限制为100的话,即使当前有10个segment也不会合并,因为合并后的segment将包含1000个文档,超过了限制。

MinMergeDocs用于确定一个当内存中文档达到多少的时候才写入文件,该项对segment的数量和大小不会有什么影响,它仅仅影响内存的使用,进一步影响写索引的效率。

### 进行搜索的流程

根据我们创建的倒排索引，搜索不就是对构建链表的遍历，合并吗？

其实并没有那么简单，google搜索引擎，当输入关键词，返回大量结果，那么你最想要的是那个，总不能在一个个找吧。

##### 1. 用户输入查询语句

查询语句的语法根据全文检索系统的实现而不同。最基本的有比如：AND, OR, NOT等。

举个例子，用户输入语句：lucene AND learned NOT hadoop。

说明用户想找一个包含lucene和learned然而不包括hadoop的文档。

当然这些分析，是基于词法和语法分析

##### 2. 对查询语句进行词法分析，语法分析，及语言处理

1. 词法分析主要是识别单词或词组
2. 语法分析主要是根据查询语句的语法规则来形成一棵语法树

现查询语句不满足语法规则，则会报错。如lucene NOT AND learned，则会出错。

1. 语言处理同索引过程中的语言处理几乎相同。

如learned变成learn等。
 经过第二步，我们得到一棵经过语言处理的语法树。

##### 3. 搜索索引，得到符合语法树的文档

首先，在反向索引表中，分别找出包含lucene，learn，hadoop的文档链表。

其次，对包含lucene，learn的链表进行合并操作，得到既包含lucene又包含learn的文档链表。

然后，将此链表与hadoop的文档链表进行差操作，去除包含hadoop的文档，从而得到既包含lucene又包含learn而且不包含hadoop的文档链表。

##### 4. 根据得到的文档和查询语句的相关性，对结果进行排序

对于查询结果应该按照与查询语句的相关性进行排序，越相关者越靠前。

如何计算相关性呢？

不如我们把查询语句看作一片短小的文档，对文档与文档之间的相关性(relevance)进行打分(scoring)，分数高的相关性好，就应该排在前面。

那么又怎么对文档之间的关系进行打分呢？

首先，一个文档有很多词(Term)组成 ，如search, lucene, full-text, this, a, what等。

其次对于文档之间的关系，不同的Term重要性不同 ，比如对于本篇文档，search, Lucene, full-text就相对重要一些，this, a , what可能相对不重要一些。所以如果两篇文档都包含search, Lucene，fulltext，这两篇文档的相关性好一些，然而就算一篇文档包含this, a, what，另一篇文档不包含this, a, what，也不能影响两篇文档的相关性。

因而判断文档之间的关系，首先找出哪些词(Term)对文档之间的关系最重要，如search, Lucene, fulltext。然后判断这些词(Term)之间的关系。

找出词(Term) 对文档的重要性的过程称为计算词的权重(Term weight) 的过程。

计算词的权重(term weight)有两个参数，第一个是词(Term)，第二个是文档(Document)。

词的权重(Term weight)表示此词(Term)在此文档中的重要程度，越重要的词(Term)有越大的权重(Term weight)，因而在计算文档之间的相关性中将发挥更大的作用。

判断词(Term) 之间的关系从而得到文档相关性的过程应用一种叫做向量空间模型的算法(Vector Space Model) 。

###### 1. 计算权重(Term weight)的过程。

影响一个词(Term)在一篇文档中的重要性主要有两个因素：

Term Frequency (tf)：即此Term在此文档中出现了多少次。tf 越大说明越重要。

Document Frequency (df)：即有多少文档包含次Term。df 越大说明越不重要。

例如，然而在一篇英语文档中，the出现的次数更多，就说明越重要吗？the并不能代表这篇文档。



```bash
Wt,d = tft,d x log(n/dft)
```

wt,d 为词t在文档d中的重要程度。

###### 2. 判断Term之间的关系从而得到文档相关性的过程，也即向量空间模型的算法(VSM)

我们把文档看作一系列词(Term)，每一个词(Term)都有一个权重(Term weight)，不同的词(Term)根据自己在文档中的权重来影响文档相关性的打分计算。

于是我们把所有此文档中词(term)的权重(term weight) 看作一个向量。

Document = {term1, term2, …… ,term N}

Document Vector = {weight1, weight2, …… ,weight N}

同样我们把查询语句看作一个简单的文档，也用向量来表示。

Query = {term1, term 2, …… , term N}

Query Vector = {weight1, weight2, …… , weight N}

我们把所有搜索出的文档向量及查询向量放到一个N维空间中，每个词(term)是一维。

![img](https:////upload-images.jianshu.io/upload_images/14019352-a35d5e9e25d70bac.png?imageMogr2/auto-orient/strip|imageView2/2/w/432/format/webp)

VSM.png

我们认为两个向量之间的夹角越小，相关性越大。

所以我们计算夹角的余弦值作为相关性的打分，夹角越小，余弦值越大，打分越高，相关性越大。

###### VSM是基于词与词之间是相互独立的词袋模型，N代表的是整个文档集的词汇量，其中每一篇文档都是一个N维向量词汇表中的每一个词的 ID 对应着向量中的一个位置,词的权重为向量位置上的值。如果文档中没有该词,那么该位置上的值为 0。

举个例子，整个文档集有11个Term，共有三篇文档搜索出来。其中各自的权重(Term weight)，如下表格。

![img](https:////upload-images.jianshu.io/upload_images/14019352-f4ab5e82e30f1ca5.png?imageMogr2/auto-orient/strip|imageView2/2/w/546/format/webp)

VSM2.png

于是文档二相关性最高，先返回，其次是文档一，最后是文档三。

最后总结下lucene的查询结果流程：

![img](https:////upload-images.jianshu.io/upload_images/14019352-74f33412c26f2183.png?imageMogr2/auto-orient/strip|imageView2/2/w/525/format/webp)



作者：张晓天a
链接：https://www.jianshu.com/p/0cfe042412a2
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。