# 开发环境

我们将构建两个应用程序：

1. 一个链上应用：部署在以太坊上的一组智能合约。
2. 一个链下应用：与智能合约交互的前端应用程序。

虽然前端应用程序的开发是本书的一部分，但它不会是我们的主要焦点。我们构建它仅仅是为了演示如何将智能合约与前端应用程序集成。因此，前端应用程序是可选的，但我仍会提供代码。

## 以太坊简介

以太坊是一个允许任何人在其上运行应用程序的区块链。它可能看起来像一个云服务提供商，但有几个不同之处：
1. 你不需要为托管应用程序付费，但你需要为部署付费。
2. 你的应用程序是不可变的。也就是说：部署后你将无法修改它。
3. 用户将为使用你的应用程序付费。

为了更好地理解这些要点，让我们看看以太坊由什么组成。

以太坊（和任何其他区块链）的核心是一个数据库。以太坊数据库中最有价值的数据是*账户状态*。账户是一个以太坊地址，与之相关的数据包括：

1. 余额：账户的以太币余额。
2. 代码：部署在此地址的智能合约的字节码。
3. 存储：智能合约用来存储数据的空间。
4. Nonce：用于防止重放攻击的序列整数。

以太坊的主要工作是以安全的方式构建和维护这些数据，不允许未经授权的访问。

以太坊也是一个网络，一个独立构建和维护状态的计算机网络。网络的主要目标是**去中心化数据库访问**：不能有单一权威可以单方面修改数据库中的任何内容。这是通过*共识*实现的，共识是网络中所有节点遵循的一组规则。如果一方决定滥用规则，它将被排除在网络之外。

> 有趣的事实：区块链可以使用 MySQL！除了性能之外，没有什么能阻止这一点。而以太坊使用 [LevelDB](https://github.com/google/leveldb)，一个快速的键值数据库。

每个以太坊节点还运行 EVM，即以太坊虚拟机。虚拟机是一个可以运行其他程序的程序，而 EVM 是一个执行智能合约的程序。用户通过交易与合约交互：除了简单地发送以太币外，交易还可以包含智能合约调用数据。它包括：

1. 编码的合约函数名。
2. 函数参数。

交易被打包成区块，然后由矿工挖掘区块。网络中的每个参与者都可以验证任何交易和任何区块。

从某种意义上说，智能合约类似于 JSON API，但你调用的是智能合约函数而不是端点，并提供函数参数。与 API 后端类似，智能合约执行编程逻辑，可以选择修改智能合约存储。与 JSON API 不同，你需要发送交易来改变区块链状态，并且你需要为发送的每个交易付费。

最后，以太坊节点暴露了一个 JSON-RPC API。通过这个 API，我们可以与节点交互：获取账户余额、估算 gas 成本、获取区块和交易、发送交易，以及执行合约调用而不发送交易（这用于从智能合约读取数据）。[这里](https://eth.wiki/json-rpc/API)你可以找到可用端点的完整列表。

> 交易也是通过 JSON-RPC API 发送的，参见 [eth_sendTransaction](https://ethereum.org/en/developers/docs/apis/json-rpc/#eth_sendtransaction)。

## 本地开发环境

目前使用的智能合约开发环境有多种：
1. [Truffle](https://trufflesuite.com)
2. [Hardhat](https://hardhat.org)
3. [Foundry](https://github.com/foundry-rs/foundry)

Truffle 是三者中最古老的，也是最不受欢迎的。Hardhat 是它改进后的后代，是最广泛使用的工具。Foundry 是新秀，它对测试提供了不同的视角。

虽然 HardHat 仍然是一个流行的解决方案，但越来越多的项目正在转向 Foundry。这有多个原因：
1. 使用 Foundry，我们可以用 Solidity 编写测试。这更加方便，因为在开发过程中我们不需要在 JavaScript（Truffle 和 HardHat 使用 JS 进行测试和自动化）和 Solidity 之间切换。用 Solidity 编写测试更加方便，因为你拥有所有的原生功能（例如，你不需要为大数使用特殊类型，也不需要在字符串和 [BigNumber](https://docs.ethers.io/v5/api/utils/bignumber/) 之间转换）。
2. Foundry 在测试期间不运行节点。这使得测试和迭代功能更快！Truffle 和 HardHat 在运行测试时都会启动一个节点；Foundry 在内部 EVM 上执行测试。

考虑到这些因素，我们将使用 Foundry 作为我们主要的智能合约开发和测试工具。

### Foundry

[Foundry](https://github.com/foundry-rs/foundry) 是一套用于以太坊应用程序开发的工具。具体来说，我们将使用：
1. [Forge](https://github.com/foundry-rs/foundry/tree/master/forge)，一个 Solidity 测试框架。
2. [Anvil](https://github.com/foundry-rs/foundry/tree/master/anvil)，一个为与 Forge 开发而设计的本地以太坊节点。我们将使用它来将我们的合约部署到本地节点，并通过前端应用程序连接到它。
3. [Cast](https://github.com/foundry-rs/foundry/tree/master/cast)，一个具有大量有用功能的 CLI 工具。

Forge 让智能合约开发者的生活变得更加轻松。使用 Forge，我们不需要运行本地节点来测试合约。相反，Forge 在其内部 EVM 上运行测试，这更快，不需要发送交易和挖掘区块。

Forge 让我们可以用 Solidity 编写测试！Forge 还使模拟区块链状态变得更容易：我们可以轻松地伪造我们的以太币或代币余额，从其他地址执行合约，在任何地址部署任何合约等。

然而，我们仍然需要一个本地节点来部署我们的合约。为此，我们将使用 Anvil。前端应用程序使用 JavaScript Web3 库与以太坊节点交互（发送交易、查询状态、估算交易 gas 成本等）——这就是为什么我们需要运行一个本地节点。

### Ethers.js

[Ethers.js](https://github.com/ethers-io/ethers.js/) 是一套用 JavaScript 编写的以太坊实用工具。这是去中心化应用程序开发中使用的两个最流行的 JavaScript 库之一（另一个是 [web3.js](https://github.com/ChainSafe/web3.js)）。这些库允许我们通过 JSON-API 与以太坊节点交互，并提供多个实用函数，使开发者的生活更轻松。

### MetaMask

[MetaMask](https://metamask.io/) 是你浏览器中的以太坊钱包。它是一个浏览器扩展，可以创建和安全存储私钥。MetaMask 是数百万用户使用的主要以太坊钱包应用程序。我们将使用它来签署我们将发送到本地节点的交易。

### React

[React](https://reactjs.org/) 是一个著名的用于构建前端应用程序的 JavaScript 库。你不需要了解 React，我会提供一个模板应用程序。

## 设置项目

要设置项目，创建一个新文件夹并在其中运行 `forge init`：
```shell
$ mkdir uniswapv3clone
$ cd uniswapv3clone
$ forge init
  ```

> 如果你使用的是 Visual Studio Code，在 forge init 中添加 `--vscode` 标志：`forge init --vscode`。Forge 将使用 VSCode 特定的设置初始化项目。

Forge 将在 `src`、`test` 和 `script` 文件夹中创建示例合约——这些可以删除。

要设置前端应用程序： 
```shell
$ npx create-react-app ui
```

它位于一个子文件夹中，这样就不会与文件夹名称发生冲突。
