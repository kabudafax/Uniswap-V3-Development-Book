# 提供流动性

理论已经够多了，让我们开始编码吧！

创建一个新文件夹（我的叫做 `uniswapv3-code`），然后在其中运行 `forge init --vscode` —— 这将初始化一个 Forge 项目。`--vscode` 标志告诉 Forge 为 Forge 项目配置 Solidity 扩展。

接下来，删除默认的合约及其测试：

- `script/Contract.s.sol`
- `src/Contract.sol`
- `test/Contract.t.sol`

就是这样！让我们创建我们的第一个合约！

## 池子合约

正如你在介绍中了解到的，Uniswap 部署了多个池子合约，每个合约都是一对代币的交换市场。Uniswap 将其所有合约分为两类：

- 核心合约，
- 和外围合约。

核心合约，顾名思义，是实现核心逻辑的合约。这些是最小的、用户不友好的、低级别的合约。它们的目的是做一件事，并尽可能可靠和安全地做到这一点。在 Uniswap V3 中，有 2 个这样的合约：

1. 池子合约，实现去中心化交易所的核心逻辑。
2. 工厂合约，作为池子合约的注册表，并使池子的部署更容易。

我们将从池子合约开始，它实现了 Uniswap 99% 的核心功能。

创建 `src/UniswapV3Pool.sol`：

```solidity
pragma solidity ^0.8.14;

contract UniswapV3Pool {}
```
让我们思考一下合约将存储哪些数据：

由于每个池子合约都是两个代币的交换市场，我们需要跟踪这两个代币地址。这些地址将是静态的，在池子部署期间一次性设置（因此，它们将是不可变的）。

每个池子合约都是一组流动性头寸。我们将它们存储在一个映射中，其中键是唯一的头寸标识符，值是存储头寸信息的结构体。

每个池子合约还需要维护一个 tick 注册表 —— 这将是一个映射，其中键是 tick 索引，值是存储 tick 信息的结构体。

由于 tick 范围是有限的，我们需要在合约中将限制存储为常量。

回想一下，池子合约存储流动性数量 $L$。所以我们需要为它设置一个变量。

最后，我们需要跟踪当前价格和相关的 tick。我们将它们存储在一个存储槽中以优化 gas 消耗：这些变量经常一起读取和写入，所以利用 Solidity 的状态变量打包特性 是有意义的。

总的来说，这是我们开始的内容：

```solidity
// src/lib/Tick.sol
library Tick {
    struct Info {
        bool initialized;
        uint128 liquidity;
    }
    ...
}

// src/lib/Position.sol
library Position {
    struct Info {
        uint128 liquidity;
    }
    ...
}

// src/UniswapV3Pool.sol
contract UniswapV3Pool {
    using Tick for mapping(int24 => Tick.Info);
    using Position for mapping(bytes32 => Position.Info);
    using Position for Position.Info;

    int24 internal constant MIN_TICK = -887272;
    int24 internal constant MAX_TICK = -MIN_TICK;

    // 池子代币，不可变
    address public immutable token0;
    address public immutable token1;

    // 打包一起读取的变量
    struct Slot0 {
        // 当前 sqrt(P)
        uint160 sqrtPriceX96;
        // 当前 tick
        int24 tick;
    }
    Slot0 public slot0;

    // 流动性数量，L。
    uint128 public liquidity;

    // Ticks 信息
    mapping(int24 => Tick.Info) public ticks;
    // 头寸信息
    mapping(bytes32 => Position.Info) public positions;

    ...
```

Uniswap V3 使用了许多辅助合约，Tick 和 Position 是其中的两个。using A for B 是 Solidity 的一个特性，它允许你用库合约 A 中的函数扩展类型 B。这简化了复杂数据结构的管理。

为了简洁，我将省略对 Solidity 语法和特性的详细解释。Solidity 有 很好的文档，如果有不清楚的地方，不要犹豫查阅它！

然后我们将在构造函数中初始化一些变量：

