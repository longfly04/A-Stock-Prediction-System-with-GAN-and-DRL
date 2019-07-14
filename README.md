A Stock Prediction System with GAN and DRL 
====
基于机器学习的股市预测系统
----

本系统参考了 https://github.com/borisbanushev/stockpredictionai 并尝试在A股市场中预测股价。

先写下实现的思路，以后慢慢改。

数据来源：https://tushare.pro/

For more information ： https://longfly04.github.io/A-Stock-Prediction-System-with-GAN-and-DRL/

# 1. 目录



# 2. 系统框架

<center><img src='doc\模型框架.jpg' width=1060></img></center>

1.系统需要获取股价信息，包括基本面、技术指标、财经新闻、上市公司财报和金融银行拆借利率等信息。训练模型使用90%训练集，9%交叉验证，1%测试。应用模型使用95%训练，5%验证，生成5%预测数据。


2.个股基本面、技术指标特征进行归一化。消息面特征经过BERT，取300维特征，归一化。


3.向量拼接后分为两流：

stream1 输入VAE，输出结果经PCA，降到80%，再进入到XGBoost，90%训练，9%验证，1%测试。

stream2 直接进到boost的数据，90%训练，9%验证，1%测试。

两个stream的output进入可视化，相当于对比特征工程对数据压缩后，数据意义表达层面有没有问题。

（但是为什么要这么做还是没搞懂。。。）

两个训练结果进本地保存，并附加时间戳。

4.stream1的输出进入GAN模型，Generator是LSTM，400维，Discriminator是CNN，padding是1维的么？主模型每50次迭代输入1次日志，每500次迭代为一次训练。

训练结果进可视化。

训练完成的模型参数进本地保存，并附加时间戳。


5.计算的D-loss和G-loss需要保存，predic-real-loss也需要保存，作为RL的输入。


6.将GAN的超参数集封装为一个对象HyperParameters，交给RL进行调优。


7.第五步的loss之和作为奖励，输入到Q值中，用PPO这种直接采取行动的方式调整HyperParameters。


每50次迭代记录一次日志，并同时将RL的参数保存到本地。

（不知道会调成什么样子。。。）


8.最后，让它跑起来。



# 3. 数据获取

通过tushare获取上证50成分股中部分股票每日成交信息和财务信息，包括：日线行情、资金流向、利润、资产负债情况、现金流量、财务报表等信息。数据的时间跨度为十年，从2008年1月1日至2018年12月31日。同时，获取了这个时间跨度内的宏观经济数据，包括利率、银行间同业拆借利率，民间借贷利率；市场交易数据，包括：沪深港通资金流向、北向资金流入、大宗交易、股票质押回购等信息。数据维度共576维。

StockData类：

1.getStockList():获取股票基本信息：股票列表、上市公司基本信息

2.getStockMarket():获取股票行情：代码、交易日、开盘价、收盘价、最高价、最低价、昨收价、涨跌幅、成交量、成交额

3.getDailyIndicator():获取每日指标：股票代码、交易日期、收盘价、换手率、量比、市盈率、市净率、市销率、总股本、流通股本、市值、流通市值。

4.getStockCurrent():个股资金流向:股票代码、交易日期、小单买入量（手）、小单买入金额（万元）、小单卖出量（手）、小单卖出金额（万元）、中单买入量（手）、中单买入金额（万元）、中单卖出量（手）、中单卖出金额（万元）、大单买入量（手）、大单买入金额（万元）、大单卖出量（手）、大单卖出金额（万元）、特大单买入量（手）、特大单买入金额（万元）、特大单卖出量（手）、特大单卖出金额（万元）、净流入量（手）、净流入额（万元）

StockFinance类：获取财务数据 API参考tushare

1.getProfit():获取上市公司财务利润

2.getBalanceSheet():获取上市公司资产负债表

3.getCashflow():获取上市公司现金流量表

4.getForecast():获取业绩预告数据

5.getExpress():获取上市公司业绩快报

6.getDividend():获取分红送股

7.getFinacialIndicator():获取上市公司财务指标数据 

8.getFinacialAudit():获取上市公司定期财务审计意见数据 

9.getFinacialMain():获得上市公司主营业务构成，分地区和产品两种方式

Market类:获取市场参考数据

1.getMoneyflow_HSGT():获取沪股通、深股通、港股通每日资金流向数据，每次最多返回300条记录，总量不限制 

2.getSecuritiesMarginTrading():获取融资融券每日交易汇总数据 

3.getPledgeState():获取股权质押统计数据 

4.getRepurchase():获取上市公司回购股票数据

5.getDesterilization():获取限售股解禁 

6.getBlockTrade():获取大宗交易 

7.getStockHolder():获取上市公司增减持数据，了解重要股东近期及历史上的股份增减变化 

Index类:指数

1.getIndexBasic():获取指数基础信息。

2.getIndexDaily():获取指数每日行情，还可以通过bar接口获取。  

3.getIndexWeight():获取各类指数成分和权重，月度数据  

4.getStockMarketIndex():上证综指指标数据

Futures类：期货

1.getFuturesDaily():获取期货日线

2.getFururesHolding():获取每日成交量

3.getFuturesWSR():获取仓单日报


Interest类：利率

1.getShibor():上海银行间同业拆放利率（Shanghai Interbank Offered Rate，简称Shibor），以位于上海的全国银行间同业拆借中心为技术平台计算、发布并命名，是由信用等级较高的银行组成报价团自主报出的人民币同业拆出利率计算确定的算术平均利率，是单利、无担保、批发性利率。

2.getShiborQuote():Shibor报价数据 

3.getShibor_LPR():贷款基础利率（Loan Prime Rate，简称LPR），是基于报价行自主报出的最优贷款利率计算并发布的贷款市场参考利率。目前，对社会公布1年期贷款基础利率。 

4.getLibor():Libor（London Interbank Offered Rate ），即伦敦同业拆借利率，是指伦敦的第一流银行之间短期资金借贷的利率，是国际金融市场中大多数浮动利率的基础利率。 

5.getHibor():HIBOR (Hongkong InterBank Offered Rate)，是香港银行同行业拆借利率。指香港货币市场上，银行与银行之间的一年期以下的短期资金借贷利率，从伦敦同业拆借利率（LIBOR）变化出来的。 

