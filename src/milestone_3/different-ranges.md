# 不同的价格区间

根据我们的实现方式，我们的Pool合约只创建包含当前价格的价格区间：

```solidity
// src/UniswapV3Pool.sol
function mint() {
    ...
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

    liquidity += uint128(amount);
    ...
}
```

从这段代码中，你也可以看到我们总是更新流动性跟踪器（它只跟踪当前可用的流动性，即当前价格下可用的流动性）。

然而，在现实中，价格区间也可以在当前价格的**下方或上方**创建。是的：Uniswap V3的设计允许流动性提供者提供不会立即使用的流动性。当当前价格进入这些"休眠"价格区间时，这种流动性会被"注入"。

以下是可能存在的价格区间类型：

1. 活跃价格区间，即包含当前价格的区间。
2. 位于当前价格下方的价格区间。这个区间的上限价格刻度低于当前价格刻度。
3. 位于当前价格上方的价格区间。这个区间的下限价格刻度高于当前价格刻度。

## 限价订单

关于非活跃流动性（即不在当前价格提供的流动性）的一个有趣事实是，它充当了*限价订单*的角色。

在交易中，限价订单是当价格跨越交易者选择的水平时执行的订单。例如，你可以下一个限价订单，当ETH价格下降到1000美元时买入1个ETH。同样，你也可以使用限价订单来卖出资产。使用Uniswap V3，你可以通过在当前价格下方或上方放置流动性来获得类似的行为。让我们看看这是如何工作的：

![当前价格外的流动性区间](images/ranges_outside_current_price.png)

如果你在当前价格下方提供流动性（即你选择的价格区间完全位于当前价格下方）或上方，那么你的全部流动性将**只由一种资产**组成——这种资产将是两种资产中较便宜的那个。在我们的例子中，我们正在建立一个ETH作为代币$$x$$和USDC作为代币$$y$$的资金池，我们将价格定义为：

$$P = \frac{y}{x}$$

如果我们在当前价格下方放置流动性，那么流动性将完全由USDC组成，因为在我们添加流动性的地方，USDC的价格低于当前价格。同样，当我们在当前价格上方放置流动性时，流动性将由ETH组成，因为在那个范围内ETH更便宜。

回想一下介绍中的这个图示：

![价格区间耗尽](../milestone_1/images/range_depleted.png)

如果我们从这个区间买入所有可用的ETH，该区间将只包含另一种代币USDC，价格将移动到曲线的右侧。我们定义的价格（$$\frac{y}{x}$$）将**增加**。如果这个区间右侧有一个价格区间，它需要有ETH流动性，而且只有ETH，没有USDC：它需要为下一次交换提供ETH。如果我们继续买入并提高价格，我们可能也会"耗尽"下一个价格区间，这意味着买入所有的ETH并卖出USDC。同样，价格区间最终只剩下USDC，当前价格移出了这个区间。

类似地，如果我们买入USDC代币，我们将价格向左移动并从资金池中移除USDC代币。下一个价格区间将只包含USDC代币以满足我们的需求，并且，类似于上述情况，如果我们从中买入所有USDC，它最终将只包含ETH代币。

注意这个有趣的事实：当跨越整个价格区间时，其流动性从一种代币交换为另一种代币。如果我们设置一个非常窄的价格区间，一个在价格移动过程中快速被跨越的区间，我们就得到了一个限价订单！例如，如果你想以较低的价格买入ETH，你需要在较低的价格放置一个只包含USDC的价格区间，并等待当前价格跨越它。之后，你需要移除你的流动性，并将其转换为ETH！

我希望这个例子没有让你感到困惑！我认为这是一个很好的方式来解释价格区间的动态。

## 更新`mint`函数

为了支持所有类型的价格区间，我们需要知道当前价格是低于、位于还是高于用户指定的价格区间，并相应地计算代币数量。如果价格区间高于当前价格，我们希望流动性由代币$$x$$组成：

```solidity
// src/UniswapV3Pool.sol
function mint() {
    ...
    if (slot0_.tick < lowerTick) {
        amount0 = Math.calcAmount0Delta(
            TickMath.getSqrtRatioAtTick(lowerTick),
            TickMath.getSqrtRatioAtTick(upperTick),
            amount
        );
    ...
```

当价格区间包含当前价格时，我们希望两种代币的数量与价格成比例（这是我们之前实现的场景）：

```solidity
} else if (slot0_.tick < upperTick) {
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

    liquidity = LiquidityMath.addLiquidity(liquidity, int128(amount));
```

注意，这是我们唯一想要更新`liquidity`的场景，因为该变量跟踪的是立即可用的流动性。

在所有其他情况下，当价格区间低于当前价格时，我们希望该区间只包含代币$$y$$：

```solidity
} else {
    amount1 = Math.calcAmount1Delta(
        TickMath.getSqrtRatioAtTick(lowerTick),
        TickMath.getSqrtRatioAtTick(upperTick),
        amount
    );
}
```
就是这样！
