# 多池交换

我们现在进入这个里程碑的核心部分——在我们的合约中实现多池交换。在这个里程碑中，我们不会修改Pool合约，因为它是一个只应实现核心功能的核心合约。多池交换是一个实用功能，我们将在Manager和Quoter合约中实现它。

## 更新Manager合约

### 单池和多池交换

在我们当前的实现中，Manager合约中的`swap`函数只支持单池交换，并在参数中接收池地址：

```solidity
function swap(
    address poolAddress_,
    bool zeroForOne,
    uint256 amountSpecified,
    uint160 sqrtPriceLimitX96,
    bytes calldata data
) public returns (int256, int256) { ... }
```

我们将把它分成两个函数：单池交换和多池交换。这些函数将有不同的参数集：

```solidity
struct SwapSingleParams {
    address tokenIn;
    address tokenOut;
    uint24 tickSpacing;
    uint256 amountIn;
    uint160 sqrtPriceLimitX96;
}

struct SwapParams {
    bytes path;
    address recipient;
    uint256 amountIn;
    uint256 minAmountOut;
}
```

1. `SwapSingleParams`接收池参数、输入金额和限制价格——这与我们之前的内容基本相同。注意，不再需要`data`。

2. `SwapParams`接收路径、输出金额接收者、输入金额和最小输出金额。后一个参数替代了`sqrtPriceLimitX96`，因为在进行多池交换时，我们无法使用Pool合约中的滑点保护（它使用限制价格）。我们需要实现另一种滑点保护，它检查最终输出金额并与`minAmountOut`进行比较：当最终输出金额小于`minAmountOut`时，滑点保护失败。

### 核心交换逻辑

让我们实现一个内部的`_swap`函数，它将被单池和多池交换函数调用。它将准备参数并调用`Pool.swap`。

```solidity
function _swap(
    uint256 amountIn,
    address recipient,
    uint160 sqrtPriceLimitX96,
    SwapCallbackData memory data
) internal returns (uint256 amountOut) {
    ...
```

`SwapCallbackData`是一个新的数据结构，包含我们在交换函数和`uniswapV3SwapCallback`之间传递的数据：
```solidity
struct SwapCallbackData {
    bytes path;
    address payer;
}
```

`path`是交换路径，`payer`是在交换中提供输入代币的地址——在多池交换过程中，我们会有不同的支付者。

在`_swap`中，我们首先要做的是使用`Path`库提取池参数：

```solidity
// function _swap(...) {
(address tokenIn, address tokenOut, uint24 tickSpacing) = data
    .path
    .decodeFirstPool();
```

然后我们确定交换方向：

```solidity
bool zeroForOne = tokenIn < tokenOut;
```

然后我们进行实际的交换：
```solidity
// function _swap(...) {
(int256 amount0, int256 amount1) = getPool(
    tokenIn,
    tokenOut,
    tickSpacing
).swap(
        recipient,
        zeroForOne,
        amountIn,
        sqrtPriceLimitX96 == 0
            ? (
                zeroForOne
                    ? TickMath.MIN_SQRT_RATIO + 1
                    : TickMath.MAX_SQRT_RATIO - 1
            )
            : sqrtPriceLimitX96,
        abi.encode(data)
    );
```

这部分与我们之前的内容相同，但这次我们调用`getPool`来找到池。`getPool`是一个对代币进行排序并调用`PoolAddress.computeAddress`的函数：

```solidity
function getPool(
    address token0,
    address token1,
    uint24 tickSpacing
) internal view returns (IUniswapV3Pool pool) {
    (token0, token1) = token0 < token1
        ? (token0, token1)
        : (token1, token0);
    pool = IUniswapV3Pool(
        PoolAddress.computeAddress(factory, token0, token1, tickSpacing)
    );
}
```

进行交换后，我们需要确定哪个金额是输出金额：
```solidity
// function _swap(...) {
amountOut = uint256(-(zeroForOne ? amount1 : amount0));
```

就是这样。现在让我们看看单池交换是如何工作的。

### 单池交换

`swapSingle`简单地作为`_swap`的包装器：

```solidity
function swapSingle(SwapSingleParams calldata params)
    public
    returns (uint256 amountOut)
{
    amountOut = _swap(
        params.amountIn,
        msg.sender,
        params.sqrtPriceLimitX96,
        SwapCallbackData({
            path: abi.encodePacked(
                params.tokenIn,
                params.tickSpacing,
                params.tokenOut
            ),
            payer: msg.sender
        })
    );
}
```

注意，我们在这里构建了一个单池路径：单池交换实际上是只有一个池的多池交换 🙂。

### 多池交换

多池交换只比单池交换稍微复杂一些。让我们来看看：

```solidity
function swap(SwapParams memory params) public returns (uint256 amountOut) {
    address payer = msg.sender;
    bool hasMultiplePools;
    ...
```

第一次交换由用户支付，因为是用户提供输入代币。

然后，我们开始遍历路径中的池：

```solidity
...
while (true) {
    hasMultiplePools = params.path.hasMultiplePools();

    params.amountIn = _swap(
        params.amountIn,
        hasMultiplePools ? address(this) : params.recipient,
        0,
        SwapCallbackData({
            path: params.path.getFirstPool(),
            payer: payer
        })
    );
    ...
```

