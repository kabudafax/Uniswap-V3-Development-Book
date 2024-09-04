# 交换费用

正如我在介绍中提到的，交换费用是Uniswap的核心机制。流动性提供者需要为他们提供的流动性获得报酬，否则他们就会将其用在其他地方。为了激励他们，每次交换时都会收取少量费用。这些费用然后按比例（与其在总池流动性中的份额成比例）分配给所有流动性提供者。

为了更好地理解费用收集和分配的机制，让我们看看它们是如何工作的。

## 如何收集交换费用

![流动性范围和费用](images/liquidity_ranges_fees.png)

交换费用只在价格范围被使用（用于交易）时才会被收集。所以我们需要跟踪价格范围边界被跨越的时刻。这是价格范围开始被使用的时候，也是我们想要开始为其收集费用的时候：

1. 当价格上升并且从左到右跨越一个tick时；

2. 当价格下降并且从右到左跨越一个tick时。

这是价格范围停止被使用的时候：

1. 当价格上升并且从右到左跨越一个tick时；

2. 当价格下降并且从左到右跨越一个tick时。

![流动性范围使用/停用](images/liquidity_range_engaged.png)

除了知道价格范围何时开始/停止使用外，我们还想跟踪每个价格范围累积了多少费用。

为了简化费用核算，Uniswap V3跟踪**1单位流动性产生的全局费用**。然后根据全局费用计算价格范围费用：从全局费用中减去价格范围外累积的费用。当跨越tick时（当交换移动价格时会跨越tick；费用在交换期间收集）会跟踪价格范围外累积的费用。采用这种方法，我们不需要在每次交换时更新每个头寸累积的费用——这使我们能够节省大量gas并使与池的交互更便宜。

让我们回顾一下，以便在继续之前有一个清晰的画面：

1. 费用由交换代币的用户支付。从输入代币中扣除一小部分并累积在池的余额中。

2. 每个池都有`feeGrowthGlobal0X128`和`feeGrowthGlobal1X128`状态变量，用于跟踪每单位流动性累积的总费用（即费用金额除以池的流动性）。

3. 请注意，此时实际头寸并未更新，以优化gas使用。

4. Tick保留了其外部累积的费用记录。当添加新头寸并激活tick（向先前空的tick添加流动性）时，tick记录了其外部累积了多少费用（按惯例，我们假设所有费用都累积在**tick以下**）。

5. 每当激活一个tick时，其外部累积的费用会更新为全局累积的费用与自上次跨越以来tick外部累积的费用之间的差额。

6. 有了知道其外部累积了多少费用的tick，我们就能计算出一个头寸内部累积了多少费用（头寸是两个tick之间的范围）。

7. 知道一个头寸内部累积了多少费用，我们就能计算流动性提供者有资格获得的费用份额。如果一个头寸没有参与交换，它内部累积的费用将为零，为这个范围提供流动性的流动性提供者将没有从中获得利润。

现在，让我们看看如何计算头寸累积的费用（第6步）。

## 计算头寸累积的费用

要计算头寸累积的总费用，我们需要考虑两种情况：当前价格在头寸内部和当前价格在头寸外部。在这两种情况下，我们都从全局收集的费用中减去头寸的下限和上限tick外部收集的费用。然而，根据当前价格，我们计算这些费用的方式不同。

当当前价格在头寸内部时，我们减去到此刻为止tick外部收集的费用：

![价格范围内外累积的费用](images/fees_inside_and_outside_price_range.png)

当当前价格在头寸外部时，我们需要在从全局收集的费用中减去它们之前更新上限或下限tick收集的费用。我们只为计算更新它们，不覆盖tick中的值，因为tick没有被跨越。

这是我们如何更新tick外部收集的费用：

$$f_{o}(i) = f_{g} - f_{o}(i)$$

tick外部收集的费用（$f_{o}(i)$）是全局收集的费用（$f_{g}$）与上次跨越时tick外部收集的费用之间的差额。我们在跨越tick时某种程度上重置了计数器。

要计算头寸内部收集的费用：

$$f_{r} = f_{g} - f_{b}(i_{l}) - f_{a}(i_{u})$$

我们从所有价格范围全局收集的费用（$f_{g}$）中减去其下限tick以下收集的费用（$f_{b}(i_{l})$）和上限tick以上收集的费用（$f_{a}(i_{u})$）。这就是我们在上面的插图中看到的。

