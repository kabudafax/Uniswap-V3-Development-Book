# 通用交换

这将是本里程碑中最困难的章节。在更新代码之前，我们需要理解Uniswap V3中交换算法的工作原理。

你可以将交换视为填充订单：用户向池子提交一个订单，以购买指定数量的代币。池子将使用可用的流动性将输入数量"转换"为另一种代币的输出数量。如果当前价格范围内没有足够的流动性，它将尝试在其他价格范围内寻找流动性（使用我们在上一章实现的函数）。

我们现在将在`swap`函数中实现这个逻辑，但目前仅限于当前价格范围内——我们将在下一个里程碑中实现跨tick交换。

```solidity
function swap(
    address recipient,
    bool zeroForOne,
    uint256 amountSpecified,
    bytes calldata data
) public returns (int256 amount0, int256 amount1) {
    ...
```

在`swap`函数中，我们添加了两个新参数：`zeroForOne`和`amountSpecified`。`zeroForOne`是控制交换方向的标志：当为`true`时，`token0`被交换成`token1`；当为`false`时，则相反。例如，如果`token0`是ETH，`token1`是USDC，将`zeroForOne`设置为`true`意味着用ETH购买USDC。`amountSpecified`是用户想要出售的代币数量。

## 填充订单

由于在Uniswap V3中，流动性存储在多个价格范围内，Pool合约需要找到所有满足用户"填充订单"所需的流动性。这是通过按用户选择的方向迭代已初始化的ticks来完成的。

在继续之前，我们需要定义两个新的结构：
```solidity
struct SwapState {
    uint256 amountSpecifiedRemaining;
    uint256 amountCalculated;
    uint160 sqrtPriceX96;
    int24 tick;
}

struct StepState {
    uint160 sqrtPriceStartX96;
    int24 nextTick;
    uint160 sqrtPriceNextX96;
    uint256 amountIn;
    uint256 amountOut;
}
```

`SwapState`维护当前交换的状态。`amountSpecifiedRemaining`跟踪池子需要购买的剩余代币数量。当它为零时，交换完成。`amountCalculated`是合约计算的输出数量。`sqrtPriceX96`和`tick`是交换完成后的新当前价格和tick。

`StepState`维护当前交换步骤的状态。这个结构跟踪"填充订单"的**一次迭代**的状态。`sqrtPriceStartX96`跟踪迭代开始时的价格。`nextTick`是将为交换提供流动性的下一个已初始化tick，`sqrtPriceNextX96`是下一个tick的价格。`amountIn`和`amountOut`是当前迭代的流动性可以提供的数量。

> 在我们实现跨tick交换（即发生在多个价格范围内的交换）之后，迭代的概念将会更加清晰。

```solidity
// src/UniswapV3Pool.sol

function swap(...) {
    Slot0 memory slot0_ = slot0;

    SwapState memory state = SwapState({
        amountSpecifiedRemaining: amountSpecified,
        amountCalculated: 0,
        sqrtPriceX96: slot0_.sqrtPriceX96,
        tick: slot0_.tick
    });
    ...
```

在填充订单之前，我们初始化一个`SwapState`实例。我们将循环直到`amountSpecifiedRemaining`为0，这意味着池子有足够的流动性从用户那里购买`amountSpecified`数量的代币。
```solidity
...
while (state.amountSpecifiedRemaining > 0) {
    StepState memory step;

    step.sqrtPriceStartX96 = state.sqrtPriceX96;

    (step.nextTick, ) = tickBitmap.nextInitializedTickWithinOneWord(
        state.tick,
        1,
        zeroForOne
    );

    step.sqrtPriceNextX96 = TickMath.getSqrtRatioAtTick(step.nextTick);
```

在循环中，我们设置一个应该为交换提供流动性的价格范围。这个范围从`state.sqrtPriceX96`到`step.sqrtPriceNextX96`，其中后者是下一个已初始化tick的价格（由`nextInitializedTickWithinOneWord`返回——我们在前一章节中了解了这个函数）。

```solidity
(state.sqrtPriceX96, step.amountIn, step.amountOut) = SwapMath
    .computeSwapStep(
        state.sqrtPriceX96,
        step.sqrtPriceNextX96,
        liquidity,
        state.amountSpecifiedRemaining
    );
```

