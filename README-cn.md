

Python   平台   pypi   MIT 许可证   建造   代码验证   下载   粘合剂  

PyPortfolioOpt 是一个实现投资组合优化方法的库，包括经典的均值方差优化技术和 Black-Litterman 分配，以及该领域的最新发展，如收缩和分层风险平价。

它功能强大且易于扩展，无论是普通投资者还是寻求简单原型设计工具的专业人士，都能轻松上手。无论您是注重基本面、已识别出少量被低估的投资标的的投资者，还是拥有一篮子策略的算法交易员，PyPortfolioOpt 都能帮助您以风险高效的方式组合您的 alpha 来源。

PyPortfolioOpt 已在《开源软件杂志》上发表🎉

PyPortfolioOpt 目前由Tuan Tran维护。

前往ReadTheDocs 上的文档深入了解该项目，或查看食谱以查看一些示例，了解从下载数据到构建投资组合的整个过程。



目录
目录
入门
为了发展
一个简单的例子
经典投资组合优化方法概述
特征
预期回报
风险模型（协方差）
目标函数
添加约束或不同的目标
Black-Litterman分配
其他优化器
相比现有实现的优势
项目原则和设计决策
测试
引用 PyPortfolioOpt
贡献
取得联系
入门
如果您想在浏览器中以交互方式使用 PyPortfolioOpt，可以在此处启动 Binder 。设置需要一些时间，但它可以让您尝试食谱，而无需处理所有要求。

注意：macOS 用户需要安装命令行工具。

注意：如果您使用的是 Windows，则首先需要安装 C++。（下载、安装说明）

该项目可在 PyPI 上获取，这意味着您可以：

pip install PyPortfolioOpt
（您可能需要遵循cvxopt和cvxpy的单独安装说明）。

不过，最佳做法是在虚拟环境中使用依赖管理器。我目前的建议是先设置好Poetry，然后运行

poetry add PyPortfolioOpt
否则，克隆/下载项目并在项目目录中运行：

python setup.py install
PyPortfolioOpt 支持 Docker。使用 构建你的第一个容器docker build -f docker/Dockerfile . -t pypfopt。你可以使用该镜像运行测试，甚至启动 Jupyter 服务器。

# iPython interpreter:
docker run -it pypfopt poetry run ipython

# Jupyter notebook server:
docker run -it -p 8888:8888 pypfopt poetry run jupyter notebook --allow-root --no-browser --ip 0.0.0.0
# click on http://127.0.0.1:8888/?token=xxx

# Pytest
docker run -t pypfopt poetry run pytest

# Bash
docker run -it pypfopt bash
欲了解更多信息，请阅读本指南。

为了发展
如果您想进行重大更改以将其与您的专有系统集成，那么克隆此存储库并仅使用源代码可能是有意义的。

git clone https://github.com/robertmartin8/PyPortfolioOpt
或者，您可以尝试：

pip install -e git+https://github.com/robertmartin8/PyPortfolioOpt.git
一个简单的例子
以下是现实生活中的股票数据示例，展示了找到最大化夏普比率（风险调整后收益的衡量标准）的多头投资组合是多么容易。

import pandas as pd
from pypfopt import EfficientFrontier
from pypfopt import risk_models
from pypfopt import expected_returns

# Read in price data
df = pd.read_csv("tests/resources/stock_prices.csv", parse_dates=True, index_col="date")

# Calculate expected returns and sample covariance
mu = expected_returns.mean_historical_return(df)
S = risk_models.sample_cov(df)

# Optimize for maximal Sharpe ratio
ef = EfficientFrontier(mu, S)
raw_weights = ef.max_sharpe()
cleaned_weights = ef.clean_weights()
ef.save_weights_to_file("weights.csv")  # saves to file
print(cleaned_weights)
ef.portfolio_performance(verbose=True)
这将输出以下权重：

{'GOOG': 0.03835,
 'AAPL': 0.0689,
 'FB': 0.20603,
 'BABA': 0.07315,
 'AMZN': 0.04033,
 'GE': 0.0,
 'AMD': 0.0,
 'WMT': 0.0,
 'BAC': 0.0,
 'GM': 0.0,
 'T': 0.0,
 'UAA': 0.0,
 'SHLD': 0.0,
 'XOM': 0.0,
 'RRC': 0.0,
 'BBY': 0.01324,
 'MA': 0.35349,
 'PFE': 0.1957,
 'JPM': 0.0,
 'SBUX': 0.01082}

Expected annual return: 30.5%
Annual volatility: 22.2%
Sharpe Ratio: 1.28
这很有趣，但本身没什么用。不过，PyPortfolioOpt 提供了一种方法，可以让你把上述连续权重转换为可以购买的实际配置。只需输入最近的价格和所需的投资组合规模（本例中为 10,000 美元）：

from pypfopt.discrete_allocation import DiscreteAllocation, get_latest_prices


latest_prices = get_latest_prices(df)