6.getWenZhouIndex():温州指数 ，即温州民间融资综合利率指数，该指数及时反映民间金融交易活跃度和交易价格。该指数样板数据主要采集于四个方面：由温州市设立的几百家企业测报点，把各自借入的民间资本利率通过各地方金融办不记名申报收集起来；对各小额贷款公司借出的利率进行加权平均；融资性担保公司如典当行在融资过程中的利率，由温州经信委和商务局负责测报；民间借贷服务中心的实时利率。(可选，2012年开放数据)

7.getGuangZhouIndex():广州民间借贷利率(可选)

News类：获取经济和金融新闻

get_news.py：获取新闻，获取主流新闻网站的快讯新闻数据 

1.getNews():新闻资讯数据从2018年10月7日开始有数据 之前没有数据

2.getCompanyPublic():上市公司公告 几乎都是空的

3.getCCTVNews():CCTV 接口限制每分钟100次



# 4. 数据特征工程

## 4.1. 概述

特征工程的好坏决定了模型的上限，调参只是逼近这个上限而已。

这个阶段，我计划将收集的数据以每日股价收盘价作为标签，获取技术指标，并作为数据特征。传统技术指标分析主要是针对股价，包括了7日均线，21日均线，MACD平滑异同移动均线，BOLLING布林线等，实际上，股价的波动在影响因素上，取决于非常多的客观交易行为，传统技术指标只是对股价的时间统计特征进行体现，并没有将市场的细节展现出来。本系统我将尝试增加自定义的技术指标，试图追踪交易量、大单买入卖出、换手率等指标的统计特征，并对以上技术指标进行频率分析，以期待获得更多市场行为信息。

基本面分析主要参考上市公司财务数据和公告，但是由于公告涉及到文本分析且数据量较少，这部分仅保留财务数据。另外，宏观经济数据也作为基本面分析的一部分。

特征组合，删除重复特征以及空值处理。由于特征存在大量空值，所以要分情况对空值进行处理，主要的方式是以空值前一日的value填充。即便如此，股价数据集仍然是一个庞大的稀疏矩阵。

在股价趋势预测任务中，暂时不适用于将每日财经新闻进行情感分析并作为事件特征，因为每日新闻数据量较大，对于个股的影响需要提取相关系数矩阵，计算量惊人。考虑到新闻的综合影响会直接体现在交易数据中，所以在技术指标中增加交易数据的统计特征，平衡特征的数量。



## 4.2. 基本面分析

概括起来，基本分析主要包括以下三个方面内容：

（1）宏观经济分析。研究经济政策（货币政策、财政政策、税收政策、产业政策等等）、经济指标（国内生产总值、失业率、通胀率、利率、汇率等等）对股票市场的影响。

（2）行业分析。分析产业前景、区域经济发展对上市公司的影响。

（3）公司分析。具体分析上市公司行业地位、市场前景、财务状况。

这里，基本面分析参考的指标主要包括：


|外汇|期权合约信息|期货|市场参考数据|宏观经济
|---|----|-----|---------|---|
|外汇基础信息（海外）|期权合约信息|期货合约信息|沪深港通资金流向|利率数据
|外汇日线行情|期权日线行情|期货合约信息 |沪深股通十大成交股|shibor利率
||期货交易日历 |港股通十大成交股||shibor报价数据
||期货日线行情 |融资融券交易汇总||LPR贷款基础利率
||每日持仓排名 |融资融券交易明细||Libor利率
||仓单日报 |前十大股东||Hibor利率
||每日结算参数 |前十大流通股东||温州民间借贷利率
||南华期货指数行情 |龙虎榜每日明细||广州民间借贷利率
||期权|龙虎榜机构交易明细
|||股权质押统计数据
|||股权质押明细数据
|||股票回购
|||概念股分类表
|||概念股明细列表
|||限售股解禁
|||大宗交易
|||股票开户数据
|||股票开户数据（旧）
|||股东人数
|||股东增减持


## 4.3. 技术指标分析

技术指标分析是主要的分析手段，技术指标主要包括：

| 指标 | 格式 | 含义 |
| --- | --- | --- |
| open | float | 开盘价 |
|high|float|最高价
|low|float|最低价
|close|float|收盘价
|pre_close|float|昨收价
|change|float|涨跌额
|pct_chg|float|涨跌幅 （未复权，如果是复权请用 通用行情接口 ）
|vol|float|成交量 （手）
|amount|float|成交额 （千元）
|close|float|当日收盘价
|turnover_rate|float|换手率（%）
|turnover_rate_f|float|换手率（自由流通股）
|volume_ratio|float|量比
|pe|float|市盈率（总市值/净利润）
|pe_ttm|float|市盈率（TTM）
|pb|float|市净率（总市值/净资产）
|ps|float|市销率
|ps_ttm|float|市销率（TTM）
|total_share|float|总股本 （万股）
|float_share|float|流通股本 （万股）
|free_share|float|自由流通股本 （万）
|total_mv|float|总市值 （万元）
|circ_mv|float|流通市值（万元）
|buy_sm_vol|int|小单买入量（手）
|buy_sm_amount|float|小单买入金额（万元）
|sell_sm_vol|int|小单卖出量（手）
|sell_sm_amount|float|小单卖出金额（万元）
|buy_md_vol|int|中单买入量（手）
|buy_md_amount|float|中单买入金额（万元）
|sell_md_vol|int|中单卖出量（手）
|sell_md_amount|float|中单卖出金额（万元）
|buy_lg_vol|int|大单买入量（手）
|buy_lg_amount|float|大单买入金额（万元）
|sell_lg_vol|int|大单卖出量（手）
|sell_lg_amount|float|大单卖出金额（万元）
|buy_elg_vol|int|特大单买入量（手）
|buy_elg_amount|float|特大单买入金额（万元）
|sell_elg_vol|int|特大单卖出量（手）
|sell_elg_amount|float|特大单卖出金额（万元）
|net_mf_vol|int|净流入量（手）
|net_mf_amount|float|净流入额（万元）


### 平滑异同平均线指标——MACD

MACD指标又叫指数平滑异同移动平均线，是由查拉尔·阿佩尔（Gerald Apple）所创造的,是一种研判股票买卖时机、跟踪股价运行趋势的技术分析工具。

#### MACD指标的原理

MACD指标是根据均线的构造原理，对股票价格的收盘价进行平滑处理，求出算术平均值以后再进行计算，是一种趋向类指标。

MACD 指标是运用快速（短期）和慢速（长期）移动平均线及其聚合与分离的征兆，加以双重平滑运算。而根据移动平均线原理发展出来的MACD，一则去除了移动平均线频繁发出假信号的缺陷，二则保留了移动平均线的效果，因此，MACD指标具有均线趋势性、稳重性、安定性等特点，是用来研判买卖股票的时机，预测股票价格涨跌的技术分析指标。

