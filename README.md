# ZKSwap design spec

Version: **1.23**, Update Date: **2020-11-28**


The design spec describes ZKSwap protocol and difference with ZKSync. Thanks a lot to ZKSync, which provides the ZK Rollup framework for ERC20 token transfer. ZKSwap is L2 token swap protocol using ZK Rollup on Ethereum and based on PlonK proof system.

### 1. Swap Protocol

The swap protocol is exactly same with Uniswap, but the swap path is limited to 1. That's the say, currently only one token can be swapped to another token if and only if the pair for those two tokens is created.

### 2. Account and Pair Account

The whole L2 state is represented by one merkle tree, which is leveled by Account and Token. The height of Account Merkle tree is ACCOUNT_MERKLE_DEPTH and the height of Token Merkle tree is TOKEN_MERKLE_DEPTH.

<img src="./figs/account.png" alt="account" style="zoom:36%;" />

The Account node include the following fields:

|       Name        |  Type  | Size (bytes) |             Comments             |
| :---------------: | :----: | :----------: | :------------------------------: |
|    PubKeyHash     |   -    |      20      | Layer2 Address（Public Key Hash) |
|      Address      |   -    |      20      |          Layer1 Address          |
|       NONCE       | uint32 |      4       |   Nonce of Layer2 Transaction    |
|   **Token0 ID** | uint16 |      2       |   Token0 ID   |
|   **Token1 ID** | uint16 |      2       |   Token1 ID   |
|       **Reserve0**       | uint128 |      16       |   Reserve value of Token0    |
|       **Reserve1**       | uint128 |      16       |   Reserve value of Token1     |
|   **LP token ID** | uint16 |      2       |   Liquidity Provider Token ID   |
|       **LP Total Amount**       | uint128 |      16       |   Total amount of Liquidity Provider Token     |
| Balance Tree Root |   -    |      Fr      |           Fr of Bn256            |

The Token0 ID, Token1 ID, Reserve0, Reserve1, LP token ID and LP Total Amount fields are added for swap releated operation, comparing with zkSync's Account. 

ZKSwap protocol supports two accounts type: one is Account and the other is Pair Account. For Account, the additional information is useless and empty. For Pair Account, the additional fields should be filled correctly and the **PubKeyHash field is fixed to zero**. 

### 3. Token Management

ZKSwap smart contract helps maintain one list of tokens. And the list is divided into two parts: one is for normal ETH/ERC20 tokens and the other is for LP ERC20 tokens.

#### a. Normal ERC20 token

Totally, 128 ERC20 tokens are supported with index from 0 to 127. The index 0 is reserved for ETH.

#### b. LP ERC20 token

Totally, 2048-128=1920 LP ERC20 tokens are supported and LP ERC20 token starts from index 128.

In Layer2, each token is assigned one ID. The ID is synchronized between L1 and L2 through ZK rollup transactions.

### 4. Swap Operations

Totally there are four swap related operations. CreatePair helps create swap pair, which composed by two different tokens. AddLiquidity and RemoveLiquidity operations are helps add and remove liquidity. Swap operation is used to swap one token to the other token.

#### a. CreatePair

CreatePair operation starts from L1 transaction. The ZKSwap smart contract helps create one pair ERC20 smart contract and one  associated LP token. The pair ERC20 smart contract address is used in L2 to identify the swap pair. And the LP token address is used in L2 binding with LP token ID.

<img src="./figs/create_pair.png" alt="create_pair" style="zoom:36%;" />

#### b. AddLiquidity

When one pair is created, one user can add liquidity to the pair by providing two specified assets. The AddLiqudity starts from L2.

<img src="./figs/add_liquidity.png" alt="add_liquidity" style="zoom:36%;" />

By adding liquidity, one account is rewarded by liquidity token. Even liquidity token is maintained in L2, one liquidity token can be withdraw to L1. 

#### c. RemoveLiquidity

Remove liquidity can be used to remove specified tokens from one pair. The RemoveLiquidity starts from L2, which is opposite with AddLiquidity.

#### d. Swap 

Swap is used to swap one token for another token, which provides by one pair account. From one pair account, the swap ratio depends on balances of two specified tokens.

<img src="./figs/swap.png" alt="swap" style="zoom:36%;" />

### 5. Circuit Design

The world state of L2 is executed by transactions and verified again by circuit. When one tx is applied on circuit, it is "divided" into OperationBranch and Operation Arguments. OperationBranch describe the merkel path. And Operation Arguments describes all other information besides branches.

<img src="./figs/witness_logic.png" alt="witness_logic" style="zoom:30%;" />

Operation is one important structure, which is the "smallest" circuit block.

<img src="./figs/operation.png" alt="operation" style="zoom:36%;" />

One logic transaction is splitted into serveral operations. And one block can support dynamically different operation numbers:

<img src="./figs/circuit_proof_arch.png" alt="operation" style="zoom:30%;" />

#### 5.a CreatePair

CreatePair circuit tries to create one pair account with specified tokens information.

#### 5.b AddLiquidity

<img src="./figs/add_liquidity_branch.png" alt="add_liquidity_branch" style="zoom:36%;" />

#### 5.c RemoveLiquidity

The remove liquidity is reverse operationn of AddLiquidity.

#### 5.d Swap

<img src="./figs/swap_branch.png" alt="swap_branch" style="zoom:36%;" />

### 6. Fee Model

No any other fee for ZKSwap protocol, but swap operation.

|            OP name             | OP number | Fee  |    FeeTo     |          Comments          |
| :----------------------------: | :-------: | :--: | :----------: | :------------------------: |
|              Noop              |     0     |  0   |              |                            |
|        Deposit         |     1     |  0   |              |          Layer 1           |
| TransferToNew  |     2     |  0   |     |                            |
|        Withdraw        |     3     |  0   |     |                            |
|       Close        |     4     |  0   |              | Dedicated and not supported |
|        Transfer        |     5     |  0   |     |                            |
|        FullExit        |     6     |  0   |              | Layer 1 |
| ChangePubKey |     7     |  0   |              |                            |
|     CreatePair     |     8     |  0   |              |          Layer 1           |
|   AddLiquidity   |     9     |  0   |     |                            |
| RemoveLiquidity |    10     |  0   |     |                            |
|          Swap          |    11     | 0.3% | LP/Operator | 0.25% LP + 0.05% Operator |

### 7. Exodus Mode

If L1 request is NOT handled in 3 days, ZKSwap protocol enters exodus mode. If the mode is on, users can and should provide proof of assets on L2 and withdraw from L1. 

Basically there are two kinds of assets from exodus mode point of view: one is normal ERC20 token and the other is liquidity tokens in one pair.

#### a. Normal ERC20 Token

For normal ERC20 token, one merkle path should be verified with the latest merkle root. 

<img src="./figs/exit_normal.png" alt="exit_normal" style="zoom:36%;" />

#### b. Liquidity Tokens

For liquidity tokens, more merkle paths should be provided. The information of one specified pair should be correct. And LP token of one account is correct.

<img src="./figs/exit_pair.png" alt="exit_pair" style="zoom:36%;" />



