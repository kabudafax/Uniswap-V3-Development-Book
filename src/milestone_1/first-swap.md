# 首次交换

现在我们有了流动性，我们可以进行我们的第一次交换了！

## 计算交换金额

当然，第一步是弄清楚如何计算交换金额。再次，让我们选择并硬编码一些我们将用来交换 ETH 的 USDC 数量。让它是 42！我们将用 42 USDC 购买 ETH。

在决定我们想要出售多少代币后，我们需要计算我们将获得多少代币作为交换。在 Uniswap V2 中，我们会使用当前池子储备，但在 Uniswap V3 中，我们有 $L$ 和 $\sqrt{P}$，并且我们知道在价格范围内交换时，只有 $\sqrt{P}$ 会改变，而 $L$ 保持不变（当交换仅在一个价格范围内进行时，Uniswap V3 的行为与 V2 完全相同）。我们还知道：

$$L = \frac{\Delta y}{\Delta \sqrt{P}}$$

而且...我们知道 $\Delta y$！这就是我们要交易的 42 USDC！因此，我们可以找出出售 42 USDC 将如何影响当前的 $\sqrt{P}$，给定 $L$：

$$\Delta \sqrt{P} = \frac{\Delta y}{L}$$

在 Uniswap V3 中，我们选择**我们希望交易达到的价格**（回想一下，交换会改变当前价格，即它沿曲线移动当前价格）。知道目标价格后，合约将计算它需要从我们这里获取的输入代币数量和相应的它将给我们的输出代币数量。

让我们将我们的数字代入上面的公式：

$$\Delta \sqrt{P} = \frac{42 \enspace USDC}{1517882343751509868544} = 2192253463713690532467206957$$

将此加到当前的 $\sqrt{P}$ 后，我们将得到目标价格：

$$\sqrt{P_{target}} = \sqrt{P_{current}} + \Delta \sqrt{P}$$

$$\sqrt{P_{target}} = 5604469350942327889444743441197$$

> 要在 Python 中计算目标价格：

> ```python
> amount_in = 42 * eth
> price_diff = (amount_in * q96) // liq
> price_next = sqrtp_cur + price_diff
> print("New price:", (price_next / q96) ** 2)
> print("New sqrtP:", price_next)
> print("New tick:", price_to_tick((price_next / q96) ** 2))
> # New price: 5003.913912782393
> # New sqrtP: 5604469350942327889444743441197
> # New tick: 85184
> ```

在找到目标价格后，我们可以使用前一章的金额计算函数来计算代币数量：

$$ x = \frac{L(\sqrt{p_b}-\sqrt{p_a})}{\sqrt{p_b}\sqrt{p_a}}$$

$$ y = L(\sqrt{p_b}-\sqrt{p_a}) $$

> 在 Python 中：

> ```python
> amount_in = calc_amount1(liq, price_next, sqrtp_cur)
> amount_out = calc_amount0(liq, price_next, sqrtp_cur)
> 
> print("USDC in:", amount_in / eth)
> print("ETH out:", amount_out / eth)
> # USDC in: 42.0
> # ETH out: 0.008396714242162444
> ```

为了验证金额，让我们回顾另一个公式：

$$\Delta x = \Delta \frac{1}{\sqrt{P}} L$$

使用这个公式，我们可以找到我们正在购买的 ETH 数量 $\Delta x$，知道价格变化 $\Delta\frac{1}{\sqrt{P}}$ 和流动性 $L$。但要小心：$\Delta \frac{1}{\sqrt{P}}$ 不是 $\frac{1}{\Delta \sqrt{P}}$！前者是 ETH 价格的变化，可以使用以下表达式找到：

$$\Delta \frac{1}{\sqrt{P}} = \frac{1}{\sqrt{P_{target}}} - \frac{1}{\sqrt{P_{current}}}$$

幸运的是，我们已经知道所有的值，所以我们可以直接代入（这可能不适合你的屏幕！）：

$$\Delta \frac{1}{\sqrt{P}} = \frac{1}{5604469350942327889444743441197} - \frac{1}{5602277097478614198912276234240}$$

$$= -6.982190286589445\text{e-}35 * 2^{96} $$
$$= -0.00000553186106731426$$

现在，让我们找到 $\Delta x$：

$$\Delta x = -0.00000553186106731426 * 1517882343751509868544 = -8396714242162698 $$

这是 0.008396714242162698 ETH，非常接近我们上面找到的数量！注意，这个数量是负数，因为我们正在从池子中移除它。

## 实现交换

交换在 `swap` 函数中实现：

```solidity
function swap(address recipient)
    public
    returns (int256 amount0, int256 amount1)
{

```

目前，它只接受一个接收者，即代币的接收者。

