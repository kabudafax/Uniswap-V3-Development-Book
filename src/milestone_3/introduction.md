# 跨价格区间交换

到目前为止,我们已经取得了巨大的进展,我们的Uniswap V3实现已经非常接近原始版本了！然而,我们的实现仅支持在一个价格区间内进行交换——这正是我们将在这个里程碑中改进的地方。

在这个里程碑中,我们将:

1. 更新`mint`函数,以便在不同的价格区间提供流动性；

2. 更新`swap`函数,当当前价格区间的流动性不足时,可以跨越价格区间；

3. 学习如何在智能合约中计算流动性；

4. 在`mint`和`swap`函数中实现滑点保护；

5. 更新UI应用程序,允许在不同的价格区间添加流动性；

6. 进一步了解定点数。

在这个里程碑中,我们将完成交换功能,这是Uniswap的核心功能！

让我们开始吧！

> 你可以在[这个Github分支](https://github.com/Jeiwan/uniswapv3-code/tree/milestone_3)中找到本章的完整代码。
>
> 这个里程碑在现有合约中引入了很多代码变更。[在这里你可以看到自上一个里程碑以来的所有变更](https://github.com/Jeiwan/uniswapv3-code/compare/milestone_2...milestone_3)

> 如果你有任何问题,欢迎在[这个里程碑的GitHub讨论区](https://github.com/Jeiwan/uniswapv3-book/discussions/categories/milestone-3-cross-tick-swaps)中提出！
