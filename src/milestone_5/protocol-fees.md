# 协议费用

在实现Uniswap的过程中，你可能会问自己："Uniswap是如何盈利的？"事实上，它并不盈利（至少截至2022年9月）。

在我们目前构建的实现中，交易者为提供流动性的流动性提供者支付费用，而Uniswap Labs作为开发这个DEX的公司，并不参与这个过程。无论是交易者还是流动性提供者都不需要为使用Uniswap DEX向Uniswap Labs支付费用。这是为什么呢？

其实Uniswap Labs有一种方式可以开始从DEX中获利。然而，这个机制尚未启用（同样，截至2022年9月）。每个Uniswap池都有一个*协议费用*收集机制。协议费用是从交换费用中收取的：交换费用中的一小部分被扣除并保存为协议费用，以便后续由Factory合约所有者（Uniswap Labs）收集。协议费用的大小预计将由UNI代币持有者决定，但必须在交换费用的$$1/4$$到$$1/10$$之间（包括这两个值）。

为了简洁起见，我们不会在我们的实现中添加协议费用，但让我们看看它们在Uniswap中是如何实现的。

协议费用大小存储在`slot0`中：

```solidity
// UniswapV3Pool.sol
struct Slot0 {
    ...
    // the current protocol fee as a percentage of the swap fee taken on withdrawal
    // represented as an integer denominator (1/x)%
    uint8 feeProtocol;
    ...
}
```

需要一个全局累加器来跟踪累积的费用：
```solidity
// accumulated protocol fees in token0/token1 units
struct ProtocolFees {
    uint128 token0;
    uint128 token1;
}
ProtocolFees public override protocolFees;
```

协议费用在`setFeeProtocol`函数中设置：

```solidity
function setFeeProtocol(uint8 feeProtocol0, uint8 feeProtocol1) external override lock onlyFactoryOwner {
    require(
        (feeProtocol0 == 0 || (feeProtocol0 >= 4 && feeProtocol0 <= 10)) &&
            (feeProtocol1 == 0 || (feeProtocol1 >= 4 && feeProtocol1 <= 10))
    );
    uint8 feeProtocolOld = slot0.feeProtocol;
    slot0.feeProtocol = feeProtocol0 + (feeProtocol1 << 4);
    emit SetFeeProtocol(feeProtocolOld % 16, feeProtocolOld >> 4, feeProtocol0, feeProtocol1);
}
```

如你所见，允许为每个代币单独设置协议费用。这两个值是两个`uint8`，它们被打包存储在一个`uint8`中：`feeProtocol1`向左移4位（这等同于乘以16）并加上`feeProtocol0`。要解包`feeProtocol0`，取`slot0.feeProtocol`除以16的余数；`feeProtocol1`只需将`slot0.feeProtocol`向右移4位。这种打包方式之所以有效，是因为`feeProtocol0`和`feeProtocol1`都不能大于10。

在开始交换之前，我们需要根据交换方向选择其中一个协议费用（交换费用和协议费用是在输入代币上收取的）：

```solidity
function swap(...) {
    ...
    uint8 feeProtocol = zeroForOne ? (slot0_.feeProtocol % 16) : (slot0_.feeProtocol >> 4);
    ...
```

为了累积协议费用，我们在计算交换步骤金额后立即从交换费用中扣除它们：

```solidity
...
while (...) {
    (..., step.feeAmount) = SwapMath.computeSwapStep(...);

    if (cache.feeProtocol > 0) {
        uint256 delta = step.feeAmount / cache.feeProtocol;
        step.feeAmount -= delta;
        state.protocolFee += uint128(delta);
    }

    ...
}
...
```

在交换完成后，需要更新全局协议费用累加器：
```solidity
if (zeroForOne) {
    if (state.protocolFee > 0) protocolFees.token0 += state.protocolFee;
} else {
    if (state.protocolFee > 0) protocolFees.token1 += state.protocolFee;
}
```

最后，Factory合约所有者可以通过调用`collectProtocol`来收集累积的协议费用：

```solidity
function collectProtocol(
    address recipient,
    uint128 amount0Requested,
    uint128 amount1Requested
) external override lock onlyFactoryOwner returns (uint128 amount0, uint128 amount1) {
    amount0 = amount0Requested > protocolFees.token0 ? protocolFees.token0 : amount0Requested;
    amount1 = amount1Requested > protocolFees.token1 ? protocolFees.token1 : amount1Requested;

    if (amount0 > 0) {
        if (amount0 == protocolFees.token0) amount0--;
        protocolFees.token0 -= amount0;
        TransferHelper.safeTransfer(token0, recipient, amount0);
    }
    if (amount1 > 0) {
        if (amount1 == protocolFees.token1) amount1--;
        protocolFees.token1 -= amount1;
        TransferHelper.safeTransfer(token1, recipient, amount1);
    }

    emit CollectProtocol(msg.sender, recipient, amount0, amount1);
}
```