---
title: "Providing Liquidity"
weight: 3
# bookFlatSection: false
# bookToc: false
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

{{< katex display >}} {{</ katex >}}

# Providing Liquidity

Enough of theory, let's start coding!

Create a new folder (mine is called `uniswapv3-code`), and run `forge init --vscode` in it–this will initialize a Forge
project. The `--vscode` flag tells Forge to configure the Solidity extension for Forge projects.

Next, remove the default contract and its test:
- `script/Contract.s.sol`
- `src/Contract.sol`
- `test/Contract.t.sol`

And that's it! Let's create our first contract!

## Pool Contract

As you've learned from the introduction, Uniswap deploys multiple Pool contracts, each of which is an exchange market of
a pair of tokens. Uniswap groups all its contract into two categories:

- core contracts, and
- periphery contracts.

Core contracts are, as the name implies, the contracts that implement core logic. These are minimal, user-*un*friendly,
low-level contracts. Their purpose is to do one thing. In Uniswap V3, there are 2 such contracts:
1. Pool contract, which implements the core logic of a decentralized exchange.
1. Factory contract, which serves as a registry of Pool contracts and a contract that makes deployment of pools easier.

We'll begin with the pool contract. Create `src/UniswapV3Pool.sol`:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.14;

contract UniswapV3Pool {}
```

Let's think about what data the contract will store:
1. Since every pool contract is an exchange market of two tokens, we need to track the two token addresses. And these
addresses will be static, set once and forever during contract initialization.
1. Each pool contract is a set of liquidity positions, a data structure to manage positions identified by: liquidity
provider's address, and upper and lower bounds of the position.
1. Each pool contract will also need to maintain a ticks registry and information about each tick–the amount of liquidity
provided by each tick.
1. Since the tick range is limited, we need to store the limits in the contract, as constants.
1. And as we discussed in the introduction, pool contracts store the amount of liquidity, $L$, and $\sqrt{P}$ instead of
token reserves. So we'll need to store them in the contract as well.

Here's what our pool contract with all the state variables:

```solidity
contract UniswapV3Pool {
    using Tick for mapping(int24 => Tick.Info);
    using Position for mapping(bytes32 => Position.Info);
    using Position for Position.Info;

    int24 internal constant MIN_TICK = -887272;
    int24 internal constant MAX_TICK = -MIN_TICK;

    // Pool tokens, immutable
    address public immutable token0;
    address public immutable token1;

    // First slot will contain essential data
    struct Slot0 {
        // Current sqrt(P)
        uint160 sqrtPriceX96;
        // Current tick
        int24 tick;
    }
    Slot0 public slot0;

    // Amount of liquidity, L.
    uint128 public liquidity;

    // Ticks info
    mapping(int24 => Tick.Info) public ticks;
    // Positions info
    mapping(bytes32 => Position.Info) public positions;

    ...
```

Uniswap V3 uses many helper contracts and `Tick` and `Position` are two of them. `using A for B` is a feature of Solidity
that lets you extend type `B` with functions from library contract `A`. This simplifies managing complex data structures.

> For brevity, I'll omit detailed explanation of Solidity syntax and features. Solidity has [great documentation](https://docs.soliditylang.org/en/latest/),
don't hesitate referring to it if something is not clear!

We'll then initialize them in the constructor:

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

Notice that we're passing $\sqrt{P}$ and the current tick index without providing liquidity–this sets the current price.

> We're setting both $\sqrt{P}$ and tick index in the constructor for simplicity. Later on, we'll implement the conversion
between $\sqrt{P}$ and tick indexes.

This is our starting point, and our goal in this chapter is to make our first swap.

## Minting

The process of providing liquidity in Uniswap V2 is called *minting*. The reason is that the V2 pool contract mints
tokens (LP-tokens) in exchange for liquidity. V3 doesn't do that, but it still uses the same name for the function. Let's
use it as well:

```solidity
function mint(
    address owner,
    int24 lowerTick,
    int24 upperTick,
    uint128 amount
) external returns (uint256 amount0, uint256 amount1) {
    ...
```

Our `mint` function will take:
1. Owner's address, to track the owner of the liquidity.
1. Upper and lower ticks, to set the bounds of a price range.
1. The amount of liquidity we have provided.

When adding initial liquidity to a pool, this function adds a new tick and a position.

We begin with checking the ticks:
```solidity
if (
    lowerTick >= upperTick ||
    lowerTick < MIN_TICK ||
    upperTick > MAX_TICK
) revert InvalidTickRange();
```

And ensuring that some amount of liquidity is provided:
```solidity
if (amount == 0) revert ZeroLiquidity();
```

Then, add a tick and a position:
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

The `ticks.update` function is:
```solidity
// src/libs/Tick.sol
...
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
...
```
It initialized a tick if it has 0 liquidity before and adds new liquidity to it. As you can see, we're calling this
function on both lower and upper ticks, thus liquidity is added to both of them–we'll see why later on.

The `position.update` function is:
```solidity
// src/libs/Position.sol
function update(Info storage self, uint128 liquidityDelta) internal {
    uint128 liquidityBefore = self.liquidity;
    uint128 liquidityAfter = liquidityBefore + liquidityDelta;

    self.liquidity = liquidityAfter;
}
```
Similar to the tick update function, it adds liquidity to a specific position. And to get a position we call:
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

Each position is uniquely identified by three keys: owner address, lower tick index, and upper tick index. We're storing
positions in a `bytes32 => Info` map and are using hashes of concatenated owner address, lower tick, and upper tick as
keys. This is cheaper than storing three nested maps.

We're not done yet! Next, we need to calculate the amounts that the user must deposit. Luckily, we have 

part: we don't want to calculate $\sqrt{x*y}$ in the contract, so we need to calculate the token amounts corresponding to
the price range and the amount of liquidity selected by the user.

```solidity
amount0 = SqrtPriceMath.getAmount0Delta(
    _slot0.sqrtPriceX96,
    14214576466144435, // upperTick, sqrt(p(i_c+100)), wei
    amount
);

amount1 = SqrtPriceMath.getAmount1Delta(
    14073146103223374, // lowerTick, sqrt(p(i_c-100)), wei
    _slot0.sqrtPriceX96,
    amount
);
```

Two values are hardcoded in this piece:
1. `14214576466144435` is the square root of the price at the upper bound.
1. `14073146103223374` is the square root of the price at the lower bound.

We do this so we don't need to implement the tick-to-price calculation in the contract to not over-complicate this chapter.
We'll return to this a little bit later!

Let's look at `getAmount0Delta`:
```solidity
// src/lib/SqrtPriceMath.sol
function getAmount0Delta(
    uint160 sqrtPriceAX96,
    uint160 sqrtPriceBX96,
    uint128 liquidity
) internal pure returns (uint256) {
    if (sqrtPriceAX96 > sqrtPriceBX96)
        (sqrtPriceAX96, sqrtPriceBX96) = (sqrtPriceBX96, sqrtPriceAX96);

    uint256 numerator1 = uint256(liquidity) << FixedPoint96.RESOLUTION;
    uint256 numerator2 = sqrtPriceBX96 - sqrtPriceAX96;

    require(sqrtPriceAX96 > 0);

    return
        Math.divRoundingUp(
            numerator1 * numerator2,
            (sqrtPriceAX96 * sqrtPriceBX96)
        );
}
```

This logic implements the $\Delta x$ formula we found in the theoretical part. First, we sort the prices so the difference
between them is positive. Then, we unpack `liquidity`: it's a Q64.96 number, we want to turn it into a `uint256` for
further calculations. Then, we implement the $\Delta x$ formula and round the result up to nearest integer–we're doing
this because we want the amount to include the full price range. If we don't do this, the amount might be too low to satisfy
the swap along the full price range.

Now, let's look at `getAmount1Delta`:
```solidity
// src/lib/SqrtPriceMath.sol
function getAmount1Delta(
    uint160 sqrtPriceAX96,
    uint160 sqrtPriceBX96,
    uint128 liquidity
) internal pure returns (uint256) {
    if (sqrtPriceAX96 > sqrtPriceBX96)
        (sqrtPriceAX96, sqrtPriceBX96) = (sqrtPriceBX96, sqrtPriceAX96);

    return
        Math.mulDivRoundingUp(
            liquidity,
            sqrtPriceBX96 - sqrtPriceAX96,
            FixedPoint96.Q96
        );
}
```
As you might've guessed, it implements the $\Delta y$ formula. The result is divided by `FixedPoint96.Q96` to adjust
for the fractional part of `liquidity`.


Now, we're ready to take tokens from user. This is done via a callback:
```solidity
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
```

First, we record current token balances (if either of them is deposited). Then we call `uniswapV3MintCallback` method
on the caller–this is the callback. It's expected that the caller (whoever executes `mint`) is a contract because
non-contract addresses cannot implement functions in Ethereum. There are two reason doing this like that:

1. Pool contract is a core contract, and core contracts are user-unfriendly. It's expected that core contracts are only
user by other contracts which make interaction with a pool easier.
1. We don't want to calculate the square root of reserves in the contract because it's an expensive operation. But we still
need to be sure that the liquidity deposited by user is correct. To achieve this, we calculate $\Delta x$ and $\Delta y$,
which doesn't require calculating square roots. But this approach forces us to use a callback to let the caller know the
actual amounts they need to deposit.

In production, Pool contracts are called from the Router contract, which handles all the nuances. We'll implement it in
a later chapter.

Finally, we're firing a `Mint` event:
```solidity
emit Mint(msg.sender, owner, lowerTick, upperTick, amount, amount0, amount1);
```

Events is how contract data is indexed in Ethereum for later search. It's a good practice to fire an event whenever
contract's state is changed to let blockchain explorer know when this happened. Events also carry useful information.
In our case it's: caller's address, liquidity position owner's address, upper and lower ticks, new liquidity, and token
amounts. This information will be stored as a log, and anyone else will be able to collect all contract events and
reproduce activity of the contract without traversing and analyzing all blocks and transactions.

And we're done! Phew! Now, let's test minting.

## Testing

At this point we don't know if everything works correctly. Before deploying our contract anywhere we're going to write
a bunch of tests to ensure the contract works correctly. Luckily to us, Forge is a great testing framework and it'll
make testing a breeze. 

Create a new test file:
```solidity
//test/UniswapV3Pool.t.sol
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

Let's run it:
```shell
$ forge test
Running 1 test for test/UniswapV3Pool.t.sol:UniswapV3PoolTest
[PASS] testExample() (gas: 279)
Test result: ok. 1 passed; 0 failed; finished in 5.07ms
```

It passes! Of course it is! So far, our test only checks that `true` is `true`!

Test contract are just contract that inherit from `forge-std/Test.sol`. This contract is a set of testing utilities, we'll
get acquainted with them step by step. If you don't want wait, open `lib/forge-std/src/Test.sol` and skim through it!

Test contracts follow a specific convention:
1. `setUp` function is used to set up test cases. In each test cases, we want to have configured environment, like
deployed contracts, minted tokens, initialized pools–we'll do all this in `setUp`.
1. Every test case starts with `test` prefix, e.g. `testMint()`. This will let Forge distinguish test cases from helper
functions (we can add any function we want).

Let's test minting!

### Test Tokens

To test minting we need tokens. This is not a problem because we can deploy any contract in tests! Moreover, Forge can
install open-source contracts as dependencies. Specifically, we need an ERC20 contract with minting functionality. We'll
use the ERC20 contract from [solmate](https://github.com/Rari-Capital/solmate), a collection of gas-optimized contracts
(however, we don't care about gas optimization at this moment) and we'll extend it in a contract that allows public
minting.

Let's install `solmate`:
```shell
$ forge install rari-capital/solmate
```

Then, let's create `ERC20Mintable.sol` contract in `test` folder (we'll use the contract only in tests):
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

Our `ERC20Mintable` inherits all functionality from `solmate/tokens/ERC20.sol` and we additionally implement public `mint`
method which will allows us to mint any number of tokens.

### Minting

Now, we're ready to test minting.

First, let's deploy all the required contracts:
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

        pool = new UniswapV3Pool(
            address(token0),
            address(token1),
            14143684505937315, // 0.0002 ETH/USDC, 1.0001^(85174/2)
            85174
        );
    }

    ...
```
First, we deploy ETH and USDC tokens. They both have 18 decimals. After that, we deploy and initialize our pool.