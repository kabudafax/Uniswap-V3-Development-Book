# Tick舍入

让我们回顾一下为支持不同的tick间距需要做的其他更改。

大于1的tick间距不允许用户选择任意价格范围：tick索引必须是tick间距的倍数。例如，对于tick间距60，我们可以有以下tick：0、60、120、180等。因此，当用户选择一个范围时，我们需要将其"舍入"，使其边界成为池子tick间距的倍数。

## JavaScript中的`nearestUsableTick`

在[Uniswap V3 SDK](https://github.com/Uniswap/v3-sdk)中，执行此操作的函数称为[nearestUsableTick](https://github.com/Uniswap/v3-sdk/blob/b6cd73a71f8f8ec6c40c130564d3aff12c38e693/src/utils/nearestUsableTick.ts)：

```javascript
/**
 * Returns the closest tick that is nearest a given tick and usable for the given tick spacing
 * @param tick the target tick
 * @param tickSpacing the spacing of the pool
 */
export function nearestUsableTick(tick: number, tickSpacing: number) {
  invariant(Number.isInteger(tick) && Number.isInteger(tickSpacing), 'INTEGERS')
  invariant(tickSpacing > 0, 'TICK_SPACING')
  invariant(tick >= TickMath.MIN_TICK && tick <= TickMath.MAX_TICK, 'TICK_BOUND')
  const rounded = Math.round(tick / tickSpacing) * tickSpacing
  if (rounded < TickMath.MIN_TICK) return rounded + tickSpacing
  else if (rounded > TickMath.MAX_TICK) return rounded - tickSpacing
  else return rounded
}
```

从本质上讲，它只是：
```javascript
Math.round(tick / tickSpacing) * tickSpacing
```

其中`Math.round`是舍入到最接近的整数：当小数部分小于0.5时，它向下舍入到较小的整数；当大于0.5时，它向上舍入到较大的整数；当等于0.5时，它也向上舍入到较大的整数。

因此，在网页应用中，我们在构建`mint`参数时将使用`nearestUsableTick`：

```javascript
const mintParams = {
  tokenA: pair.token0.address,
  tokenB: pair.token1.address,
  tickSpacing: pair.tickSpacing,
  lowerTick: nearestUsableTick(lowerTick, pair.tickSpacing),
  upperTick: nearestUsableTick(upperTick, pair.tickSpacing),
  amount0Desired, amount1Desired, amount0Min, amount1Min
}
```

> 实际上，每当用户调整价格范围时都应该调用它，因为我们希望用户看到将要创建的实际价格。在我们简化的应用中，我们使其不那么用户友好。

然而，我们还希望在Solidity测试中有一个类似的函数，但我们使用的数学库都没有实现它。

## Solidity中的`nearestUsableTick`

在我们的智能合约测试中，我们需要一种方法来舍入tick并将舍入后的价格转换为$\sqrt{P}$。在前面的章节中，我们选择使用[ABDKMath64x64](https://github.com/abdk-consulting/abdk-libraries-solidity)来处理测试中的定点数学运算。然而，该库并没有实现我们需要移植`nearestUsableTick`的舍入函数，所以我们需要自己实现它：

```solidity
function divRound(int128 x, int128 y)
    internal
    pure
    returns (int128 result)
{
    int128 quot = ABDKMath64x64.div(x, y);
    result = quot >> 64;

    // Check if remainder is greater than 0.5
    if (quot % 2**64 >= 0x8000000000000000) {
        result += 1;
    }
}
```

该函数执行多个操作：

1. 它将两个Q64.64数相除；

2. 然后将结果舍入到小数点前一位（`result = quot >> 64`），此时小数部分丢失（即结果向下舍入）；

3. 然后将商除以$2^{64}$，取余数，并与`0x8000000000000000`（在Q64.64中为0.5）进行比较；

4. 如果余数大于或等于0.5，则将结果向上舍入到较大的整数。

我们得到的是根据JavaScript中`Math.round`规则舍入的整数。然后我们可以重新实现`nearestUsableTick`：

```solidity
function nearestUsableTick(int24 tick_, uint24 tickSpacing)
    internal
    pure
    returns (int24 result)
{
    result =
        int24(divRound(int128(tick_), int128(int24(tickSpacing)))) *
        int24(tickSpacing);

    if (result < TickMath.MIN_TICK) {
        result += int24(tickSpacing);
    } else if (result > TickMath.MAX_TICK) {
        result -= int24(tickSpacing);
    }
}
```
就是这样！
