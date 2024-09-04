# Quoter合约

为了将我们更新后的Pool合约集成到前端应用中，我们需要一种方法来计算交换金额，而不实际进行交换。用户将输入他们想要出售的金额，我们希望计算并向他们展示他们将获得的交换金额。我们将通过Quoter合约来实现这一点。

由于Uniswap V3中的流动性分散在多个价格范围内，我们无法使用公式计算交换金额（这在Uniswap V2中是可能的）。Uniswap V3的设计迫使我们使用不同的方法：为了计算交换金额，我们将启动一个真实的交换，并在回调函数中中断它，获取Pool合约计算的金额。也就是说，我们必须模拟一个真实的交换来计算输出金额！

再次，我们将为此制作一个辅助合约：

```solidity
contract UniswapV3Quoter {
    struct QuoteParams {
        address pool;
        uint256 amountIn;
        bool zeroForOne;
    }

    function quote(QuoteParams memory params)
        public
        returns (
            uint256 amountOut,
            uint160 sqrtPriceX96After,
            int24 tickAfter
        )
    {
        ...
```

Quoter是一个只实现了一个公共函数——`quote`的合约。Quoter是一个通用合约，可以与任何池子配合使用，所以它将池子地址作为参数。其他参数（`amountIn`和`zeroForOne`）是模拟交换所必需的。

```solidity
try
    IUniswapV3Pool(params.pool).swap(
        address(this),
        params.zeroForOne,
        params.amountIn,
        abi.encode(params.pool)
    )
{} catch (bytes memory reason) {
    return abi.decode(reason, (uint256, uint160, int24));
}
```

这个合约唯一做的事情就是调用池子的`swap`函数。预期这个调用会回滚（即抛出错误）——我们将在交换回调中执行这个操作。在回滚的情况下，回滚原因会被解码并返回；`quote`永远不会回滚。注意，在额外数据中，我们只传递了池子地址——在交换回调中，我们将使用它来获取交换后池子的`slot0`。
```solidity
function uniswapV3SwapCallback(
    int256 amount0Delta,
    int256 amount1Delta,
    bytes memory data
) external view {
    address pool = abi.decode(data, (address));

    uint256 amountOut = amount0Delta > 0
        ? uint256(-amount1Delta)
        : uint256(-amount0Delta);

    (uint160 sqrtPriceX96After, int24 tickAfter) = IUniswapV3Pool(pool)
        .slot0();
```

在交换回调中，我们收集我们需要的值：输出数量、新价格和相应的tick。接下来，我们需要保存这些值并回滚：
```solidity
assembly {
    let ptr := mload(0x40)
    mstore(ptr, amountOut)
    mstore(add(ptr, 0x20), sqrtPriceX96After)
    mstore(add(ptr, 0x40), tickAfter)
    revert(ptr, 96)
}
```

为了优化gas，这部分是用[Yul](https://docs.soliditylang.org/en/latest/assembly.html)实现的，Yul是Solidity中用于内联汇编的语言。让我们来分解一下：

1. `mload(0x40)`读取下一个可用内存槽的指针（EVM中的内存以32字节为一个槽组织）；

2. 在那个内存槽，`mstore(ptr, amountOut)`写入`amountOut`；

3. `mstore(add(ptr, 0x20), sqrtPriceX96After)`在`amountOut`之后写入`sqrtPriceX96After`；

4. `mstore(add(ptr, 0x40), tickAfter)`在`sqrtPriceX96After`之后写入`tickAfter`；

5. `revert(ptr, 96)`回滚调用并返回地址`ptr`（我们上面写入的数据的开始）处的96字节数据（我们写入内存的值的总长度）。

所以，我们正在连接我们需要的值的字节表示（正是`abi.encode()`所做的）。注意，偏移量始终是32字节，即使`sqrtPriceX96After`占20字节（`uint160`）而`tickAfter`占3字节（`int24`）。这是为了我们可以使用`abi.decode()`来解码数据：它的对应物`abi.encode()`将所有整数编码为32字节的字。

然后...完成了。

## 回顾

让我们回顾一下以更好地理解算法：

1. `quote`调用池子的`swap`，带有输入数量和交换方向；

2. `swap`执行真实的交换，它运行循环来填充用户指定的输入数量；

3. 为了从用户那里获取代币，`swap`在调用者上调用交换回调；

4. 调用者（Quote合约）实现回调，在其中它以输出数量、新价格和新tick回滚；

5. 回滚冒泡到初始的`quote`调用；

6. 在`quote`中，捕获回滚，解码回滚原因并作为调用`quote`的结果返回。

我希望这是清楚的！

## Quoter的限制

这个设计有一个重要的限制：由于`quote`调用Pool合约的`swap`函数，而`swap`函数不是纯函数或视图函数（因为它修改合约状态），`quote`也不能是纯函数或视图函数。`swap`修改状态，`quote`也是如此，即使不是在Quoter合约中。但我们将`quote`视为一个getter，一个只读取合约数据的函数。这种不一致意味着当调用`quote`时，EVM将使用[CALL](https://www.evm.codes/#f1)操作码而不是[STATICCALL](https://www.evm.codes/#fa)。这不是一个大问题，因为Quoter在交换回调中回滚，而回滚会重置调用期间修改的状态——这保证了`quote`不会修改Pool合约的状态（不会发生实际交易）。

这个问题带来的另一个不便是，从客户端库（Ethers.js、Web3.js等）调用`quote`将触发一个交易。为了解决这个问题，我们需要强制库进行静态调用。我们将在本里程碑的后面看到如何在Ethers.js中做到这一点。
