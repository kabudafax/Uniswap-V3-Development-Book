# 通用铸造

现在，我们准备更新`mint`函数，这样我们就不需要再硬编码值，而可以计算它们了。

## 索引已初始化的Ticks

回想一下，在`mint`函数中，我们更新TickInfo映射以存储ticks处可用流动性的信息。现在，我们还需要在位图索引中索引新初始化的ticks——我们稍后将在交换过程中使用此索引来查找下一个已初始化的tick。

首先，我们需要更新`Tick.update`函数：

```solidity
// src/lib/Tick.sol
function update(
    mapping(int24 => Tick.Info) storage self,
    int24 tick,
    uint128 liquidityDelta
) internal returns (bool flipped) {
    ...
    flipped = (liquidityAfter == 0) != (liquidityBefore == 0);
    ...
}
```

现在它返回一个`flipped`标志，当向空的tick添加流动性或从tick中移除全部流动性时，该标志会被设置为true。

然后，在`mint`函数中，我们更新位图索引：

```solidity
// src/UniswapV3Pool.sol
...
bool flippedLower = ticks.update(lowerTick, amount);
bool flippedUpper = ticks.update(upperTick, amount);

if (flippedLower) {
    tickBitmap.flipTick(lowerTick, 1);
}

if (flippedUpper) {
    tickBitmap.flipTick(upperTick, 1);
}
...
```

> 再次强调，我们将tick间距设置为1，直到我们在里程碑4中引入不同的值。

## 代币数量计算

`mint`函数中最大的变化是切换到代币数量计算。在里程碑1中，我们硬编码了这些值：

```solidity
    amount0 = 0.998976618347425280 ether;
    amount1 = 5000 ether;
```

现在我们将使用里程碑1中的公式在Solidity中计算它们。让我们回顾一下这些公式：

$$\Delta x = \frac{L(\sqrt{p(i_u)} - \sqrt{p(i_c)})}{\sqrt{p(i_u)}\sqrt{p(i_c)}}$$
$$\Delta y = L(\sqrt{p(i_c)} - \sqrt{p(i_l)})$$

$\Delta x$是`token0`或代币$x$的数量。让我们在Solidity中实现它：

```solidity
// src/lib/Math.sol
function calcAmount0Delta(
    uint160 sqrtPriceAX96,
    uint160 sqrtPriceBX96,
    uint128 liquidity
) internal pure returns (uint256 amount0) {
    if (sqrtPriceAX96 > sqrtPriceBX96)
        (sqrtPriceAX96, sqrtPriceBX96) = (sqrtPriceBX96, sqrtPriceAX96);

    require(sqrtPriceAX96 > 0);

    amount0 = divRoundingUp(
        mulDivRoundingUp(
            (uint256(liquidity) << FixedPoint96.RESOLUTION),
            (sqrtPriceBX96 - sqrtPriceAX96),
            sqrtPriceBX96
        ),
        sqrtPriceAX96
    );
}
```

> 这个函数与我们Python脚本中的`calc_amount0`完全相同。

第一步是对价格进行排序，以确保在相减时不会发生下溢。接下来，我们将`liquidity`转换为Q96.64数，方法是将其乘以2**96。然后，根据公式，我们将其乘以价格的差值，并除以较大的价格。之后，我们再除以较小的价格。除法的顺序并不重要，但我们想要进行两次除法，因为价格的乘法可能会溢出。

我们使用`mulDivRoundingUp`来在一个操作中完成乘法和除法。这个函数基于`PRBMath`中的`mulDiv`：

```solidity
function mulDivRoundingUp(
    uint256 a,
    uint256 b,
    uint256 denominator
) internal pure returns (uint256 result) {
    result = PRBMath.mulDiv(a, b, denominator);
    if (mulmod(a, b, denominator) > 0) {
        require(result < type(uint256).max);
        result++;
    }
}
```

`mulmod`是一个Solidity函数，它将两个数（`a`和`b`）相乘，将结果除以`denominator`，并返回余数。如果余数为正，我们就向上取整结果。

接下来是$\Delta y$：

```solidity
function calcAmount1Delta(
    uint160 sqrtPriceAX96,
    uint160 sqrtPriceBX96,
    uint128 liquidity
) internal pure returns (uint256 amount1) {
    if (sqrtPriceAX96 > sqrtPriceBX96)
        (sqrtPriceAX96, sqrtPriceBX96) = (sqrtPriceBX96, sqrtPriceAX96);

    amount1 = mulDivRoundingUp(
        liquidity,
        (sqrtPriceBX96 - sqrtPriceAX96),
        FixedPoint96.Q96
    );
}
```

> 这个函数与我们Python脚本中的`calc_amount1`完全相同。

同样，我们使用`mulDivRoundingUp`来避免乘法过程中的溢出。

就是这样！我们现在可以使用这些函数来计算代币数量：

```solidity
// src/UniswapV3Pool.sol
function mint(...) {
    ...
    Slot0 memory slot0_ = slot0;

    amount0 = Math.calcAmount0Delta(
        slot0_.sqrtPriceX96,
        TickMath.getSqrtRatioAtTick(upperTick),
        amount
    );

    amount1 = Math.calcAmount1Delta(
        slot0_.sqrtPriceX96,
        TickMath.getSqrtRatioAtTick(lowerTick),
        amount
    );
    ...
}
```
其他一切保持不变。你需要更新池测试中的数量，由于四舍五入，它们会略有不同。
