# 我们将构建什么

本书的目标是构建一个 Uniswap V3 的克隆版。然而，我们不会构建一个完全相同的副本。主要原因是 Uniswap 是一个包含许多细节和辅助机制的大型项目——详细解释所有这些内容会使本书内容过于庞大，并使读者更难完成阅读。相反，我们将构建 Uniswap 的核心，即其最困难和最重要的机制。这包括流动性管理、交换、费用、外围合约、报价合约和 NFT 合约。之后，我相信你将能够阅读 Uniswap V3 的源代码，并理解本书范围之外的所有机制。

## 智能合约

完成本书后，你将实现以下合约：
1. `UniswapV3Pool`——实现流动性管理和交换的核心池合约。这个合约非常接近[原始合约](https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Pool.sol)，但是一些实现细节不同，为了简化也省略了一些内容。例如，我们的实现将只处理"精确输入"交换，即已知输入金额的交换。原始实现还支持已知*输出*金额的交换（即当你想购买特定数量的代币时）。
2. `UniswapV3Factory`——部署新池并保存所有已部署池记录的注册合约。除了更改所有者和费用的能力外，这个合约与[原始合约](https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Factory.sol)基本相同。
3. `UniswapV3Manager`——一个使与池合约交互更容易的外围合约。这是[SwapRouter](https://github.com/Uniswap/v3-periphery/blob/main/contracts/SwapRouter.sol)的一个非常简化的实现。同样，如你所见，我不区分"精确输入"和"精确输出"交换，只实现前者。
4. `UniswapV3Quoter` 是一个很酷的合约，允许在链上计算交换价格。这是 [Quoter](https://github.com/Uniswap/v3-periphery/blob/main/contracts/lens/Quoter.sol) 和 [QuoterV2](https://github.com/Uniswap/v3-periphery/blob/main/contracts/lens/QuoterV2.sol) 的最小化复制版。再次强调，只支持"精确输入"交换。
5. `UniswapV3NFTManager` 允许将流动性头寸转换为 NFT。这是 [NonfungiblePositionManager](https://github.com/Uniswap/v3-periphery/blob/main/contracts/NonfungiblePositionManager.sol) 的简化实现。

## 前端应用程序

对于本书，我还构建了一个简化版的 [Uniswap UI](https://app.uniswap.org/) 克隆。这是一个非常简单的克隆版，而且我的 React 和前端技能非常有限，但它展示了前端应用程序如何使用 Ethers.js 和 MetaMask 与智能合约交互。