```solidity
    constructor(
        address token0_,
        address token1_,
        uint160 sqrtPriceX96,
        int24 tick
    ) {
        token0 = token0_;
        token1 = token1_;

        slot0 = Slot0({sqrtPriceX96: sqrtPriceX96, tick: tick});
    }
}
```

在这里，我们设置了代币地址不可变量，并设置了当前价格和 tick —— 我们不需要为后者提供流动性。

这是我们的起点，我们在本章的目标是使用预先计算和硬编码的值进行我们的第一次交换。

## 铸造

在 Uniswap V2 中，提供流动性的过程被称为*铸造*。原因是 V2 池子合约会铸造代币（LP-代币）以换取流动性。V3 不这样做，但它仍然使用相同的名称来命名函数。让我们也使用它：

```solidity
function mint(
    address owner,
    int24 lowerTick,
    int24 upperTick,
    uint128 amount
) external returns (uint256 amount0, uint256 amount1) {
    ...
```
我们的 mint 函数将接受：

所有者地址，用于跟踪流动性的所有者。
上限和下限 tick，用于设置价格范围的边界。
我们想要提供的流动性数量。
注意，用户指定的是 $L$，而不是实际的代币数量。这当然不是很方便，但请记住，Pool 合约是一个核心合约——它不打算对用户友好，因为它应该只实现核心逻辑。在后面的章节中，我们将制作一个辅助合约，在调用 Pool.mint 之前将代币数量转换为 $L$。

让我们简要概述一下铸造的工作方式：

用户指定一个价格范围和流动性数量；
合约更新 ticks 和 positions 映射；
合约计算用户必须发送的代币数量（我们将预先计算并硬编码它们）；
合约从用户那里获取代币并验证是否设置了正确的数量。
让我们从检查 ticks 开始：

```solidity
if (
    lowerTick >= upperTick ||
    lowerTick < MIN_TICK ||
    upperTick > MAX_TICK
) revert InvalidTickRange();
```

并确保提供了一些流动性数量：

```solidity
if (amount == 0) revert ZeroLiquidity();
```

然后，添加一个 tick 和一个头寸：

```solidity
ticks.update(lowerTick, amount);
ticks.update(upperTick, amount);

Position.Info storage position = positions.get(
    owner,
    lowerTick,
    upperTick
);
position.update(amount);
```

ticks.update 函数是：

```solidity
// src/lib/Tick.sol
function update(
    mapping(int24 => Tick.Info) storage self,
    int24 tick,
    uint128 liquidityDelta
) internal {
    Tick.Info storage tickInfo = self[tick];
    uint128 liquidityBefore = tickInfo.liquidity;
    uint128 liquidityAfter = liquidityBefore + liquidityDelta;

    if (liquidityBefore == 0) {
        tickInfo.initialized = true;
    }

    tickInfo.liquidity = liquidityAfter;
}
```

如果 tick 的流动性为 0，它会初始化该 tick，并向其添加新的流动性。如你所见，我们在上下限 tick 上都调用了这个函数，因此流动性被添加到两者中。

position.update 函数是：

```solidity
// src/libs/Position.sol
function update(Info storage self, uint128 liquidityDelta) internal {
    uint128 liquidityBefore = self.liquidity;
    uint128 liquidityAfter = liquidityBefore + liquidityDelta;

    self.liquidity = liquidityAfter;
}
```

类似于 tick 更新函数，它向特定头寸添加流动性。要获取头寸，我们调用：

```solidity
// src/libs/Position.sol
...
function get(
    mapping(bytes32 => Info) storage self,
    address owner,
    int24 lowerTick,
    int24 upperTick
) internal view returns (Position.Info storage position) {
    position = self[
        keccak256(abi.encodePacked(owner, lowerTick, upperTick))
    ];
}
...
```

每个头寸都由三个键唯一标识：所有者地址、下限 tick 索引和上限 tick 索引。我们对这三个进行哈希处理以使数据存储更便宜：当哈希处理后，每个键将占用 32 字节，而不是当 owner、lowerTick 和 upperTick 是单独的键时占用 96 字节。