da = DiscreteAllocation(weights, latest_prices, total_portfolio_value=10000)
allocation, leftover = da.greedy_portfolio()
print("Discrete allocation:", allocation)
print("Funds remaining: ${:.2f}".format(leftover))
12 out of 20 tickers were removed
Discrete allocation: {'GOOG': 1, 'AAPL': 4, 'FB': 12, 'BABA': 4, 'BBY': 2,
                      'MA': 20, 'PFE': 54, 'SBUX': 1}
Funds remaining: $11.89
免责声明：本项目不构成任何投资建议，作者对您后续的投资决策不承担任何责任。更多信息请参阅许可证。

经典投资组合优化方法概述
哈里·马科维茨（Harry Markowitz）1952年的论文堪称经典，它将投资组合优化从一门艺术变成了一门科学。其核心观点是，通过组合不同预期收益和波动率的资产，人们可以确定一个数学上最优的配置，从而最大限度地降低目标收益的风险——所有这些最优投资组合的集合被称为有效前沿。



尽管该领域已取得长足发展，但半个多世纪过去了，马科维茨的核心思想仍然至关重要，并在许多投资组合管理公司中日常运用。均值-方差优化的主要缺点在于，理论处理需要了解资产的预期收益和未来风险特征（协方差）。显然，如果我们知道股票的预期收益，生活会容易得多，但关键在于，股票收益的预测难度是出了名的。作为替代方案，我们可以根据历史数据推导出预期收益和协方差的估计值——虽然我们确实失去了马科维茨提供的理论保证，但我们的估计值越接近实际值，我们的投资组合就越好。

因此，该项目提供了四组主要功能（当然它们是密切相关的）

预期收益估计
风险估计（即资产收益的协方差）
待优化的目标函数
优化器。
PyPortfolioOpt 的一个关键设计目标是模块化——用户应该能够交换他们的组件，同时仍然使用 PyPortfolioOpt 提供的框架。

特征
在本节中，我们将详细介绍 PyPortfolioOpt 的一些可用功能。更多示例请参阅此处的Jupyter 笔记本。另一个不错的资源是测试。

在ReadTheDocs上可以找到更全面的版本，以及适合更高级用户的扩展。

预期回报
平均历史收益：
最简单、最常见的方法，即每项资产的预期收益等于其历史收益的平均值。
易于解释且非常直观
指数加权平均历史收益：
与平均历史收益相似，但它对近期价格的权重呈指数级增长
在估计未来回报时，资产最近的回报可能比 10 年前的回报更有分量。
资本资产定价模型（CAPM）：
基于市场贝塔值预测回报的简单模型
这在金融领域广泛使用！
风险模型（协方差）
协方差矩阵不仅编码了一项资产的波动性，还编码了它与其他资产的相关性。这一点很重要，因为为了获得多元化的收益（从而提高单位风险的回报），投资组合中的资产应尽可能保持不相关。

样本协方差矩阵：
协方差矩阵的无偏估计
相对容易计算
多年来的事实标准
然而，它的估计误差很高，这在均值方差优化中尤其危险，因为优化器可能会赋予这些错误估计过多的权重。
半协方差：一种关注下行变化的风险度量。
指数协方差：对样本协方差的改进，赋予近期数据更大的权重
协方差收缩：将样本协方差矩阵与结构化估计器相结合的技术，旨在减少错误权重的影响。PyPortfolioOpt 提供了由 提供的高效矢量化实现的包装器sklearn.covariance。
手动收缩
Ledoit Wolf 收缩法，用于选择最佳收缩参数。我们提供三个收缩目标：constant_variance、single_factor和constant_correlation。
Oracle 近似收缩
最小协方差行列式：
协方差的稳健估计
实施于sklearn.covariance


（此图是使用生成的plotting.plot_covariance）

目标函数
最大夏普比率：这会产生一个切线投资组合，因为在收益与风险的图表上，该投资组合对应于有效前沿的切线，其y轴截距等于无风险利率。这是默认选项，因为它可以找到单位风险下的最佳收益。
最小波动率。如果你想了解波动率到底能低到什么程度，这个方法或许有用，但实际上，我认为使用最大化夏普比率的投资组合更合理。
有效回报，又称马科维茨投资组合，在给定目标回报的情况下，将风险降至最低——这是马科维茨 1952 年的主要关注点
有效风险：给定目标风险的夏普最大化投资组合。
最大二次效用。您可以提供自己的风险规避水平并计算适当的投资组合。
添加约束或不同的目标
多头/空头：默认情况下，PyPortfolioOpt 中的所有均值方差优化方法都是仅限多头的，但可以通过改变权重界限来初始化它们以允许空头头寸：
ef = EfficientFrontier(mu, S, weight_bounds=(-1, 1))
市场中性：对于efficient_risk和efficient_return方法，PyPortfolioOpt 提供了一个选项来构建市场中性投资组合（即权重总和为零）。对于最大夏普投资组合和最小波动率投资组合，这是不可能的，因为它们在那些情况下不随杠杆率变化。市场中性要求权重为负：
ef = EfficientFrontier(mu, S, weight_bounds=(-1, 1))
ef.efficient_return(target_return=0.2, market_neutral=True)
最小/最大仓位规模：您可能希望任何证券的占比不超过投资组合的 10%。这很容易编码：
ef = EfficientFrontier(mu, S, weight_bounds=(0, 0.1))
均值-方差优化的一个问题是它会导致许多零权重。虽然这些零权重在样本内是“最优的”，但大量研究表明，这一特性会导致均值-方差投资组合在样本外表现不佳。为此，我引入了一个目标函数，可以减少任何目标函数中可忽略不计的权重数量。本质上，它对gamma小权重添加了一个惩罚项（参数为），其项类似于机器学习中的 L2 正则化。可能需要尝试多个gamma值才能达到所需的不可忽略权重数量。对于包含 20 只证券的测试投资组合，gamma ~ 1就足够了。

