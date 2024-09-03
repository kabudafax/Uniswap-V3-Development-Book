# 介绍

在这个里程碑中，我们将构建一个池子合约，它可以从用户那里接收流动性，并在一个价格范围内进行交换。为了尽可能保持简单，我们将只在一个价格范围内提供流动性，并且只允许在一个方向上进行交换。此外，我们将手动计算所有必需的数学计算，以在开始使用 Solidity 中的数学库之前获得更好的直觉。

让我们模拟我们将要构建的情况：

1. 将会有一个 ETH/USDC 池子合约。ETH 将是 \\(x\\) 储备，USDC 将是 \\(y\\) 储备。

2. 我们将把当前价格设置为 1 ETH 兑换 5000 USDC。

3. 我们将提供流动性的范围是 1 ETH 兑换 4545-5500 USDC。

4. 我们将从池子中购买一些 ETH。在这一点上，由于我们只有一个价格范围，我们希望交易的价格保持在价格范围内。

视觉上，这个模型看起来像这样：

![用 USDC 购买 ETH 的可视化](images/buy_eth_model.png)

在开始编码之前，让我们弄清楚数学并计算模型的所有参数。为了保持简单，我将在 Python 中进行数学计算，然后再在 Solidity 中实现它们。这将允许我们专注于数学，而不用深入研究 Solidity 中的数学细节。这也意味着，在智能合约中，我们将硬编码所有的金额。这将允许我们从一个简单的最小可行产品开始。

为了方便起见，我把所有的 Python 计算放在了 [unimath.py](https://github.com/Jeiwan/uniswapv3-code/blob/main/unimath.py) 中。

> 你可以在 [这个 Github 分支](https://github.com/Jeiwan/uniswapv3-code/tree/milestone_1) 中找到这个里程碑的完整代码。

> 如果你有任何问题，欢迎在 [这个里程碑的 GitHub 讨论](https://github.com/Jeiwan/uniswapv3-book/discussions/categories/milestone-1-first-swap) 中提出！