MACD 指标主要是通过EMA、DIF和DEA（或叫MACD、DEM）这三值之间关系的研判，DIF和DEA连接起来的移动平均线的研判以及DIF减去DEM值而绘制成的柱状图（BAR）的研判等来分析判断行情，预测股价中短期趋势的主要的股市技术分析指标。其中，DIF是核心，DEA是辅助。DIF是快速平滑移动平均线（EMA1）和慢速平滑移动平均线（EMA2）的差。BAR柱状图在股市技术软件上是用红柱和绿柱的收缩来研判行情。

#### MACD指标的计算方法

MACD在应用上，首先计算出快速移动平均线（即EMA1）和慢速移动平均线（即EMA2），以此两个数值，来作为测量两者（快慢速线）间的离差值（DIF）的依据，然后再求DIF的N周期的平滑移动平均线DEA（也叫MACD、DEM）线。

以EMA1的参数为12日，EMA2的参数为26日，DIF的参数为9日为例来看看MACD的计算过程

1、计算移动平均值（EMA）

12日EMA的算式为

EMA（12）=前一日EMA（12）×11/13＋今日收盘价×2/13

26日EMA的算式为

EMA（26）=前一日EMA（26）×25/27＋今日收盘价×2/27

2、计算离差值（DIF）

DIF=今日EMA（12）－今日EMA（26）

3、计算DIF的9日EMA

根据离差值计算其9日的EMA，即离差平均值，是所求的MACD值。为了不与指标原名相混淆，此值又名DEA或DEM。

今日DEA（MACD）=前一日DEA×8/10＋今日DIF×2/10

计算出的DIF和DEA的数值均为正值或负值。

理论上，在持续的涨势中，12 日EMA线在26日 EMA线之上，其间的正离差值（+DIF）会越来越大；反之，在跌势中离差值可能变为负数（—DIF），也会越来越大，而在行情开始好转时，正负离差值将会缩小。指标MACD正是利用正负的离差值（±DIF）与离差值的N日平均线（N日EMA）的交叉信号作为买卖信号的依据，即再度以快慢速移动线的交叉原理来分析买卖信号。另外，MACD指标在股市软件上还有个辅助指标——BAR柱状线，其公式为：BAR=2×(DIF－DEA)，我们还是可以利用BAR 柱状线的收缩来决定买卖时机。

离差值DIF 和离差平均值DEA是研判MACD的主要工具。其计算方法比较烦琐，由于目前这些计算值都会在股市分析软件上由计算机自动完成，因此，投资者只要了解其运算过程即可，而更重要的是掌握它的研判功能。另外，和其他指标的计算一样，由于选用的计算周期的不同，MACD指标也包括日MACD指标、周MACD指标、月MACD指标年MACD指标以及分钟MACD指标等各种类型。经常被用于股市研判的是日MACD指标和周MACD指标。虽然它们的计算时的取值有所不同，但基本的计算方法一样。

在实践中，将各点的 DIF和DEA（MACD）连接起来就会形成在零轴上下移动的两条快速（短期）和慢速（长期）线，此即为MACD图。


### 随机指标——KDJ

   KDJ指标又叫随机指标，是由乔治·蓝恩博士（George Lane）最早提出的，是一种相当新颖、实用的技术分析指标，它起先用于期货市场的分析，后被广泛用于股市的中短期趋势分析，是期货和股票市场上最常用的技术分析工具。


#### KDJ指标的原理

随机指标KDJ 一般是根据统计学的原理，通过一个特定的周期（常为9日、9周等）内出现过的最高价、最低价及最后一个计算周期的收盘价及这三者之间的比例关系，来计算最后一个计算周期的未成熟随机值RSV，然后根据平滑移动平均线的方法来计算K值、D值与J值，并绘成曲线图来研判股票走势。

随机指标KDJ 是以最高价、最低价及收盘价为基本数据进行计算，得出的K值、D值和J值分别在指标的坐标上形成的一个点，连接无数个这样的点位，就形成一个完整的、能反映价格波动趋势的KDJ指标。它主要是利用价格波动的真实波幅来反映价格走势的强弱和超买超卖现象，在价格尚未上升或下降之前发出买卖信号的一种技术工具。它在设计过程中主要是研究最高价、最低价和收盘价之间的关系，同时也融合了动量观念、强弱指标和移动平均线的一些优点，因此，能够比较迅速、快捷、直观地研判行情。

随机指标KDJ 最早是以KD指标的形式出现，而KD指标是在威廉指标的基础上发展起来的。不过威廉指标只判断股票的超买超卖的现象，在KDJ指标中则融合了移动平均线速度上的观念，形成比较准确的买卖信号依据。在实践中，K线与D线配合J线组成KDJ指标来使用。由于KDJ线本质上是一个随机波动的观念，故其对于掌握中短期行情走势比较准确。

 

#### KDJ指标的计算方法

指标KDJ的计算比较复杂，首先要计算周期（n日、n周等）的RSV值，即未成熟随机指标值，然后再计算K值、D值、J值等。以日KDJ数值的计算为例，其计算公式为

n日RSV=（Cn－Ln）÷（Hn－Ln）×100

式中，Cn为第n日收盘价；Ln为n日内的最低价；Hn为n日内的最高价。RSV值始终在1—100间波动。

其次，计算K值与D值：

当日K值=2/3×前一日K值＋1/3×当日RSV

当日D值=2/3×前一日D值＋1/3×当日K值

若无前一日K 值与D值，则可分别用50来代替。

以9日为周期的KD线为例。首先须计算出最近9日的RSV值，即未成熟随机值，计算公式为

9日RSV=（C－L9）÷（H9－L9）×100

式中，C为第9日的收盘价；L9为9日内的最低价；H9为9日内的最高价。

K值=2/3×前一日 K值＋1/3×当日RSV

D值=2/3×前一日K值＋1/3×当日RSV

若无前一日K值与D值，则可以分别用50代替。

需要说明的是，式中的平滑因子1/3和2/3是可以人为选定的,不过目前已经约定俗成，固定为1/3和2/3。在大多数股市分析软件中，平滑因子已经被设定为1/3和2/3，不需要作改动。另外，一般在介绍KD时，往往还附带一个J指标。

J指标的计算公式为：

J=3D—2K

实际上，J的实质是反映K值和D值的乖离程度，从而领先KD值找出头部或底部。J值范围可超过100。

