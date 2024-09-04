# 流动性计算

在Uniswap V3的所有数学计算中，我们还没有在Solidity中实现的是流动性计算。在Python脚本中，我们有这些函数：

```python
def liquidity0(amount, pa, pb):
    if pa > pb:
        pa, pb = pb, pa
    return (amount * (pa * pb) / q96) / (pb - pa)


def liquidity1(amount, pa, pb):
    if pa > pb:
        pa, pb = pb, pa
    return amount * q96 / (pb - pa)
```

让我们在Solidity中实现它们，这样我们就可以在`Manager.mint()`函数中计算流动性。

## 实现代币X的流动性计算

我们将要实现的函数允许我们在已知代币数量和价格范围的情况下计算流动性（$$L = \sqrt{xy}$$）。幸运的是，我们已经知道所有的公式。让我们回顾一下这个：

$$\Delta x = \Delta \frac{1}{\sqrt{P}}L$$

在前一章中，我们使用这个公式来计算交换数量（在这种情况下是$$\Delta x$$），现在我们将使用它来找到$$L$$：

$$L = \frac{\Delta x}{\Delta \frac{1}{\sqrt{P}}}$$

或者，简化后：

$$L = \frac{\Delta x \sqrt{P_u} \sqrt{P_l}}{\sqrt{P_u} - \sqrt{P_l}}$$

> 我们在[流动性数量计算](https://uniswapv3book.com/docs/milestone_1/calculating-liquidity/#liquidity-amount-calculation)中推导了这个公式。

在Solidity中，我们将再次使用`PRBMath`来处理乘法和除法时的溢出：

```solidity
function getLiquidityForAmount0(
    uint160 sqrtPriceAX96,
    uint160 sqrtPriceBX96,
    uint256 amount0
) internal pure returns (uint128 liquidity) {
    if (sqrtPriceAX96 > sqrtPriceBX96)
        (sqrtPriceAX96, sqrtPriceBX96) = (sqrtPriceBX96, sqrtPriceAX96);

    uint256 intermediate = PRBMath.mulDiv(
        sqrtPriceAX96,
        sqrtPriceBX96,
        FixedPoint96.Q96
    );
    liquidity = uint128(
        PRBMath.mulDiv(amount0, intermediate, sqrtPriceBX96 - sqrtPriceAX96)
    );
}
```

## 实现代币Y的流动性计算

同样，我们将使用[流动性数量计算](https://uniswapv3book.com/docs/milestone_1/calculating-liquidity/#liquidity-amount-calculation)中的另一个公式来在已知$$y$$的数量和价格范围的情况下找到$$L$$：

$$\Delta y = \Delta\sqrt{P} L$$

$$L = \frac{\Delta y}{\sqrt{P_u}-\sqrt{P_l}}$$


```solidity
function getLiquidityForAmount1(
    uint160 sqrtPriceAX96,
    uint160 sqrtPriceBX96,
    uint256 amount1
) internal pure returns (uint128 liquidity) {
    if (sqrtPriceAX96 > sqrtPriceBX96)
        (sqrtPriceAX96, sqrtPriceBX96) = (sqrtPriceBX96, sqrtPriceAX96);

    liquidity = uint128(
        PRBMath.mulDiv(
            amount1,
            FixedPoint96.Q96,
            sqrtPriceBX96 - sqrtPriceAX96
        )
    );
}
```

我希望这很清楚！

## 寻找公平流动性

你可能会想，为什么有两种计算$$L$$的方法，而我们一直只有一个$$L$$，它是通过$$L = \sqrt{xy}$$计算的，哪种方法是正确的？答案是：它们都是正确的。

在上述公式中，我们基于不同的参数计算$$L$$：价格范围和任一代币的数量。不同的价格范围和不同的代币数量将导致不同的$$L$$值。并且有一种情况下我们需要计算这两个$$L$$并选择其中一个。回想一下`mint`函数中的这段代码：

```solidity
if (slot0_.tick < lowerTick) {
    amount0 = Math.calcAmount0Delta(...);
} else if (slot0_.tick < upperTick) {
    amount0 = Math.calcAmount0Delta(...);

    amount1 = Math.calcAmount1Delta(...);

    liquidity = LiquidityMath.addLiquidity(liquidity, int128(amount));
} else {
    amount1 = Math.calcAmount1Delta(...);
}
```

事实证明，在计算流动性时我们也需要遵循这个逻辑：

1. 如果我们正在为高于当前价格的范围计算流动性，我们使用$$\Delta x$$版本的公式；
2. 当为低于当前价格的范围计算流动性时，我们使用$$\Delta y$$的公式；
3. 当价格范围包含当前价格时，我们计算**两者**并选择较小的那个。

> 再次，我们在[流动性数量计算](https://uniswapv3book.com/docs/milestone_1/calculating-liquidity/#liquidity-amount-calculation)中讨论了这些想法。

让我们现在实现这个逻辑。

当当前价格低于价格范围的下限时：

```solidity
function getLiquidityForAmounts(
    uint160 sqrtPriceX96,
    uint160 sqrtPriceAX96,
    uint160 sqrtPriceBX96,
    uint256 amount0,
    uint256 amount1
) internal pure returns (uint128 liquidity) {
    if (sqrtPriceAX96 > sqrtPriceBX96)
        (sqrtPriceAX96, sqrtPriceBX96) = (sqrtPriceBX96, sqrtPriceAX96);

    if (sqrtPriceX96 <= sqrtPriceAX96) {
        liquidity = getLiquidityForAmount0(
            sqrtPriceAX96,
            sqrtPriceBX96,
            amount0
        );
```

当当前价格在范围内时，我们选择较小的$$L$$：

```solidity
} else if (sqrtPriceX96 <= sqrtPriceBX96) {
    uint128 liquidity0 = getLiquidityForAmount0(
        sqrtPriceX96,
        sqrtPriceBX96,
        amount0
    );
    uint128 liquidity1 = getLiquidityForAmount1(
        sqrtPriceAX96,
        sqrtPriceX96,
        amount1
    );

    liquidity = liquidity0 < liquidity1 ? liquidity0 : liquidity1;
```

最后：
```solidity
} else {
    liquidity = getLiquidityForAmount1(
        sqrtPriceAX96,
        sqrtPriceBX96,
        amount1
    );
}
```
完成。
