## 总体构架说明

这个爬虫是基于scrapy框架，但因为懒得建立一个复杂的scrapy project，所以只是写成一个个零散的文件。同时这个爬虫要依赖于redis数据库，其数据存储和代理池维护等均是利用redis完成的。

要应对各种网站的封IP操作，需要准备好代理IP。网上的免费代理IP基本不在考虑范围之内，可用性太低。而付费的代理IP主要分为两种：

- 隧道代理，主要优点是省事，不用自己维护代理池，一开始我就图省事直接用阿布云的隧道代理（我早几年写爬虫的时候经常用这个），后来感觉隧道代理调用频率的限制太大，爬得太慢，就没再用了
- 通过API提取代理IP和端口，再为每个请求设置不同代理，这个的缺点就是需要费点事，要自己维护代理池，但代理IP的使用没有限制，因此效率可以比隧道代理高得多，我后来时候的代理主要是这种方式，具体的厂商是[蜻蜓代理](https://proxy.horocn.com/) 

两种代理的价格都是在一天10+元左右，而一天的时间就足够把全部的数据都爬下来了。

我利用从蜻蜓代码上不断获取代理IP，并存到代理池中，每次请求时从代理池中随机取出一个代理IP，而当以下任一条件满足时将代理IP移除：存活时间超过100秒、IP被网站封了、请求出现其他异常（如超时）。

这里的所有爬虫都支持断点续爬。其中百度和必应的爬虫和总体方案介绍所说的一样，直接搜索应用名称得到搜索结果。但对于七麦网，我一开始并不是通过七麦网的sitemap得到应用链接，而是基于需要爬的应用包名放到七麦网下搜索，得到对应的应用链接，再去爬相应的链接。后来我才发现七麦网提供的sitemap包括了全部安卓应用的链接，因此又写了另一个爬虫，去爬取剩下的应用。因此这里关于七麦网会包含好几个爬虫，虽然不大优雅，但这个是我复赛期间真实使用的爬虫，之后有空可能会改一改把几个爬虫合一起。

## 爬虫代码说明

这里我将依次介绍本文件夹下全部代码文件。

首先是三个用来import的文件：

- qimai_encryptor.py 这里面包含了用于七麦网参数加密的类。加密方法改写/参考自https://github.com/longxiaofei/qimai-spider/blob/master/qimai.py 。
- spider_util.py 写了一个类，继承自scrapy.Spider，用于支持我更换代理。之后所有的爬虫都会继承自这个类。
- settings.py 主要是一些scrapy的配置选项。此外还包含redis数据库选项REDIS_SETTINGS和代理设置选项

接着是两个辅助爬虫运行的文件：

- import_rawdata.py 将需要爬取的应用名称和应用包名放到redis数据库中。如之前总体方案介绍所说，七麦网的数据被分为两次爬取，而第二次需要爬取的应用也依赖于第一次的结果，因此总共需要导入两次数据。第一次是在所有爬虫开始之前，第二次是在第一部分的七麦网数据爬取完成之后（原则上也可以跳过第一次爬取，直接将全部的应用放到第二次中，还能更快一些）。
- get_qingting_proxy.py 里面是一个死循环，每10秒从蜻蜓代理获取代理IP放入各个代理池，同时移除全部存活时间超过100秒的代理IP。这里之所以要多个代理池是为了同时运行多个网站的爬虫，而多个网站的代理应该要互不干扰（比如虽然IP被七麦网封了，但仍能正常爬取百度的数据）。这个文件需要一直运行着，以不断补充新的代理。

最后才是各个爬虫文件，这些爬虫使用 `scrapy runspider xxxx.py` 这样的命令运行。

- baidu_search.py 百度搜索的爬虫。
- bing_search.py 必应搜索的爬虫。必应其实我无法完全避开它的反爬机制，但同一个应用名称多爬个很多次总能爬到，并且是否设代理IP好像还不影响爬取的成功率。这个我没啥好的办法，只能多跑几次。
- qimai_apkname2appid.py 根据应用包名获取应用在七麦网的APPID
- qimai_appbaseinfo.py 根据上一步爬到的APPID爬取对应的应用数据
- qimai_addition_appbaseinfo.py 根据新补充进行来的APPID爬取对应的应用数据