现在，当当前价格高于下限tick时（即头寸被使用），我们不需要更新下限tick以下累积的费用，可以直接从下限tick中获取。当当前价格低于上限tick时，上限tick外部收集的费用也是如此。在其他两种情况下，我们需要考虑更新后的费用：

1. 当获取下限tick以下收集的费用，且当前价格也低于该tick时（下限tick最近没有被跨越）。

2. 当获取上限tick以上的费用，且当前价格也高于该tick时（上限tick最近没有被跨越）。

我希望这不会太令人困惑。幸运的是，我们现在知道了开始编码所需的一切！

## 累积交换费用

为了保持简单，我们将逐步将费用添加到我们的代码库中。我们将从累积交换费用开始。

### 添加所需的状态变量

我们需要做的第一件事是将费用金额参数添加到Pool中——每个池在部署期间都将配置一个固定且不可变的费用。在上一章中，我们添加了统一和简化池部署的Factory合约。所需的池参数之一是tick间距。现在，我们将用费用金额替换它，并将费用金额与tick间距关联起来：费用金额越大，tick间距越大。这是为了让低波动性池（稳定币池）有较低的费用。

让我们更新Factory：

```solidity
// src/UniswapV3Factory.sol
contract UniswapV3Factory is IUniswapV3PoolDeployer {
    ...
    mapping(uint24 => uint24) public fees; // `tickSpacings` replaced by `fees`

    constructor() {
        fees[500] = 10;
        fees[3000] = 60;
    }

    function createPool(
        address tokenX,
        address tokenY,
        uint24 fee
    ) public returns (address pool) {
        ...
        parameters = PoolParameters({
            factory: address(this),
            token0: tokenX,
            token1: tokenY,
            tickSpacing: fees[fee],
            fee: fee
        });
        ...
    }
}
```

费用金额是基点的百分之一。也就是说，1个费用单位是0.0001%，500是0.05%，3000是0.3%。

下一步是开始在Pool中累积费用。为此，我们将添加两个全局费用累积器变量：

```solidity
// src/UniswapV3Pool.sol
contract UniswapV3Pool is IUniswapV3Pool {
    ...
    uint24 public immutable fee;
    uint256 public feeGrowthGlobal0X128;
    uint256 public feeGrowthGlobal1X128;
}
```

索引为0的变量跟踪以`token0`累积的费用，索引为1的变量跟踪以`token1`累积的费用。

### 收集费用

现在我们需要更新`SwapMath.computeSwapStep`——这是我们计算交换金额的地方，也是我们将计算和扣除交换费用的地方。在函数中，我们将所有出现的`amountRemaining`替换为`amountRemainingLessFee`：

```solidity
uint256 amountRemainingLessFee = PRBMath.mulDiv(
    amountRemaining,
    1e6 - fee,
    1e6
);
```

因此，我们从输入代币金额中扣除费用，并从较小的输入金额计算输出金额。

该函数现在还返回在此步骤中收集的费用金额——根据是否达到范围的上限，计算方式有所不同：

```solidity
bool max = sqrtPriceNextX96 == sqrtPriceTargetX96;
if (!max) {
    feeAmount = amountRemaining - amountIn;
} else {
    feeAmount = Math.mulDivRoundingUp(amountIn, fee, 1e6 - fee);
}
```
当未达到上限时，当前价格范围有足够的流动性来完成交换，因此我们只需返回我们需要完成的金额与实际完成金额之间的差额。注意，这里不涉及`amountRemainingLessFee`，因为实际最终金额是在`amountIn`中计算的（它是基于可用流动性计算的）。

当达到目标价格时，我们不能从整个`amountRemaining`中扣除费用，因为当前价格范围没有足够的流动性来完成交换。因此，费用金额是从当前价格范围已完成的金额（`amountIn`）中扣除的。

在`SwapMath.computeSwapStep`返回后，我们需要更新交换累积的费用。注意，只有一个变量来跟踪它们，因为在开始交换时，我们已经知道输入代币（在交换过程中，费用只在`token0`或`token1`中收集，而不是两者都收集）：

```solidity
SwapState memory state = SwapState({
    ...
    feeGrowthGlobalX128: zeroForOne
        ? feeGrowthGlobal0X128
        : feeGrowthGlobal1X128
});

(...) = SwapMath.computeSwapStep(...);

state.feeGrowthGlobalX128 += PRBMath.mulDiv(
    step.feeAmount,
    FixedPoint128.Q128,
    state.liquidity
);
```