在每次迭代中，我们用以下参数调用`_swap`：

1. `params.amountIn`跟踪输入金额。在第一次交换时，它是用户提供的金额。在后续的交换中，它是前一次交换返回的金额。

2. `hasMultiplePools ? address(this) : params.recipient`——如果路径中有多个池，接收者是Manager合约，它将在交换之间存储代币。如果路径中只有一个池（最后一个），接收者是参数中指定的接收者（通常是发起交换的用户）。

3. `sqrtPriceLimitX96`设置为0以禁用Pool合约中的滑点保护。

4. 最后一个参数是我们传递给`uniswapV3SwapCallback`的内容——我们稍后会看到。

进行一次交换后，我们需要继续处理路径中的下一个池或返回：

```solidity
    ...

    if (hasMultiplePools) {
        payer = address(this);
        params.path = params.path.skipToken();
    } else {
        amountOut = params.amountIn;
        break;
    }
}
```

这里我们正在更改支付者并从路径中移除已处理的池。

最后，新的滑点保护：

```solidity
if (amountOut < params.minAmountOut)
    revert TooLittleReceived(amountOut);
```

### 交换回调

让我们看看更新后的交换回调：


```solidity
function uniswapV3SwapCallback(
    int256 amount0,
    int256 amount1,
    bytes calldata data_
) public {
    SwapCallbackData memory data = abi.decode(data_, (SwapCallbackData));
    (address tokenIn, address tokenOut, ) = data.path.decodeFirstPool();

    bool zeroForOne = tokenIn < tokenOut;

    int256 amount = zeroForOne ? amount0 : amount1;

    if (data.payer == address(this)) {
        IERC20(tokenIn).transfer(msg.sender, uint256(amount));
    } else {
        IERC20(tokenIn).transferFrom(
            data.payer,
            msg.sender,
            uint256(amount)
        );
    }
}
```

回调期望接收编码的`SwapCallbackData`，其中包含路径和支付者地址。它从路径中提取池代币，确定交换方向（`zeroForOne`），以及合约需要转出的金额。然后，根据支付者地址的不同，它会有不同的行为：

1. 如果支付者是当前合约（这在进行连续交换时会发生），它会从当前合约的余额中将代币转移到下一个池（调用此回调的池）。

2. 如果支付者是不同的地址（发起交换的用户），它会从用户的余额中转移代币。

## 更新Quoter合约

Quoter是另一个需要更新的合约，因为我们希望也用它来查找多池交换中的输出金额。与Manager类似，我们将有两种`quote`函数变体：单池和多池。让我们先看看前者。

### 单池报价

我们只需要在当前的`quote`实现中做几处更改：

1. 将其重命名为`quoteSingle`；

2. 将参数提取到一个结构体中（这主要是一个美化的改变）；

3. 在参数中，不再使用池地址，而是使用两个代币地址和一个tick间距。

```solidity
// src/UniswapV3Quoter.sol
struct QuoteSingleParams {
    address tokenIn;
    address tokenOut;
    uint24 tickSpacing;
    uint256 amountIn;
    uint160 sqrtPriceLimitX96;
}

function quoteSingle(QuoteSingleParams memory params)
    public
    returns (
        uint256 amountOut,
        uint160 sqrtPriceX96After,
        int24 tickAfter
    )
{
    ...
```

在函数主体中，我们唯一的变化是使用`getPool`来查找池地址：
```solidity
    ...
    IUniswapV3Pool pool = getPool(
        params.tokenIn,
        params.tokenOut,
        params.tickSpacing
    );

    bool zeroForOne = params.tokenIn < params.tokenOut;
    ...
```

### 多池报价

多池报价的实现类似于多池交换，但使用的参数较少。

```solidity
function quote(bytes memory path, uint256 amountIn)
    public
    returns (
        uint256 amountOut,
        uint160[] memory sqrtPriceX96AfterList,
        int24[] memory tickAfterList
    )
{
    sqrtPriceX96AfterList = new uint160[](path.numPools());
    tickAfterList = new int24[](path.numPools());
    ...
```

作为参数，我们只需要一个输入金额和一个交换路径。该函数返回与`quoteSingle`类似的值，但"交换后价格"和"交换后tick"是在每次交换后收集的，因此我们需要返回数组。

```solidity
uint256 i = 0;
while (true) {
    (address tokenIn, address tokenOut, uint24 tickSpacing) = path
        .decodeFirstPool();

    (
        uint256 amountOut_,
        uint160 sqrtPriceX96After,
        int24 tickAfter
    ) = quoteSingle(
            QuoteSingleParams({
                tokenIn: tokenIn,
                tokenOut: tokenOut,
                tickSpacing: tickSpacing,
                amountIn: amountIn,
                sqrtPriceLimitX96: 0
            })
        );

    sqrtPriceX96AfterList[i] = sqrtPriceX96After;
    tickAfterList[i] = tickAfter;
    amountIn = amountOut_;
    i++;

    if (path.hasMultiplePools()) {
        path = path.skipToken();
    } else {
        amountOut = amountIn;
        break;
    }
}
```

循环的逻辑与更新后的`swap`函数中的逻辑相同：

1. 获取当前池的参数；

2. 在当前池上调用`quoteSingle`；

3. 保存返回的值；

4. 如果路径中还有更多的池，则重复；否则返回。
