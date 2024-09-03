# 管理器合约

在部署我们的池子合约之前，我们需要解决一个问题。如你所记得的，Uniswap V3 合约分为两类：

1. 核心合约，实现核心功能，不提供用户友好的接口。
2. 外围合约，为核心合约实现用户友好的接口。

池子合约是一个核心合约，它不应该是用户友好和灵活的。它期望调用者进行所有的计算（价格、金额）并提供适当的调用参数。它也不使用 ERC20 的 `transferFrom` 来从调用者转移代币。相反，它使用两个回调：

1. `uniswapV3MintCallback`，在铸造流动性时调用；
2. `uniswapV3SwapCallback`，在交换代币时调用。

在我们的测试中，我们在测试合约中实现了这些回调。由于只有合约可以实现它们，普通用户（非合约地址）无法调用池子合约。这没问题。但现在不再是这样了 🙂。

我们在本书中的下一步是将池子合约部署到本地区块链，并从前端应用程序与之交互。因此，我们需要构建一个合约，让非合约地址与池子交互。让我们现在就做这个！

## 工作流程

管理器合约将这样工作：

1. 为了铸造流动性，我们将批准管理器合约花费代币。
2. 然后我们将调用管理器合约的 `mint` 函数，并传递铸造参数，以及我们想要提供流动性的池子地址。
3. 管理器合约将调用池子的 `mint` 函数并实现 `uniswapV3MintCallback`。它将有权限将我们的代币发送到池子合约。
4. 为了交换代币，我们也将批准管理器合约花费代币。
5. 然后我们将调用管理器合约的 `swap` 函数，类似于铸造，它将把调用传递给池子。
管理器合约将把我们的代币发送到池子合约，池子合约将交换它们并将输出金额发送给我们。

因此，管理器合约将作为用户和池子之间的中介。

## 向回调传递数据

在实现管理器合约之前，我们需要升级池子合约。

管理器合约将与任何池子一起工作，并允许任何地址调用它。为了实现这一点，我们需要升级回调：我们希望向它们传递不同的池子地址和用户地址。让我们看看我们当前的 `uniswapV3MintCallback` 实现（在测试合约中）：

```solidity
function uniswapV3MintCallback(uint256 amount0, uint256 amount1) public {
    if (transferInMintCallback) {
        token0.transfer(msg.sender, amount0);
        token1.transfer(msg.sender, amount1);
    }
}
```
这里的关键点：

该函数转移属于测试合约的代币——我们希望它通过使用 transferFrom 从调用者转移代币。
该函数知道 token0 和 token1，这对每个池子都会不同。
想法：我们需要改变回调的参数，以便我们可以传递用户和池子地址。

现在，让我们看看交换回调：

```solidity
function uniswapV3SwapCallback(int256 amount0, int256 amount1) public {
    if (amount0 > 0 && transferInSwapCallback) {
        token0.transfer(msg.sender, uint256(amount0));
    }

    if (amount1 > 0 && transferInSwapCallback) {
        token1.transfer(msg.sender, uint256(amount1));
    }
}
```

同样，它从测试合约转移代币，并且知道 token0 和 token1。

为了向回调传递额外的数据，我们需要首先将其传递给 mint 和 swap（因为回调是从这些函数调用的）。然而，由于这些额外的数据在函数中没有使用，为了不使它们的参数更混乱，我们将使用 abi.encode() 对额外的数据进行编码。

让我们将额外的数据定义为一个结构：

```solidity
// src/UniswapV3Pool.sol

struct CallbackData {
    address token0;
    address token1;
    address payer;
}
```

然后将编码后的数据传递给回调：

```solidity
function mint(
    address owner,
    int24 lowerTick,
    int24 upperTick,
    uint128 amount,
    bytes calldata data // <--- 新行
) external returns (uint256 amount0, uint256 amount1) {
    
    IUniswapV3MintCallback(msg.sender).uniswapV3MintCallback(
        amount0,
        amount1,
        data // <--- 新行
    );
    
}


function swap(address recipient, bytes calldata data) // <--- 添加了 `data`
    public
    returns (int256 amount0, int256 amount1)
{
    
    IUniswapV3SwapCallback(msg.sender).uniswapV3SwapCallback(
        amount0,
        amount1,
        data // <--- 新行
    );
    
}
```

现在，我们可以在测试合约的回调中读取额外的数据。

```solidity
function uniswapV3MintCallback(
    uint256 amount0,
    uint256 amount1,
    bytes calldata data
) public {
    if (transferInMintCallback) {
        UniswapV3Pool.CallbackData memory extra = abi.decode(
            data,
            (UniswapV3Pool.CallbackData)
        );

        IERC20(extra.token0).transferFrom(extra.payer, msg.sender, amount0);
        IERC20(extra.token1).transferFrom(extra.payer, msg.sender, amount1);
    }
}
```

试着自己更新其余的代码，如果变得太困难，可以随时查看 这个提交。


## 实现管理器合约

除了实现回调，管理器合约不会做太多事情：它只会简单地将调用重定向到池子合约。目前这是一个非常简约的合约：

```solidity
pragma solidity ^0.8.14;

import "../src/UniswapV3Pool.sol";
import "../src/interfaces/IERC20.sol";

contract UniswapV3Manager {
    function mint(
        address poolAddress_,
        int24 lowerTick,
        int24 upperTick,
        uint128 liquidity,
        bytes calldata data
    ) public {
        UniswapV3Pool(poolAddress_).mint(
            msg.sender,
            lowerTick,
            upperTick,
            liquidity,
            data
        );
    }

    function swap(address poolAddress_, bytes calldata data) public {
        UniswapV3Pool(poolAddress_).swap(msg.sender, data);
    }

    function uniswapV3MintCallback(...) {...}
    function uniswapV3SwapCallback(...) {...}
}
```

这些回调与测试合约中的回调相同，只是没有 transferInMintCallback 和 transferInSwapCallback 标志，因为管理器合约总是转移代币。

好了，我们现在完全准备好部署并与前端应用程序集成了！