这里我们根据流动性的数量调整累积的费用，以便后续公平地在流动性提供者之间分配费用。

### 更新Tick中的费用跟踪器

接下来，如果在交换过程中跨越了一个tick（跨越tick意味着我们进入了一个新的价格范围），我们需要更新该tick中的费用跟踪器：

```solidity
if (state.sqrtPriceX96 == step.sqrtPriceNextX96) {
    int128 liquidityDelta = ticks.cross(
        step.nextTick,
        (
            zeroForOne
                ? state.feeGrowthGlobalX128
                : feeGrowthGlobal0X128
        ),
        (
            zeroForOne
                ? feeGrowthGlobal1X128
                : state.feeGrowthGlobalX128
        )
    );
    ...
}
```

由于我们此时还没有更新`feeGrowthGlobal0X128/feeGrowthGlobal1X128`状态变量，我们根据交换方向将`state.feeGrowthGlobalX128`作为其中一个费用参数传递。`cross`函数按照我们上面讨论的方式更新费用跟踪器：

```solidity
// src/lib/Tick.sol
function cross(
    mapping(int24 => Tick.Info) storage self,
    int24 tick,
    uint256 feeGrowthGlobal0X128,
    uint256 feeGrowthGlobal1X128
) internal returns (int128 liquidityDelta) {
    Tick.Info storage info = self[tick];
    info.feeGrowthOutside0X128 =
        feeGrowthGlobal0X128 -
        info.feeGrowthOutside0X128;
    info.feeGrowthOutside1X128 =
        feeGrowthGlobal1X128 -
        info.feeGrowthOutside1X128;
    liquidityDelta = info.liquidityNet;
}
```

> 我们还没有添加`feeGrowthOutside0X128/feeGrowthOutside1X128`变量的初始化——我们将在后面的步骤中完成这个。

### 更新全局费用跟踪器

最后，在交换完成后，我们可以更新全局费用跟踪器：

```solidity
if (zeroForOne) {
    feeGrowthGlobal0X128 = state.feeGrowthGlobalX128;
} else {
    feeGrowthGlobal1X128 = state.feeGrowthGlobalX128;
}
```

再次强调，在交换过程中，只有其中一个会被更新，因为费用是从输入代币中收取的，根据交换方向，这可能是`token0`或`token1`中的任一个。

这就是关于交换的全部内容！现在让我们看看当添加流动性时，费用会发生什么变化。

## 头寸管理中的费用跟踪

当添加或移除流动性时（我们还没有实现后者），我们也需要初始化或更新费用。费用需要在tick（tick外部累积的费用——我们刚刚添加的`feeGrowthOutside`变量）和头寸（头寸内部累积的费用）中都进行跟踪。对于头寸，我们还需要跟踪和更新作为费用收集的代币数量——换句话说，我们将每单位流动性的费用转换为代币数量。后者是必要的，这样当流动性提供者移除流动性时，他们可以获得作为交换费用收集的额外代币。

让我们再次逐步进行。

### Tick中费用跟踪器的初始化

在`Tick.update`函数中，每当初始化一个tick（向先前空的tick添加流动性）时，我们都会初始化其费用跟踪器。然而，我们只在tick低于当前价格时这样做，即当它在当前价格范围内时：

```solidity
// src/lib/Tick.sol
function update(
    mapping(int24 => Tick.Info) storage self,
    int24 tick,
    int24 currentTick,
    int128 liquidityDelta,
    uint256 feeGrowthGlobal0X128,
    uint256 feeGrowthGlobal1X128,
    bool upper
) internal returns (bool flipped) {
    ...
    if (liquidityBefore == 0) {
        // by convention, assume that all previous fees were collected below
        // the tick
        if (tick <= currentTick) {
            tickInfo.feeGrowthOutside0X128 = feeGrowthGlobal0X128;
            tickInfo.feeGrowthOutside1X128 = feeGrowthGlobal1X128;
        }

        tickInfo.initialized = true;
    }
    ...
}
```

如果它不在当前价格范围内，其费用跟踪器将为0，并且它们将在下次跨越该tick时更新（参见我们上面更新的`cross`函数）。

### 更新头寸费用和代币数量