如果我们使用三个键，我们需要三个映射。每个键将单独存储，并且会占用 32 字节，因为 Solidity 将值存储在 32 字节的槽中（当不应用打包时）。

接下来，继续铸造，我们需要计算用户必须存入的数量。幸运的是，我们已经在前面的部分中弄清楚了公式并计算了确切的数量。所以，我们将硬编码它们：

```solidity
amount0 = 0.998976618347425280 ether;
amount1 = 5000 ether;
```

> 我们将在后面的章节中用实际计算替换这些。

我们还将根据添加的 amount 更新池子的 liquidity。

```solidity
liquidity += uint128(amount);
```

## 铸造

在 Uniswap V2 中，提供流动性的过程被称为*铸造*。原因是 V2 池子合约会铸造代币（LP-代币）以换取流动性。V3 不这样做，但它仍然使用相同的名称来命名函数。让我们也使用它：

```solidity
function mint(
    address owner,
    int24 lowerTick,
    int24 upperTick,
    uint128 amount
) external returns (uint256 amount0, uint256 amount1) {
    ...
```

我们的 mint 函数将接受：

所有者地址，用于跟踪流动性的所有者。
上限和下限 tick，用于设置价格范围的边界。
我们想要提供的流动性数量。
注意，用户指定的是 $L$，而不是实际的代币数量。这当然不是很方便，但请记住，Pool 合约是一个核心合约——它不打算对用户友好，因为它应该只实现核心逻辑。在后面的章节中，我们将制作一个辅助合约，在调用 Pool.mint 之前将代币数量转换为 $L$。

让我们简要概述一下铸造的工作方式：

用户指定一个价格范围和流动性数量；
合约更新 ticks 和 positions 映射；
合约计算用户必须发送的代币数量（我们将预先计算并硬编码它们）；
合约从用户那里获取代币并验证是否设置了正确的数量。
让我们从检查 ticks 开始：

```solidity
if (
    lowerTick >= upperTick ||
    lowerTick < MIN_TICK ||
    upperTick > MAX_TICK
) revert InvalidTickRange();
```

并确保提供了一些流动性数量：

```solidity
if (amount == 0) revert ZeroLiquidity();
```

然后，添加一个 tick 和一个头寸：

```solidity
ticks.update(lowerTick, amount);
ticks.update(upperTick, amount);

Position.Info storage position = positions.get(
    owner,
    lowerTick,
    upperTick
);
position.update(amount);
```

ticks.update 函数是：

```solidity
// src/lib/Tick.sol
function update(
    mapping(int24 => Tick.Info) storage self,
    int24 tick,
    uint128 liquidityDelta
) internal {
    Tick.Info storage tickInfo = self[tick];
    uint128 liquidityBefore = tickInfo.liquidity;
    uint128 liquidityAfter = liquidityBefore + liquidityDelta;

    if (liquidityBefore == 0) {
        tickInfo.initialized = true;
    }

    tickInfo.liquidity = liquidityAfter;
}
```

如果 tick 的流动性为 0，它会初始化该 tick，并向其添加新的流动性。如你所见，我们在上下限 tick 上都调用了这个函数，因此流动性被添加到两者中。

position.update 函数是：

```solidity
// src/libs/Position.sol
function update(Info storage self, uint128 liquidityDelta) internal {
    uint128 liquidityBefore = self.liquidity;
    uint128 liquidityAfter = liquidityBefore + liquidityDelta;

    self.liquidity = liquidityAfter;
}
```

类似于 tick 更新函数，它向特定头寸添加流动性。要获取头寸，我们调用：

```solidity
// src/libs/Position.sol
...
function get(
    mapping(bytes32 => Info) storage self,
    address owner,
    int24 lowerTick,
    int24 upperTick
) internal view returns (Position.Info storage position) {
    position = self[
        keccak256(abi.encodePacked(owner, lowerTick, upperTick))
    ];
}
...
```

