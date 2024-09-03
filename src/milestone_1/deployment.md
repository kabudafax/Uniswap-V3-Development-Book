# 部署

好了，我们的资金池合约已经完成。现在，让我们看看如何将它部署到本地以太坊网络，以便稍后可以从前端应用程序中使用它。

## 选择本地区块链网络

智能合约开发需要运行一个本地网络，在开发和测试期间将合约部署到该网络。以下是我们对这样一个网络的要求：

1. 真实区块链。它必须是一个真实的以太坊网络，而不是模拟。我们希望确保我们的合约在本地网络中的运行方式与在主网中完全相同。

2. 速度。我们希望交易能立即被打包，这样我们可以快速迭代。

3. 以太币。为了支付交易费用，我们需要一些以太币，并且我们希望本地网络允许我们生成任意数量的以太币。

4. 作弊码。除了提供标准API外，我们还希望本地网络允许我们做更多的事情。例如，我们希望能够在任何地址部署合约、从任何地址执行交易（模拟其他地址）、直接更改合约状态等。

目前有多种解决方案：

1. [Ganache](https://trufflesuite.com/ganache/) 来自 Truffle Suite。

2. [Hardhat](https://hardhat.org/)，这是一个开发环境，除了其他有用的功能外，还包括一个本地节点。

3. [Anvil](https://github.com/foundry-rs/foundry/tree/master/anvil) 来自 Foundry。

这些都是可行的解决方案，每一个都能满足我们的需求。话虽如此，项目已经慢慢从 Ganache（这是最古老的解决方案）迁移到 Hardhat（现在似乎是使用最广泛的），而现在又出现了新的选择：Foundry。Foundry 也是这些解决方案中唯一使用 Solidity 编写测试的（其他使用 JavaScript）。此外，Foundry 还允许用 Solidity 编写部署脚本。因此，既然我们决定在所有地方都使用 Solidity，我们将使用 Anvil 来运行本地开发区块链，并用 Solidity 编写部署脚本。

## 运行本地区块链

Anvil 不需要配置，我们可以用一个简单的命令运行它：

```shell
$ anvil --code-size-limit 50000

                             _   _
                            (_) | |
      __ _   _ __   __   __  _  | |
     / _` | | '_ \  \ \ / / | | | |
    | (_| | | | | |  \ V /  | | | |
     \__,_| |_| |_|   \_/   |_| |_|

    0.1.0 (d89f6af 2022-06-24T00:15:17.897682Z)
    https://github.com/foundry-rs/foundry
...
Listening on 127.0.0.1:8545

```
我们将编写不适合以太坊合约大小限制（24576 字节）的大型合约，因此我们需要告诉 Anvil 允许更大的智能合约。

Anvil 运行一个单独的以太坊节点，所以这不是一个网络，但这没关系。默认情况下，它创建 10 个账户，每个账户有 10,000 ETH。它在启动时打印地址和相关的私钥——我们在从 UI 部署和与合约交互时将使用其中一个地址。

Anvil 在 127.0.0.1:8545 暴露 JSON-RPC API 接口——这个接口是与以太坊节点交互的主要方式。你可以在这里找到完整的 API 参考。以下是如何通过 curl 调用它：

```shell
$ curl -X POST -H 'Content-Type: application/json' \
  --data '{"id":1,"jsonrpc":"2.0","method":"eth_chainId"}' \
  http://127.0.0.1:8545
{"jsonrpc":"2.0","id":1,"result":"0x7a69"}
$ curl -X POST -H 'Content-Type: application/json' \
  --data '{"id":1,"jsonrpc":"2.0","method":"eth_getBalance","params":["0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266","latest"]}' \
  http://127.0.0.1:8545
{"jsonrpc":"2.0","id":1,"result":"0x21e19e0c9bab2400000"}

```
你也可以使用 cast（Foundry 的一部分）来做这个：

```shell
$ cast chain-id
31337
$ cast balance 0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266
10000000000000000000000
```

现在，让我们将资金池和管理器合约部署到本地网络。

## 首次部署

本质上，部署合约意味着：

1. 将源代码编译成 EVM 字节码。
2. 发送一个包含字节码的交易。
3. 创建一个新地址，执行字节码的构造函数部分，并在该地址上存储部署的字节码。当您的合约创建交易被挖矿时，以太坊节点会自动完成这一步。

部署通常包括多个步骤：准备参数、部署辅助合约、部署主合约、初始化合约等。脚本可以帮助自动化这些步骤，我们将用 Solidity 编写脚本！

创建 `scripts/DeployDevelopment.sol` 合约，内容如下：

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.14;

import "forge-std/Script.sol";

contract DeployDevelopment is Script {
    function run() public {
      ...
    }
}

```

它看起来与测试合约非常相似，唯一的区别是它继承自 Script 合约，而不是 Test。按照惯例，我们需要定义 run 函数，它将作为我们部署脚本的主体。在 run 函数中，我们首先定义部署参数：

```solidity
uint256 wethBalance = 1 ether;
uint256 usdcBalance = 5042 ether;
int24 currentTick = 85176;
uint160 currentSqrtP = 5602277097478614198912276234240;
```

这些是我们之前使用的相同值。注意，我们将铸造 5042 USDC——其中 5000 USDC 将作为流动性提供给资金池，42 USDC 将在交换中出售。

接下来，我们定义将作为部署交易执行的一系列步骤（实际上，每个步骤都将是一个单独的交易）。为此，我们使用 startBroadcast/endBroadcast 作弊码：

```solidity
vm.startBroadcast();
...
vm.stopBroadcast();
```

这些作弊码是由 Foundry 提供的。我们通过继承 forge-std/Script.sol 在脚本合约中获得了它们。

在 broadcast() 作弊码之后或 startBroadcast()/stopBroadcast() 之间的所有内容都会被转换为交易，这些交易会被发送到执行脚本的节点。

在广播作弊码之间，我们将放置实际的部署步骤。首先，我们需要部署代币：

```solidity
ERC20Mintable token0 = new ERC20Mintable("Wrapped Ether", "WETH", 18);
ERC20Mintable token1 = new ERC20Mintable("USD Coin", "USDC", 18);
```

我们不能在没有代币的情况下部署资金池，所以我们需要先部署它们。

由于我们正在部署到本地开发网络，我们需要自己部署代币。在主网和公共测试网络（Ropsten、Goerli、Sepolia）中，代币已经创建。因此，要部署到这些网络，我们需要编写特定于网络的部署脚本。

下一步是部署资金池合约：

```solidity
UniswapV3Pool pool = new UniswapV3Pool(
    address(token0),
    address(token1),
    currentSqrtP,
    currentTick
);
```

接下来是 Manager 合约的部署：

```solidity
UniswapV3Manager manager = new UniswapV3Manager();
```

最后，我们可以为我们的地址铸造一些 ETH 和 USDC：

```solidity
token0.mint(msg.sender, wethBalance);
token1.mint(msg.sender, usdcBalance);
```

在 Foundry 脚本中，msg.sender 是在 broadcast 块内发送交易的地址。我们在运行脚本时可以设置它。

最后，在脚本的末尾，添加一些 console.log 调用来打印已部署合约的地址：

```solidity
console.log("WETH address", address(token0));
console.log("USDC address", address(token1));
console.log("Pool address", address(pool));
console.log("Manager address", address(manager));
```

好了，让我们运行脚本（确保 Anvil 在另一个终端窗口中运行）：

```shell
$ forge script scripts/DeployDevelopment.s.sol --broadcast --fork-url http://localhost:8545 --private-key $PRIVATE_KEY  --code-size-limit 50000
```

我们再次增加了智能合约代码大小，以便编译器不会失败。

--broadcast 启用交易的广播。默认情况下它不启用，因为并非每个脚本都发送交易。--fork-url 设置要发送交易的节点地址。--private-key 设置发送者钱包：需要私钥来签署交易。你可以选择 Anvil 启动时打印的任何一个私钥。我选择了第一个：

0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80

部署需要几秒钟。最后，你会看到它发送的交易列表。它还会将交易收据保存到 broadcast 文件夹。在 Anvil 中，你还会看到许多带有 eth_sendRawTransaction、eth_getTransactionByHash 和 eth_getTransactionReceipt 的行——在向 Anvil 发送交易后，Forge 使用 JSON-RPC API 来检查它们的状态并获取交易执行结果（收据）。

恭喜！你刚刚部署了一个智能合约！

## 与合约交互，ABI

现在，让我们看看如何与已部署的合约进行交互。

每个合约都暴露了一组公共函数。对于资金池合约，这些是 mint(...) 和 swap(...)。此外，Solidity 为公共变量创建 getter，所以我们也可以调用 token0()、token1()、positions() 等。然而，由于合约是编译后的字节码，函数名在编译过程中丢失，不存储在区块链上。相反，每个函数都由一个选择器标识，这是函数签名哈希的前 4 个字节。用伪代码表示：

```solidity
hash("transfer(address,address,uint256)")[0:4]
```

EVM 使用 Keccak 哈希算法，该算法被标准化为 SHA-3。具体来说，Solidity 中的哈希函数是 keccak256。

知道了这一点，让我们对已部署的合约进行两次调用：一次是通过 curl 进行低级调用，另一次使用 cast。

## 代币余额

让我们检查部署者地址的 WETH 余额。函数的签名是 balanceOf(address)（如 ERC-20 中定义的）。要找到这个函数的 ID（它的选择器），我们将对它进行哈希处理并取前四个字节：

```shell
$ cast keccak "balanceOf(address)"| cut -b 1-10
0x70a08231
```

要传递地址，我们只需将其附加到函数选择器后（并添加左填充到 32 位，因为地址在函数调用数据中占 32 字节）：

0x70a08231000000000000000000000000f39fd6e51aad88f6f4ce6ab8827279cfffb92266

0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266 是我们要检查余额的地址。这是我们的地址，Anvil 中的第一个账户。

接下来，我们执行 eth_call JSON-RPC 方法来进行调用。注意，这不需要发送交易——这个端点用于从合约中读取数据。

```shell
$ params='{"from":"0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266","to":"0xe7f1725e7734ce288f8367e1bb143e90bb3f0512","data":"0x70a08231000000000000000000000000f39fd6e51aad88f6f4ce6ab8827279cfffb92266"}'
$ curl -X POST -H 'Content-Type: application/json' \
  --data '{"id":1,"jsonrpc":"2.0","method":"eth_call","params":['"$params"',"latest"]}' \
  http://127.0.0.1:8545
{"jsonrpc":"2.0","id":1,"result":"0x00000000000000000000000000000000000000000000011153ce5e56cf880000"}

```

"to" 地址是 USDC 代币。它由部署脚本打印，在你的情况下可能不同。

以太坊节点返回原始字节结果，要解析它们，我们需要知道返回值的类型。对于 balanceOf 函数，返回值的类型是 uint256。使用 cast，我们可以将其转换为十进制数，然后转换为 ether：

```shell
$ cast --to-dec 0x00000000000000000000000000000000000000000000011153ce5e56cf880000| cast --from-wei
5042.000000000000000000

```

余额是正确的！我们向我们的地址铸造了 5042 USDC。

当前 Tick 和价格
上面的例子是低级合约调用的演示。通常，你不会通过 curl 进行调用，而是使用一个使其更容易的工具或库。Cast 可以在这里再次帮助我们！

让我们使用 cast 获取资金池的当前价格和 tick：

```shell
$ cast call POOL_ADDRESS "slot0()"| xargs cast --abi-decode "a()(uint160,int24)"
5602277097478614198912276234240
85176
```

很好！第一个值是当前的 $\sqrt{P}$，第二个值是当前的 tick。

由于 --abi-decode 需要完整的函数签名，我们必须指定 "a()"，即使我们只想解码函数输出。

ABI
为了简化与合约的交互，Solidity 编译器可以输出 ABI，即应用程序二进制接口。

ABI 是一个 JSON 文件，包含合约所有公共方法和事件的描述。这个文件的目的是使编码函数参数和解码返回值变得更容易。要使用 Forge 获取 ABI，使用以下命令：

```shell
$ forge inspect UniswapV3Pool abi
```

请随意浏览文件以更好地理解其内容。