下一步是计算头寸累积的费用和代币。由于头寸是两个tick之间的范围，我们将使用我们在上一步中添加到tick的费用跟踪器来计算这些值。下一个函数可能看起来有些混乱，但它实现了我们之前看到的精确的价格范围费用公式：

```solidity
// src/lib/Tick.sol
function getFeeGrowthInside(
    mapping(int24 => Tick.Info) storage self,
    int24 lowerTick_,
    int24 upperTick_,
    int24 currentTick,
    uint256 feeGrowthGlobal0X128,
    uint256 feeGrowthGlobal1X128
)
    internal
    view
    returns (uint256 feeGrowthInside0X128, uint256 feeGrowthInside1X128)
{
    Tick.Info storage lowerTick = self[lowerTick_];
    Tick.Info storage upperTick = self[upperTick_];

    uint256 feeGrowthBelow0X128;
    uint256 feeGrowthBelow1X128;
    if (currentTick >= lowerTick_) {
        feeGrowthBelow0X128 = lowerTick.feeGrowthOutside0X128;
        feeGrowthBelow1X128 = lowerTick.feeGrowthOutside1X128;
    } else {
        feeGrowthBelow0X128 =
            feeGrowthGlobal0X128 -
            lowerTick.feeGrowthOutside0X128;
        feeGrowthBelow1X128 =
            feeGrowthGlobal1X128 -
            lowerTick.feeGrowthOutside1X128;
    }

    uint256 feeGrowthAbove0X128;
    uint256 feeGrowthAbove1X128;
    if (currentTick < upperTick_) {
        feeGrowthAbove0X128 = upperTick.feeGrowthOutside0X128;
        feeGrowthAbove1X128 = upperTick.feeGrowthOutside1X128;
    } else {
        feeGrowthAbove0X128 =
            feeGrowthGlobal0X128 -
            upperTick.feeGrowthOutside0X128;
        feeGrowthAbove1X128 =
            feeGrowthGlobal1X128 -
            upperTick.feeGrowthOutside1X128;
    }

    feeGrowthInside0X128 =
        feeGrowthGlobal0X128 -
        feeGrowthBelow0X128 -
        feeGrowthAbove0X128;
    feeGrowthInside1X128 =
        feeGrowthGlobal1X128 -
        feeGrowthBelow1X128 -
        feeGrowthAbove1X128;
}
```

在这里，我们计算两个tick之间（价格范围内）累积的费用。为此，我们首先计算下限tick以下累积的费用，然后计算上限tick以上累积的费用。最后，我们从全局累积的费用中减去这些费用。这就是我们之前看到的公式：

$$f_{r} = f_{g} - f_{b}(i_{l}) - f_{a}(i_{u})$$

在计算tick上方和下方收集的费用时，我们根据价格范围是否被使用（当前价格是否在价格范围的边界tick之间）采用不同的方法。当它被使用时，我们简单地使用tick的当前费用跟踪器；当它未被使用时，我们需要使用tick的更新后的费用跟踪器——你可以在上面代码的两个`else`分支中看到这些计算。

在找到头寸内累积的费用后，我们就可以更新头寸的费用和代币数量跟踪器了：

```solidity
// src/lib/Position.sol
function update(
    Info storage self,
    int128 liquidityDelta,
    uint256 feeGrowthInside0X128,
    uint256 feeGrowthInside1X128
) internal {
    uint128 tokensOwed0 = uint128(
        PRBMath.mulDiv(
            feeGrowthInside0X128 - self.feeGrowthInside0LastX128,
            self.liquidity,
            FixedPoint128.Q128
        )
    );
    uint128 tokensOwed1 = uint128(
        PRBMath.mulDiv(
            feeGrowthInside1X128 - self.feeGrowthInside1LastX128,
            self.liquidity,
            FixedPoint128.Q128
        )
    );

    self.liquidity = LiquidityMath.addLiquidity(
        self.liquidity,
        liquidityDelta
    );
    self.feeGrowthInside0LastX128 = feeGrowthInside0X128;
    self.feeGrowthInside1LastX128 = feeGrowthInside1X128;

    if (tokensOwed0 > 0 || tokensOwed1 > 0) {
        self.tokensOwed0 += tokensOwed0;
        self.tokensOwed1 += tokensOwed1;
    }
}
```

