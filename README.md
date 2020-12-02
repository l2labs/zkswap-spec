# ZKSwap design spec

Version: **1.23**, Update Date: **2020-12-01**


The design spec describes ZKSwap protocol and difference with ZKSync. Thanks to ZKSync, which provides the ZK Rollup framework for ERC20 token transfer. ZKSwap is a L2 token swap protocol using ZK Rollup on Ethereum and based on the PLONK proof system.

### 1. Swap Protocol

The swap protocol is exactly same as Uniswap, but the swap path is limited to 1. That is to say, currently only one token can be swapped to another token if and only if the pair for those two tokens is created.

### 2. Account and Pair Account

The whole Layer 2 state is represented by one Merkle tree, which is leveled by Account and Token. The height of Account Merkle tree is `ACCOUNT_MERKLE_DEPTH` and the height of Token Merkle tree is `TOKEN_MERKLE_DEPTH`.

![account](./figs/account.png)

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

The Token0 ID, Token1 ID, Reserve0, Reserve1, LP token ID and LP Total Amount fields are added for swap related operation, comparing with ZKSync's Account. 

ZKSwap protocol supports two account types: one is Account and the other is Pair Account. For Account, the additional information is useless and empty. For Pair Account, the additional fields should be filled correctly and the **PubKeyHash field is set to zero**. 

### 3. Token Management

The ZKSwap smart contract maintains a single list for tokens. The list is divided into two parts: one is for normal ETH/ERC20 tokens and the other is for LP ERC20 tokens.

#### a. Normal ERC20 token

In total 128 ERC20 tokens are supported with index from 0 to 127. The index 0 is reserved for ETH.

#### b. LP ERC20 token

In total 2,048 – 128 + 1 = 1,921 LP ERC20 tokens are supported and LP ERC20 tokens from index 128 to index 2,048.

In Layer2, each token is assigned one ID. The ID is synchronized between L1 and L2 through ZK rollup transactions.

### 4. Swap Operations

There are four swap related operations. **CreatePair** helps create the swap pair, which is composed of two different tokens. **AddLiquidity** and **RemoveLiquidity** operations are used to add and remove liquidity. **Swap** operation is used to swap one token for the other token.

#### a. CreatePair

The CreatePair operation starts from a L1 transaction. The ZKSwap smart contract helps create one pair for the ERC20 smart contract and one associated LP token. The pair ERC20 smart contract address is used in L2 to identify the swap pair. And the LP token address is used in L2 binding with the LP token ID.

![create_pair](./figs/create_pair.png)

#### b. AddLiquidity

When one pair is created, a user can add liquidity to the pair by providing the two specified assets. The AddLiqudity operation starts from L2.

![add_liquidity](./figs/add_liquidity.png)

By adding liquidity, one account is rewarded with the liquidity token. The liquidity token is maintained in L2, and the liquidity token can be withdrawn to L1. 

#### c. RemoveLiquidity

Remove liquidity can be used to remove specified tokens from one pair. The RemoveLiquidity operation starts from L2, which is opposite from AddLiquidity.

#### d. Swap 

Swap is used to swap one token for another token, in the one pair account. From one pair account, the swap ratio depends on balances of the two specified tokens.

![swap](figs/swap.png)

### 5. Circuit Design

The global state of L2 is updated by transaction execution and verified again by circuit. When one transaction is applied on circuit, it is "divided" into OperationBranch and Operation Arguments. OperationBranch describes the Merkel path. Operation Arguments describe all other information besides branches.

![witness_logic](./figs/witness_logic.png)

Operation is an important structure, which is the "smallest" circuit block.

![operation](./figs/operation.png)

One logic transaction is split into several operations. And one block can support dynamically different operation numbers:

![circuit_proof_arch](./figs/circuit_proof_arch.png)

#### 5.a CreatePair

The CreatePair circuit tries to create one pair account with the specified token information.

#### 5.b AddLiquidity

![add_liquidity_branch](./figs/add_liquidity_branch.png)

#### 5.c RemoveLiquidity

The RemoveLiquidity operation is the reverse of AddLiquidity.

#### 5.d Swap

![swap_branch](./figs/swap_branch.png)

### 6. Fee Model

In ZKSwap protocol there is a fee for the Swap operation, but no fees for the other operations.

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

If a L1 request is NOT handled in 3 days, ZKSwap protocol enters exodus mode, which applies to the entire ZKSwap platform. If exodus mode is on, users can and should provide proof of assets on L2 and withdraw to L1.

Basically, there are two kinds of assets from the exodus mode point of view: one is a normal ERC20 token and the other is liquidity tokens in one pair.

#### a. Normal ERC20 Token

For normal ERC20 token, one Merkle path should be verified with the latest Merkle root. 

![exit_normal](./figs/exit_normal.png)

#### b. Liquidity Tokens

For liquidity tokens, more Merkle paths should be provided. The information of one specified pair should be correct. And the LP token of one account is correct.

![exit_pair](./figs/exit_pair.png)



