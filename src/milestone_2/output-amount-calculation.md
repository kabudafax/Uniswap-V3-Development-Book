## 输出数量计算

我们的Uniswap数学公式集合还缺少最后一块：计算卖出ETH（即卖出代币$x$）时输出数量的公式。在上一个里程碑中，我们有一个类似的公式用于买入ETH（买入代币$x$）的情况：

$$\Delta \sqrt{P} = \frac{\Delta y}{L}$$

这个公式用于找出卖出代币$y$时价格的变化。然后我们将这个变化加到当前价格上，以找到目标价格：

$$\sqrt{P_{target}} = \sqrt{P_{current}} + \Delta \sqrt{P}$$

现在，我们需要一个类似的公式来找出卖出代币$x$（在我们的例子中是ETH）和买入代币$y$（在我们的例子中是USDC）时的目标价格。

回想一下，代币$x$的变化可以通过以下公式计算：

$$\Delta x = \Delta \frac{1}{\sqrt{P}}L$$

从这个公式，我们可以找到目标价格：

$$\Delta x = (\frac{1}{\sqrt{P_{target}}} - \frac{1}{\sqrt{P_{current}}}) L$$

$$= \frac{L}{\sqrt{P_{target}}} - \frac{L}{\sqrt{P_{current}}}$$

从这里，我们可以通过基本的代数变换找到$\sqrt{P_{target}}$：

$$\sqrt{P_{target}} = \frac{\sqrt{P}L}{\Delta x \sqrt{P} + L}$$

知道了目标价格，我们可以用类似于上一个里程碑中的方法找到输出数量。

让我们用新公式更新我们的Python脚本：

```python
# Swap ETH for USDC
amount_in = 0.01337 * eth

print(f"\nSelling {amount_in/eth} ETH")

price_next = int((liq * q96 * sqrtp_cur) // (liq * q96 + amount_in * sqrtp_cur))

print("New price:", (price_next / q96) ** 2)
print("New sqrtP:", price_next)
print("New tick:", price_to_tick((price_next / q96) ** 2))

amount_in = calc_amount0(liq, price_next, sqrtp_cur)
amount_out = calc_amount1(liq, price_next, sqrtp_cur)

print("ETH in:", amount_in / eth)
print("USDC out:", amount_out / eth)
```

它的输出：
```shell
Selling 0.01337 ETH
New price: 4993.777388290041
New sqrtP: 5598789932670289186088059666432
New tick: 85163
ETH in: 0.013369999999998142
USDC out: 66.80838889019013
```

这意味着使用我们在上一步提供的流动性，当卖出0.01337 ETH时，我们将获得66.8 USDC。

这看起来不错，但Python已经足够了！我们将在Solidity中实现所有的数学计算。