J 指标是个辅助指标，最早的KDJ指标只有两条线，即K线和D线，指标也被称为KD指标，随着股市分析技术的发展，KD指标逐渐演变成KDJ指标，从而提高了KDJ指标分析行情的能力。另外，在一些股市重要的分析软件上，KDJ指标的K、D、J参数已经被简化成仅仅一个，即周期数（如日、周、月等），而且，随着股市软件分析技术的发展，投资者只需掌握KDJ形成的基本原理和计算方法，无须去计算K、D、J的值，更为重要的是利用KDJ指标去分析、研判股票行情。

和其他指标的计算一样，由于选用的计算周期的不同，KDJ指标也包括日KDJ指标、周KDJ指标、月KDJ指标年KDJ指标以及分钟KDJ指标等各种类型。经常被用于股市研判的是日KDJ指标和周KDJ指标。虽然它们的计算时的取值有所不同，但基本的计算方法一样。




### 威廉指标——WR

威廉指标WR又叫威廉超买超卖指标，简称威廉指标，是由拉瑞·威廉（Larry William）在1973年发明的，是目前股市技术分析中比较常用的短期研判指标。



#### 威廉指标的原理

威廉指标主要是通过分析一段时间内股价最高价、最低价和收盘价之间的关系，来判断股市的超买超卖现象，预测股价中短期的走势。它主要是利用振荡点来反映市场的超买超卖行为，分析多空双方力量的对比，从而提出有效的信号来研判市场中短期行为的走势。

威廉指标是属于研究股价波幅的技术分析指标，在公式设计上和随机指标的原理比较相似，两者都是从研究股价波幅出发，通过分析一段时间的股票的最高价、最低价和收盘价等这三者关系，来反映市场的买卖气势的强弱，借以考察阶段性市场气氛、判断价格和理性投资价值标准相背离的程度。

和股市其他技术分析指标一样，威廉指标可以运用于行情的各个周期的研判，大体而言，威廉指标可分为日、周、月、年、5 分钟、15分钟、30分钟、60分钟等各种周期。虽然各周期的威廉指标的研判有所区别，但基本原理相差不多。如日威廉指标是表示当天的收盘价在过去的一段日子里的全部价格范围内所处的相对位置，把这些日子里的最高价减去当日收市价，再将其差价除以这段日子的全部价格范围就得出当日的威廉指标。

威廉指标在计算时首先要决定计算参数，此数可以采取一个买卖循环周期的半数。以日为买卖的周期为例，通常所选用的买卖循环周期为8日、14日、28日或56日等，扣除周六和周日，实际交易日为6日、10日、20日或40日等，取其一半则为3日、5日、10日或20日等。

#### W%R指标的计算方法

W%R指标的计算主要是利用分析周期内的最高价、最低价及周期结束的收盘价等三者之间的关系展开的。以日威廉指标为例，其计算公式为：

W%R=（Hn—C）÷（Hn—Ln）×100

其中：C为计算日的收盘价，Ln为N周期内的最低价，Hn为N周期内的最高价，公式中的N为选定的计算时间参数，一般为4或14。

以计算周期为14日为例，其计算过程如下：

W%R（14日）=（H14—C）÷（H14—L14）×100

其中，C为第14天的收盘价，H14为14日内的最高价，L14为14日内的最低价。

威廉指标是表示当天的收盘价在过去一段时间里的全部价格范围内所处的相对位置，因此，计算出的W%R值位于0——100之间。越接近0值，表明目前的价位越接近过去14日内的最低价；越接近100值，表明目前的价位越接近过去14日内的最高价，从这点出发，对于威廉指标的研判可能比较更容易理解。

由于计算方法的不同，威廉指标的刻度在有些书中与随机指标W%R 和相对强弱指标RSI一样，顺序是一样的，即上界为100、下界为0。而在我国沪深股市通用的股市分析软件（钱龙、分析家等分析软件系统）中，W%R的刻度与RSI的刻度相反。为方便投资者，这里介绍的W%R的刻度与钱龙（分析家）软件相一致，即上界为0、下界为100。

另外，和其他指标的计算一样，由于选用的计算周期的不同，W%R指标也包括日W%R指标、周W%R指标、月W%R指标和年W%R指标以及分钟W%R指标等各种类型。经常被用于股市研判的是日W%R指标和周W%R指标。虽然它们的计算时的取值有所不同，但基本的计算方法一样。


### 相对强弱指标——RSI

相对强弱指标RSI又叫力度指标，其英文全称为“Relative Strength Index”，由威尔斯·魏尔德﹝Welles Wilder﹞所创造的，是目前股市技术分析中比较常用的中短线指标。


#### RSI指标的原理

相对强弱指标RSI是根据股票市场上供求关系平衡的原理，通过比较一段时期内单个股票价格的涨跌的幅度或整个市场的指数的涨跌的大小来分析判断市场上多空双方买卖力量的强弱程度，从而判断未来市场走势的一种技术指标。

从它构造的原理来看，与MACD、 TRIX等趋向类指标相同的是，RSI指标是对单个股票或整个市场指数的基本变化趋势作出分析，而与MACD、TRIX等不同的是，RSI指标是先求出单个股票若干时刻的收盘价或整个指数若干时刻收盘指数的强弱，而不是直接对股票的收盘价或股票市场指数进行平滑处理。

相对强弱指标RSI是一定时期内市场的涨幅与涨幅加上跌幅的比值。它是买卖力量在数量上和图形上的体现，投资者可根据其所反映的行情变动情况及轨迹来预测未来股价走势。在实践中，人们通常将其与移动平均线相配合使用，借以提高行情预测的准确性。

#### RSI指标的计算方法

相对强弱指标RSI的计算公式有两种

其一：

假设A为N日内收盘价的正数之和，B为N日内收盘价的负数之和乘以（—1）

这样，A和B均为正，将A、B代入RSI计算公式，则

RSI（N）=A÷（A＋B）×100

其二：

RS（相对强度）=N日内收盘价涨数和之均值÷N日内收盘价跌数和之均值

RSI（相对强弱指标）=100－100÷（1+RS）

这两个公式虽然有些不同，但计算的结果一样。

以14日RSI指标为例，从当起算，倒推包括当日在内的15个收盘价，以每一日的收盘价减去上一日的收盘价，得到14个数值，这些数值有正有负。这样，RSI指标的计算公式具体如下：

A=14个数字中正数之和

B=14个数字中负数之和乘以（—1）