每个头寸都由三个键唯一标识：所有者地址、下限 tick 索引和上限 tick 索引。我们对这三个进行哈希处理以使数据存储更便宜：当哈希处理后，每个键将占用 32 字节，而不是当 owner、lowerTick 和 upperTick 是单独的键时占用 96 字节。

如果我们使用三个键，我们需要三个映射。每个键将单独存储，并且会占用 32 字节，因为 Solidity 将值存储在 32 字节的槽中（当不应用打包时）。

接下来，继续铸造，我们需要计算用户必须存入的数量。幸运的是，我们已经在前面的部分中弄清楚了公式并计算了确确切的数量。所以，我们将硬编码它们：

```solidity
amount0 = 0.998976618347425280 ether;
amount1 = 5000 ether;
```

我们将在后面的章节中用实际计算替换这些。

我们还将根据添加的 amount 更新池子的 liquidity。

```solidity
liquidity += uint128(amount);
```

现在，我们准备从用户那里获取代币。这是通过回调完成的：

```solidity
function mint(...) ... {
    ...

    uint256 balance0Before;
    uint256 balance1Before;
    if (amount0 > 0) balance0Before = balance0();
    if (amount1 > 0) balance1Before = balance1();
    IUniswapV3MintCallback(msg.sender).uniswapV3MintCallback(
        amount0,
        amount1
    );
    if (amount0 > 0 && balance0Before + amount0 > balance0())
        revert InsufficientInputAmount();
    if (amount1 > 0 && balance1Before + amount1 > balance1())
        revert InsufficientInputAmount();

    ...
}

function balance0() internal returns (uint256 balance) {
    balance = IERC20(token0).balanceOf(address(this));
}

function balance1() internal returns (uint256 balance) {
    balance = IERC20(token1).balanceOf(address(this));
}
```

首先，我们记录当前的代币余额。然后我们在调用者上调用 uniswapV3MintCallback 方法——这就是回调。预期调用者（无论谁调用 mint）是一个合约，因为在以太坊中非合约地址无法实现函数。在这里使用回调，虽然完全不友好，但让合约能够使用其当前状态计算代币数量——这是至关重要的，因为我们不能信任用户。

预期调用者实现 uniswapV3MintCallback 并在此函数中将代币转移到 Pool 合约。调用回调函数后，我们继续检查 Pool 合约余额是否发生变化：我们要求它们分别至少增加 amount0 和 amount1 ——这意味着调用者已将代币转移到池子。

最后，我们触发一个 Mint 事件：

```solidity
emit Mint(msg.sender, owner, lowerTick, upperTick, amount, amount0, amount1);
```

事件是以太坊中如何索引合约数据以供后续搜索的方式。每当合约状态发生变化时触发事件是一个好习惯，让区块链浏览器知道这何时发生。事件还携带有用信息。在我们的情况下，它是调用者的地址、流动性头寸所有者的地址、上下限 tick、新的流动性和代币数量。这些信息将作为日志存储，其他人将能够收集所有合约事件并重现合约的活动，而无需遍历和分析所有区块和交易。

我们完成了！呼！现在，让我们测试铸造。

## 测试

在这一点上，我们不知道一切是否正常工作。在将我们的合约部署到任何地方之前，我们将编写一系列测试以确保合约正常工作。幸运的是，Forge 是一个很棒的测试框架，它将使测试变得轻而易举。

创建一个新的测试文件：

```solidity
// test/UniswapV3Pool.t.sol
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.14;

import "forge-std/Test.sol";

contract UniswapV3PoolTest is Test {
    function setUp() public {}

    function testExample() public {
        assertTrue(true);
    }
}
```

让我们运行它：

```shell
$ forge test
Running 1 test for test/UniswapV3Pool.t.sol:UniswapV3PoolTest
[PASS] testExample() (gas: 279)
Test result: ok. 1 passed; 0 failed; finished in 5.07ms
```

