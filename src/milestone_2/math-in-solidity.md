# Solidity中的数学运算

由于Solidity不支持带小数部分的数字，Solidity中的数学运算有些复杂。Solidity给我们提供了整数和无符号整数类型，但这对于较为复杂的数学计算来说是不够的。

另一个困难是gas消耗：算法越复杂，消耗的gas就越多。因此，如果我们需要进行高级数学运算（如`exp`、`ln`和`sqrt`），我们希望它们尽可能地节省gas。

另一个大问题是下溢/上溢的可能性。当乘以`uint256`数字时，存在上溢的风险：结果数字可能大到无法容纳在256位中。

所有这些困难迫使我们使用第三方数学库，这些库实现了高级数学运算，理想情况下还优化了它们的gas消耗。如果我们需要的算法没有现成的库，我们就必须自己实现，这在需要实现独特计算时是一项困难的任务。

## 重用数学合约

在我们的Uniswap V3实现中，我们将使用两个第三方数学合约：

1. [PRBMath](https://github.com/paulrberg/prb-math)，这是一个优秀的固定点高级数学算法库。我们将使用`mulDiv`函数来处理乘法然后除法整数时的溢出问题。

2. 来自原始Uniswap V3仓库的[TickMath](https://github.com/Uniswap/v3-core/blob/main/contracts/libraries/TickMath.sol)。这个合约实现了两个函数，`getSqrtRatioAtTick`和`getTickAtSqrtRatio`，用于将$\sqrt{P}$转换为ticks，反之亦然。

让我们关注后者。

在我们的合约中，我们需要将ticks转换为相应的$\sqrt{P}$，反之亦然。公式如下：

$$\sqrt{P(i)} = \sqrt{1.0001^i} = 1.0001^{\frac{i}{2}}$$

$$i = log_{\sqrt{1.0001}}\sqrt{P(i)}$$

这些是复杂的数学运算（至少对Solidity来说是这样），它们需要高精度，因为我们不希望在计算价格时出现舍入误差。为了获得更好的精度和优化，我们需要一个独特的实现。

如果你查看[getSqrtRatioAtTick](https://github.com/Uniswap/v3-core/blob/8f3e4645a08850d2335ead3d1a8d0c64fa44f222/contracts/libraries/TickMath.sol#L23-L54)和[getTickAtSqrtRatio](https://github.com/Uniswap/v3-core/blob/8f3e4645a08850d2335ead3d1a8d0c64fa44f222/contracts/libraries/TickMath.sol#L61-L204)的原始代码，你会发现它们相当复杂：有很多魔法数字（如`0xfffcb933bd6fad37aa2d162d1a594001`），乘法和位运算。在这一点上，我们不打算分析代码或重新实现它，因为这是一个非常高级且有些不同的主题。我们将按原样使用该合约。在后面的里程碑中，我们将分解这些计算。