RSI（14）=A÷（A＋B）×100

式中：A为14日中股价向上波动的大小

B为14日中股价向下波动的大小

A＋B为股价总的波动大小

RSI 的计算公式实际上就是反映了某一阶段价格上涨所产生的波动占总的波动的百分比率，百分比越大，强势越明显；百分比越小，弱势越明显。RSI的取值介于0— 100之间。在计算出某一日的RSI值以后，可采用平滑运算法计算以后的RSI值，根据RSI值在坐标图上连成的曲线，即为RSI线。

以日为计算周期为例，计算RSI值一般是以5日、10日、14日为一周期。另外也有以6日、12日、24日为计算周期。一般而言，若采用的周期的日数短，RSI指标反应可能比较敏感；日数较长，可能反应迟钝。目前，沪深股市中RSI所选用的基准周期为6日和12日。

和其他指标的计算一样，由于选用的计算周期的不同，RSI 指标也包括日RSI指标、周RSI指标、月RSI指标年RSI指标以及分钟RSI指标等各种类型。经常被用于股市研判的是日RSI指标和周RSI指标。虽然它们的计算时的取值有所不同，但基本的计算方法一样。另外，随着股市软件分析技术的发展，投资者只需掌握RSI形成的基本原理和计算方法，无须去计算指标的数值，更为重要的是利用RSI指标去分析、研判股票行情。

 ### 中间意愿指标——CR

CR指标又叫中间意愿指标，它和AR、BR指标又很多相似之处，但更有自己独特的研判功能，是分析股市多空双方力量对比、把握买卖股票时机的一种中长期技术分析工具。


#### CR指标的原理

CR指标同AR、BR指标有很多相似的地方，如计算公式和研判法则等，但它与AR、BR指标最大不同的地方在于理论的出发点有不同之处。CR指标的理论出发点是：中间价是股市最有代表性的价格。

为避免AR、 BR指标的不足，在选择计算的均衡价位时，CR指标采用的是上一计算周期的中间价。理论上，比中间价高的价位其能量为“强”，比中间价低的价位其能量为“ 弱”。CR指标以上一个计算周期（如N日）的中间价比较当前周期（如日）的最高价、最低价，计算出一段时期内股价的“强弱”，从而在分析一些股价的异常波动行情时，有其独到的功能。

另外，CR指标不但能够测量人气的热度、价格动量的潜能，而且能够显示出股价的压力带和支撑带，为分析预测股价未来的变化趋势，判断买卖股票的时机提供重要的参考。

#### CR指标的计算方法

由于选用的计算周期不同，CR指标也包括日CR指标、周CR指标、月CR指标、年CR指标以及分钟CR指标等很多种类型。经常被用于股市研判的是日CR指标和周CR指标。虽然它们计算时取值有所不同，但基本的计算方法一样。

以日CR指标为例，其计算公式为：

CR（N日）=P1÷P2×100

式中，P1=∑（H－YM），表示N日以来多方力量的总和

P2=∑（YM－L），表示N日以来空方力量的总和

H表示今日的最高价，L表示今日的最低价

YM表示昨日（上一个交易日）的中间价

CR计算公式中的中间价其实也是一个指标，它是通过对昨日（YM）交易的最高价、最低价、开盘家和收盘价进行加权平均而得到的，其每个价格的权重可以人为地选定。目前比较常用地中间价计算方法有四种：

1、M=（2C+H+L）÷4

2、M=（C+H+L+O）÷4

3、M=（C+H+L）÷3

4、M=（H+L）÷2

式中，C为收盘价，H为最高价，L为最低价，O为开盘价

从四种中间价的计算方法来看，对四种价格的重视程度是不一样的，三种都是选用了收盘价，可见，收盘价在技术分析中的重要性。

和其他技术指标一样，在实战中，投资者不需要进行CR指标的计算，主要是了解CR的计算方法，以便更加深入地掌握CR指标的实质，为运用指标打下基础。

## 相关技术指标图表

分析股价数据特征，主要对开盘价、收盘价、最高价、最低价、涨跌幅、成交额、换手率、量比、市盈率、市净率、市销率、流通股本、总股本、小、中、大、特大单买入卖出额等特征进行分析，并且对以上特征应用了指数移动平均、差分自相关等数据处理手段。

* 股价、成交量特征
<center><img src='project\feature_engineering\30_price_amount.png' width=1060></img></center>


* MACD 异步移动均线
<center><img src='project\feature_engineering\31_MACD.png' width=1060></img></center>


* 布林线和随机指标KDJ
<center><img src='project\feature_engineering\32_boll_kdj.png' width=1060></img></center>


* 开盘价和相关强弱指数RSI
<center><img src='project\feature_engineering\33_open_rsi.png' width=1060></img></center>


* 威廉指标WR和中间意愿指标CR
<center><img src='project\feature_engineering\34_cr_ma.png' width=1060></img></center>


* 短线超买超卖指标CCI、成交量变异率VR和波动幅度TR
<center><img src='project\feature_engineering\35_cci_tr_vr.png' width=1060></img></center>


* 动向指标DMI
<center><img src='project\feature_engineering\36_close_DMI.png' width=1060></img></center>


* 傅里叶变换和逆变换
<center><img src='project\feature_engineering\40_Fourier_transforms.png' width=1060></img></center>

* 股价频谱分析

<center><img src='project\feature_engineering\41_Fourier_components.png' width=1060></img></center>


* 收盘价逐日相关性 price correlation
<center><img src='project\feature_engineering\50_Close_price_correlations.png' width=1060></img></center>


## 4.4. 特征重要性分析


* 特征重要性分析 xgboost回归
<center><img src='project\feature_engineering\61_Feature_importance.png' width=1060></img></center>

* 特征重要性误差分析
<center><img src='project\feature_engineering\60_Training_Vs_Validation_Error.png' width=1060></img></center>


我们可以看到，在去除强相关性的特征后，验证集的误差随着训练次数的迭代有所增加，表明模型出现了过拟合。

由于在股价数据中，包含有大量与股价特征强相关的数据，比如开盘价、股价动量（即为前一日的收盘价），股价七日均值等，在使用xgboost进行回归时，这些特征会因强相关特性而独占重要性指标，比如股价动量重要性可达到0.8以上，这样会削弱其他指标的重要性度量。

所以在实际操作中，为了将强相关因素排除，更多的利用其他指标构建股价的特征关系，我们删除了部分特征，并得到了以上特征重要性直方图。以上最重要的特征其重要性也在0.2以下，我们尽可能的实现特征与股价强相关的剥离，并试图发现股价与其他特征的关联。

