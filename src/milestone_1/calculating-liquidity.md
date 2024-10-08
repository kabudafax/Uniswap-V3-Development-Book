# 计算流动性

没有流动性就无法进行交易，为了进行我们的第一次交换，我们需要向池合约中注入一些流动性。以下是我们需要知道的向池合约添加流动性的信息：

1. 价格范围。作为流动性提供者，我们希望在特定的价格范围内提供流动性，它只会在这个范围内使用。

2. 流动性数量，即两种代币的数量。我们需要将这些数量的代币转移到池合约中。

在这里，我们将手动计算这些，但在后面的章节中，合约将为我们完成这项工作。让我们从价格范围开始。

## 价格范围计算

回想一下，在Uniswap V3中，整个价格范围被划分为刻度：每个刻度对应一个价格并有一个索引。在我们的第一个池实现中，我们将以每1 ETH 5000美元的价格用USDC购买ETH。购买ETH将从池中移除一定数量的ETH，并将价格略微推高到5000美元以上。我们希望在包含这个价格的范围内提供流动性。并且我们希望确保最终价格保持在**这个范围内**（我们将在后面的里程碑中进行多范围交换）。

我们需要找到三个刻度：

1. 当前刻度将对应当前价格（1 ETH = 5000 USDC）。

2. 我们提供流动性的价格范围的上下界。让下限价格为4545美元，上限价格为5500美元。

从理论介绍中，我们知道：

$$\sqrt{P} = \sqrt{\frac{y}{x}}$$

由于我们同意使用ETH作为$x$储备，USDC作为$y$储备，每个刻度的价格为：

$$\sqrt{P_c} = \sqrt{\frac{5000}{1}} = \sqrt{5000} \approx 70.71$$

$$\sqrt{P_l} = \sqrt{\frac{4545}{1}} \approx 67.42$$

$$\sqrt{P_u} = \sqrt{\frac{5500}{1}} \approx 74.16$$

其中$P_c$是当前价格，$P_l$是范围的下限，$P_u$是范围的上限。

现在，我们可以找到对应的刻度。我们知道价格和刻度通过以下公式相连：

$$\sqrt{P(i)}=1.0001^{\frac{i}{2}}$$

因此，我们可以通过以下方式找到刻度$i$：

$$i = log_{\sqrt{1.0001}} \sqrt{P(i)}$$

> 这个公式中的平方根相互抵消，但由于我们使用$\sqrt{p}$工作，我们需要保留它们。

让我们找到这些刻度：

1. 当前刻度：$i_c = log_{\sqrt{1.0001}} 70.71 = 85176$

2. 下限刻度：$i_l = log_{\sqrt{1.0001}} 67.42 = 84222$

3. 上限刻度：$i_u = log_{\sqrt{1.0001}} 74.16 = 86129$

> 为了计算这些，我使用了Python：

> ```python
> import math
>
> def price_to_tick(p):
>     return math.floor(math.log(p, 1.0001))
>
> price_to_tick(5000)
> > 85176
>```

这就是价格范围计算的全部内容！

