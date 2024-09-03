# 多池交换

在实现跨刻度交换后，我们已经接近了真实的 Uniswap V3 交换。我们实现的一个重要限制是它只允许在单个池内进行交换——如果一对代币没有对应的池，那么这两个代币之间就无法进行交换。但在 Uniswap 中并非如此，因为它允许多池交换。在本章中，我们将为我们的实现添加多池交换功能。

以下是我们的计划：

1. 首先，我们将学习并实现 Factory 合约；

2. 然后，我们将了解链式或多池交换的工作原理，并实现 Path 库；

3. 接着，我们将更新前端应用以支持多池交换；

4. 我们将实现一个基本的路由器，用于在两个代币之间找到路径；

5. 在此过程中，我们还将学习刻度间距（tick spacing），这是一种优化交换的方法。

完成本章后，我们的实现将能够处理多池交换，例如，通过不同的稳定币将 WBTC 换成 WETH：WETH → USDC → USDT → WBTC。

让我们开始吧！

> 你可以在[这个 Github 分支](https://github.com/Jeiwan/uniswapv3-code/tree/milestone_4)中找到本章的完整代码。
>
> 这个里程碑对现有合约引入了许多代码更改。[在这里你可以看到自上一个里程碑以来的所有变更](https://github.com/Jeiwan/uniswapv3-code/compare/milestone_3...milestone_4)

> 如果你有任何问题，欢迎在[本里程碑的 GitHub 讨论区](https://github.com/Jeiwan/uniswapv3-book/discussions/categories/milestone-4-multi-pool-swaps)中提出！