在训练时，我将训练数据和验证数据比例设置为0.9，但是当我将训练/验证比例设置为0.98时，模型的特征重要性发生了比较明显的变化。在接下来对vae进行训练时，也出现了由于训练/验证比例的变化而造成的结果漂移。

**训练模型的数据/验证比例变化会明显的影响结果，这让我很难理解。有可能是模型并没有发现数据的时间序列特性，也可能是特征本身并没有对股价有明显的影响。**




# 5. 训练模型

我始终反复思考建立股价预测模型的本源是什么：我们想预测一个公司股票的走势，根本目的在于根据交易者以及上市公司本身行为去判断这些行为对股价的影响，以便于提前预测股市未来波动并作出及时的投资策略。一般情况下，投资者会根据股价以及外界信息对投资行为进行执行，上市公司根据法律法规和自身经营情况，对外披露自己的财务、股东、以及盈利情况。这些都是会影响到股价波动的因素。

我们想通过一个模型，在长期对某一只股票的建模基础上，敏感应对以上提到的市场变化，并提前将股价波动预测出来。

我计划首先建立以日交易数据为基础训练集的股价中长期预测模型，这个模型依赖上市公司财务数据、宏观经历数据、以及市场参考数据等，建模更加宏观，忽略每日的波动情况。

在以上基础上，建立以每分钟或者每10分钟交易数据为基础的短期股价预测模型，这个模型更依赖市场随时的变化，以及外界的新闻、突发消息等。期待这个模型能够对当日出现的股价波动和走势进行预判和模拟。

## 模型训练思路

首先，我们需要一些基准模型对股价预测的结果进行评估，除了使用真实的测试数据之外，我们尝试使用几种比较常用的模型对股价进行预测。在特征工程阶段，我们使用了xgboost模型对参数的重要性进行了计算，并找出了25个与股价相关性很高的特征。在模型训练阶段，我们想通过n天的股价训练数据，预测出第n+1天的股价。如果模型的容量足够大，我们甚至可以从n天的股价中预测出n+3、n+7、n+21天的股价波动走势，这样就实现了我们这个项目的最终目的。

我们用监督模型对n+1天的股价进行预测。

1.支持向量机回归模型（SVC）

2.Xgboost集成树模型

3.LSTM网络模型

以上三种模型都是有监督的模型。

其次，我们再尝试使用生成模型对股价进行预测。

1.VAE变分自编码器

2.GAN生成对抗网络

以上两种生成模型本质上都是从高斯分布中采样，并通过模型生成数据样本。如果用贝叶斯理论可以证明其本质上都是一致的，模型都在尝试对真实分布和生成分布的KL散度进行优化和学习。

最后我们想通过DRL对生成模型的参数进行学习，不断尝试优化模型。

## Scikit Learn 库

TODO

## XGBOOST库

TODO

## Keras框架
Keras 是一个用 Python 编写的高级神经网络 API，它能够以 TensorFlow, CNTK, 或者 Theano 作为后端运行。Keras 的开发重点是支持快速的实验。能够以最小的时延把你的想法转换为实验结果，是做好研究的关键。

由于本项目基于快速成型的模型搭建原则，所以应用Keras这种高层API更加方便。

>允许简单而快速的原型设计（由于用户友好，高度模块化，可扩展性）。
同时支持卷积神经网络和循环神经网络，以及两者的组合。
在 CPU 和 GPU 上无缝运行。

**Keras的特征** 

>用户友好。 Keras 是为人类而不是为机器设计的 API。它把用户体验放在首要和中心位置。Keras 遵循减少认知困难的最佳实践：它提供一致且简单的 API，将常见用例所需的用户操作数量降至最低，并且在用户错误时提供清晰和可操作的反馈。

>模块化。 模型被理解为由独立的、完全可配置的模块构成的序列或图。这些模块可以以尽可能少的限制组装在一起。特别是神经网络层、损失函数、优化器、初始化方法、激活函数、正则化方法，它们都是可以结合起来构建新模型的模块。

>易扩展性。 新的模块是很容易添加的（作为新的类和函数），现有的模块已经提供了充足的示例。由于能够轻松地创建可以提高表现力的新模块，Keras 更加适合高级研究。

>基于 Python 实现。 Keras 没有特定格式的单独配置文件。模型定义在 Python 代码中，这些代码紧凑，易于调试，并且易于扩展。


## 使用LSTM网络生成股价数据

LSTM模型对于时间序列数据的训练来说，具备先天的优势。在本项目中，我借鉴了大神的模型：https://github.com/jaungiers/LSTM-Neural-Network-for-Time-Series-Prediction

模型的预测结果如下：

<center><img src='project\lstm\lstm_prediction_point_by_point.png' width=1060></img></center>

上图是对股价逐个点进行预测，虽然看上去与实际价格吻合的很好，但是这个结果具有一定的欺骗性，因为模型只需要预测一个点，那么，在下一次预测中，这个预测点实际上就被真实数据取代了，那么模型只需要给出实际误差不是很大的结果就可以莫混过关。

<center><img src='project\lstm\lstm_prediction_full_window.png' width=1060></img></center>

上图是对整个测试序列进行预测，这个结果明显看出，模型并没有学到什么，对于股价波动没有丝毫的敏感性。

<center><img src='project\lstm\lstm_prediction_multiple.png' width=1060></img></center>

上图是以序列为单位进行预测，可以看出虽然在序列的开始节点是比较准确的，但是模型依然对波动不敏感，而且每个序列的第一个点的预测结果跟上面逐个点预测的原因是一样的，再继续使用模型预测波动趋势就会发现模型并没有对股价的涨跌做出反应，这说明模型学习的效果并不理想。

项目使用的是三层叠加的LSTM网络，每层网络100维，最后使用一个全连接层对股价预测输出结果。

我仔细研究了模型思路，发现了一些可以改进的地方：

**1.原来的项目中只是仅仅使用了股价的收盘价和成交量为训练数据，只有二维，这显然无法表达股价波动的多种影响因素。我使用了Tushare中获取的股价数据，进行特征整合和去重、特征工程之后，得到了488维的股价数据，也就是说，每一日的股价，伴随有488个不同层面的特征数据对其进行描述。我认为，只有当我们获得了全部数据，才能够真正的发现数据背后的规律。**

