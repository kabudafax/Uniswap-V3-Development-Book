# 交换路径

让我们想象一下，我们只有这些池：WETH/USDC、USDC/USDT和WBTC/USDT。如果我们想将WETH换成WBTC，我们需要进行多次交换（WETH→USDC→USDT→WBTC），因为没有WETH/WBTC池。我们可以手动完成这个过程，或者我们可以改进我们的合约来处理这种链式或多池交换。当然，我们会选择后者！

在进行多池交换时，我们将前一次交换的输出发送到下一次交换的输入。例如：

1. 在WETH/USDC池中，我们卖出WETH并买入USDC；

2. 在USDC/USDT池中，我们卖出上一次交换得到的USDC并买入USDT；

3. 在WBTC/USDT池中，我们卖出上一个池中得到的USDT并买入WBTC。

我们可以将这个系列转化为一个路径：

```
WETH/USDC,USDC/USDT,WBTC/USDT
```

并在我们的合约中遍历这样的路径，以在一个交易中执行多次交换。然而，回想一下上一章，我们不需要知道池地址，相反，我们可以从池参数中推导出它们。因此，上述路径可以转化为一系列代币：

```
WETH, USDC, USDT, WBTC
```

回想一下，tick间距是另一个识别池的参数（除了代币之外）。因此，上述路径变为：
```
WETH, 60, USDC, 10, USDT, 60, WBTC
```

其中60和10是tick间距。我们在波动较大的交易对（如ETH/USDC、WBTC/USDT）中使用60，在稳定币交易对（USDC/USDT）中使用10。

现在，有了这样的路径，我们可以遍历它来为每个池构建池参数：

1. `WETH, 60, USDC`；

2. `USDC, 10, USDT`；

3. `USDT, 60, WBTC`。

知道这些参数后，我们可以使用`PoolAddress.computeAddress`（我们在上一章中实现的）来推导出池地址。

> 我们也可以在单个池内进行交换时使用这个概念：路径将只包含一个池的参数。因此，我们可以在所有交换中普遍使用交换路径。

让我们构建一个库来处理交换路径。

## Path库

在代码中，交换路径是一个字节序列。在Solidity中，可以这样构建路径：

```solidity
bytes.concat(
    bytes20(address(weth)),
    bytes3(uint24(60)),
    bytes20(address(usdc)),
    bytes3(uint24(10)),
    bytes20(address(usdt)),
    bytes3(uint24(60)),
    bytes20(address(wbtc))
);
```

它看起来像这样：
```shell
0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2 # weth address
  00003c                                   # 60
  A0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48 # usdc address
  00000a                                   # 10
  dAC17F958D2ee523a2206206994597C13D831ec7 # usdt address
  00003c                                   # 60
  2260FAC5E5542a773Aa44fBCfeDf7C193bc2C599 # wbtc address
```

这些是我们需要实现的函数：

1. 计算路径中的池数量；

2. 确定路径是否有多个池；

3. 从路径中提取第一个池的参数；

4. 在路径中进行到下一对；

5. 解码第一个池的参数。

### 计算路径中的池数量

让我们从计算路径中的池数量开始：

```solidity
// src/lib/Path.sol
library Path {
    /// @dev The length the bytes encoded address
    uint256 private constant ADDR_SIZE = 20;
    /// @dev The length the bytes encoded tick spacing
    uint256 private constant TICKSPACING_SIZE = 3;

    /// @dev The offset of a single token address + tick spacing
    uint256 private constant NEXT_OFFSET = ADDR_SIZE + TICKSPACING_SIZE;
    /// @dev The offset of an encoded pool key (tokenIn + tick spacing + tokenOut)
    uint256 private constant POP_OFFSET = NEXT_OFFSET + ADDR_SIZE;
    /// @dev The minimum length of a path that contains 2 or more pools;
    uint256 private constant MULTIPLE_POOLS_MIN_LENGTH =
        POP_OFFSET + NEXT_OFFSET;

    ...
```