接下来，我们计算当前价格范围可以提供的数量，以及交换将导致的新的当前价格。

```solidity
    state.amountSpecifiedRemaining -= step.amountIn;
    state.amountCalculated += step.amountOut;
    state.tick = TickMath.getTickAtSqrtRatio(state.sqrtPriceX96);
}
```

循环的最后一步是更新SwapState。`step.amountIn`是价格范围可以从用户那里购买的代币数量；`step.amountOut`是池子可以卖给用户的相关的另一种代币的数量。`state.sqrtPriceX96`是交换后将设置的当前价格（回想一下，交易会改变当前价格）。

## SwapMath合约

让我们仔细看看`SwapMath.computeSwapStep`。

```solidity
// src/lib/SwapMath.sol
function computeSwapStep(
    uint160 sqrtPriceCurrentX96,
    uint160 sqrtPriceTargetX96,
    uint128 liquidity,
    uint256 amountRemaining
)
    internal
    pure
    returns (
        uint160 sqrtPriceNextX96,
        uint256 amountIn,
        uint256 amountOut
    )
{
    ...
```

这是交换的核心逻辑。该函数在一个价格范围内计算交换数量，并考虑可用的流动性。它将返回：新的当前价格以及输入和输出代币数量。尽管输入数量是由用户提供的，我们仍然计算它，以了解一次`computeSwapStep`调用处理了用户指定输入数量的多少。

```solidity
bool zeroForOne = sqrtPriceCurrentX96 >= sqrtPriceTargetX96;

sqrtPriceNextX96 = Math.getNextSqrtPriceFromInput(
    sqrtPriceCurrentX96,
    liquidity,
    amountRemaining,
    zeroForOne
);
```

通过检查价格，我们可以确定交换的方向。知道方向后，我们可以计算交换`amountRemaining`代币后的价格。我们稍后会回到这个函数。

在找到新价格后，我们可以使用我们已有的函数计算交换的输入和输出数量（这些函数与我们在`mint`函数中用于从流动性计算代币数量的函数相同）：
```solidity
amountIn = Math.calcAmount0Delta(
    sqrtPriceCurrentX96,
    sqrtPriceNextX96,
    liquidity
);
amountOut = Math.calcAmount1Delta(
    sqrtPriceCurrentX96,
    sqrtPriceNextX96,
    liquidity
);
```

如果方向相反，则交换这些数量：
```solidity
if (!zeroForOne) {
    (amountIn, amountOut) = (amountOut, amountIn);
}
```

这就是`computeSwapStep`的全部内容！

## 通过交换数量找到价格

现在让我们看看`Math.getNextSqrtPriceFromInput`——这个函数根据另一个$\sqrt{P}$、流动性和输入数量计算$\sqrt{P}$。它告诉我们在给定当前价格和流动性的情况下，交换指定输入数量的代币后价格将会是多少。

好消息是我们已经知道这些公式：回想一下我们在Python中是如何计算`price_next`的：

```python
# When amount_in is token0
price_next = int((liq * q96 * sqrtp_cur) // (liq * q96 + amount_in * sqrtp_cur))
# When amount_in is token1
price_next = sqrtp_cur + (amount_in * q96) // liq
```

我们将在Solidity中实现这个：
```solidity
// src/lib/Math.sol
function getNextSqrtPriceFromInput(
    uint160 sqrtPriceX96,
    uint128 liquidity,
    uint256 amountIn,
    bool zeroForOne
) internal pure returns (uint160 sqrtPriceNextX96) {
    sqrtPriceNextX96 = zeroForOne
        ? getNextSqrtPriceFromAmount0RoundingUp(
            sqrtPriceX96,
            liquidity,
            amountIn
        )
        : getNextSqrtPriceFromAmount1RoundingDown(
            sqrtPriceX96,
            liquidity,
            amountIn
        );
}
```