它通过了！当然，它会通过！到目前为止，我们的测试只检查 true 是否为 true！

测试合约只是继承自 forge-std/Test.sol 的合约。这个合约是一组测试工具，我们将逐步熟悉它们。如果你不想等待，打开 lib/forge-std/src/Test.sol 并浏览一下。

测试合约遵循特定的约定：

setUp 函数用于设置测试用例。在每个测试用例中，我们希望有一个配置好的环境，比如部署的合约、铸造的代币和初始化的池子——我们将在 setUp 中完成所有这些。

每个测试用例都以 test 前缀开始，例如 testMint()。这将让 Forge 区分测试用例和辅助函数（我们也可以有任何我们想要的函数）。

现在让我们实际测试铸造。

测试代币
要测试铸造，我们需要代币。这不是问题，因为我们可以在测试中部署任何合约！此外，Forge 可以将开源合约安装为依赖项。具体来说，我们需要一个具有铸造功能的 ERC20 合约。我们将使用 Solmate（一个气体优化合约集合）中的 ERC20 合约，并制作一个继承自 Solmate 合约并公开铸造功能的 ERC20 合约（默认情况下是公开的）。

让我们安装 solmate：

```shell
$ forge install rari-capital/solmate
```

然后，让我们在 test 文件夹中创建 ERC20Mintable.sol 合约（我们只会在测试中使用这个合约）：

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.14;

import "solmate/tokens/ERC20.sol";

contract ERC20Mintable is ERC20 {
    constructor(
        string memory _name,
        string memory _symbol,
        uint8 _decimals
    ) ERC20(_name, _symbol, _decimals) {}

    function mint(address to, uint256 amount) public {
        _mint(to, amount);
    }
}
```

我们的 ERC20Mintable 继承了 solmate/tokens/ERC20.sol 的所有功能，我们额外实现了公共的 mint 方法，这将允许我们铸造任意数量的代币。

### 铸造

现在，我们准备测试铸造。

首先，让我们部署所有必需的合约：

```solidity
// test/UniswapV3Pool.t.sol
...
import "./ERC20Mintable.sol";
import "../src/UniswapV3Pool.sol";

