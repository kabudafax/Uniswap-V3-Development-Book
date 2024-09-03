# NFT 头寸

这是本书的点睛之笔。在这个里程碑中，我们将学习如何扩展 Uniswap 合约并将其集成到第三方协议中。这种可能性是核心合约只包含关键功能的直接结果，这允许将其集成到其他合约中，而无需向核心合约添加新功能。

Uniswap V3 的一个额外特性是将流动性头寸转换为 NFT 代币的能力。以下是这种代币的一个例子：

<p align="center">
<img src="images/nft_example.png" alt="Uniswap V3 NFT 示例"/>
</p>

它显示了代币符号、池手续费、头寸 ID、下限和上限刻度、代币地址，以及提供头寸的曲线段。

> 你可以在[这个 OpenSea 集合](https://opensea.io/collection/uniswap-v3-positions)中查看所有 Uniswap V3 NFT 头寸。

在这个里程碑中，我们将添加流动性头寸的 NFT 代币化功能！

让我们开始吧！

> 你可以在[这个 Github 分支](https://github.com/Jeiwan/uniswapv3-code/tree/milestone_6)中找到本章的完整代码。
>
> 这个里程碑对现有合约引入了许多代码更改。[在这里你可以看到自上一个里程碑以来的所有变更](https://github.com/Jeiwan/uniswapv3-code/compare/milestone_5...milestone_6)

> 如果你有任何问题，欢迎在[本里程碑的 GitHub 讨论区](https://github.com/Jeiwan/uniswapv3-book/discussions/categories/milestone-6-nft-positions)中提出！