**2.模型使用叠加的LSTM网络可以很好的发现序列数据中的深层结构，但是网络应该将这个序列的结构信息也传递给全连接层，这样才能真正的捕捉时间序列信息，但是原模型并没有将序列整个输出，而仅仅将最后一个 LSTM cell的hidden state输出给全连接层，这样会造成模型学习的知识丢失。**

**3.由于LSTM网络传递的是三维的数据，在我的实验数据中，数据维度是（2364，50，488），也就是有2364个样本，每个样本有488个特征，以50个时间步为一个训练数据进行训练。这样网络最后输出的结果也是三维的，所以我在Dense层之前添加了一个Flatten层，用来展平三维数据到二维，然后增加了一个Dropout层避免过拟合，这样再输出到Dense层之后，全连接的网络会学习到整个序列在时间上的结构信息，更加有利于准确预测结果。**

综上，我修改了原项目中的模型结构和数据结构，主要是增加了新的网络层，并对数据维度进行了扩展，我使用了2008年到2018年十年的股价日收盘价数据进行训练，模型参数总量为169万，预测结果有了很大的提升。

<center><img src='project\lstm\predict_seq_to_seq.png' width=1060></img></center>

我们可以看到，虽然在股价的绝对值上，预测结果有较大的偏差，但是股价的走势以及每日走势细节。尤其是对股价的剧烈波动都基本能够准确捕捉，可以说模型已经基本学习到了股价波动的趋势。

相比于原模型来说，在以序列为单位进行预测这项任务上，新的模型更能学习到股价趋势。

这个模型只是我实现模型的第一个版本，但是基本可以证明我的思路是正确的，这个版本有一个明显的问题：

**我对预测数据的切分，仅仅是按照序列长度进行了等分，相当于模型只是对结果进行了int(test_len/seq_len)个预测，这样的预测很容易受到噪声影响。**

所以我又对预测结果进行了改进，借鉴了滑动窗口的思路，将预测结果和滑动窗口结合，对同一个点的结果进行平均，消除噪声的影响。

然后，得到的结果，有那么一点萌 （—.—）

<center><img src='project\lstm\predict_seq_to_seq_avg.png' width=1060></img></center>

看来时间窗口平滑真的很平滑。

但是不得不说，股价的趋势仍然可以很好的预测出来。如果以这样的预测结果进行量化交易，可以通过对预测值求导的方式，估算出涨跌趋势，从而指导交易。

另外，对于这个模型，还有一个改进的方法。

之前对于训练数据和测试数据，都是在时间窗口内进行的标准化。标准化会对数据的进行重新分布，将数据集的均值和方差都设为标准正态分布。那么问题来了，训练集和测试集实际上每个样本数据都是基于不同的均值和方差的，但是标准化之后它们就具有相同的均值和方差。这样会损失一些信息。如果我们在训练之前，将整个训练集和测试集统一进行一次标准化，使得整体的均值方差保持一致，这样，基于时间的均值和方差序列信息就会更好的保留，并且可能会在一定程度上提高预测的精确度。以上，感觉表述不清，我们用实验来验证一下。

<center><img src='project\lstm\predict_seq_to_seq_general_normed.png' width=1060></img></center>

上图是全局标准化之后的结果，我们可以看到，在绝对值上面，相对于窗口标准化的预测结果，股价更加接近于真实数据。在这里我发现应该是窗口预测的原因，两个窗口衔接的位置容易出现比较大的误差。这在实际当中是不可能出现的。于是我又有了一个对窗口衔接处做修正的方案：

<center><img src='project\lstm\predict_seq_to_seq_general_normed_rectified.png' width=1060></img></center>

上图对窗口的衔接处做了处理，如果前一个窗口的最后一个数据和下一个窗口的第一个数据之间的偏差超过了涨跌停板的限制时，我们人为的将后面一个窗口的数据都修正到涨跌停板的范围内，这样，既不会影响在窗口内模型预测的结果，也使得结果数据更加自然、真实。

<center><img src='project\lstm\predict_seq_to_seq_avg_general_normed.png' width=1060></img></center>

上面是全局标准化之后的滑动平均之后的结果，发现也是可以捕获基本的趋势，但是在股价绝对值上面，差别较大。


以上，是基于LSTM网络时间序列预测模型对股价数据的预测情况。基本上，学术和金融界对个这个模型的研究到这里就差不多了。但是我仍然在思考一个问题：

**股价预测模型更多的是需要通过历史数据预测未来走势，这并不是简单的时间序列监督学习问题，更多的是将历史数据作为未来股价的特征，并对模型进行训练。虽然LSTM网络就可以捕捉历史信息，但是实际训练时，我们是将每个时间点数据拼接起来作为时间序列，并将这个序列输入模型，预测时，我们也是通过测试集的特征，预测股价这个结果。如果在实际应用中，我并不知道未来一段时间的特征，那我就无法预测未来的股价，我们的模型实际上就没有做到预知未来。**

**所以，我在思考，有没有可能将未来股价作为标签，现在的数据作为特征，直接在训练数据上就体现出“用过去预测未来”这种理念呢？**

带着这个问题，我需要重新规划我的数据集。数据集中，训练集和测试集应该这样划分：

1.过去的数据特征应该可以映射到未来的股价上，即投资者们可能在投资策略和布局上有自己一套计划，这个计划可能短到几天，长到几年，如果我们能够将过去的特征与未来的股价波动构建联系，那么就是说我们发现了众多投资者的策略的规律，这样可以很好的预测未来股价。

2.这种划分还需要避免以下问题，那就是不能够人为的定义训练时间窗口。因为我们的模型的理想状态是通过训练自己发现股票投资者的投资策略和股价的波动的历史原因，而不是我们定义一个时间窗口，单纯以时间窗口的数据进行训练模型。

想破了脑袋，也想不出如何规划这样的训练数据：必须以全部时间步的训练数据为特征，并且标签为未来的未知数据。这样的训练模型，恐怕不是监督学习能搞定的。

综上，我感到监督学习已经无能为力。这是由监督学习的框架和理念决定的，**它无法产生新的数据，只能在原有数据上不断近似和逼近。**

但是通过使用LSTM模型对高维特征进行分析，我们仍然可以看到，模型能够学习到一些价格波动规律，这就说明股价在一定程度上是可预测的！我们需要综合各种深度学习模型并应用到实际需求当中。监督模型可以挖掘特定的时间序列中特征集合与标签之间的内在联系，无监督模型可以自己去发现样本集的特征、分布等高维的规律，并用编码或者神经网络去表征。我们既需要发现在一段时间内股价与特征之间的关系，也需要发现股价的历史信息中的分布信息等高维信息，并以此为依据去预测未来的股价波动。这是监督模型和非监督模型的结合。

