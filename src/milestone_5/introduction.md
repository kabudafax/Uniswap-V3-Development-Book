# 手续费和价格预言机

在这个里程碑中，我们将为我们的 Uniswap 实现添加两个新功能。它们有一个共同点：它们都是建立在我们已经构建的基础之上——这就是为什么我们将它们推迟到这个里程碑。然而，它们的重要性并不相同。

我们将添加交换手续费和价格预言机：

- 交换手续费是我们正在实现的 DEX 设计中的一个关键机制。它们是使整个系统粘合在一起的胶水。交换手续费激励流动性提供者提供流动性，而没有流动性就无法进行交易，这一点我们已经了解过了。

- 另一方面，价格预言机是 DEX 的一个可选的实用功能。DEX 在进行交易的同时，也可以充当价格预言机——即为其他服务提供代币价格。这不会影响实际的交换，但为其他链上应用提供了有用的服务。

好了，让我们开始构建吧！

> 你可以在[这个 Github 分支](https://github.com/Jeiwan/uniswapv3-code/tree/milestone_5)中找到本章的完整代码。
>
> 这个里程碑对现有合约引入了许多代码更改。[在这里你可以看到自上一个里程碑以来的所有变更](https://github.com/Jeiwan/uniswapv3-code/compare/milestone_4...milestone_5)

> 如果你有任何问题，欢迎在[本里程碑的 GitHub 讨论区](https://github.com/Jeiwan/uniswapv3-book/discussions/categories/milestone-5-fees-and-price-oracle)中提出！
