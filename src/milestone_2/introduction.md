# 第二次交换

好了,现在事情变得真实起来了。到目前为止,我们的实现看起来太过人为和静态。我们计算并硬编码了所有的金额,以使学习曲线不那么陡峭,现在我们准备让它变得动态。我们将实现第二次交换,这是一个相反方向的交换:卖出ETH来购买USDC。为此,我们将大幅改进我们的智能合约:

1. 我们需要在Solidity中实现数学计算。然而,由于Solidity只支持整数除法,在Solidity中实现数学运算很棘手,所以我们将使用第三方库。

2. 我们需要让用户选择交换方向,而池合约需要支持双向交换。我们将改进合约,使其更接近多范围交换,这是我们将在下一个里程碑中实现的。

3. 最后,我们将更新UI以支持双向交换AND输出金额计算！这将要求我们实现另一个合约,Quoter。

在这个里程碑结束时,我们将拥有一个几乎像真正的DEX一样工作的应用程序！

让我们开始吧！

> 你可以在[这个Github分支](https://github.com/Jeiwan/uniswapv3-code/tree/milestone_2)中找到本章的完整代码。
>
> 这个里程碑在现有合约中引入了很多代码变更。[在这里你可以看到自上一个里程碑以来的所有变更](https://github.com/Jeiwan/uniswapv3-code/compare/milestone_1...milestone_2)

> 如果你有任何问题,欢迎在[这个里程碑的GitHub讨论区](https://github.com/Jeiwan/uniswapv3-book/discussions/categories/milestone-2-second-swap)中提出！
