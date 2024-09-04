# 用户界面

让我们使我们的网页应用更像一个真正的去中心化交易所（DEX）。我们现在可以移除硬编码的交换金额，让用户输入任意金额。此外，我们现在可以让用户在两个方向上进行交换，所以我们还需要一个按钮来交换代币输入。更新后，交换表单将看起来像这样：

```jsx
<form className="SwapForm">
  <SwapInput
    amount={zeroForOne ? amount0 : amount1}
    disabled={!enabled || loading}
    readOnly={false}
    setAmount={setAmount_(zeroForOne ? setAmount0 : setAmount1, zeroForOne)}
    token={zeroForOne ? pair[0] : pair[1]} />
  <ChangeDirectionButton zeroForOne={zeroForOne} setZeroForOne={setZeroForOne} disabled={!enabled || loading} />
  <SwapInput
    amount={zeroForOne ? amount1 : amount0}
    disabled={!enabled || loading}
    readOnly={true}
    token={zeroForOne ? pair[1] : pair[0]} />
  <button className='swap' disabled={!enabled || loading} onClick={swap_}>Swap</button>
</form>
```

每个输入框都有一个与之关联的金额，这取决于由`zeroForOne`状态变量控制的交换方向。下方的输入框始终是只读的，因为它的值是由Quoter合约计算得出的。

`setAmount_`函数做两件事：它更新上方输入框的值，并调用Quoter合约来计算下方输入框的值：

```js
const updateAmountOut = debounce((amount) => {
  if (amount === 0 || amount === "0") {
    return;
  }

  setLoading(true);

  quoter.callStatic
    .quote({ pool: config.poolAddress, amountIn: ethers.utils.parseEther(amount), zeroForOne: zeroForOne })
    .then(({ amountOut }) => {
      zeroForOne ? setAmount1(ethers.utils.formatEther(amountOut)) : setAmount0(ethers.utils.formatEther(amountOut));
      setLoading(false);
    })
    .catch((err) => {
      zeroForOne ? setAmount1(0) : setAmount0(0);
      setLoading(false);
      console.error(err);
    })
})

const setAmount_ = (setAmountFn) => {
  return (amount) => {
    amount = amount || 0;
    setAmountFn(amount);
    updateAmountOut(amount)
  }
}
```
注意在`quoter`上调用的`callStatic`——这就是我们在上一章讨论的内容：我们需要强制Ethers.js进行静态调用。由于`quote`不是`pure`或`view`函数，Ethers.js会尝试在交易中调用`quote`。

就是这样！用户界面现在允许我们指定任意金额并在任一方向进行交换！