首先，我们需要找到目标价格和 tick，并计算代币数量。再次，我们将简单地硬编码我们之前计算的值，以保持事情尽可能简单：

```solidity
int24 nextTick = 85184;
uint160 nextPrice = 5604469350942327889444743441197;

amount0 = -0.008396714242162444 ether;
amount1 = 42 ether;
```

接下来，我们需要更新当前的 tick 和 sqrtP，因为交易会影响当前价格：

```solidity
(slot0.tick, slot0.sqrtPriceX96) = (nextTick, nextPrice);
```

然后，合约将代币发送给接收者，并让调用者将输入金额转入合约：

```solidity
IERC20(token0).transfer(recipient, uint256(-amount0));

uint256 balance1Before = balance1();
IUniswapV3SwapCallback(msg.sender).uniswapV3SwapCallback(
    amount0,
    amount1
);
if (balance1Before + uint256(amount1) < balance1())
    revert InsufficientInputAmount();
```

再次，我们使用回调来将控制权传递给调用者，让它转移代币。之后，我们检查池子的余额是否正确并包含输入金额。

最后，合约发出一个 Swap 事件，使交换可被发现。该事件包含了关于交换的所有信息：

```solidity
emit Swap(
    msg.sender,
    recipient,
    amount0,
    amount1,
    slot0.sqrtPriceX96,
    liquidity,
    slot0.tick
);
```
就是这样！该函数简单地将一定数量的代币发送到指定的接收者地址，并期望换取一定数量的其他代币。在本书的过程中，这个函数将变得更加复杂。

## 测试交换

现在，我们可以测试交换函数了。在同一个测试文件中，创建 `testSwapBuyEth` 函数并设置测试用例。这个测试用例使用与 `testMintSuccess` 相同的参数：

```solidity
function testSwapBuyEth() public {
    TestCaseParams memory params = TestCaseParams({
        wethBalance: 1 ether,
        usdcBalance: 5000 ether,
        currentTick: 85176,
        lowerTick: 84222,
        upperTick: 86129,
        liquidity: 1517882343751509868544,
        currentSqrtP: 5602277097478614198912276234240,
        shouldTransferInCallback: true,
        mintLiqudity: true
    });
    (uint256 poolBalance0, uint256 poolBalance1) = setupTestCase(params);
```

然而，接下来的步骤将会不同。

我们不会测试流动性是否正确地添加到池子中，因为我们在其他测试用例中已经测试了这个功能。

要进行测试交换，我们需要 42 USDC：

```solidity
token1.mint(address(this), 42 ether);
```

在进行交换之前，我们需要确保我们可以在池子合约请求时向其转移代币：

```solidity
function uniswapV3SwapCallback(int256 amount0, int256 amount1) public {
    if (amount0 > 0) {
        token0.transfer(msg.sender, uint256(amount0));
    }

    if (amount1 > 0) {
        token1.transfer(msg.sender, uint256(amount1));
    }
}
```

由于交换期间的金额可以是正数（发送到池子的金额）和负数（从池子中取出的金额），在回调中，我们只想发送正数金额，即我们正在交易的金额。

现在，我们可以调用 swap：

```solidity
(int256 amount0Delta, int256 amount1Delta) = pool.swap(address(this));
```

该函数返回在交换中使用的代币数量，我们可以立即检查它们：

```solidity
assertEq(amount0Delta, -0.008396714242162444 ether, "invalid ETH out");
assertEq(amount1Delta, 42 ether, "invalid USDC in");
```

然后，我们需要确保代币从调用者转移：

```solidity
assertEq(
    token0.balanceOf(address(this)),
    uint256(userBalance0Before - amount0Delta),
    "invalid user ETH balance"
);
assertEq(
    token1.balanceOf(address(this)),
    0,
    "invalid user USDC balance"
);
```

并发送到池子合约：

```solidity
assertEq(
    token0.balanceOf(address(pool)),
    uint256(int256(poolBalance0) + amount0Delta),
    "invalid pool ETH balance"
);
assertEq(
    token1.balanceOf(address(pool)),
    uint256(int256(poolBalance1) + amount1Delta),
    "invalid pool USDC balance"
);
```

最后，我们检查池子状态是否正确更新：

```solidity
(uint160 sqrtPriceX96, int24 tick) = pool.slot0();
assertEq(
    sqrtPriceX96,
    5604469350942327889444743441197,
    "invalid current sqrtP"
);
assertEq(tick, 85184, "invalid current tick");
assertEq(
    pool.liquidity(),
    1517882343751509868544,
    "invalid current liquidity"
);
```

注意，交换不会改变当前的流动性——在后面的章节中，我们将看到它何时会改变。

家庭作业
编写一个测试，使其因 InsufficientInputAmount 错误而失败。请记住，这里有一个隐藏的 bug 🙂。