这个函数处理两个方向的交换。由于计算方法不同，我们将在单独的函数中实现它们。
```solidity
function getNextSqrtPriceFromAmount0RoundingUp(
    uint160 sqrtPriceX96,
    uint128 liquidity,
    uint256 amountIn
) internal pure returns (uint160) {
    uint256 numerator = uint256(liquidity) << FixedPoint96.RESOLUTION;
    uint256 product = amountIn * sqrtPriceX96;

    if (product / amountIn == sqrtPriceX96) {
        uint256 denominator = numerator + product;
        if (denominator >= numerator) {
            return
                uint160(
                    mulDivRoundingUp(numerator, sqrtPriceX96, denominator)
                );
        }
    }

    return
        uint160(
            divRoundingUp(numerator, (numerator / sqrtPriceX96) + amountIn)
        );
}
```
在这个函数中，我们实现了两个公式。在第一个`return`处，它实现了我们在Python中实现的相同公式。这是最精确的公式，但在将`amountIn`乘以`sqrtPriceX96`时可能会溢出。这个公式是（我们在"输出数量计算"中讨论过）：

$$\sqrt{P_{target}} = \frac{\sqrt{P}L}{\Delta x \sqrt{P} + L}$$

当它溢出时，我们使用一个替代公式，这个公式精度较低：

$$\sqrt{P_{target}} = \frac{L}{\Delta x + \frac{L}{\sqrt{P}}}$$

这实际上就是将前一个公式的分子和分母都除以$\sqrt{P}$，以消除分子中的乘法。

另一个函数的数学计算更简单：

```solidity
function getNextSqrtPriceFromAmount1RoundingDown(
    uint160 sqrtPriceX96,
    uint128 liquidity,
    uint256 amountIn
) internal pure returns (uint160) {
    return
        sqrtPriceX96 +
        uint160((amountIn << FixedPoint96.RESOLUTION) / liquidity);
}
```
## 完成交换

现在，让我们回到`swap`函数并完成它。

到目前为止，我们已经循环遍历了下一个初始化的ticks，填充了用户指定的`amountSpecified`，计算了输入和输出数量，并找到了新的价格和tick。由于在这个里程碑中，我们只实现一个价格范围内的交换，这就足够了。现在我们需要更新合约的状态，向用户发送代币，并获取交换的代币。
```solidity
if (state.tick != slot0_.tick) {
    (slot0.sqrtPriceX96, slot0.tick) = (state.sqrtPriceX96, state.tick);
}
```

首先，我们设置新的价格和tick。由于这个操作会写入合约的存储，为了优化gas消耗，我们只在新的tick不同时才执行这个操作。

```solidity
(amount0, amount1) = zeroForOne
    ? (
        int256(amountSpecified - state.amountSpecifiedRemaining),
        -int256(state.amountCalculated)
    )
    : (
        -int256(state.amountCalculated),
        int256(amountSpecified - state.amountSpecifiedRemaining)
    );
```

接下来，我们根据交换方向和在交换循环中计算的数量来计算交换金额。

```solidity
if (zeroForOne) {
    IERC20(token1).transfer(recipient, uint256(-amount1));

    uint256 balance0Before = balance0();
    IUniswapV3SwapCallback(msg.sender).uniswapV3SwapCallback(
        amount0,
        amount1,
        data
    );
    if (balance0Before + uint256(amount0) > balance0())
        revert InsufficientInputAmount();
} else {
    IERC20(token0).transfer(recipient, uint256(-amount0));

    uint256 balance1Before = balance1();
    IUniswapV3SwapCallback(msg.sender).uniswapV3SwapCallback(
        amount0,
        amount1,
        data
    );
    if (balance1Before + uint256(amount1) > balance1())
        revert InsufficientInputAmount();
}
```

接下来，我们根据交换方向与用户交换代币。这部分与里程碑2中的内容相同，只是增加了处理另一个交换方向的逻辑。

就是这样！交换完成了！

## 测试

测试不会有太大变化，我们只需要将`amountSpecified`和`zeroForOne`传递给`swap`函数。不过，输出数量会有微小的变化，因为现在是在Solidity中计算的。

我们现在可以测试相反方向的交换了！我将把这个作为作业留给你（只需确保选择一个小的输入数量，以便我们的单一价格范围可以处理整个交换）。如果感到困难，不要犹豫查看[我的测试](https://github.com/Jeiwan/uniswapv3-code/blob/milestone_2/test/UniswapV3Pool.t.sol)！