ef = EfficientFrontier(mu, S)
ef.add_objective(objective_functions.L2_reg, gamma=1)
ef.max_sharpe()
Black-Litterman分配
从 v0.5.0 开始，我们支持 Black-Litterman 资产配置模型，该模型允许您将先前估计的收益（例如市场隐含收益）与您自己的观点相结合，形成后验估计。与仅使用历史平均收益相比，这可以更好地估计预期收益。查看文档，了解该理论的讨论以及输入格式方面的建议。

S = risk_models.sample_cov(df)
viewdict = {"AAPL": 0.20, "BBY": -0.30, "BAC": 0, "SBUX": -0.2, "T": 0.131321}
bl = BlackLittermanModel(S, pi="equal", absolute_views=viewdict, omega="default")
rets = bl.bl_returns()

ef = EfficientFrontier(rets, S)
ef.max_sharpe()
其他优化器
上述功能主要涉及通过二次规划解决均值-方差优化问题（尽管 已处理cvxpy）。但是，我们也提供不同的优化器：

均值半方差优化
均值-CVaR优化
分层风险平价，使用聚类算法选择不相关资产
Markowitz 临界线算法（CLA）
请参阅文档以了解更多信息。

相比现有实现的优势
包括经典方法（Markowitz 1952 和 Black-Litterman）、建议的最佳实践（例如协方差收缩），以及许多最新发展和新颖特征，如 L2 正则化、收缩协方差、分层风险平价。
对 pandas 数据框的原生支持：轻松输入您的每日价格数据。
广泛的实践测试，使用真实数据。
易于与您的专有策略和模型相结合。
对缺失数据和不同长度的价格序列具有稳健性（例如，FB 数据仅追溯到 2012 年，而 AAPL 数据可追溯到 1980 年）。
项目原则和设计决策
应该可以轻松地将优化过程的各个组件与用户的专有改进进行交换。
可用性就是一切：不言自明比保持一致更好。
除非投资组合优化能够实际应用于实际资产价格，否则它毫无意义。
任何已实施的措施都应经过测试。
内联文档是好的，专用（独立）文档更好。两者并不互相排斥。
格式化不应该妨碍编码：因此，我将所有格式化决定都推迟给了Black。
测试
测试是用 pytest 编写的（在我看来，它比其他工具更直观unittest），并且我已尽力确保接近 100% 的覆盖率。只需导航到包目录，然后pytest在命令行中运行即可运行测试。

PyPortfolioOpt 提供了 20 个股票每日回报的测试数据集：

['GOOG', 'AAPL', 'FB', 'BABA', 'AMZN', 'GE', 'AMD', 'WMT', 'BAC', 'GM',
'T', 'UAA', 'SHLD', 'XOM', 'RRC', 'BBY', 'MA', 'PFE', 'JPM', 'SBUX']
这些股票代码是经过非正式筛选后选定的，以满足以下几个标准：

流动性适中
不同的表现和波动性
不同数量的数据来测试稳健性
目前，测试尚未探索所有边缘情况以及目标函数和参数的组合。然而，每种方法和参数都已测试，确保其按预期工作。

引用 PyPortfolioOpt
如果您使用 PyPortfolioOpt 发表作品，请引用JOSS 论文。

引用字符串：

Martin, R. A., (2021). PyPortfolioOpt: portfolio optimization in Python. Journal of Open Source Software, 6(61), 3066, https://doi.org/10.21105/joss.03066
BibTex::

@article{Martin2021,
  doi = {10.21105/joss.03066},
  url = {https://doi.org/10.21105/joss.03066},
  year = {2021},
  publisher = {The Open Journal},
  volume = {6},
  number = {61},
  pages = {3066},
  author = {Robert Andrew Martin},
  title = {PyPortfolioOpt: portfolio optimization in Python},
  journal = {Journal of Open Source Software}
}
贡献
欢迎大家积极贡献。更多信息请参阅贡献指南。

我要感谢自 2018 年 PyPortfolioOpt 发布以来为其做出贡献的所有人。特别感谢：

Tuan Tran（现在是主要维护者！）
菲利普·席勒
卡尔·皮斯内尔
费利佩·施奈德
王定元
帕特·纽厄尔
阿迪亚·布特拉
托马斯·施梅尔策
里奇·卡普托
尼古拉斯·克努德