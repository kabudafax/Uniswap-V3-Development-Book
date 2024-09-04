# 闪电贷费用

在前一章中，我们实现了闪电贷并使其免费。然而，Uniswap在闪电贷上收取交换费用，我们将把这一点添加到我们的实现中：闪电贷借款人偿还的金额必须包含费用。

以下是更新后的`flash`函数的样子：

```solidity
function flash(
    uint256 amount0,
    uint256 amount1,
    bytes calldata data
) public {
    uint256 fee0 = Math.mulDivRoundingUp(amount0, fee, 1e6);
    uint256 fee1 = Math.mulDivRoundingUp(amount1, fee, 1e6);

    uint256 balance0Before = IERC20(token0).balanceOf(address(this));
    uint256 balance1Before = IERC20(token1).balanceOf(address(this));

    if (amount0 > 0) IERC20(token0).transfer(msg.sender, amount0);
    if (amount1 > 0) IERC20(token1).transfer(msg.sender, amount1);

    IUniswapV3FlashCallback(msg.sender).uniswapV3FlashCallback(
        fee0,
        fee1,
        data
    );

    if (IERC20(token0).balanceOf(address(this)) < balance0Before + fee0)
        revert FlashLoanNotPaid();
    if (IERC20(token1).balanceOf(address(this)) < balance1Before + fee1)
        revert FlashLoanNotPaid();

    emit Flash(msg.sender, amount0, amount1);
}
```

变化的是，我们现在对调用者请求的金额计算费用，然后期望池子余额增加相应的费用金额。