这里需要注意的最后一点是，Uniswap使用[Q64.96数字](https://en.wikipedia.org/wiki/Q_%28number_format%29)来存储$\sqrt{P}$。这是一个定点数，整数部分有64位，小数部分有96位。在我们上面的计算中，价格是浮点数：`70.71`、`67.42`和`74.16`。我们需要将它们转换为Q64.96。幸运的是，这很简单：我们需要将这些数字乘以$2^{96}$（Q数是二进制定点数，所以我们需要将我们的小数乘以Q64.96的基数，即$2^{96}$）。我们将得到：

$$\sqrt{P_c} = 5602277097478614198912276234240$$

$$\sqrt{P_l} = 5314786713428871004159001755648$$

$$\sqrt{P_u} = 5875717789736564987741329162240$$

> 在Python中：

> ```python
> q96 = 2**96
> def price_to_sqrtp(p):
>     return int(math.sqrt(p) * q96)
>
> price_to_sqrtp(5000)
> > 5602277097478614198912276234240
> ```

> 注意我们在转换为整数之前进行乘法。否则，我们将失去精度。

## 代币数量计算

下一步是决定我们想要存入池中的代币数量。答案是我们想要多少就多少。这些数量并没有严格定义，我们可以存入足够的数量，以便在不使当前价格离开我们投入流动性的价格范围的情况下购买少量ETH。在开发和测试过程中，我们将能够铸造任何数量的代币，所以获得我们想要的数量不是问题。

对于我们的第一次交换，让我们存入1 ETH和5000 USDC。

> 请记住，当前池储备的比例表示当前现货价格。因此，如果我们想向池中投入更多代币并保持相同的价格，数量必须成比例，例如：2 ETH和10,000 USDC；10 ETH和50,000 USDC等。

## 流动性数量计算

接下来，我们需要根据我们将存入的数量计算$L$。这是一个棘手的部分，所以请仔细听！

从理论介绍中，你记得：

$$L = \sqrt{xy}$$

然而，这个公式是针对无限曲线的🙂 但我们想要将流动性投入到有限的价格范围内，这只是那个无限曲线的一个片段。我们需要专门为我们要存入流动性的价格范围计算$L$。我们需要一些更高级的计算。

为了计算价格范围的$L$，让我们看一个我们之前讨论过的有趣事实：价格范围可能会耗尽。可以从价格范围中买走一种代币的全部数量，使池中只剩下另一种代币。

![范围耗尽示例](images/range_depleted.png)

在点$a$和$b$，范围内只有一种代币：在点$a$处是ETH，在点$b$处是USDC。

也就是说，我们想找到一个$L$，使价格能够移动到任一点。我们希望有足够的流动性让价格达到价格范围的任一边界。因此，我们希望$L$基于$\Delta x$和$\Delta y$的最大数量来计算。

现在，让我们看看边缘处的价格是多少。当从池中买入ETH时，价格上涨；当买入USDC时，价格下跌。回想一下，价格是$\frac{y}{x}$。所以，在点$a$，价格是范围内的最低点；在点$b$，价格是最高点。

>实际上，在这些点上价格并没有定义，因为池中只有一种储备，但我们需要理解的是，点$b$附近的价格高于起始价格，而点$a$处的价格低于起始价格。

现在，将上图中的曲线分成两段：一段在起始点的左侧，一段在起始点的右侧。我们将计算**两个**$L$，每段一个。为什么？因为池中的两种代币各自贡献了**其中一段**：左段完全由代币$x$组成，右段完全由代币$y$组成。这源于在交换过程中，价格向任一方向移动的事实：它要么上涨，要么下跌。为了使价格移动，只需要其中一种代币：

1. 当价格上涨时，只需要代币$x$进行交换（我们正在买入代币$x$，所以我们只想从池中取出代币$x$）；

2. 当价格下跌时，只需要代币$y$进行交换。

因此，当前价格左侧曲线段的流动性仅由代币$x$组成，并且仅根据提供的代币$x$数量计算。同样，当前价格右侧曲线段的流动性仅由代币$y$组成，并且仅根据提供的代币$y$数量计算。

![曲线上的流动性](images/curve_liquidity.png)

这就是为什么在提供流动性时，我们计算两个$L$并选择其中一个。选择哪一个？较小的那个。为什么？因为较大的那个已经包含了较小的那个！我们希望新的流动性**均匀**分布在曲线上，因此我们希望在当前价格的左右两侧添加相同的$L$。如果我们选择较大的那个，用户需要提供更多的流动性来补偿较小的那个的不足。当然，这是可行的，但这会使智能合约变得更复杂。

> 较大的$L$的剩余部分会怎样？嗯，什么都不会发生。在选择较小的$L$之后，我们可以简单地将其转换为导致较大$L$的代币的较小数量——这将调整它。之后，我们将得到能产生相同$L$的代币数量。

这里我需要你注意的最后一个细节是：**新的流动性不能改变当前价格**。也就是说，它必须与当前储备的比例成比例。这就是为什么两个$L$可能不同——当比例没有保持时。我们选择较小的$L$来重新建立比例。

我希望在我们用代码实现这个之后，这会更有意义！现在，让我们看看公式。

让我们回顾一下$\Delta x$和$\Delta y$是如何计算的：

$$\Delta x = \Delta \frac{1}{\sqrt{P}} L$$

$$\Delta y = \Delta \sqrt{P} L$$

我们可以通过用实际价格替换delta P来扩展这些公式（我们从上面知道它们）：

$$\Delta x = (\frac{1}{\sqrt{P_c}} - \frac{1}{\sqrt{P_b}}) L$$

$$\Delta y = (\sqrt{P_c} - \sqrt{P_a}) L$$

$P_a$是点$a$处的价格，$P_b$是点$b$处的价格，$P_c$是当前价格（见上图）。注意，由于价格计算为$\frac{y}{x}$（即它是以$y$表示的$x$的价格），点$b$处的价格高于当前价格和点$a$处的价格。点$a$处的价格是三者中最低的。

让我们从第一个公式中找到$L$：

$$\Delta x = (\frac{1}{\sqrt{P_c}} - \frac{1}{\sqrt{P_b}}) L$$

$$\Delta x = \frac{L}{\sqrt{P_c}} - \frac{L}{\sqrt{P_b}}$$

$$\Delta x = \frac{L(\sqrt{P_b} - \sqrt{P_c})}{\sqrt{P_b} \sqrt{P_c}}$$

$$L = \Delta x \frac{\sqrt{P_b} \sqrt{P_c}}{\sqrt{P_b} - \sqrt{P_c}}$$

从第二个公式：

$$\Delta y = (\sqrt{P_c} - \sqrt{P_a}) L$$

$$L = \frac{\Delta y}{\sqrt{P_c} - \sqrt{P_a}}$$

所以，这就是我们的两个$L$，每个段一个：

$$L = \Delta x \frac{\sqrt{P_b} \sqrt{P_c}}{\sqrt{P_b} - \sqrt{P_c}}$$

$$L = \frac{\Delta y}{\sqrt{P_c} - \sqrt{P_a}}$$

现在，让我们将我们之前计算的价格代入它们：

$$L = \Delta x \frac{\sqrt{P_b}\sqrt{P_c}}{\sqrt{P_b}-\sqrt{P_c}} = 1 ETH * \frac{5875... * 5602...}{5875... - 5602...}$$

转换为Q64.96后，我们得到：

$$L = 1519437308014769733632$$

对于另一个$L$：

$$L = \frac{\Delta y}{\sqrt{P_c}-\sqrt{P_a}} = \frac{5000USDC}{5602... - 5314...}$$

$$L = 1517882343751509868544$$

在这两个中，我们将选择较小的那个。

> 在Python中：

> ```python
> sqrtp_low = price_to_sqrtp(4545)
> sqrtp_cur = price_to_sqrtp(5000)
> sqrtp_upp = price_to_sqrtp(5500)
> 
> def liquidity0(amount, pa, pb):
>     if pa > pb:
>         pa, pb = pb, pa
>     return (amount * (pa * pb) / q96) / (pb - pa)
>
> def liquidity1(amount, pa, pb):
>     if pa > pb:
>         pa, pb = pb, pa
>     return amount * q96 / (pb - pa)
>
> eth = 10**18
> amount_eth = 1 * eth
> amount_usdc = 5000 * eth
> 
> liq0 = liquidity0(amount_eth, sqrtp_cur, sqrtp_upp)
> liq1 = liquidity1(amount_usdc, sqrtp_cur, sqrtp_low)
> liq = int(min(liq0, liq1))
> > 1517882343751509868544
> ```

## 再次计算代币数量

由于我们选择了要存入的数量，这些数量可能是错误的。我们不能在任何价格范围内存入任何数量；流动性数量需要均匀分布在我们存入的价格范围的曲线上。因此，即使用户选择了数量，合约也需要重新计算它们，实际数量会略有不同（至少是因为四舍五入）。

幸运的是，我们已经知道公式：

$$\Delta x = \frac{L(\sqrt{P_b} - \sqrt{P_c})}{\sqrt{P_b} \sqrt{P_c}}$$

$$\Delta y = L(\sqrt{P_c} - \sqrt{P_a})$$

> 在Python中：

> ```python
> def calc_amount0(liq, pa, pb):
>     if pa > pb:
>         pa, pb = pb, pa
>     return int(liq * q96 * (pb - pa) / pa / pb)
> 
> 
> def calc_amount1(liq, pa, pb):
>     if pa > pb:
>         pa, pb = pb, pa
>     return int(liq * (pb - pa) / q96)
>
> amount0 = calc_amount0(liq, sqrtp_upp, sqrtp_cur)
> amount1 = calc_amount1(liq, sqrtp_low, sqrtp_cur)
> (amount0, amount1)
> > (998976618347425408, 5000000000000000000000)
> ```

> 如你所见，这些数字接近我们想要提供的数量，但ETH略小。

> **提示**：使用`cast --from-wei AMOUNT`将wei转换为ether，例如：
> `cast --from-wei 998976618347425280`将给你`0.998976618347425280`。