我们首先定义几个常量：

1. `ADDR_SIZE`是地址的大小，20字节；

2. `TICKSPACING_SIZE`是tick间距的大小，3字节（`uint24`）；

3. `NEXT_OFFSET`是下一个代币地址的偏移量——为了得到它，我们跳过一个地址和一个tick间距；

4. `POP_OFFSET`是池键的偏移量（代币地址 + tick间距 + 代币地址）；

5. `MULTIPLE_POOLS_MIN_LENGTH`是包含2个或更多池的路径的最小长度（一组池参数 + tick间距 + 代币地址）。

要计算路径中的池数量，我们减去一个地址的大小（路径中的第一个或最后一个代币），并将剩余部分除以`NEXT_OFFSET`（地址 + tick间距）：

```solidity
function numPools(bytes memory path) internal pure returns (uint256) {
    return (path.length - ADDR_SIZE) / NEXT_OFFSET;
}
```

### 确定路径是否有多个池

要检查路径中是否有多个池，我们需要将路径的长度与`MULTIPLE_POOLS_MIN_LENGTH`进行比较：

```solidity
function hasMultiplePools(bytes memory path) internal pure returns (bool) {
    return path.length >= MULTIPLE_POOLS_MIN_LENGTH;
}
```

### 从路径中提取第一个池参数

为了实现其他函数，我们需要一个辅助库，因为Solidity没有原生的字节操作函数。具体来说，我们需要一个函数来从字节数组中提取子数组，以及几个将字节转换为`address`和`uint24`的函数。

幸运的是，有一个很棒的开源库叫做[solidity-bytes-utils](https://github.com/GNSPS/solidity-bytes-utils)。要使用这个库，我们需要在`Path`库中扩展`bytes`类型：

```solidity
library Path {
    using BytesLib for bytes;
    ...
}
```

现在我们可以实现`getFirstPool`了：
```solidity
function getFirstPool(bytes memory path)
    internal
    pure
    returns (bytes memory)
{
    return path.slice(0, POP_OFFSET);
}
```

该函数简单地返回编码为字节的第一个"代币地址 + tick间距 + 代币地址"段。

### 在路径中进行到下一对

当我们遍历路径并丢弃已处理的池时，我们将使用下一个函数。注意，我们移除的是"代币地址 + tick间距"，而不是完整的池参数，因为我们需要另一个代币地址来计算下一个池地址。

```solidity
function skipToken(bytes memory path) internal pure returns (bytes memory) {
    return path.slice(NEXT_OFFSET, path.length - NEXT_OFFSET);
}
```

### 解码第一个池参数

最后，我们需要解码路径中第一个池的参数：

```solidity
function decodeFirstPool(bytes memory path)
    internal
    pure
    returns (
        address tokenIn,
        address tokenOut,
        uint24 tickSpacing
    )
{
    tokenIn = path.toAddress(0);
    tickSpacing = path.toUint24(ADDR_SIZE);
    tokenOut = path.toAddress(NEXT_OFFSET);
}
```

不幸的是，`BytesLib`没有实现`toUint24`函数，但我们可以自己实现它！`BytesLib`有多个`toUintXX`函数，所以我们可以取其中一个并将其转换为`uint24`版本：

```solidity
library BytesLibExt {
    function toUint24(bytes memory _bytes, uint256 _start)
        internal
        pure
        returns (uint24)
    {
        require(_bytes.length >= _start + 3, "toUint24_outOfBounds");
        uint24 tempUint;

        assembly {
            tempUint := mload(add(add(_bytes, 0x3), _start))
        }

        return tempUint;
    }
}
```

我们在一个新的库合约中做这个，然后我们可以在我们的Path库中与`BytesLib`一起使用它：

```solidity
library Path {
    using BytesLib for bytes;
    using BytesLibExt for bytes;
    ...
}
```