在计算欠付的代币时，我们将头寸累积的费用乘以流动性——这与我们在交换过程中所做的相反。最后，我们更新费用跟踪器，并将代币数量添加到先前跟踪的数量中。

现在，每当修改头寸（在添加或移除流动性期间），我们都会计算头寸收集的费用并更新头寸：

```solidity
// src/UniswapV3Pool.sol
function mint(...) {
    ...
    bool flippedLower = ticks.update(params.lowerTick, ...);
    bool flippedUpper = ticks.update(params.upperTick, ...);
    ...
    (uint256 feeGrowthInside0X128, uint256 feeGrowthInside1X128) = ticks
        .getFeeGrowthInside(
            params.lowerTick,
            params.upperTick,
            slot0_.tick,
            feeGrowthGlobal0X128_,
            feeGrowthGlobal1X128_
        );

    position.update(
        params.liquidityDelta,
        feeGrowthInside0X128,
        feeGrowthInside1X128
    );
    ...
}
```

## 移除流动性

我们现在准备添加我们尚未实现的唯一核心功能——移除流动性。与铸造相对，我们将这个函数称为`burn`。这个函数将允许流动性提供者从他们之前添加流动性的头寸中移除部分或全部流动性。除此之外，它还将计算流动性提供者有资格获得的费用代币。然而，实际的代币转移将在一个单独的函数——`collect`中完成。

### 销毁流动性

销毁流动性与铸造相对。我们当前的设计和实现使这成为一个无麻烦的任务：销毁流动性简单地说就是带负号的铸造。它就像添加一个负数量的流动性。

> 为了实现`burn`，我需要重构代码并将所有与头寸管理相关的内容（更新tick和头寸，以及代币数量计算）提取到`_modifyPosition`函数中，该函数被`mint`和`burn`函数共同使用。

```solidity
function burn(
    int24 lowerTick,
    int24 upperTick,
    uint128 amount
) public returns (uint256 amount0, uint256 amount1) {
    (
        Position.Info storage position,
        int256 amount0Int,
        int256 amount1Int
    ) = _modifyPosition(
            ModifyPositionParams({
                owner: msg.sender,
                lowerTick: lowerTick,
                upperTick: upperTick,
                liquidityDelta: -(int128(amount))
            })
        );

    amount0 = uint256(-amount0Int);
    amount1 = uint256(-amount1Int);

    if (amount0 > 0 || amount1 > 0) {
        (position.tokensOwed0, position.tokensOwed1) = (
            position.tokensOwed0 + uint128(amount0),
            position.tokensOwed1 + uint128(amount1)
        );
    }

    emit Burn(msg.sender, lowerTick, upperTick, amount, amount0, amount1);
}
```

在`burn`函数中，我们首先更新一个头寸并从中移除一定数量的流动性。然后，我们更新头寸所欠的代币数量——它们现在包括通过费用累积的数量以及之前作为流动性提供的数量。我们也可以将此视为将头寸流动性转换为头寸所欠的代币数量——这些数量不再用作流动性，可以通过调用`collect`函数自由赎回：

```solidity
function collect(
    address recipient,
    int24 lowerTick,
    int24 upperTick,
    uint128 amount0Requested,
    uint128 amount1Requested
) public returns (uint128 amount0, uint128 amount1) {
    Position.Info storage position = positions.get(
        msg.sender,
        lowerTick,
        upperTick
    );

    amount0 = amount0Requested > position.tokensOwed0
        ? position.tokensOwed0
        : amount0Requested;
    amount1 = amount1Requested > position.tokensOwed1
        ? position.tokensOwed1
        : amount1Requested;

    if (amount0 > 0) {
        position.tokensOwed0 -= amount0;
        IERC20(token0).transfer(recipient, amount0);
    }

    if (amount1 > 0) {
        position.tokensOwed1 -= amount1;
        IERC20(token1).transfer(recipient, amount1);
    }

    emit Collect(
        msg.sender,
        recipient,
        lowerTick,
        upperTick,
        amount0,
        amount1
    );
}
```

这个函数简单地从池中转移代币，并确保只能转移有效的数量（一个人不能转出超过他们销毁的数量加上他们赚取的费用）。

还有一种方法可以只收集费用而不销毁流动性：销毁0数量的流动性，然后调用`collect`。在销毁过程中，头寸将被更新，它所欠的代币数量也将被更新。

就是这样！我们的池实现现在完成了！