contract UniswapV3PoolTest is Test {
    ERC20Mintable token0;
    ERC20Mintable token1;
    UniswapV3Pool pool;

    function setUp() public {
        token0 = new ERC20Mintable("Ether", "ETH", 18);
        token1 = new ERC20Mintable("USDC", "USDC", 18);
    }

    ...
```

在 setUp 函数中，我们部署代币但不部署池子！这是因为我们所有的测试用例都将使用相同的代币，但每个测试用例都将有一个唯一的池子。

为了使池子的设置更清晰和简单，我们将在一个单独的函数 setupTestCase 中完成这个操作，该函数接受一组测试用例参数。在我们的第一个测试用例中，我们将测试成功的流动性铸造。以下是测试用例参数的样子：

```solidity
function testMintSuccess() public {
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
```

1. 我们计划向池子存入 1 ETH 和 5000 USDC。
2. 我们希望当前 tick 为 85176，下限和上限 tick 分别为 84222 和 86129（我们在上一章中计算了这些值）。
3. 我们指定了预先计算的流动性和当前的 $\sqrt{P}$。
4. 我们还希望存入流动性（mintLiquidity 参数）并在池子合约请求时转移代币（shouldTransferInCallback）。我们不想在每个测试用例中都这样做，所以我们希望有这些标志。
 接下来，我们用上述参数调用 setupTestCase：

```solidity
function setupTestCase(TestCaseParams memory params)
    internal
    returns (uint256 poolBalance0, uint256 poolBalance1)
{
    token0.mint(address(this), params.wethBalance);
    token1.mint(address(this), params.usdcBalance);

    pool = new UniswapV3Pool(
        address(token0),
        address(token1),
        params.currentSqrtP,
        params.currentTick
    );

    if (params.mintLiqudity) {
        (poolBalance0, poolBalance1) = pool.mint(
            address(this),
            params.lowerTick,
            params.upperTick,
            params.liquidity
        );
    }

    shouldTransferInCallback = params.shouldTransferInCallback;
}
```

在这个函数中，我们铸造代币并部署池子。此外，当设置 mintLiquidity 标志时，我们在池子中铸造流动性。最后，我们设置 shouldTransferInCallback 标志，以便在铸造回调中读取：

```solidity
function uniswapV3MintCallback(uint256 amount0, uint256 amount1) public {
    if (shouldTransferInCallback) {
        token0.transfer(msg.sender, amount0);
        token1.transfer(msg.sender, amount1);
    }
}
```

是测试合约提供流动性并在池子上调用 mint 函数，没有用户。测试合约将充当用户，因此它可以实现铸造回调函数。

像这样设置测试用例并不是强制性的，你可以按照最舒适的方式来做。测试合约只是合约。

在 testMintSuccess 中，我们希望确保池子合约：

1. 从我们这里获取正确数量的代币；
2. 创建具有正确键和流动性的头寸；
3. 初始化我们指定的上限和下限 tick；
4. 具有正确的 $\sqrt{P}$ 和 $L$。
让我们来做这个。

铸造发生在 setupTestCase 中，所以我们不需要再次执行。该函数还返回我们提供的数量，所以让我们检查它们：

```solidity
(uint256 poolBalance0, uint256 poolBalance1) = setupTestCase(params);

uint256 expectedAmount0 = 0.998976618347425280 ether;
uint256 expectedAmount1 = 5000 ether;
assertEq(
    poolBalance0,
    expectedAmount0,
    "incorrect token0 deposited amount"
);
assertEq(
    poolBalance1,
    expectedAmount1,
    "incorrect token1 deposited amount"
);
```

我们期望特定的预先计算的数量。我们还可以检查这些数量是否已转移到池子：

```solidity
assertEq(token0.balanceOf(address(pool)), expectedAmount0);
assertEq(token1.balanceOf(address(pool)), expectedAmount1);
```

接下来，我们需要检查池子为我们创建的头寸。还记得 positions 映射中的键是一个哈希吗？我们需要手动计算它，然后从合约中获取我们的头寸：

```solidity
bytes32 positionKey = keccak256(
    abi.encodePacked(address(this), params.lowerTick, params.upperTick)
);
uint128 posLiquidity = pool.positions(positionKey);
assertEq(posLiquidity, params.liquidity);
```

由于 Position.Info 是一个 结构体，当获取时它会被解构：每个字段都被分配给一个单独的变量。

接下来是 ticks。同样，这很简单：

```solidity
(bool tickInitialized, uint128 tickLiquidity) = pool.ticks(
    params.lowerTick
);
assertTrue(tickInitialized);
assertEq(tickLiquidity, params.liquidity);

(tickInitialized, tickLiquidity) = pool.ticks(params.upperTick);
assertTrue(tickInitialized);
assertEq(tickLiquidity, params.liquidity);
```

最后，$\sqrt{P}$ 和 $L$：

```solidity
(uint160 sqrtPriceX96, int24 tick) = pool.slot0();
assertEq(
    sqrtPriceX96,
    5602277097478614198912276234240,
    "invalid current sqrtP"
);
assertEq(tick, 85176, "invalid current tick");
assertEq(
    pool.liquidity(),
    1517882343751509868544,
    "invalid current liquidity"
);
```

如你所见，用 Solidity 编写测试并不难！

## 失败情况

当然，仅测试成功场景是不够的。我们还需要测试失败的情况。提供流动性时可能出现什么问题？这里有几个提示：

1. 上限和下限 tick 太大或太小。
2. 提供了零流动性。
3. 流动性提供者没有足够的代币。

我将让你来实现这些场景！随时查看![仓库](https://github.com/Jeiwan/uniswapv3-code/blob/milestone_1/test/UniswapV3Pool.t.sol)中的代码。