这个时候，我想终于到了VAE和GAN等生成模型出场的时候了。

### 直接使用原版VAE对股价模型进行编码

利用Keras自带的vae模型，尝试对股价特征进行编码时，出现无法训练，训练loss为负数，验证集loss直接发散的情况。我尝试修改了隐层的参数，用更大维度的隐层编码，发现结果并没有变化。

首先是用训练集/验证集为0.9的情况下对模型进行训练的结果：

<center><img src='project\vae\train_val_loss_0.9.png' width=1060></img></center>

<center><img src='project\vae\encode_0.9.png' width=1060></img></center>

训练结果很糟糕，模型没有被训练出来，但是encode的前两个维度貌似有聚集的特征。

然后用训练集/验证集为0.98的情况下对模型进行训练：


<center><img src='project\vae\train_val_loss.png' width=1060></img></center>


<center><img src='project\vae\encode.png' width=1060></img></center>

验证loss出现很大的波动，而且训练误差为负数，编码反而更加随机。


综合考虑可能有如下原因：

1.股价数据是时间序列数据：原版的变分模型没有对时间序列信息进行编码的能力，仅仅是对图像编码的移植。

2.股价的特征选择问题：一个上市公司的股价波动与其财务、市场信息的关联性较小，特征无法代表股价的走势。


3.模型还没有发现股价走势的深层信息，或者是无法拟合股价的波动特性。

原版的VAE模型是用来处理图像的，主要是用来生成nmist数据集中的手写数字，在直接用于时间序列数据中时，出现很多问题。我想，我们需要修改原本的模型结构，使其能够处理时序数据。

TODO


## 使用GAN网络生成股价预测数据

使用了大概一周的时间系统学习了GAN这个庞大的生成模型体系，这里还是要非常感谢苏神的博客提供的通俗易懂且鞭辟入里的解释。

https://kexue.fm/content.html

GAN中的对抗和生成两个模型，看上去只需要将各自的结果进行比较，然后分别根据比较结果优化自己模型，最后实现生成结果与真实结果逐步逼近即可。但实际上，这个过程中包含了非常丰富的数学推导，极其广泛的数理基础以及可以推广到很多方面的应用前景。

目前GAN网络在图像编码和生成方面取得了卓有成效的进展，我们可以看到人脸图片的生成过程，包括从一个男性脸逐渐变成婴儿脸的逐步渐进的变化情况。我们将其理解为，模型学习到了人脸数据集中不同人脸的特征，并将这些特征编码，再将这个编码空间映射到一个高斯分布空间中，确保整个编码空间可以被采样到，这样话，我们就能够通过人工设定采样参数的方式获取到这个编码，从而生成我们想要的人脸图像。这个思路是将人脸特征信息从隐参数空间中挖掘出来，成为一个能被大家所利用的特征，我们就可以使用这个特征空间，来做很多有趣的事情。

作为股价生成任务中的模型，我始终在思考如何将图像处理任务迁移到时间序列模型的生成任务上来，在前人所做的研究中，SeqGAN完成了文本信息的GAN网络，通过LSTM模型作为生成器，CNN网络作为判别器，使用Policy Gradient强化学习算法的loss作为优化对象进行seq to seq的生成任务，取得了一定的成果。但是文本信息生成任务与股价信息生成任务还是有一些区别。主要是问答系统或者翻译系统的seq to seq模型基于的是编码，以编码为传递信息的主要媒介，再看seqGAN模型，也是将这个编码映射到高斯空间，中间也加入了一些便于模型收敛的技巧。而股价模型更多的是通过历史交易数据预测未来，历史数据与未来数据有一定的时间连续性，而且股价特征既有消息面、技术面也有基本面，如何捕捉这些特征并预测未来股价，与传统的文本序列生成模型有很大不同。这也是我始终在纠结和思考的问题。

之前，我使用了LSTM模型，以50个交易日为一个period作为训练样本，训练了能够捕捉股价波动趋势的模型，取得了想要的效果。但是模型最大的问题是：

1.股价预测任务不是监督学习任务，我们在无法获得未来股价的特征时，显然也就无法预测未来股价。

2.股价的波动趋势能够被模型学习到，主要原因可能是开盘价、最高最低价等与收盘价强相关的信息作为特征，模型可以比较容易的推测出结果，模型就不需要学习上市公司财报、财务数据等基本面数据或者成交量等技术指标，所以，这个模型可能没有学习到股价的真正特征。

针对以上问题，我也是想破了脑袋叫破了喉咙。决定以四步走的形式，彻底解决这个问题：

1.重新规划训练模型。这一次，我想让模型学习到历史数据对未来股价的影响，也就是说，我能够在只知道现在和历史的成交信息的基础上，预测出未来n-m个交易日的股价。在训练数据上，以n为窗口，1为步长划分数据，在n这个窗口内，取m=λn，λ一般取0.9，也就是说将窗口内m个数据特征作为训练数据，而n-m个标签作为这个窗口训练的标签，这样的话，模型就可以训练出m天的数据对未来n-m股价的映射。仍然通过LSTM网络进行训练和测试。

2.如果步骤1可以得到比较满意的效果，那么说明模型可以学到历史数据对未来股价的影响情况，那么，我们就可以构建一个序列生成模型GAN，以LSTM为生成器，CNN为判别器，同时还对股价特征空间进行编码。

3.在这个基础上，我们再利用Bert模型对市场信息、上市公司财报等数据进行编码和情感分析，可以得到一个关于市场积极程度的编码向量，利用这个编码向量对GAN模型生成向量进行加权，就可以将市场短期的情绪加入到模型生成中，从而训练出既能够把握市场长期趋势，又能够考虑到短期的财经新闻和消息对股价影响的模型。

4.最后，对于多重的模型，我们既要交替训练，又要使用强化学习不同模型loss的权重进行探索和调试，期待最后收敛到一个合理的范围。

以上，就是我目前对于整个系统的全部思路，可以说是一边学习，一边实验，一边调整思路，好在一开始的夙愿和想法的大方向，都没有变，只是在细节和实现路径上，不断的试错和修正。

那么接下来，任何开源的项目都没有做过的模型，就要开始搭建了。

# 6. 评估和调参

## keras强化学习框架

* Keras-rl库

# 7. 可视化

TODO

# 8. 参考资料

感谢支持，赚个积分。

https://tushare.pro/register?reg=266868 