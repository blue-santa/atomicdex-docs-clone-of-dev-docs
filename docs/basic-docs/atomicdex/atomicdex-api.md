# DEX API

## Note About Rational Number Type

MM2 now offers the [num-rational crate](https://crates.io/crates/num-rational) feature. This is used to represent order volumes and prices.

Komodo highly recommends that the developer use the rational number type when calculating an order's price and volume. This avoids rounding and precision errors when calculating numbers such as `1/3`, as these cannot be represented as a finite decimal.

The MM2 API typically will return both the rational number type as well as the decimal representation, but the decimal representation should be considered only a convenience feature for readability.
    
The number is represented in JSON as follows: 

```json
[[1,[0,1]],[1,[1]]]
```

The first item, `[1,[0,1]]`, is the `numerator`. 

The second item, `[1,[1]]`, is the `denominator`.

The `numerator` and `denominator` are BigInteger numbers represented as a sign and uint32 array (where numbers are 32 bit parts of big integer in little endian order). 

`[1,[0,1]]` represents `+0000000000000000000000000000000010000000000000000000000000000000` = `4294967296`

`[-1,[1,1]]` represents `-1000000000000000000000000000000010000000000000000000000000000000` = `-4294967297`

<!--

Sidd: The following was removed from the first sentence above. The readability here is difficult, and if all versions of MM2 going forward offer num-rational crate, there is no need to include the version number, I believe? Let me know if we should still keep it.


Starting from 2.0.1319_mm2_059381c81 version MM2

-->

## buy

**buy base rel price volume**

The `buy` method issues a buy request and attempts to match an order from the orderbook based on the provided arguments.

::: tip

Buy and sell methods always create the `taker` order first. Therefore, you must pay an additional 1/777 fee of the trade amount during the swap when taking liquidity from the market. If your order is not matched in 30 seconds, the order is automatically converted to a `maker` request and stays on the orderbook until the request is matched or cancelled. To always act as a maker, please use the [setprice method.](../../../basic-docs/atomicdex/atomicdex-api.html#setprice)

:::

#### Arguments

| Structure | Type                       | Description                                                                   |
| --------- | -------------------------- | ----------------------------------------------------------------------------- |
| base      | string                     | the name of the coin the user desires to receive                              |
| rel       | string                     | the name of the coin the user desires to sell                                 |
| price     | numeric string or rational | the price in `rel` the user is willing to pay per one unit of the `base` coin |
| volume    | numeric string or rational | the amount of coins the user is willing to receive of the `base` coin         |

#### Response

| Structure              | Type     | Description                                                                                                                                                                                                     |
| ---------------------- | -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| result                 | object   | the resulting order object                                                                                                                                                                                      |
| result.action          | string   | the action of the request (`Buy`)                                                                                                                                                                               |
| result.base            | string   | the base currency of request                                                                                                                                                                                    |
| result.base_amount     | string   | the resulting amount of base currency that will be received if the order matches (in decimal representation)                                                                                                    |
| result.base_amount_rat | rational | the resulting amount of base currency that will be received if the order matches (in rational representation)                                                                                                   |
| result.rel             | string   | the rel currency of the request                                                                                                                                                                                 |
| result.rel_amount      | string   | the maximum amount of `rel` coin that will be spent to buy the `base_amount` (according to `price`, in decimal representation)                                                                                  |
| result.rel_amount_rat  | rational | the maximum amount of `rel` coin that will be spent to buy the `base_amount` (according to `price`, in rational representation)                                                                                 |
| result.method          | string   | this field is used for internal P2P interactions; the value is always equal to "request"                                                                                                                        |
| result.dest_pub_key    | string   | reserved for future use. `dest_pub_key` will allow the user to choose the P2P node that will be eligible to match with the request. This value defaults to a "zero pubkey", which means `anyone` can be a match |
| result.sender_pubkey   | string   | the public key of this node                                                                                                                                                                                     |
| result.uuid            | string   | the request uuid                                                                                                                                                                                                |

#### :pushpin: Examples

#### Command (decimal representation)

```bash
curl --url "http://127.0.0.1:7783" --data "{\"userpass\":\"$userpass\",\"method\":\"buy\",\"base\":\"HELLO\",\"rel\":\"WORLD\",\"volume\":"\"1\"",\"price\":"\"1\""}"
```

#### Command (rational representation)

```bash
curl --url "http://127.0.0.1:7783" --data "{\"userpass\":\"$userpass\",\"method\":\"buy\",\"base\":\"HELLO\",\"rel\":\"WORLD\",\"volume\":[[1,[1]],[1,[1]]],\"price\":[[1,[1]],[1,[1]]]}"
```

<div style="margin-top: 0.5rem;">

<collapse-text hidden title="Response">

#### Response (success)

```json
{
  "result": {
    "action": "Buy",
    "base": "HELLO",
    "base_amount": "1",
    "base_amount_rat": [[1,[1]],[1,[1]]],
    "dest_pub_key": "0000000000000000000000000000000000000000000000000000000000000000",
    "method": "request",
    "rel": "WORLD",
    "rel_amount": "1",
    "rel_amount_rat": [[1,[1]],[1,[1]]],
    "sender_pubkey": "c213230771ebff769c58ade63e8debac1b75062ead66796c8d793594005f3920",
    "uuid": "288743e2-92a5-471e-92d5-bb828a2303c3"
  }
}
```

#### Response (error)

```json
{
  "error": "rpc:278] utxo:884] REL balance 12.88892991 is too low, required 21.15"
}
```

#### Response (error)

```json
{
  "error":"rpc:275] lp_ordermatch:665] The WORLD amount 40000/3 is larger than available 47.60450107, balance: 47.60450107, locked by swaps: 0.00000000"
}
```

</collapse-text>

</div>

## cancel\_all\_orders

**cancel_order cancel_by**

The `cancel_all_orders` cancels the active orders created by the MM2 node by specified condition.

#### Arguments

| Structure            | Type   | Description                                                                       |
| -------------------- | ------ | --------------------------------------------------------------------------------- |
| cancel_by            | object | orders matching this condition are cancelled                                  |
| cancel_by.type       | string | `All` to cancel all orders; `Pair` to cancel all orders for specific coin pairs; `Coin` to cancel all orders for a specific coin |
| cancel_by.data       | object | additional data the cancel condition; present with `Pair` and `Coin` types           |
| cancel_by.data.base  | string | base coin of the pair; `Pair` type only                                           |
| cancel_by.data.rel   | string | rel coin of the pair; `Pair` type only                                            |
| cancel_by.data.ticker| string | order will be cancelled if it uses `ticker` as base or rel; `Coin` type only      |

#### Response

| Structure                 | Type                     | Description                                                                                                    |
| ------------------------- | ------------------------ | -------------------------------------------------------------------------------------------------------------- |
| result                    | object                   |                                                                                                                |
| result.cancelled          | array of strings (uuids) | uuids of cancelled orders                                                                                      |
| result.currently_matching | array of strings (uuids) | uuids of the orders being matched with other orders; these are not cancelled even if they fit cancel condition |

#### :pushpin: Examples

#### Command (All orders)

```bash
curl --url "http://127.0.0.1:7783" --data "{\"userpass\":\"$userpass\",\"method\":\"cancel_all_orders\",\"cancel_by\":{\"type\":\"All\"}}"
```

#### Command (Cancel by pair)

```bash
curl --url "http://127.0.0.1:7783" --data "{\"userpass\":\"$userpass\",\"method\":\"cancel_all_orders\",\"cancel_by\":{\"type\":\"Pair\",\"data\":{\"base\":\"RICK\",\"rel\":\"MORTY\"}}}"
```

#### Command (Cancel by coin)

```bash
curl --url "http://127.0.0.1:7783" --data "{\"userpass\":\"$userpass\",\"method\":\"cancel_all_orders\",\"cancel_by\":{\"type\":\"Coin\",\"data\":{\"ticker\":\"RICK\"}}}"
```

<div style="margin-top: 0.5rem;">

<collapse-text hidden title="Response">

#### Response (1 order cancelled)

```json
{
  "result": {
    "cancelled": ["2aae69d1-0167-493e-ad15-c6a8b43546d6"],
    "currently_matching": []
  }
}
```

#### Response (1 order cancelled and 1 is currently matching)

```json
{
  "result": {
    "cancelled": ["2aae69d1-0167-493e-ad15-c6a8b43546d6"],
    "currently_matching": ["e9a6f422-e378-437f-bb74-ba4307a90e68"]
  }
}
```

</collapse-text>

</div>

## cancel\_order

**cancel_order uuid**

The `cancel_order` cancels the active order created by the MM2 node.

#### Arguments

| Structure | Type   | Description                                      |
| --------- | ------ | ------------------------------------------------ |
| uuid      | string | the uuid of the order the user desires to cancel |

#### Response

| Structure | Type   | Description                       |
| --------- | ------ | --------------------------------- |
| result    | string | indicates the status of operation |

#### :pushpin: Examples

#### Command

```bash
curl --url "http://127.0.0.1:7783" --data "{\"userpass\":\"$userpass\",\"method\":\"cancel_order\",\"uuid\":\"6a242691-6c05-474a-85c1-5b3f42278f41\"}"
```

<div style="margin-top: 0.5rem;">

<collapse-text hidden title="Response">

#### Response (success)

```json
{ "result": "success" }
```

#### Response (error)

```json
{ "error": "Order with uuid 6a242691-6c05-474a-85c1-5b3f42278f42 is not found" }
```

</collapse-text>

</div>

## coins\_needed\_for\_kick\_start

**coins_needed_for_kick_start()**

If MM2 is stopped while making a swap/having the active order it will attempt to kick-start them on next launch and continue from the point where it's stopped. `coins_needed_for_kick_start` returns the tickers of coins that should be activated ASAP after MM2 is started to continue the interrupted swaps. Consider calling this method on MM2 startup and activate the returned coins using `enable` or `electrum` methods.

#### Arguments

| Structure | Type | Description |
| --------- | ---- | ----------- |
| (none)    |      |             |

#### Response

| Structure | Type             | Description                                                              |
| --------- | ---------------- | ------------------------------------------------------------------------ |
| result    | array of strings | tickers of coins that should be activated to kick-start swaps and orders |

#### :pushpin: Examples

#### Command

```bash
curl --url "http://127.0.0.1:7783" --data "{\"userpass\":\"$userpass\",\"method\":\"coins_needed_for_kick_start\"}"
```

<div style="margin-top: 0.5rem;">

<collapse-text hidden title="Response">

#### Response (BTC and KMD should be activated ASAP in this case)

```json
{ "result": ["BTC", "KMD"] }
```

#### Response (no swaps and orders waiting to be started)

```json
{ "result": [] }
```

</collapse-text>

</div>

## disable\_coin

**disable_coin coin**

The `disable_coin` method deactivates the previously enabled coin. MM2 also cancels all active orders that use the selected coin. The method will return an error in the following cases:
- The coin is not enabled
- The coin is used by active swaps
- The coin is used by a currently matching order. In this case, other orders might still be cancelled

#### Arguments

| Structure | Type   | Description                   |
| --------- | ------ | ----------------------------- |
| coin      | string | the ticker of coin to disable |

#### Response

| Structure                  | Type             | Description                                                                        |
| -------------------------- | ---------------- | ---------------------------------------------------------------------------------- |
| result.coin                | string           | the ticker of deactivated coin                                                     |
| result.cancelled_orders    | array of strings | uuids of cancelled orders                                                          |
| swaps                      | array of strings | uuids of active swaps that use the selected coin; present only in error cases    |
| orders.matching            | array of strings | uuids of matching orders that use the selected coin; present only in error cases |
| orders.cancelled           | array of strings | uuids of orders that were successfully cancelled despite the error                 |

#### :pushpin: Examples

#### Command

```bash
curl --url "http://127.0.0.1:7783" --data "{\"userpass\":\"$userpass\",\"method\":\"disable_coin\",\"coin\":\"RICK\"}"
```

<div style="margin-top: 0.5rem;">

<collapse-text hidden title="Response">

#### Response (success)

```json
{
  "result": {
    "cancelled_orders":["e5fc7c81-7574-4d3f-b64a-47227455d62a"],
    "coin":"RICK"
  }
}
```

#### Response (error - coin is not enabled)

```json
{
  "error": "No such coin: RICK"
}
```

#### Response (error - active swap is using the coin)

```json
{
  "error": "There're active swaps using RICK",
  "swaps": ["d88d0a0e-f8bd-40ab-8edd-fe20801ef349"]
}
```

#### Response (error - the order is matched at the moment, but another order is cancelled)

```json
{
  "error":"There're currently matching orders using RICK",
  "orders": {
    "matching": ["d88d0a0e-f8bd-40ab-8edd-fe20801ef349"],
    "cancelled": ["c88d0a0e-f8bd-40ab-8edd-fe20801ef349"]
  }
}
```

</collapse-text>

</div>

## electrum

**electrum coin servers (mm2 tx_history=false)**

::: warning Important

This command must be executed at the initiation of each MM2 instance. Also, AtomicDEX software requires the `mm2` parameter to be set for each `coin`; the methods to activate the parameter vary. See below for further information.

:::

::: tip

Electrum mode is available for utxo-based coins only; this includes Bitcoin and Bitcoin-based forks. Electrum mode is not available for ETH/ERC20.

:::

The `electrum` method enables a `coin` by connecting the user's software instance to the `coin` blockchain using electrum technology (e.g. lite mode). This allows the user to avoid syncing the entire blockchain to their local machine.

Each `coin` can be enabled only once, and in either Electrum or Native mode. The DEX software does not allow a `coin` to be active in both modes at once.

#### Notes on the MM2 Parameter

For each `coin`, Komodo software requires the user/developer to set the `mm2` parameter. This can be achieved either in the [coins](../../../basic-docs/atomicdex/atomicdex-tutorials/atomicdex-walkthrough.html#setting-up-the-coin-list) for more details), or via the [electrum](../../../basic-docs/atomicdex/atomicdex-api.html#electrum) and [enable](../../../basic-docs/atomicdex/atomicdex-api.html#enable) methods.

The value of the `mm2` parameter informs the software as to whether the `coin` is expected to function.

- `0` = `non-functioning`
- `1` = `functioning`

::: tip

GUI software developers may refer to the `coins` file [in this link](https://github.com/jl777/coins) for the default coin json configuration.

:::

Volunteers are welcome to test coins with AtomicDEX software at any time. After testing a coin, please create a pull request with the desired coin configuration and successful swap details using the guide linked below.

[Guide to Submitting Coin Test Results](https://github.com/jl777/coins#0-the-coin-must-be-tested-with-barterdex-atomic-swaps)

##### Examples of the Parameter Settings

Set the value of the `mm2` parameter in the [coins](../../../basic-docs/atomicdex/atomicdex-tutorials/atomicdex-walkthrough.html#setting-up-the-coin-list) file as follows:

```bash
"mm2" : 1
```

For terminal interface examples, see the examples section below.

#### Arguments

| Structure                         | Type                                             | Description                                                                                                                                                          |
| --------------------------------- | ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| coin                              | string                                           | the name of the coin you want to enable                                                                                                                              |
| servers                           | array of objects                                 | the list of Electrum servers to which you want to connect                                                                                                            |
| servers.url                       | string                                           | server url                                                                                                                                                           |
| servers.protocol                  | string                                           | the transport protocol that MM2 will use to connect to the server. Possible values: `TCP`, `SSL`. Default value: `TCP`                                               |
| servers.disable_cert_verification | bool                                             | when set to true, this disables server SSL/TLS certificate verification (e.g. to use self-signed certificate). Default value is `false`. <b>Use at your own risk</b> |
| mm2                               | number (required if not set in the `coins` file) | this property informs the AtomicDEX software as to whether the coin is expected to function; accepted values are either `0` or `1`                                   |
| tx_history                        | bool                                             | whether the node should enable `tx_history` preloading as a background process; this must be set to `true` if you plan to use the `my_tx_history` API                |

#### Response

| Structure              | Type             | Description                                                                                              |
| ---------------------- | ---------------- | -------------------------------------------------------------------------------------------------------- |
| address                | string           | the address of the user's `coin` wallet, based on the user's passphrase                                  |
| balance                | string (numeric) | the amount of `coin` the user holds in their wallet                                                      |
| locked_by_swaps        | string (numeric) | the number of coins locked by ongoing swaps. There is a time gap between the start of the swap and the sending of the actual swap transaction (MM2 locks the coins virtually to prevent the user from using the same funds across several ongoing swaps) |
| coin                   | string           | the ticker of the enabled coin                                                                             |
| required_confirmations | number           | MM2 will wait for the this number of transaction confirmations during the swap                  |
| result                 | string           | the result of the request; this value either indicates `success`, or an error, or another type of failure |

#### :pushpin: Examples

#### Command

```bash
curl --url "http://127.0.0.1:7783" --data "{\"userpass\":\"$userpass\",\"method\":\"electrum\",\"coin\":\"HELLOWORLD\",\"servers\":[{\"url\":\"localhost:20025\",\"protocol\":\"SSL\",\"disable_cert_verification\":true},{\"url\":\"localhost:10025\"}]}"
```

<div style="margin-top: 0.5rem;">

<collapse-text hidden title="Response">

#### Response (Success)

```json
{
  "coin": "HELLOWORLD",
  "address": "RQNUR7qLgPUgZxYbvU9x5Kw93f6LU898CQ",
  "balance": "10",
  "locked_by_swaps": "0",
  "required_confirmations":1,
  "result": "success"
}
```

</collapse-text>

</div>

#### Command (With `mm2` argument)

```bash
curl --url "http://127.0.0.1:7783" --data "{\"userpass\":\"$userpass\",\"method\":\"electrum\",\"coin\":\"HELLOWORLD\",\"servers\":[{\"url\":\"localhost:20025\",\"protocol\":\"SSL\",\"disable_cert_verification\":true},{\"url\":\"localhost:10025\"}],\"mm2\":1}"
```

<div style="margin-top: 0.5rem;">

<collapse-text hidden title="Response">

#### Response (Success)

```bash
{
  "coin": "HELLOWORLD",
  "address": "RQNUR7qLgPUgZxYbvU9x5Kw93f6LU898CQ",
  "balance": "10",
  "locked_by_swaps": "0",
  "required_confirmations":1,
  "result": "success"
}
```

#### Response (Error, `mm2` is not set)

```bash
{
  "error":"lp_coins:943] lp_coins:693] mm2 param is not set neither in coins config nor enable request, assuming that coin is not supported"
}
```

</collapse-text>

</div>

## enable

**enable coin (urls swap_contract_address mm2 tx_history=false)**

::: warning Important

AtomicDEX software requires the `mm2` parameter to be set for each `coin`; the methods to activate the parameter vary. See below for further information.

:::

The `enable` method enables a coin by connecting the user's software instance to the `coin` blockchain using the `native` coin daemon.

Each `coin` can be enabled only once, and in either Electrum or Native mode. The DEX software does not allow a `coin` to be active in both modes at once.

For utxo-based coins the daemon of this blockchain must also be running on the user's machine for `enable` to function.

ETH/ERC20 coins are also enabled by the `enable` method, but a local installation of an ETH node is not required.

#### Notes on the mm2 Parameter

Please refer to the `mm2` explanatory section in the `electrum` method for information about setting the `mm2` parameter and testing new coins.

[Link to mm2 explanatory section](../../../basic-docs/atomicdex/atomicdex-api.html#notes-on-the-mm2-parameter)

For terminal interface examples using the `mm2` parameter with the `enable` method, see the examples section below.

#### Using AtomicDEX Software on an ETH-Based Network

The following information can assist the user/developer in connecting AtomicDEX software to the Ethereum network:

- Swap smart contract on the ETH mainnet: [0x8500AFc0bc5214728082163326C2FF0C73f4a871](https://etherscan.io/address/0x8500AFc0bc5214728082163326C2FF0C73f4a871)
  - Main-net nodes maintained by the Komodo team: <b>http://eth1.cipig.net:8555</b>, <b>http://eth2.cipig.net:8555</b>, <b>http://eth3.cipig.net:8555</b>
- Swap smart contract on the Ropsten testnet: [0x7Bc1bBDD6A0a722fC9bffC49c921B685ECB84b94](https://ropsten.etherscan.io/address/0x7bc1bbdd6a0a722fc9bffc49c921b685ecb84b94)
  - Ropsten node maintained by the Komodo team: <b>http://195.201.0.6:8545</b>

To use AtomicDEX software on another Ethereum-based network, such as the Kovan testnet or ETC, deploy the Etomic swap contract code from the repository linked below. Use of this code requires either an ETH node setup or access to a public service such as [Infura.](https://infura.io/)

[Link to repository code for Ethereum-based networks](https://github.com/artemii235/etomic-swap)

#### Arguments

| Structure             | Type                                             | Description                                                                                                                                                                                                                                                                                                         |
| --------------------- | ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| coin                  | string                                           | the name of the coin the user desires to enable                                                                                                                                                                                                                                                                     |
| urls                  | array of strings (required for ETH/ERC20)        | urls of Ethereum RPC nodes to which the user desires to connect                                                                                                                                                                                                                                                     |
| swap_contract_address | string (required for ETH/ERC20)                  | address of etomic swap smart contract                                                                                                                                                                                                                                                                               |
| gas_station_url       | string (optional for ETH/ERC20)                  | url of [ETH gas station API](https://docs.ethgasstation.info/); MM2 uses [eth_gasPrice RPC API](https://github.com/ethereum/wiki/wiki/JSON-RPC#eth_gasprice) by default; when this parameter is set, MM2 will request the current gas price from Station for new transactions, and this often results in lower fees |
| mm2                   | number (required if not set in the `coins` file) | this property informs the AtomicDEX software as to whether the coin is expected to function; accepted values are either `0` or `1`                                                                                                                                                                                  |
| tx_history            | bool                                             | whether the node should enable `tx_history` preloading as a background process; this must be set to `true` if you plan to use the `my_tx_history` API                                                                                                                                                               |

#### Response

| Structure              | Type             | Description                                                                                              |
| ---------------------- | ---------------- | -------------------------------------------------------------------------------------------------------- |
| address                | string           | the address of the user's `coin` wallet, based on the user's passphrase                                  |
| balance                | string (numeric) | the amount of `coin` the user holds in their wallet                                                      |
| locked_by_swaps        | string (numeric) | the number of coins locked by ongoing swaps. There is a time gap between the start of the swap and the sending of the actual swap transaction (MM2 locks the coins virtually to prevent the user from using the same funds across several ongoing swaps) |
| coin                   | string           | the ticker of enabled coin                                                                             |
| required_confirmations | number           | MM2 will wait for the this number of coin's transaction confirmations during the swap                  |
| result                 | string           | the result of the request; this value either indicates `success`, or an error or other type of failure |

#### :pushpin: Examples

#### Command (for Bitcoin-based blockchains)

```bash
curl --url "http://127.0.0.1:7783" --data "{\"userpass\":\"$userpass\",\"method\":\"enable\",\"coin\":\"HELLOWORLD\"}"
```

<div style="margin-top: 0.5rem;">

<collapse-text hidden title="Response">

#### Response

```json
{
  "coin": "HELLOWORLD",
  "address": "RQNUR7qLgPUgZxYbvU9x5Kw93f6LU898CQ",
  "balance": "10",
  "locked_by_swaps": "0",
  "required_confirmations":1,
  "result": "success"
}
```

</collapse-text>

</div>

#### Command (for Ethereum and ERC20-based blockchains)

```bash
curl --url "http://127.0.0.1:7783" --data "{\"userpass\":\"$userpass\",\"method\":\"enable\",\"coin\":\"ETH\",\"urls\":[\"http://195.201.0.6:8545\"],\"swap_contract_address\":\"0x7Bc1bBDD6A0a722fC9bffC49c921B685ECB84b94\"}"
```

<div style="margin-top: 0.5rem;">

<collapse-text hidden title="Response">

#### Response

```json
{
  "coin": "ETH",
  "address": "0x3c7aad7b693e94f13b61d4be4abaeaf802b2e3b5",
  "balance": "50",
  "locked_by_swaps": "0",
  "required_confirmations":1,
  "result": "success"
}
```

</collapse-text>

</div>

#### Command (for Ethereum and ERC20-based blockchains with gas\_station\_url)

```bash
curl --url "http://127.0.0.1:7783" --data "{\"userpass\":\"$userpass\",\"method\":\"enable\",\"coin\":\"ETH\",\"urls\":[\"http://195.201.0.6:8545\"],\"swap_contract_address\":\"0x7Bc1bBDD6A0a722fC9bffC49c921B685ECB84b94\",\"gas_station_url\":\"https://ethgasstation.info/json/ethgasAPI.json\"}"
```

<div style="margin-top: 0.5rem;">

<collapse-text hidden title="Response">

#### Response

```json
{
  "coin": "ETH",
  "address": "0x3c7aad7b693e94f13b61d4be4abaeaf802b2e3b5",
  "balance": "50",
  "locked_by_swaps": "0",
  "required_confirmations":1,
  "result": "success"
}
```

</collapse-text>

</div>

#### Command (With `mm2` argument)

```bash
curl --url "http://127.0.0.1:7783" --data "{\"userpass\":\"$userpass\",\"method\":\"enable\",\"coin\":\"HELLOWORLD\",\"mm2\":1}"
```

<div style="margin-top: 0.5rem;">

<collapse-text hidden title="Response">

#### Response (Success):

```bash
{
  "coin": "HELLOWORLD",
  "address": "RQNUR7qLgPUgZxYbvU9x5Kw93f6LU898CQ",
  "balance": "10",
  "locked_by_swaps": "0",
  "result": "success"
}
```

#### Response (Error, `mm2` is not set)

```bash
{
  "error":"lp_coins:943] lp_coins:693] mm2 param is not set neither in coins config nor enable request, assuming that coin is not supported"
}
```

</collapse-text>

</div>

## get\_enabled\_coins

**get_enabled_coins**

The `get_enabled_coins` method returns data of coins that are currently enabled on the user's MM2 node.

#### Arguments

| Structure | Type | Description |
| --------- | ---- | ----------- |
| (none)    |      |             |

#### Response

| Structure      | Type             | Description                            |
| -------------- | ---------------- | -------------------------------------- |
| result         | array of objects | tickers and addresses of enabled coins |
| result.address | string           | the user's address for this coin       |
| result.ticker  | string           | the ticker name of this coin           |

#### :pushpin: Examples

#### Command

```bash
curl --url "http://127.0.0.1:7783" --data "{\"userpass\":\"$userpass\",\"method\":\"get_enabled_coins\"}"
```

<div style="margin-top: 0.5rem;">

<collapse-text hidden title="Response">

#### Response

```json
{
  "result": [
    {
      "address": "1WxswvLF2HdaDr4k77e92VjaXuPQA8Uji",
      "ticker": "BTC"
    },
    {
      "address": "R9o9xTocqr6CeEDGDH6mEYpwLoMz6jNjMW",
      "ticker": "PIZZA"
    },
    {
      "address": "R9o9xTocqr6CeEDGDH6mEYpwLoMz6jNjMW",
      "ticker": "BEER"
    },
    {
      "address": "0xbAB36286672fbdc7B250804bf6D14Be0dF69fa29",
      "ticker": "ETH"
    },
    {
      "address": "R9o9xTocqr6CeEDGDH6mEYpwLoMz6jNjMW",
      "ticker": "ETOMIC"
    },
    {
      "address": "0xbAB36286672fbdc7B250804bf6D14Be0dF69fa29",
      "ticker": "DEC8"
    },
    {
      "address": "0xbAB36286672fbdc7B250804bf6D14Be0dF69fa29",
      "ticker": "BAT"
    }
  ]
}
```

</collapse-text>

</div>

## get\_trade\_fee

**get_trade_fee coin**

The `get_trade_fee` method returns the approximate amount of the miner fee that will be paid per swap transaction.

This amount should be multiplied by 2 and deducted from the volume on `buy/sell` calls when the user is about to trade the entire balance of the selected coin.

#### Arguments

| Structure | Type   | Description                                      |
| --------- | ------ | ------------------------------------------------ |
| coin      | string | the name of the coin for the requested trade fee |

#### Response

| Structure     | Type             | Description                                                                                                                                                |
| ------------- | ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| result        | object           | an object containing the relevant information                                                                                                              |
| result.coin   | string           | the fee will be paid from the user's balance of this coin. This coin name may differ from the requested coin. For example ERC20 fees are paid by ETH (gas) |
| result.amount | string (numeric) | the approximate fee amount to be paid per swap transaction                                                                                                 |

#### :pushpin: Examples

#### Command (BTC)

```bash
curl --url "http://127.0.0.1:7783" --data "{\"userpass\":\"$userpass\",\"method\":\"get_trade_fee\",\"coin\":\"BTC\"}"
```

<div style="margin-top: 0.5rem;">

<collapse-text hidden title="Response">

#### Response

```json
{
  "result": {
    "amount": "0.00096041",
    "coin": "BTC"
  }
}
```

</collapse-text>

</div>

#### Command (ETH)

```bash
curl --url "http://127.0.0.1:7783" --data "{\"userpass\":\"$userpass\",\"method\":\"get_trade_fee\",\"coin\":\"ETH\"}"
```

<div style="margin-top: 0.5rem;">

<collapse-text hidden title="Response">

#### Response

```json
{
  "result": {
    "amount": "0.00121275",
    "coin": "ETH"
  }
}
```

</collapse-text>

</div>

#### Command (ERC20)

```bash
curl --url "http://127.0.0.1:7783" --data "{\"userpass\":\"$userpass\",\"method\":\"get_trade_fee\",\"coin\":\"BAT\"}"
```

<div style="margin-top: 0.5rem;">

<collapse-text hidden title="Response">

#### Response

```json
{
  "result": {
    "amount": "0.00121275",
    "coin": "ETH"
  }
}
```

</collapse-text>

</div>

## help

**help()**

The `help` method returns the full API documentation in the terminal.

#### Arguments

| Structure | Type | Description |
| --------- | ---- | ----------- |
| (none)    |      |             |

#### Response

| Structure                           | Type | Description |
| ----------------------------------- | ---- | ----------- |
| (returns the full docs in terminal) |      |             |

## import_swaps

**import_swaps swaps**

The `import_swaps` method imports to the local database the `swaps` data that was exported from another MM2 instance.

Use this method in combination with `my_swap_status` or `my_recent_swaps` to copy the swap history between different devices.

#### Arguments

| Structure | Type             | Description                                                             |
| --------- | ---------------- | ----------------------------------------------------------------------- |
| swaps     | array of objects | swaps data; each record has the format of the `my_swap_status` response |

#### Response

| Structure        | Type                | Description                                                  |
| ---------------- | ------------------- | ------------------------------------------------------------ |
| result.imported  | array of strings    | uuids of swaps that were successfully imported               |
| result.imported  | map                 | uuids of swaps that failed to import; includes error message |

#### :pushpin: Examples

#### Command

```bash
curl --url "http://127.0.0.1:7783" --data "{\"userpass\":\"$userpass\",\"method\":\"import_swaps\",\"swaps\":[{"error_events":["StartFailed","NegotiateFailed","TakerFeeSendFailed","MakerPaymentValidateFailed","TakerPaymentTransactionFailed","TakerPaymentDataSendFailed","TakerPaymentWaitForSpendFailed","MakerPaymentSpendFailed","TakerPaymentRefunded","TakerPaymentRefundFailed"],"events":[{"event":{"data":{"lock_duration":7800,"maker":"631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640","maker_amount":"3","maker_coin":"BEER","maker_coin_start_block":156186,"maker_payment_confirmations":0,"maker_payment_wait":1568883784,"my_persistent_pub":"02031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3","started_at":1568881184,"taker_amount":"4","taker_coin":"ETOMIC","taker_coin_start_block":175041,"taker_payment_confirmations":1,"taker_payment_lock":1568888984,"uuid":"07ce08bf-3db9-4dd8-a671-854affc1b7a3"},"type":"Started"},"timestamp":1568881185316},{"event":{"data":{"maker_payment_locktime":1568896784,"maker_pubkey":"02631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640","secret_hash":"eba736c5cc9bb33dee15b4a9c855a7831a484d84"},"type":"Negotiated"},"timestamp":1568881246025},{"event":{"data":{"block_height":0,"coin":"ETOMIC","fee_details":{"amount":"0.00001"},"from":["R9o9xTocqr6CeEDGDH6mEYpwLoMz6jNjMW"],"internal_id":"0c07be4dda88d8d75374496aa0f27e12f55363ce8d558cb5feecc828545e5f87","my_balance_change":"-0.005158","received_by_me":"0.004842","spent_by_me":"0.01","timestamp":0,"to":["R9o9xTocqr6CeEDGDH6mEYpwLoMz6jNjMW","RThtXup6Zo7LZAi8kRWgjAyi1s4u6U9Cpf"],"total_amount":"0.01","tx_hash":"0c07be4dda88d8d75374496aa0f27e12f55363ce8d558cb5feecc828545e5f87","tx_hex":"0400008085202f890146b98696761d5e8667ffd665b73e13a8400baab4b22230a7ede0e4708597ee9c000000006a473044022077acb70e5940dfe789faa77e72b34f098abbf0974ea94a0380db157e243965230220614ec4966db0a122b0e7c23aa0707459b3b4f8241bb630c635cf6e943e96362e012102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffff02f0da0700000000001976a914ca1e04745e8ca0c60d8c5881531d51bec470743f88ac68630700000000001976a91405aab5342166f8594baf17a7d9bef5d56744332788ac5e3a835d000000000000000000000000000000"},"type":"TakerFeeSent"},"timestamp":1568881250689},{"event":{"data":{"block_height":0,"coin":"BEER","fee_details":{"amount":"0.00001"},"from":["RJTYiYeJ8eVvJ53n2YbrVmxWNNMVZjDGLh"],"internal_id":"31d97b3359bdbdfbd241e7706c90691e4d7c0b7abd27f2b22121be7f71c5fd06","my_balance_change":"0","received_by_me":"0","spent_by_me":"0","timestamp":0,"to":["RJTYiYeJ8eVvJ53n2YbrVmxWNNMVZjDGLh","bPZrgpAviZUYUmECtwdocqpFy1VEV7pL7d"],"total_amount":"8240.19873868","tx_hash":"31d97b3359bdbdfbd241e7706c90691e4d7c0b7abd27f2b22121be7f71c5fd06","tx_hex":"0400008085202f8901b4679094d4bf74f52c9004107cb9641a658213d5e9950e42a8805824e801ffc7010000006b483045022100b2e49f8bdc5a4b6c404e10150872dbec89a46deb13a837d3251c0299fe1066ca022012cbe6663106f92aefce88238b25b53aadd3522df8290ced869c3cc23559cc23012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff0200a3e1110000000017a91476e1998b0cd18da5f128e5bb695c36fbe6d957e98764c987c9bf0000001976a91464ae8510aac9546d5e7704e31ce177451386455588ac753a835d000000000000000000000000000000"},"type":"MakerPaymentReceived"},"timestamp":1568881291571},{"event":{"type":"MakerPaymentWaitConfirmStarted"},"timestamp":1568881291571},{"event":{"type":"MakerPaymentValidatedAndConfirmed"},"timestamp":1568881291985},{"event":{"data":{"block_height":0,"coin":"ETOMIC","fee_details":{"amount":"0.00001"},"from":["R9o9xTocqr6CeEDGDH6mEYpwLoMz6jNjMW"],"internal_id":"95926ab204049edeadb370c17a1168d9d79ee5747d8d832f73cfddf1c74f3961","my_balance_change":"-4.00001","received_by_me":"43.59901307","spent_by_me":"47.59902307","timestamp":0,"to":["R9o9xTocqr6CeEDGDH6mEYpwLoMz6jNjMW","bKnDnsrM3eR8d9vWHfd3LP1dcV1RskE7x5"],"total_amount":"47.59902307","tx_hash":"95926ab204049edeadb370c17a1168d9d79ee5747d8d832f73cfddf1c74f3961","tx_hex":"0400008085202f8902875f5e5428c8ecfeb58c558dce6353f5127ef2a06a497453d7d888da4dbe070c010000006a4730440220416059356dc6dde0ddbee206e456698d7e54c3afa92132ecbf332e8c937e5383022068a41d9c208e8812204d4b0d21749b2684d0eea513467295e359e03c5132e719012102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffff46b98696761d5e8667ffd665b73e13a8400baab4b22230a7ede0e4708597ee9c010000006b483045022100a990c798d0f96fd5ff7029fd5318f3c742837400d9f09a002e7f5bb1aeaf4e5a0220517dbc16713411e5c99bb0172f295a54c97aaf4d64de145eb3c5fa0fc38b67ff012102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffff020084d7170000000017a9144d57b4930e6c86493034f17aa05464773625de1c877bd0de03010000001976a91405aab5342166f8594baf17a7d9bef5d56744332788ac8c3a835d000000000000000000000000000000"},"type":"TakerPaymentSent"},"timestamp":1568881296904},{"event":{"data":{"secret":"fb968d5460399f20ffd09906dc8f65c21fbb5cb8077a8e6d7126d0526586ca96","transaction":{"block_height":0,"coin":"ETOMIC","fee_details":{"amount":"0.00001"},"from":["bKnDnsrM3eR8d9vWHfd3LP1dcV1RskE7x5"],"internal_id":"68f5ec617bd9a4a24d7af0ce9762d87f7baadc13a66739fd4a2575630ecc1827","my_balance_change":"0","received_by_me":"0","spent_by_me":"0","timestamp":0,"to":["RJTYiYeJ8eVvJ53n2YbrVmxWNNMVZjDGLh"],"total_amount":"4","tx_hash":"68f5ec617bd9a4a24d7af0ce9762d87f7baadc13a66739fd4a2575630ecc1827","tx_hex":"0400008085202f890161394fc7f1ddcf732f838d7d74e59ed7d968117ac170b3adde9e0404b26a929500000000d8483045022100a33d976cf509d6f9e66c297db30c0f44cced2241ee9c01c5ec8d3cbbf3d41172022039a6e02c3a3c85e3861ab1d2f13ba52677a3b1344483b2ae443723ba5bb1353f0120fb968d5460399f20ffd09906dc8f65c21fbb5cb8077a8e6d7126d0526586ca96004c6b63049858835db1752102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ac6782012088a914eba736c5cc9bb33dee15b4a9c855a7831a484d84882102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ac68ffffffff011880d717000000001976a91464ae8510aac9546d5e7704e31ce177451386455588ac942c835d000000000000000000000000000000"}},"type":"TakerPaymentSpent"},"timestamp":1568881328643},{"event":{"data":{"error":"taker_swap:798] utxo:950] utxo:950] error"},"type":"MakerPaymentSpendFailed"},"timestamp":1568881328645},{"event":{"type":"Finished"},"timestamp":1568881328648}],"my_info":{"my_amount":"4","my_coin":"ETOMIC","other_amount":"3","other_coin":"BEER","started_at":1568881184},"recoverable":true,"success_events":["Started","Negotiated","TakerFeeSent","MakerPaymentReceived","MakerPaymentWaitConfirmStarted","MakerPaymentValidatedAndConfirmed","TakerPaymentSent","TakerPaymentSpent","MakerPaymentSpent","Finished"],"type":"Taker","uuid":"07ce08bf-3db9-4dd8-a671-854affc1b7a3"}]}"
```

<div style="margin-top: 0.5rem;">

<collapse-text hidden title="Response">

#### Response

```json
{
  "result":{
    "imported":[
      "07ce08bf-3db9-4dd8-a671-854affc1b7a3"
    ],
    "skipped":{
      "1af6bb5e-e131-4b06-b235-36fae8daab0a":"lp_swap:424] File already exists"
    }
  }
}
```

</collapse-text>

</div>

## my\_balance

**my_balance coin**

The `my_balance` method returns the current balance of the specified `coin`.

#### Arguments

| Structure | Type   | Description                                  |
| --------- | ------ | -------------------------------------------- |
| coin      | string | the name of the coin to retrieve the balance |

#### Response

| Structure       | Type             | Description                        |
| --------------- | ---------------- | ---------------------------------- |
| address         | string           | the address that holds the coins   |
| balance         | string (numeric) | the number of coins in the address |
| locked_by_swaps | string (numeric) | the number of coins locked by ongoing swaps. There is a time gap between the start of the swap and the sending of the actual swap transaction (MM2 locks the coins virtually to prevent the user from using the same funds across several ongoing swaps) |
| coin            | string           | the name of the coin               |

#### :pushpin: Examples

#### Command

```bash
curl --url "http://127.0.0.1:7783" --data "{\"userpass\":\"$userpass\",\"method\":\"my_balance\",\"coin\":\"HELLOWORLD\"}"
```

<div style="margin-top: 0.5rem;">

<collapse-text hidden title="Response">

#### Response

```json
{
  "address":"R9o9xTocqr6CeEDGDH6mEYpwLoMz6jNjMW",
  "balance":"60.00253836",
  "coin":"HELLOWORLD",
  "locked_by_swaps":"0"
}
```

</collapse-text>

</div>

## my\_orders

**my_orders()**

The `my_orders` method returns the data of all active orders created by the MM2 node.

#### Arguments

| Structure | Type | Description |
| --------- | ---- | ----------- |
| (none)    |      |             |

#### Response

| Structure    | Type           | Description                                           |
| ------------ | -------------- | ----------------------------------------------------- |
| maker_orders | map of objects | orders that are currently active in market maker mode |
| taker_orders | map of objects | orders that are currently active in market taker mode |

#### :pushpin: Examples

#### Command

```bash
curl --url "http://127.0.0.1:7783" --data "{\"userpass\":\"$userpass\",\"method\":\"my_orders\"}"
```

<div style="margin-top: 0.5rem;">

<collapse-text hidden title="Response">

#### Response

```json
{
  "result":{
    "maker_orders":{
      "ea77dcc3-a711-4c3d-ac36-d45fc5e1ee0c":{
        "available_amount":"1",
        "base":"BEER",
        "cancellable":true,
        "created_at":1568808684710,
        "matches":{
          "60aaacca-ed31-4633-9326-c9757ea4cf78":{
            "connect":{
              "dest_pub_key":"c213230771ebff769c58ade63e8debac1b75062ead66796c8d793594005f3920",
              "maker_order_uuid":"fedd5261-a57e-4cbf-80ac-b3507045e140",
              "method":"connect",
              "sender_pubkey":"5a2f1c468b7083c4f7649bf68a50612ffe7c38b1d62e1ece3829ca88e7e7fd12",
              "taker_order_uuid":"60aaacca-ed31-4633-9326-c9757ea4cf78"
            },
            "connected":{
              "dest_pub_key":"5a2f1c468b7083c4f7649bf68a50612ffe7c38b1d62e1ece3829ca88e7e7fd12",
              "maker_order_uuid":"fedd5261-a57e-4cbf-80ac-b3507045e140",
              "method":"connected",
              "sender_pubkey":"c213230771ebff769c58ade63e8debac1b75062ead66796c8d793594005f3920",
              "taker_order_uuid":"60aaacca-ed31-4633-9326-c9757ea4cf78"
            },
            "last_updated":1560529572571,
            "request":{
              "action":"Buy",
              "base":"BEER",
              "base_amount":"1",
              "dest_pub_key":"0000000000000000000000000000000000000000000000000000000000000000",
              "method":"request",
              "rel":"PIZZA",
              "rel_amount":"1",
              "sender_pubkey":"5a2f1c468b7083c4f7649bf68a50612ffe7c38b1d62e1ece3829ca88e7e7fd12",
              "uuid":"60aaacca-ed31-4633-9326-c9757ea4cf78"
            },
            "reserved":{
              "base":"BEER",
              "base_amount":"1",
              "dest_pub_key":"5a2f1c468b7083c4f7649bf68a50612ffe7c38b1d62e1ece3829ca88e7e7fd12",
              "maker_order_uuid":"fedd5261-a57e-4cbf-80ac-b3507045e140",
              "method":"reserved",
              "rel":"PIZZA",
              "rel_amount":"1",
              "sender_pubkey":"c213230771ebff769c58ade63e8debac1b75062ead66796c8d793594005f3920",
              "taker_order_uuid":"60aaacca-ed31-4633-9326-c9757ea4cf78"
            }
          }
        },
        "max_base_vol":"1",
        "max_base_vol_rat":[[1,[1]],[1,[1]]],
        "min_base_vol":"0",
        "min_base_vol_rat":[[0,[]],[1,[1]]],
        "price":"1",
        "price_rat":[[1,[1]],[1,[1]]],
        "rel":"ETOMIC",
        "started_swaps":[
          "60aaacca-ed31-4633-9326-c9757ea4cf78"
        ],
        "uuid":"ea77dcc3-a711-4c3d-ac36-d45fc5e1ee0c"
      }
    },
    "taker_orders":{
      "ea199ac4-b216-4a04-9f08-ac73aa06ae37":{
        "cancellable":true,
        "created_at":1568811351456,
        "matches":{
          "15922925-cc46-4219-8cbd-613802e17797":{
            "connect":{
              "dest_pub_key":"5a2f1c468b7083c4f7649bf68a50612ffe7c38b1d62e1ece3829ca88e7e7fd12",
              "maker_order_uuid":"15922925-cc46-4219-8cbd-613802e17797",
              "method":"connect",
              "sender_pubkey":"c213230771ebff769c58ade63e8debac1b75062ead66796c8d793594005f3920",
              "taker_order_uuid":"45252de5-ea9f-44ae-8b48-85092a0c99ed"
            },
            "connected":{
              "dest_pub_key":"c213230771ebff769c58ade63e8debac1b75062ead66796c8d793594005f3920",
              "maker_order_uuid":"15922925-cc46-4219-8cbd-613802e17797",
              "method":"connected",
              "sender_pubkey":"5a2f1c468b7083c4f7649bf68a50612ffe7c38b1d62e1ece3829ca88e7e7fd12",
              "taker_order_uuid":"45252de5-ea9f-44ae-8b48-85092a0c99ed"
            },
            "last_updated":1560529049477,
            "reserved":{
              "base":"BEER",
              "base_amount":"1",
              "dest_pub_key":"c213230771ebff769c58ade63e8debac1b75062ead66796c8d793594005f3920",
              "maker_order_uuid":"15922925-cc46-4219-8cbd-613802e17797",
              "method":"reserved",
              "rel":"ETOMIC",
              "rel_amount":"1",
              "sender_pubkey":"5a2f1c468b7083c4f7649bf68a50612ffe7c38b1d62e1ece3829ca88e7e7fd12",
              "taker_order_uuid":"45252de5-ea9f-44ae-8b48-85092a0c99ed"
            }
          }
        },
        "request":{
          "action":"Buy",
          "base":"BEER",
          "base_amount":"1",
          "base_amount_rat":[[1,[1]],[1,[1]]],
          "dest_pub_key":"0000000000000000000000000000000000000000000000000000000000000000",
          "method":"request",
          "rel":"ETOMIC",
          "rel_amount":"1",
          "rel_amount_rat":[[1,[1]],[1,[1]]],
          "sender_pubkey":"031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3",
          "uuid":"ea199ac4-b216-4a04-9f08-ac73aa06ae37"
        }
      }
    }
  }
}
```

</collapse-text>

</div>

## my\_recent\_swaps

**(from_uuid limit=10)**

The `my_recent_swaps` method returns the data of the most recent atomic swaps executed by the MM2 node.

#### Arguments

| Structure | Type   | Description                                                             |
| --------- | ------ | ----------------------------------------------------------------------- |
| limit     | number | limits the number of returned swaps                                     |
| from_uuid | string | MM2 will skip records until this uuid, skipping the `from_uuid` as well |

#### Response

| Structure | Type             | Description                                                                                                                             |
| --------- | ---------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| swaps     | array of objects | swaps data; each record has the format of the `my_swap_status` response                                                                 |
| from_uuid | string           | the from_uuid that was set in the request; this value is null if nothing was set                                                        |
| skipped   | number           | the number of skipped records (i.e. the position of `from_uuid` in the list + 1; the value is 0 if `from_uuid` was not set              |
| limit     | number           | the limit that was set in the request; note that the actual number of swaps can differ from the specified limit (e.g. on the last page) |
| total     | number           | total number of swaps available                                                                                                         |

#### :pushpin: Examples

#### Command

```bash
curl --url "http://127.0.0.1:7783" --data "{\"userpass\":\"$userpass\",\"method\":\"my_recent_swaps\",\"from_uuid\":\"e299c6ece7a7ddc42444eda64d46b163eaa992da65ce6de24eb812d715184e4c\",\"limit\":2}"
```

<div style="margin-top: 0.5rem;">

<collapse-text hidden title="Response">

#### Response (success)

```json
{
  "result": {
    "from_uuid": "e299c6ece7a7ddc42444eda64d46b163eaa992da65ce6de24eb812d715184e4c",
    "limit": 2,
    "skipped": 1,
    "swaps": [
      {
        "error_events": [
          "StartFailed",
          "NegotiateFailed",
          "TakerFeeValidateFailed",
          "MakerPaymentTransactionFailed",
          "MakerPaymentDataSendFailed",
          "TakerPaymentValidateFailed",
          "TakerPaymentSpendFailed",
          "MakerPaymentRefunded",
          "MakerPaymentRefundFailed"
        ],
        "events": [
          {
            "event": {
              "data": {
                "lock_duration": 7800,
                "maker_amount": "1",
                "maker_coin": "BEER",
                "maker_coin_start_block": 154221,
                "maker_payment_confirmations": 1,
                "maker_payment_lock": 1561545442,
                "my_persistent_pub": "02031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3",
                "secret": "ea774bc94dce44c138920c6e9255e31d5645e60c0b64e9a059ab025f1dd2fdc6",
                "started_at": 1561529842,
                "taker": "5a2f1c468b7083c4f7649bf68a50612ffe7c38b1d62e1ece3829ca88e7e7fd12",
                "taker_amount": "1",
                "taker_coin": "PIZZA",
                "taker_coin_start_block": 141363,
                "taker_payment_confirmations": 1,
                "uuid": "6bf6e313-e610-4a9a-ba8c-57fc34a124aa"
              },
              "type": "Started"
            },
            "timestamp": 1561529842866
          },
          {
            "event": {
              "data": {
                "taker_payment_locktime": 1561537641,
                "taker_pubkey": "02631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640"
              },
              "type": "Negotiated"
            },
            "timestamp": 1561529883208
          },
          {
            "event": {
              "data": {
                "block_height": 141364,
                "coin": "PIZZA",
                "fee_details": {
                  "amount": "0.00001"
                },
                "from": ["RJTYiYeJ8eVvJ53n2YbrVmxWNNMVZjDGLh"],
                "internal_id": "a91469546211cc910fbe4a1f4668ab0353765d3d0cb03f4a67bff9326991f682",
                "my_balance_change": "0",
                "received_by_me": "0",
                "spent_by_me": "0",
                "timestamp": 1561529907,
                "to": [
                  "RJTYiYeJ8eVvJ53n2YbrVmxWNNMVZjDGLh",
                  "RThtXup6Zo7LZAi8kRWgjAyi1s4u6U9Cpf"
                ],
                "total_amount": "0.002",
                "tx_hash": "a91469546211cc910fbe4a1f4668ab0353765d3d0cb03f4a67bff9326991f682",
                "tx_hex": "0400008085202f89021c7eeec33f8eb5ff2ed6c3d09e40e04b05a9674ea2feefcc12de3f9dcc16aff8000000006b483045022100e18e3d1afa8a24ecec82c92bfc05c119bfacdbb71b5f5663a4b96cc2a41ab269022017a79a1a1f6e0220d8fa1d2cf3b1c9788272f1ad18e4987b8f1cd4418acaa5b0012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff6a0d321eb52c3c7165adf80f83b15b7a5caa3a0dfa87746239021600d47fb43e000000006b483045022100937ed900e084d57d5e3341499fc66c5574884ca71cd4331fa696c8b7a517591b02201f5f851f94c3ca0ffb4789f1af22cb95dc83564e127ed7d23f1129eb2b981a2f012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff02bcf60100000000001976a914ca1e04745e8ca0c60d8c5881531d51bec470743f88ac9c120100000000001976a91464ae8510aac9546d5e7704e31ce177451386455588ac2f0e135d000000000000000000000000000000"
              },
              "type": "TakerFeeValidated"
            },
            "timestamp": 1561529927879
          },
          {
            "event": {
              "data": {
                "block_height": 0,
                "coin": "BEER",
                "fee_details": {
                  "amount": "0.00001"
                },
                "from": ["R9o9xTocqr6CeEDGDH6mEYpwLoMz6jNjMW"],
                "internal_id": "efa90a2918e6793c8a2725c06ee34d0fa76c43bc85e680be195414e7aee36154",
                "my_balance_change": "-1.00001",
                "received_by_me": "0.0285517",
                "spent_by_me": "1.0285617",
                "timestamp": 0,
                "to": [
                  "R9o9xTocqr6CeEDGDH6mEYpwLoMz6jNjMW",
                  "bKuQbg7vgFR1C25vPqMq8ePnB25cUEAGpo"
                ],
                "total_amount": "1.0285617",
                "tx_hash": "efa90a2918e6793c8a2725c06ee34d0fa76c43bc85e680be195414e7aee36154",
                "tx_hex": "0400008085202f890cdcd071edda0d5f489b0be9c8b521ad608bb6d7f43f6e7a491843e7a4d0078f85000000006b483045022100fbc3bd09f8e1821ed671d1b1d2ed355833fb42c0bc435fef2da5c5b0a980b9a002204ef92b35576069d640ca0ac08f46645e5ade36afd5f19fb6aad19cfc9fb221fb012102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffffe6ae2a3ce221a6612d9e640bdbe10a2e477b3bc68a1aeee4a6784cb18648a785010000006a47304402202000a7e60ae2ce1529247875623ef2c5b42448dcaeac8de0f8f0e2f8e5bd8a6b0220426321a004b793172014f522efbca77a3dc92e86ce0a75330d8ceb83072ad4e7012102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffff9335553edcbac9559cae517a3e25b880a48fabf661c4ac338394972eef4572da000000006b4830450221008ded7230f2fb37a42b94f96174ec192baf4cd9e9e68fb9b6cf0463a36a6093e00220538de51ceda1617f3964a2350802377940fdfa018cc1043d77c66081b1cab0c4012102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3fffffffff91b5d3733877f84108de77fec46bee156766e1a6837fa7b580ccbc3905acb14000000006b483045022100d07cf1fd20e07aafdc942ba56f6b45baee61b93145a2bdba391e2cdb8024bf15022056ea8183990703ef05018df2fe8bd5ec678ec0f9207b0283292b2cdafc5e1e0c012102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffff147870387ca938b2b6e7daa96ba2496014f125c0e4e576273ef36ee8186c415a000000006a47304402204c5b15b641d7e34444456d2ea6663bdc8bd8216e309a7220814474f346b8425e0220634d1dd943b416b7a807704d7f7a3d46a60d88ef4e20734588a0b302c55fa82d012102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffffd2b954ae9b4a61fad9f7bc956d24e38d7b6fe313da824bd3bd91287d5a6b49d9000000006b483045022100a7387d9ab7b2c92d3cbce525e96ffac5ae3ef14f848661741ada0db17715c4a002202c1417d5e3e04b1a2d1774a83bb8d5aa1c0536c100138123089fa69921b5d976012102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffff28792a2e26d9d7be0467fac52b12ece67410b23eea845008257979bd87d083e3000000006a473044022027c40517c33cd3202d4310cfd2c75f38e6d7804b255fc3838a32ea26e5a3cb0002202b4399e1d7e655b64f699318f2bfbdced49f064ee54e9d6a678668fce51caf96012102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffffa8bf797bacd213b74a9977ae1b956afe3af33a1ee94324e010a16db891a07441000000006a473044022004cbb1d970b9f02c578b5c1d7de33361581eebc19c3cd8d2e50b0211ca4ef13702200c93b9fe5428055b6274dc8e52073c3e87f5b5e4206134d745928ccfc9393919012102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffff2b6fd82c9a68111b67ad85a614a6ecb50f7b6eac3d21d8ebefd9a6065cdf5729000000006b483045022100fdff16c595c7b4a9b4fc1e445b565f7b29fe5b7a08f79291b0ff585c7b72ac2902200c694aa124013bd419ce2349f15d10435827868d35db939b9d3c344d16e78420012102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffff6a5468dd8c83553dc51022f2a2fb772cf91c8607dc2ca1b8f203ac534612ab29000000006b483045022100ba7cc79e7ae3720238bfc5caa225dc8017d6a0d1cb1ec66abaf724fd20b3b7ab02206e8c942756604af0f63b74af495a9b3b7f4a44c489267f69a14cf2b1b953f46e012102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffff5f9f48a91d343fd5aef1d85f00850070931459ab256697afb728d1c81c1fa1d2000000006a47304402200ec85fc66f963e7504eb27361a4b4bb17de60e459da414fdc3962476de636134022056b62c15cf7f9b4e7d4e11c03e4e541dd348919b8c55efa4f1927e2fdd5ae8ea012102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffffee1f455924d3167e7f7abf452c1856e9abdcfe27dc889942dd249cb376169d38000000006b48304502210089274eed807c5d23d819f6dfa8a358a9748e56f2080be4396ef77bb19d91b17402207fc7b22c879534fffe0eeaaec8fc284e22c2756f380c05ea57b881a96b09f3af012102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffff0200e1f5050000000017a9144eb3a361d8a15d7f6a8ef9d1cf44962a90c44d548702912b00000000001976a91405aab5342166f8594baf17a7d9bef5d56744332788ac490e135d000000000000000000000000000000"
              },
              "type": "MakerPaymentSent"
            },
            "timestamp": 1561529938879
          },
          {
            "event": {
              "data": {
                "block_height": 141365,
                "coin": "PIZZA",
                "fee_details": {
                  "amount": "0.00001"
                },
                "from": ["RJTYiYeJ8eVvJ53n2YbrVmxWNNMVZjDGLh"],
                "internal_id": "7e0e38e31dbe80792ef320b8c0a7cb9259127427ef8c2fca1d796f24484046a5",
                "my_balance_change": "0",
                "received_by_me": "0",
                "spent_by_me": "0",
                "timestamp": 1561529960,
                "to": [
                  "RJTYiYeJ8eVvJ53n2YbrVmxWNNMVZjDGLh",
                  "bUN5nesdt1xsAjCtAaYUnNbQhGqUWwQT1Q"
                ],
                "total_amount": "1.01999523",
                "tx_hash": "7e0e38e31dbe80792ef320b8c0a7cb9259127427ef8c2fca1d796f24484046a5",
                "tx_hex": "0400008085202f892082f6916932f9bf674a3fb00c3d5d765303ab68461f4abe0f91cc1162546914a9010000006b483045022100999b8bb0224476b5c344a466d0051ec7a8c312574ad8956a4177a42625cb86e302205a6664396bff3f2e6fe57adb7e082a26d1b8da9ee77b3fc24aa4148fdd5c84db012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffffcad29a146b81bcaa44744efbec5149b6e3ca32bace140f75ad5794288d5bff6c000000006b483045022100b4dbfe88561c201fb8fbaf5bbf5bc0985893c909429c579425da84b02d23cc12022075f1e1e3eba38d167a6e84aac23faee5a2eb0799511e647213cee168529d4e5d012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffffa13eeacd04b3e26ae3f41530b560c615dafa0fd4235cc5b22d48ab97e7c3399c000000006a47304402201158306fe668cbf56ad3f586dc83c1cda9efab44cef46da6bc0fe242292c85ed02201d622fe283410320e760233ae81dc53df65406b09fd07f8649f1775689219c35012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff4352b9f1f01dde4209b9e91076a3cfcabdaa23d9d5a313abfe7edb67ee4273e3000000006b483045022100825242fb3c6d460580016e93718ae1f43917e53abcc1558a64a6ab6f406763dd0220543936ce4c725e5e9f03831274a8475b535171bb29e1919fcf52ba2a9c85a553012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffffcc0fa94b5973c893e61d470ae3982b0bedfd29cb0da2c60a362de438598f108c000000006b4830450221008c70a8e10ca37819e5a4d9783366e729e690d78f2fdd8a1f4812ddc14ec7d6ad022035ba8cb4d4e50684107f8af5c184582687b5d7dfda5d9be1bd45e45749c77f08012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffffb0bd3bb9fedb7bbf49ca1612c955ba6095202186cef5be6952aed3dd32da4268000000006a4730440220592216d63c199faa587a4a6cbe11ca26027368a116b50818ce30eced59ca887202201bcafcf88f9f2632151596732f839d77cbe2f2243822c8551faffecc90b5dc19012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff65cf2831fc200e55aaacbe0881ad0edfb298ee6d4421b08b048aecc151716bd1000000006a47304402202032eb1ccebc3be4b94bae343d1d168e87040d2d20977c47d073d6bf490ef6c00220067656e00c4b8930167c54078609925cec7b893a52bcb9304e6b2602f564413e012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffffeaf67880bee214acecc74b12f648c1014d6394c4abf99832d408981bb460e999000000006b483045022100b9ae1cc824149220ac517298e6f21c26939485b31d0ae19d97d986c5f8f34e4502200a90578cf2c1835dbea00484af1f225711c255f1d0a3208f2e4f1154f0db2c9a012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffffad089c3fe7987a44f150f34b7ac66972de76dd84c739bdeddf360ab029dfd4d6000000006a473044022015f0386ed67a61626fbe5ae79e0d39d38e7b4072b648e8a26e23adadc0a8e5bc02202398188fa2feb26994e5c1e7e758788de3d5f0f0096f956a0cd58804710bea6a012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffffd6c66730546c62dd003b5af1f1e5ecfd339c62db0169c1d499584e09a8a9b288000000006b4830450221008d4c73f0e3c9d913ba32fd864167649242e3e891412ab80bdd3f7ff43a238ee20220602738e98008b146256b51d0df99e222aa165f2ce351241ebc23d8a098e2b0db012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff12d9eff354f46cbd4446a0bff27a6a635ff5b1dc8a5dd8b0178bb5f89c9ec080000000006b48304502210098d3349ba9b13560748949933d2704663a5ab52cdc804afa1ac4da3e5992e0a002201525d7ad8466ad260219f3873fb7781addbd363f91e8063bfa86c7ed4e385b84012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff69e16767806ea5f069b7d46674f7aa747fcc6e541189ce7fcf92edcfd7642ff4000000006b4830450221008a5ebfe904c87f21947a44d8418190be5893993a683fde0f96df8a9487923da002205be1bbd8b518ba2f303cae23bc20806e84ffbba6a03f032385b15edb8df107f4012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640fffffffff4fbabd508178f553e676d67ce325796b03aa249b41a23c681c1ad9dedb45ae7000000006a47304402207cea6824abe1ce35e18954b858d45243e2cb57d27d782adc5b6b07ebd21a02d7022007ba0469b298c4b1a7c4a148fa16bae93d28593b34e197c10ac0d1faf9cc1bfa012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff14867aa59932d895be607fb7398f5594e39d9fa2e1c7daaed7b1390dbfdddcab000000006b4830450221009fb6e1885a3658c593809f95ecd2049f8ef9e00379686ac236b17312c9613d4c0220709fc50c9a920a19254389944db366c354708c19885d2479d9968fda0848f9f7012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff75777c692daaa21d216a1a2a7059203becfcdcf6793aa1259cdd7aadec957ab6000000006a47304402202945019860abf9b80e71f340320d114846efa4d2780ce12513e3983fb4d3f15b022019be008fb7368e3f1f022924dc7af1138b94041f46084dd27768bc8cacd1529f012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffffca037b85742e93df4eef8e8ac3b8531321c8a8e21a4a941360866ea57a973665000000006a4730440220760283a7828edcc53671fc73e29c30cdc64d60d300292761d39730f0d09f94c202201e026293e3891a6fe46e40cd21778a41e21641a261a7fbf3bf75b034d9c788d9012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffffa68edd030b4307ad87bfeff96a6db5b3ddd1a0901c488a4fe4d2093531896d75000000006b48304502210091a41e16b2c27d7ef6077e8de9df692b6013e61d72798ff9f7eba7fc983cdb65022034de29a0fb22a339e8044349913915444ab420772ab0ab423e44cfe073cb4e70012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff8c952791181993a7512e48d098a06e6197c993b83241a4bf1330c0e95f2c304d000000006b483045022100fa14b9301feb056f6e6b10446a660525cc1ff3e191b0c45f9e12dcd4f142422c02203f4a94f2a9d3ec0b74fac2156dd9b1addb8fa5b9a1cfc9e34b0802e88b1cbfa3012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff32bc4d014542abf328fecff29b9f4c243c3dd295fe42524b20bf591a3ddc26a1000000006a47304402206f92c4da6651c8959f7aed61608d26b9e46f5c1d69f4fc6e592b1f552b6067f102201c8cc221eac731867d15d483cc83322dba2f14f31d3efb26be937a68ad772394012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffffbb3877248c26b23023d7dbb83a5f8293c65c5bff4ac47935a4a31248cefffd91000000006a47304402205bab19ad082a1918e18ccb6462edc263196fb88c8fdfd6bd07a0cf031a4637810220371a621c1bdc6b957db2447a92dcf87b0309653a2db480c9ed623f34a6e6d8a9012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff6415b7356c94609b9a7a7eb06e4c306896767abbc11399779f952fb9ae197059000000006b483045022100e2d038dbb9a873f5a58ec7901d6a7e79f1b404afea3d852056f4d0746cfb821102207fb274947b10d467cd71aa948e9a50f5f4430b661b27afc347efd9d6cc409d9c012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff1aeefdf80ec8a07d657ca64a2c0aa465f58e6284755c9a263c5a807be43b4b81000000006a47304402206e7ff765ba47a8785008f64f49c8e73232d582b2b2d0a49be0880c2557de8f8602206448423a6a37ad9740eb316513b31f73599ae14f65623709fb5443ae609f3e2e012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff3c091681df17b46f280bc9d8011c1bb31397637ce945b393f70380f8cd0a8b0d010000006a47304402206ca8717000f3086d364318f56d52e2369c40b88a1cb86455a8db262b4816698a02206711caf453bfda6b1b3542e27e68c3180f92f0548326d74e30b3ed18cd2c2353012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff91f32d98b581def165495aff6b69530e1f3de7f68fabfeb93730cf9793bbcd2a000000006a47304402200a8cd5e29ee7ff136772ea1789a39a027eaa1cd92f90f9d57fd8cf77202251f402203dd2bc282a838a5730e840a0d22b4f0edbe3cb2da00466c66bc2b5c66fc8b032012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff854d9226c28a1f5fe440e08f41000f3547f304ecf9cc010d0b5bc845ef1f039a000000006b483045022100fe6cce49975cc78af1c394bc02d995710833ba08cf7f8dd5f99add2cc7db26c40220793491309c215d8314a1c142bef7ec6b9a397249bec1c00a0a5ab47dfc1208b6012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff593bc907aa71f3b0f7aa8c48bb5f650595e65a5a733a9601a8374ed978eec9a7000000006a47304402206362ae3c4cf1a19ba0e43424b03af542077b49761172c1ad26d802f54b1b6ca602206bc7edb655bb0024c0e48c1f4c18c8864f8d1ce59ae55cd81dc0bd1234430691012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff3b9da8c5ab0c0cd6b40f602ea6ed8e36a48034b182b9d1a77ffebd15fe203b94000000006b483045022100f8610eae25899663cb5fa9a4575d937da57cdfd41958794bbb4c02f8bed75da40220262d40e019ec3a57b252f4150d509cce6f8a2dbd83184a9fc2ed56aba8018b15012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff0897c8a57e15e7f3893b195d65cf6c6001b29c8c9734213d7a3131f57b9eca2e000000006b483045022100c485cbd6408cf0759bcf23c4154249882934b522a93c6b49e62412305bf7646902201cc4b668af4bb22fe57c32c4d34e822bceb12f6bd6923afdabf4894752a56ec3012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffffffdc7000f7c45b62960623fa3a277e8a55348a4fe4936fef1224b6953434a249000000006b4830450221008a51a9c26f475d5c0838afe9d51524f95adfb21a9b0a02eae31cb01dc0a31fab022071c5492fbc7270731d4a4947a69398bf99dd28c65bb69d19910bf53a515274c8012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff10ec2af7e31ca28e27177215904d9a59abf80f0652b24e3f749f14fb7b2264ec000000006b483045022100fe4269f8f5ca53ebcff6fb782142a6228f0e50498a531b7a9c0d54768af9854102207cc740a9ea359569b49d69a94215ce3e23aeda5779cebc434ad3d608e1752990012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff5e3830c088dd6ea412d778b0a700ef27c183cf03e19f3d6f71bc5eaf53b2c22e000000006b4830450221009788a7e7f2407ba2f7c504091fbdf8f8498367781e8a357616d68e2a6770b4e70220518c92f5fb21e6bfd7d870a783b2a5572ce003f2dbb237ec59df419c9a148966012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff51630ccb0ad32b24cc7ae1b3602950ba518dca6aa65ef560e57f08c23eed8d80000000006a47304402201aa556153ffeb13aa674353bf88c04a7af15c7eb32e1a835464e4b613c31dc2802200395858c29a46e9108de1f90b401ee26c296388b4073143b63f849b2cce461af012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff0200e1f5050000000017a914ab802c4d644be63fd1a72834ff63b650d6b5353987bb7e1e00000000001976a91464ae8510aac9546d5e7704e31ce177451386455588ac680e135d000000000000000000000000000000"
              },
              "type": "TakerPaymentReceived"
            },
            "timestamp": 1561529998938
          },
          {
            "event": {
              "type": "TakerPaymentWaitConfirmStarted"
            },
            "timestamp": 1561529998941
          },
          {
            "event": {
              "type": "TakerPaymentValidatedAndConfirmed"
            },
            "timestamp": 1561530000859
          },
          {
            "event": {
              "data": {
                "block_height": 0,
                "coin": "PIZZA",
                "fee_details": {
                  "amount": "0.00001"
                },
                "from": ["bUN5nesdt1xsAjCtAaYUnNbQhGqUWwQT1Q"],
                "internal_id": "235f8e7ab3c9515a17fe8ee721ef971bbee273eb90baf70788edda7b73138c86",
                "my_balance_change": "0.99999",
                "received_by_me": "0.99999",
                "spent_by_me": "0",
                "timestamp": 0,
                "to": ["R9o9xTocqr6CeEDGDH6mEYpwLoMz6jNjMW"],
                "total_amount": "1",
                "tx_hash": "235f8e7ab3c9515a17fe8ee721ef971bbee273eb90baf70788edda7b73138c86",
                "tx_hex": "0400008085202f8901a5464048246f791dca2f8cef2774125992cba7c0b820f32e7980be1de3380e7e00000000d8483045022100beca668a946fcad98da64cc56fa04edd58b4c239aa1362b4453857cc2e0042c90220606afb6272ef0773185ade247775103e715e85797816fbc204ec5128ac10a4b90120ea774bc94dce44c138920c6e9255e31d5645e60c0b64e9a059ab025f1dd2fdc6004c6b6304692c135db1752102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ac6782012088a914eb78e2f0cf001ed7dc69276afd37b25c4d6bb491882102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ac68ffffffff0118ddf505000000001976a91405aab5342166f8594baf17a7d9bef5d56744332788ac8000135d000000000000000000000000000000"
              },
              "type": "TakerPaymentSpent"
            },
            "timestamp": 1561530003429
          },
          {
            "event": {
              "type": "Finished"
            },
            "timestamp": 1561530003433
          }
        ],
        "my_info": {
          "my_amount": "1",
          "my_coin": "BEER",
          "other_amount": "1",
          "other_coin": "PIZZA",
          "started_at": 1561529842
        },
        "maker_coin": "BEER",
        "maker_amount": "1",
        "taker_coin": "PIZZA",
        "taker_amount": "1",
        "gui": null,
        "mm_version": "unknown", 
        "success_events": [
          "Started",
          "Negotiated",
          "TakerFeeValidated",
          "MakerPaymentSent",
          "TakerPaymentReceived",
          "TakerPaymentWaitConfirmStarted",
          "TakerPaymentValidatedAndConfirmed",
          "TakerPaymentSpent",
          "Finished"
        ],
        "type": "Maker",
        "uuid": "6bf6e313-e610-4a9a-ba8c-57fc34a124aa"
      },
      {
        "error_events": [
          "StartFailed",
          "NegotiateFailed",
          "TakerFeeSendFailed",
          "MakerPaymentValidateFailed",
          "TakerPaymentTransactionFailed",
          "TakerPaymentDataSendFailed",
          "TakerPaymentWaitForSpendFailed",
          "MakerPaymentSpendFailed",
          "TakerPaymentRefunded",
          "TakerPaymentRefundFailed"
        ],
        "events": [
          {
            "event": {
              "data": {
                "lock_duration": 31200,
                "maker": "5a2f1c468b7083c4f7649bf68a50612ffe7c38b1d62e1ece3829ca88e7e7fd12",
                "maker_amount": "0.01",
                "maker_coin": "BEER",
                "maker_coin_start_block": 154187,
                "maker_payment_confirmations": 1,
                "maker_payment_wait": 1561492367,
                "my_persistent_pub": "02031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3",
                "started_at": 1561481967,
                "taker_amount": "0.01",
                "taker_coin": "BCH",
                "taker_coin_start_block": 588576,
                "taker_payment_confirmations": 1,
                "taker_payment_lock": 1561513167,
                "uuid": "491df802-43c3-4c73-85ef-1c4c49315ac6"
              },
              "type": "Started"
            },
            "timestamp": 1561481968393
          },
          {
            "event": {
              "data": {
                "maker_payment_locktime": 1561544367,
                "maker_pubkey": "02631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640",
                "secret_hash": "ba5128bcca5a2f7d2310054fb8ec51b80f352ef3"
              },
              "type": "Negotiated"
            },
            "timestamp": 1561482029079
          },
          {
            "event": {
              "data": {
                "block_height": 0,
                "coin": "BCH",
                "fee_details": {
                  "amount": "0.00001"
                },
                "from": ["1WxswvLF2HdaDr4k77e92VjaXuPQA8Uji"],
                "internal_id": "9dd7c0c8124315d7884fb0c7bf8dbfd3f3bd185c62a2ee42dfbc1e3b74f21a0e",
                "my_balance_change": "0.00002287",
                "received_by_me": "0.0155402",
                "spent_by_me": "0.01556307",
                "timestamp": 0,
                "to": [
                  "1KRhTPvoxyJmVALwHFXZdeeWFbcJSbkFPu",
                  "1WxswvLF2HdaDr4k77e92VjaXuPQA8Uji"
                ],
                "total_amount": "0.01556307",
                "tx_hash": "9dd7c0c8124315d7884fb0c7bf8dbfd3f3bd185c62a2ee42dfbc1e3b74f21a0e",
                "tx_hex": "0100000001f1beda7feba9fa5c52aa38027587db50b6428bbbcc053cd4ab17461fb00b89d1000000006a473044022004ad0330210e20dea416c3ff442e50dc59970c5d1a8b4d0a7d5cc61a2edc701602204459e1ee6774f1ba8258322fff72e1e1acddeb7aed2f75657458aa3deecc9465412102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffff0207050000000000001976a914ca1e04745e8ca0c60d8c5881531d51bec470743f88ac64b61700000000001976a91405aab5342166f8594baf17a7d9bef5d56744332788ac2d53125d"
              },
              "type": "TakerFeeSent"
            },
            "timestamp": 1561482032294
          },
          {
            "event": {
              "data": {
                "block_height": 154190,
                "coin": "BEER",
                "fee_details": {
                  "amount": "0.00001"
                },
                "from": ["RJTYiYeJ8eVvJ53n2YbrVmxWNNMVZjDGLh"],
                "internal_id": "ba36c890785e3e9d4b853310ad4d79ce8175e7c4184a398128b37339321672f4",
                "my_balance_change": "0",
                "received_by_me": "0",
                "spent_by_me": "0",
                "timestamp": 1561482056,
                "to": [
                  "RJTYiYeJ8eVvJ53n2YbrVmxWNNMVZjDGLh",
                  "bF2S8qwenfVZbvUU6dWyV3oXMxEP7sHLbr"
                ],
                "total_amount": "0.99999",
                "tx_hash": "ba36c890785e3e9d4b853310ad4d79ce8175e7c4184a398128b37339321672f4",
                "tx_hex": "0400008085202f890197f703d245127e5b88471791f2820d29152046f4be133907afa8ac5542911190000000006b48304502210090e1c52aa2eba12b7c71fceab83b77f1456830a3dee1b956a831ecee5b5b353602205353a48c0129eae44b7c06a1f1651b9ceb8642374a1d5224a1e907240a978ad2012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff0240420f000000000017a914192f34528c6c8cd11eefebec27f195f3894eb11187f096e605000000001976a91464ae8510aac9546d5e7704e31ce177451386455588ac4353125d000000000000000000000000000000"
              },
              "type": "MakerPaymentReceived"
            },
            "timestamp": 1561482073479
          },
          {
            "event": {
              "type": "MakerPaymentWaitConfirmStarted"
            },
            "timestamp": 1561482073482
          },
          {
            "event": {
              "type": "MakerPaymentValidatedAndConfirmed"
            },
            "timestamp": 1561482074296
          },
          {
            "event": {
              "data": {
                "block_height": 0,
                "coin": "BCH",
                "fee_details": {
                  "amount": "0.00001"
                },
                "from": ["1WxswvLF2HdaDr4k77e92VjaXuPQA8Uji"],
                "internal_id": "bc98def88d93c270ae3cdb8a098d1b939ca499bf98f7a22b97be36bca13cdbc7",
                "my_balance_change": "-0.01001",
                "received_by_me": "0.0055302",
                "spent_by_me": "0.0155402",
                "timestamp": 0,
                "to": [
                  "1WxswvLF2HdaDr4k77e92VjaXuPQA8Uji",
                  "31k5nkp5G9QHq3zZFFba6Kq3m5FEHstkrd"
                ],
                "total_amount": "0.0155402",
                "tx_hash": "bc98def88d93c270ae3cdb8a098d1b939ca499bf98f7a22b97be36bca13cdbc7",
                "tx_hex": "01000000010e1af2743b1ebcdf42eea2625c18bdf3d3bf8dbfc7b04f88d7154312c8c0d79d010000006a4730440220030266d6d6435a4772cce2cebd91b6d4afffb920e23e9bc761434f105349cda002202335a050e2f28e4ca28862868141d3d7b553f3d30bceb83724ad70a32d04b0bd412102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffff0240420f000000000017a9140094798ed4100852f10a9ad85990f19b364f4c2d873c700800000000001976a91405aab5342166f8594baf17a7d9bef5d56744332788ac5a53125d"
              },
              "type": "TakerPaymentSent"
            },
            "timestamp": 1561482078908
          },
          {
            "event": {
              "data": {
                "secret": "66ed6c24bbb4892634eac4ce1e1ad0627d6379da4443b8d656b64d49ef2aa7a3",
                "transaction": {
                  "block_height": 0,
                  "coin": "BCH",
                  "fee_details": {
                    "amount": "0.00001"
                  },
                  "from": ["31k5nkp5G9QHq3zZFFba6Kq3m5FEHstkrd"],
                  "internal_id": "eec643315d4495aa5feb5062344fe2474223dc0f231b610afd336f908ae99ebc",
                  "my_balance_change": "0",
                  "received_by_me": "0",
                  "spent_by_me": "0",
                  "timestamp": 0,
                  "to": ["1ABMe2m1XphME4gaZNcjQFdJc6ttu2Gbz2"],
                  "total_amount": "0.01",
                  "tx_hash": "eec643315d4495aa5feb5062344fe2474223dc0f231b610afd336f908ae99ebc",
                  "tx_hex": "0100000001c7db3ca1bc36be972ba2f798bf99a49c931b8d098adb3cae70c2938df8de98bc00000000d747304402202e344f8c61f2f49f4d620d687d02448cfba631a8ce8c0f8ee774da177230a75902201f4a175e7fa40f26896f522b5c51c7c0485e0ad18d3221c885e8b96b52ed1cab412066ed6c24bbb4892634eac4ce1e1ad0627d6379da4443b8d656b64d49ef2aa7a3004c6b6304cfcc125db1752102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ac6782012088a914ba5128bcca5a2f7d2310054fb8ec51b80f352ef3882102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ac68ffffffff01583e0f00000000001976a91464ae8510aac9546d5e7704e31ce177451386455588acfd49125d"
                }
              },
              "type": "TakerPaymentSpent"
            },
            "timestamp": 1561483355081
          },
          {
            "event": {
              "data": {
                "block_height": 0,
                "coin": "BEER",
                "fee_details": {
                  "amount": "0.00001"
                },
                "from": ["bF2S8qwenfVZbvUU6dWyV3oXMxEP7sHLbr"],
                "internal_id": "858f07d0a4e74318497a6e3ff4d7b68b60ad21b5c8e90b9b485f0ddaed71d0dc",
                "my_balance_change": "0.00999",
                "received_by_me": "0.00999",
                "spent_by_me": "0",
                "timestamp": 0,
                "to": ["R9o9xTocqr6CeEDGDH6mEYpwLoMz6jNjMW"],
                "total_amount": "0.01",
                "tx_hash": "858f07d0a4e74318497a6e3ff4d7b68b60ad21b5c8e90b9b485f0ddaed71d0dc",
                "tx_hex": "0400008085202f8901f47216323973b32881394a18c4e77581ce794dad1033854b9d3e5e7890c836ba00000000d8483045022100847a65faed4bea33c5cbccff2bee7c1292871a3b130bd2f23e696bd80c07365f02202039ea02b4463afd4f1e2b20b348d64b40aaea165f8dfb483293e2b368d536fe012066ed6c24bbb4892634eac4ce1e1ad0627d6379da4443b8d656b64d49ef2aa7a3004c6b6304af46135db1752102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ac6782012088a914ba5128bcca5a2f7d2310054fb8ec51b80f352ef3882102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ac68ffffffff01583e0f00000000001976a91405aab5342166f8594baf17a7d9bef5d56744332788ac4b4a125d000000000000000000000000000000"
              },
              "type": "MakerPaymentSpent"
            },
            "timestamp": 1561483358319
          },
          {
            "event": {
              "type": "Finished"
            },
            "timestamp": 1561483358321
          }
        ],
        "my_info": {
          "my_amount": "0.01",
          "my_coin": "BCH",
          "other_amount": "0.01",
          "other_coin": "BEER",
          "started_at": 1561481967
        },
        "maker_coin": "BEER",
        "maker_amount": "0.01",
        "taker_coin": "BCH",
        "taker_amount": "0.01",
        "gui": null,
        "mm_version": "unknown", 
        "success_events": [
          "Started",
          "Negotiated",
          "TakerFeeSent",
          "MakerPaymentReceived",
          "MakerPaymentWaitConfirmStarted",
          "MakerPaymentValidatedAndConfirmed",
          "TakerPaymentSent",
          "TakerPaymentSpent",
          "MakerPaymentSpent",
          "Finished"
        ],
        "type": "Taker",
        "uuid": "491df802-43c3-4c73-85ef-1c4c49315ac6"
      }
    ],
    "total": 49
  }
}
```

Response (error)

```json
{
  "error": "lp_swap:1454] from_uuid e299c6ece7a7ddc42444eda64d46b163eaa992da65ce6de24eb812d715184e41 swap is not found"
}
```

</collapse-text>

</div>

## my\_swap\_status

**uuid**

The `my_swap_status` method returns the data of an atomic swap executed on a MM2 node.

#### Arguments

| Structure   | Type   | Description                                                 |
| ----------- | ------ | ----------------------------------------------------------- |
| params uuid | string | the uuid of swap, typically received from the buy/sell call |

#### Response

| Structure      | Type                      | Description                                                                                                                                |
| -------------- | ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| events         | array of objects          | the events that occurred during the swap                                                                                                   |
| success_events | array of strings          | a list of events that gained a `success` swap state; the contents are listed in the order in which they should occur in the `events` array |
| error_events   | array of strings          | a list of events that fell into an `error` swap state; if at least 1 of the events happens, the swap is considered a failure               |
| type           | string                    | whether the node acted as a market `Maker` or `Taker`                                                                                      |
| uuid           | string                    | swap uuid                                                                                                                                  |
| gui            | string (optional)         | information about gui; copied from MM2 configuration                                                                                       |
| mm_version     | string (optional)         | MM2 version                                                                                                                                |
| maker_coin     | string (optional)         | ticker of maker coin                                                                                                                       |
| taker_coin     | string (optional)         | ticker of taker coin                                                                                                                       |
| maker_amount   | string (numeric, optional)| the amount of coins to be swapped by maker                                                                                                 |
| taker_amount   | string (numeric, optional)| the amount of coins to be swapped by taker                                                                                                 |
| my_info        | object (optional)         | this object maps event data to make displaying swap data in a GUI simpler (`my_coin`, `my_amount`, etc.)                                   |
| recoverable    | bool                      | whether the swap can be recovered using the `recover_funds_of_swap` API command. Important note: MM2 does not record the state regarding whether the swap was recovered or not. MM2 allows as many calls to the `recover_funds_of_swap` method as necessary, in case of errors |

#### :pushpin: Examples

#### Command

```bash
curl --url "http://127.0.0.1:7783" --data "{\"method\":\"my_swap_status\",\"params\":{\"uuid\":\"d14452bb-e82d-44a0-86b0-10d4cdcb8b24\"},\"userpass\":\"$userpass\"}"
```

<div style="margin-top: 0.5rem;">

<collapse-text hidden title="Response">

#### Response (Taker swap)

```json
{
  "result": {
    "error_events": [
      "StartFailed",
      "NegotiateFailed",
      "TakerFeeSendFailed",
      "MakerPaymentValidateFailed",
      "TakerPaymentTransactionFailed",
      "TakerPaymentDataSendFailed",
      "TakerPaymentWaitForSpendFailed",
      "MakerPaymentSpendFailed",
      "TakerPaymentRefunded",
      "TakerPaymentRefundFailed"
    ],
    "events": [
      {
        "event": {
          "data": {
            "lock_duration": 31200,
            "maker": "5a2f1c468b7083c4f7649bf68a50612ffe7c38b1d62e1ece3829ca88e7e7fd12",
            "maker_amount": "0.01",
            "maker_coin": "BEER",
            "maker_coin_start_block": 154187,
            "maker_payment_confirmations": 1,
            "maker_payment_wait": 1561492367,
            "my_persistent_pub": "02031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3",
            "started_at": 1561481967,
            "taker_amount": "0.01",
            "taker_coin": "BCH",
            "taker_coin_start_block": 588576,
            "taker_payment_confirmations": 1,
            "taker_payment_lock": 1561513167,
            "uuid": "491df802-43c3-4c73-85ef-1c4c49315ac6"
          },
          "type": "Started"
        },
        "timestamp": 1561481968393
      },
      {
        "event": {
          "data": {
            "maker_payment_locktime": 1561544367,
            "maker_pubkey": "02631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640",
            "secret_hash": "ba5128bcca5a2f7d2310054fb8ec51b80f352ef3"
          },
          "type": "Negotiated"
        },
        "timestamp": 1561482029079
      },
      {
        "event": {
          "data": {
            "block_height": 0,
            "coin": "BCH",
            "fee_details": {
              "amount": "0.00001"
            },
            "from": ["1WxswvLF2HdaDr4k77e92VjaXuPQA8Uji"],
            "internal_id": "9dd7c0c8124315d7884fb0c7bf8dbfd3f3bd185c62a2ee42dfbc1e3b74f21a0e",
            "my_balance_change": "0.00002287",
            "received_by_me": "0.0155402",
            "spent_by_me": "0.01556307",
            "timestamp": 0,
            "to": [
              "1KRhTPvoxyJmVALwHFXZdeeWFbcJSbkFPu",
              "1WxswvLF2HdaDr4k77e92VjaXuPQA8Uji"
            ],
            "total_amount": "0.01556307",
            "tx_hash": "9dd7c0c8124315d7884fb0c7bf8dbfd3f3bd185c62a2ee42dfbc1e3b74f21a0e",
            "tx_hex": "0100000001f1beda7feba9fa5c52aa38027587db50b6428bbbcc053cd4ab17461fb00b89d1000000006a473044022004ad0330210e20dea416c3ff442e50dc59970c5d1a8b4d0a7d5cc61a2edc701602204459e1ee6774f1ba8258322fff72e1e1acddeb7aed2f75657458aa3deecc9465412102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffff0207050000000000001976a914ca1e04745e8ca0c60d8c5881531d51bec470743f88ac64b61700000000001976a91405aab5342166f8594baf17a7d9bef5d56744332788ac2d53125d"
          },
          "type": "TakerFeeSent"
        },
        "timestamp": 1561482032294
      },
      {
        "event": {
          "data": {
            "block_height": 154190,
            "coin": "BEER",
            "fee_details": {
              "amount": "0.00001"
            },
            "from": ["RJTYiYeJ8eVvJ53n2YbrVmxWNNMVZjDGLh"],
            "internal_id": "ba36c890785e3e9d4b853310ad4d79ce8175e7c4184a398128b37339321672f4",
            "my_balance_change": "0",
            "received_by_me": "0",
            "spent_by_me": "0",
            "timestamp": 1561482056,
            "to": [
              "RJTYiYeJ8eVvJ53n2YbrVmxWNNMVZjDGLh",
              "bF2S8qwenfVZbvUU6dWyV3oXMxEP7sHLbr"
            ],
            "total_amount": "0.99999",
            "tx_hash": "ba36c890785e3e9d4b853310ad4d79ce8175e7c4184a398128b37339321672f4",
            "tx_hex": "0400008085202f890197f703d245127e5b88471791f2820d29152046f4be133907afa8ac5542911190000000006b48304502210090e1c52aa2eba12b7c71fceab83b77f1456830a3dee1b956a831ecee5b5b353602205353a48c0129eae44b7c06a1f1651b9ceb8642374a1d5224a1e907240a978ad2012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff0240420f000000000017a914192f34528c6c8cd11eefebec27f195f3894eb11187f096e605000000001976a91464ae8510aac9546d5e7704e31ce177451386455588ac4353125d000000000000000000000000000000"
          },
          "type": "MakerPaymentReceived"
        },
        "timestamp": 1561482073479
      },
      {
        "event": {
          "type": "MakerPaymentWaitConfirmStarted"
        },
        "timestamp": 1561482073482
      },
      {
        "event": {
          "type": "MakerPaymentValidatedAndConfirmed"
        },
        "timestamp": 1561482074296
      },
      {
        "event": {
          "data": {
            "block_height": 0,
            "coin": "BCH",
            "fee_details": {
              "amount": "0.00001"
            },
            "from": ["1WxswvLF2HdaDr4k77e92VjaXuPQA8Uji"],
            "internal_id": "bc98def88d93c270ae3cdb8a098d1b939ca499bf98f7a22b97be36bca13cdbc7",
            "my_balance_change": "-0.01001",
            "received_by_me": "0.0055302",
            "spent_by_me": "0.0155402",
            "timestamp": 0,
            "to": [
              "1WxswvLF2HdaDr4k77e92VjaXuPQA8Uji",
              "31k5nkp5G9QHq3zZFFba6Kq3m5FEHstkrd"
            ],
            "total_amount": "0.0155402",
            "tx_hash": "bc98def88d93c270ae3cdb8a098d1b939ca499bf98f7a22b97be36bca13cdbc7",
            "tx_hex": "01000000010e1af2743b1ebcdf42eea2625c18bdf3d3bf8dbfc7b04f88d7154312c8c0d79d010000006a4730440220030266d6d6435a4772cce2cebd91b6d4afffb920e23e9bc761434f105349cda002202335a050e2f28e4ca28862868141d3d7b553f3d30bceb83724ad70a32d04b0bd412102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffff0240420f000000000017a9140094798ed4100852f10a9ad85990f19b364f4c2d873c700800000000001976a91405aab5342166f8594baf17a7d9bef5d56744332788ac5a53125d"
          },
          "type": "TakerPaymentSent"
        },
        "timestamp": 1561482078908
      },
      {
        "event": {
          "data": {
            "secret": "66ed6c24bbb4892634eac4ce1e1ad0627d6379da4443b8d656b64d49ef2aa7a3",
            "transaction": {
              "block_height": 0,
              "coin": "BCH",
              "fee_details": {
                "amount": "0.00001"
              },
              "from": ["31k5nkp5G9QHq3zZFFba6Kq3m5FEHstkrd"],
              "internal_id": "eec643315d4495aa5feb5062344fe2474223dc0f231b610afd336f908ae99ebc",
              "my_balance_change": "0",
              "received_by_me": "0",
              "spent_by_me": "0",
              "timestamp": 0,
              "to": ["1ABMe2m1XphME4gaZNcjQFdJc6ttu2Gbz2"],
              "total_amount": "0.01",
              "tx_hash": "eec643315d4495aa5feb5062344fe2474223dc0f231b610afd336f908ae99ebc",
              "tx_hex": "0100000001c7db3ca1bc36be972ba2f798bf99a49c931b8d098adb3cae70c2938df8de98bc00000000d747304402202e344f8c61f2f49f4d620d687d02448cfba631a8ce8c0f8ee774da177230a75902201f4a175e7fa40f26896f522b5c51c7c0485e0ad18d3221c885e8b96b52ed1cab412066ed6c24bbb4892634eac4ce1e1ad0627d6379da4443b8d656b64d49ef2aa7a3004c6b6304cfcc125db1752102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ac6782012088a914ba5128bcca5a2f7d2310054fb8ec51b80f352ef3882102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ac68ffffffff01583e0f00000000001976a91464ae8510aac9546d5e7704e31ce177451386455588acfd49125d"
            }
          },
          "type": "TakerPaymentSpent"
        },
        "timestamp": 1561483355081
      },
      {
        "event": {
          "data": {
            "block_height": 0,
            "coin": "BEER",
            "fee_details": {
              "amount": "0.00001"
            },
            "from": ["bF2S8qwenfVZbvUU6dWyV3oXMxEP7sHLbr"],
            "internal_id": "858f07d0a4e74318497a6e3ff4d7b68b60ad21b5c8e90b9b485f0ddaed71d0dc",
            "my_balance_change": "0.00999",
            "received_by_me": "0.00999",
            "spent_by_me": "0",
            "timestamp": 0,
            "to": ["R9o9xTocqr6CeEDGDH6mEYpwLoMz6jNjMW"],
            "total_amount": "0.01",
            "tx_hash": "858f07d0a4e74318497a6e3ff4d7b68b60ad21b5c8e90b9b485f0ddaed71d0dc",
            "tx_hex": "0400008085202f8901f47216323973b32881394a18c4e77581ce794dad1033854b9d3e5e7890c836ba00000000d8483045022100847a65faed4bea33c5cbccff2bee7c1292871a3b130bd2f23e696bd80c07365f02202039ea02b4463afd4f1e2b20b348d64b40aaea165f8dfb483293e2b368d536fe012066ed6c24bbb4892634eac4ce1e1ad0627d6379da4443b8d656b64d49ef2aa7a3004c6b6304af46135db1752102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ac6782012088a914ba5128bcca5a2f7d2310054fb8ec51b80f352ef3882102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ac68ffffffff01583e0f00000000001976a91405aab5342166f8594baf17a7d9bef5d56744332788ac4b4a125d000000000000000000000000000000"
          },
          "type": "MakerPaymentSpent"
        },
        "timestamp": 1561483358319
      },
      {
        "event": {
          "type": "Finished"
        },
        "timestamp": 1561483358321
      }
    ],
    "my_info": {
      "my_amount": "0.01",
      "my_coin": "BCH",
      "other_amount": "0.01",
      "other_coin": "BEER",
      "started_at": 1561481967
    },
    "maker_coin": "BEER",
    "maker_amount": "0.01",
    "taker_coin": "BCH",
    "taker_amount": "0.01",
    "gui": null,
    "mm_version": "unknown", 
    "recoverable": false,
    "success_events": [
      "Started",
      "Negotiated",
      "TakerFeeSent",
      "MakerPaymentReceived",
      "MakerPaymentWaitConfirmStarted",
      "MakerPaymentValidatedAndConfirmed",
      "TakerPaymentSent",
      "TakerPaymentSpent",
      "MakerPaymentSpent",
      "Finished"
    ],
    "type": "Taker",
    "uuid": "491df802-43c3-4c73-85ef-1c4c49315ac6"
  }
}
```

#### Response (Maker swap)

```json
{
  "result": {
    "error_events": [
      "StartFailed",
      "NegotiateFailed",
      "TakerFeeValidateFailed",
      "MakerPaymentTransactionFailed",
      "MakerPaymentDataSendFailed",
      "TakerPaymentValidateFailed",
      "TakerPaymentSpendFailed",
      "MakerPaymentRefunded",
      "MakerPaymentRefundFailed"
    ],
    "events": [
      {
        "event": {
          "data": {
            "lock_duration": 7800,
            "maker_amount": "1",
            "maker_coin": "BEER",
            "maker_coin_start_block": 154221,
            "maker_payment_confirmations": 1,
            "maker_payment_lock": 1561545442,
            "my_persistent_pub": "02031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3",
            "secret": "ea774bc94dce44c138920c6e9255e31d5645e60c0b64e9a059ab025f1dd2fdc6",
            "started_at": 1561529842,
            "taker": "5a2f1c468b7083c4f7649bf68a50612ffe7c38b1d62e1ece3829ca88e7e7fd12",
            "taker_amount": "1",
            "taker_coin": "PIZZA",
            "taker_coin_start_block": 141363,
            "taker_payment_confirmations": 1,
            "uuid": "6bf6e313-e610-4a9a-ba8c-57fc34a124aa"
          },
          "type": "Started"
        },
        "timestamp": 1561529842866
      },
      {
        "event": {
          "data": {
            "taker_payment_locktime": 1561537641,
            "taker_pubkey": "02631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640"
          },
          "type": "Negotiated"
        },
        "timestamp": 1561529883208
      },
      {
        "event": {
          "data": {
            "block_height": 141364,
            "coin": "PIZZA",
            "fee_details": {
              "amount": "0.00001"
            },
            "from": ["RJTYiYeJ8eVvJ53n2YbrVmxWNNMVZjDGLh"],
            "internal_id": "a91469546211cc910fbe4a1f4668ab0353765d3d0cb03f4a67bff9326991f682",
            "my_balance_change": "0",
            "received_by_me": "0",
            "spent_by_me": "0",
            "timestamp": 1561529907,
            "to": [
              "RJTYiYeJ8eVvJ53n2YbrVmxWNNMVZjDGLh",
              "RThtXup6Zo7LZAi8kRWgjAyi1s4u6U9Cpf"
            ],
            "total_amount": "0.002",
            "tx_hash": "a91469546211cc910fbe4a1f4668ab0353765d3d0cb03f4a67bff9326991f682",
            "tx_hex": "0400008085202f89021c7eeec33f8eb5ff2ed6c3d09e40e04b05a9674ea2feefcc12de3f9dcc16aff8000000006b483045022100e18e3d1afa8a24ecec82c92bfc05c119bfacdbb71b5f5663a4b96cc2a41ab269022017a79a1a1f6e0220d8fa1d2cf3b1c9788272f1ad18e4987b8f1cd4418acaa5b0012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff6a0d321eb52c3c7165adf80f83b15b7a5caa3a0dfa87746239021600d47fb43e000000006b483045022100937ed900e084d57d5e3341499fc66c5574884ca71cd4331fa696c8b7a517591b02201f5f851f94c3ca0ffb4789f1af22cb95dc83564e127ed7d23f1129eb2b981a2f012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff02bcf60100000000001976a914ca1e04745e8ca0c60d8c5881531d51bec470743f88ac9c120100000000001976a91464ae8510aac9546d5e7704e31ce177451386455588ac2f0e135d000000000000000000000000000000"
          },
          "type": "TakerFeeValidated"
        },
        "timestamp": 1561529927879
      },
      {
        "event": {
          "data": {
            "block_height": 0,
            "coin": "BEER",
            "fee_details": {
              "amount": "0.00001"
            },
            "from": ["R9o9xTocqr6CeEDGDH6mEYpwLoMz6jNjMW"],
            "internal_id": "efa90a2918e6793c8a2725c06ee34d0fa76c43bc85e680be195414e7aee36154",
            "my_balance_change": "-1.00001",
            "received_by_me": "0.0285517",
            "spent_by_me": "1.0285617",
            "timestamp": 0,
            "to": [
              "R9o9xTocqr6CeEDGDH6mEYpwLoMz6jNjMW",
              "bKuQbg7vgFR1C25vPqMq8ePnB25cUEAGpo"
            ],
            "total_amount": "1.0285617",
            "tx_hash": "efa90a2918e6793c8a2725c06ee34d0fa76c43bc85e680be195414e7aee36154",
            "tx_hex": "0400008085202f890cdcd071edda0d5f489b0be9c8b521ad608bb6d7f43f6e7a491843e7a4d0078f85000000006b483045022100fbc3bd09f8e1821ed671d1b1d2ed355833fb42c0bc435fef2da5c5b0a980b9a002204ef92b35576069d640ca0ac08f46645e5ade36afd5f19fb6aad19cfc9fb221fb012102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffffe6ae2a3ce221a6612d9e640bdbe10a2e477b3bc68a1aeee4a6784cb18648a785010000006a47304402202000a7e60ae2ce1529247875623ef2c5b42448dcaeac8de0f8f0e2f8e5bd8a6b0220426321a004b793172014f522efbca77a3dc92e86ce0a75330d8ceb83072ad4e7012102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffff9335553edcbac9559cae517a3e25b880a48fabf661c4ac338394972eef4572da000000006b4830450221008ded7230f2fb37a42b94f96174ec192baf4cd9e9e68fb9b6cf0463a36a6093e00220538de51ceda1617f3964a2350802377940fdfa018cc1043d77c66081b1cab0c4012102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3fffffffff91b5d3733877f84108de77fec46bee156766e1a6837fa7b580ccbc3905acb14000000006b483045022100d07cf1fd20e07aafdc942ba56f6b45baee61b93145a2bdba391e2cdb8024bf15022056ea8183990703ef05018df2fe8bd5ec678ec0f9207b0283292b2cdafc5e1e0c012102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffff147870387ca938b2b6e7daa96ba2496014f125c0e4e576273ef36ee8186c415a000000006a47304402204c5b15b641d7e34444456d2ea6663bdc8bd8216e309a7220814474f346b8425e0220634d1dd943b416b7a807704d7f7a3d46a60d88ef4e20734588a0b302c55fa82d012102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffffd2b954ae9b4a61fad9f7bc956d24e38d7b6fe313da824bd3bd91287d5a6b49d9000000006b483045022100a7387d9ab7b2c92d3cbce525e96ffac5ae3ef14f848661741ada0db17715c4a002202c1417d5e3e04b1a2d1774a83bb8d5aa1c0536c100138123089fa69921b5d976012102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffff28792a2e26d9d7be0467fac52b12ece67410b23eea845008257979bd87d083e3000000006a473044022027c40517c33cd3202d4310cfd2c75f38e6d7804b255fc3838a32ea26e5a3cb0002202b4399e1d7e655b64f699318f2bfbdced49f064ee54e9d6a678668fce51caf96012102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffffa8bf797bacd213b74a9977ae1b956afe3af33a1ee94324e010a16db891a07441000000006a473044022004cbb1d970b9f02c578b5c1d7de33361581eebc19c3cd8d2e50b0211ca4ef13702200c93b9fe5428055b6274dc8e52073c3e87f5b5e4206134d745928ccfc9393919012102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffff2b6fd82c9a68111b67ad85a614a6ecb50f7b6eac3d21d8ebefd9a6065cdf5729000000006b483045022100fdff16c595c7b4a9b4fc1e445b565f7b29fe5b7a08f79291b0ff585c7b72ac2902200c694aa124013bd419ce2349f15d10435827868d35db939b9d3c344d16e78420012102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffff6a5468dd8c83553dc51022f2a2fb772cf91c8607dc2ca1b8f203ac534612ab29000000006b483045022100ba7cc79e7ae3720238bfc5caa225dc8017d6a0d1cb1ec66abaf724fd20b3b7ab02206e8c942756604af0f63b74af495a9b3b7f4a44c489267f69a14cf2b1b953f46e012102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffff5f9f48a91d343fd5aef1d85f00850070931459ab256697afb728d1c81c1fa1d2000000006a47304402200ec85fc66f963e7504eb27361a4b4bb17de60e459da414fdc3962476de636134022056b62c15cf7f9b4e7d4e11c03e4e541dd348919b8c55efa4f1927e2fdd5ae8ea012102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffffee1f455924d3167e7f7abf452c1856e9abdcfe27dc889942dd249cb376169d38000000006b48304502210089274eed807c5d23d819f6dfa8a358a9748e56f2080be4396ef77bb19d91b17402207fc7b22c879534fffe0eeaaec8fc284e22c2756f380c05ea57b881a96b09f3af012102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffff0200e1f5050000000017a9144eb3a361d8a15d7f6a8ef9d1cf44962a90c44d548702912b00000000001976a91405aab5342166f8594baf17a7d9bef5d56744332788ac490e135d000000000000000000000000000000"
          },
          "type": "MakerPaymentSent"
        },
        "timestamp": 1561529938879
      },
      {
        "event": {
          "data": {
            "block_height": 141365,
            "coin": "PIZZA",
            "fee_details": {
              "amount": "0.00001"
            },
            "from": ["RJTYiYeJ8eVvJ53n2YbrVmxWNNMVZjDGLh"],
            "internal_id": "7e0e38e31dbe80792ef320b8c0a7cb9259127427ef8c2fca1d796f24484046a5",
            "my_balance_change": "0",
            "received_by_me": "0",
            "spent_by_me": "0",
            "timestamp": 1561529960,
            "to": [
              "RJTYiYeJ8eVvJ53n2YbrVmxWNNMVZjDGLh",
              "bUN5nesdt1xsAjCtAaYUnNbQhGqUWwQT1Q"
            ],
            "total_amount": "1.01999523",
            "tx_hash": "7e0e38e31dbe80792ef320b8c0a7cb9259127427ef8c2fca1d796f24484046a5",
            "tx_hex": "0400008085202f892082f6916932f9bf674a3fb00c3d5d765303ab68461f4abe0f91cc1162546914a9010000006b483045022100999b8bb0224476b5c344a466d0051ec7a8c312574ad8956a4177a42625cb86e302205a6664396bff3f2e6fe57adb7e082a26d1b8da9ee77b3fc24aa4148fdd5c84db012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffffcad29a146b81bcaa44744efbec5149b6e3ca32bace140f75ad5794288d5bff6c000000006b483045022100b4dbfe88561c201fb8fbaf5bbf5bc0985893c909429c579425da84b02d23cc12022075f1e1e3eba38d167a6e84aac23faee5a2eb0799511e647213cee168529d4e5d012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffffa13eeacd04b3e26ae3f41530b560c615dafa0fd4235cc5b22d48ab97e7c3399c000000006a47304402201158306fe668cbf56ad3f586dc83c1cda9efab44cef46da6bc0fe242292c85ed02201d622fe283410320e760233ae81dc53df65406b09fd07f8649f1775689219c35012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff4352b9f1f01dde4209b9e91076a3cfcabdaa23d9d5a313abfe7edb67ee4273e3000000006b483045022100825242fb3c6d460580016e93718ae1f43917e53abcc1558a64a6ab6f406763dd0220543936ce4c725e5e9f03831274a8475b535171bb29e1919fcf52ba2a9c85a553012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffffcc0fa94b5973c893e61d470ae3982b0bedfd29cb0da2c60a362de438598f108c000000006b4830450221008c70a8e10ca37819e5a4d9783366e729e690d78f2fdd8a1f4812ddc14ec7d6ad022035ba8cb4d4e50684107f8af5c184582687b5d7dfda5d9be1bd45e45749c77f08012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffffb0bd3bb9fedb7bbf49ca1612c955ba6095202186cef5be6952aed3dd32da4268000000006a4730440220592216d63c199faa587a4a6cbe11ca26027368a116b50818ce30eced59ca887202201bcafcf88f9f2632151596732f839d77cbe2f2243822c8551faffecc90b5dc19012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff65cf2831fc200e55aaacbe0881ad0edfb298ee6d4421b08b048aecc151716bd1000000006a47304402202032eb1ccebc3be4b94bae343d1d168e87040d2d20977c47d073d6bf490ef6c00220067656e00c4b8930167c54078609925cec7b893a52bcb9304e6b2602f564413e012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffffeaf67880bee214acecc74b12f648c1014d6394c4abf99832d408981bb460e999000000006b483045022100b9ae1cc824149220ac517298e6f21c26939485b31d0ae19d97d986c5f8f34e4502200a90578cf2c1835dbea00484af1f225711c255f1d0a3208f2e4f1154f0db2c9a012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffffad089c3fe7987a44f150f34b7ac66972de76dd84c739bdeddf360ab029dfd4d6000000006a473044022015f0386ed67a61626fbe5ae79e0d39d38e7b4072b648e8a26e23adadc0a8e5bc02202398188fa2feb26994e5c1e7e758788de3d5f0f0096f956a0cd58804710bea6a012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffffd6c66730546c62dd003b5af1f1e5ecfd339c62db0169c1d499584e09a8a9b288000000006b4830450221008d4c73f0e3c9d913ba32fd864167649242e3e891412ab80bdd3f7ff43a238ee20220602738e98008b146256b51d0df99e222aa165f2ce351241ebc23d8a098e2b0db012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff12d9eff354f46cbd4446a0bff27a6a635ff5b1dc8a5dd8b0178bb5f89c9ec080000000006b48304502210098d3349ba9b13560748949933d2704663a5ab52cdc804afa1ac4da3e5992e0a002201525d7ad8466ad260219f3873fb7781addbd363f91e8063bfa86c7ed4e385b84012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff69e16767806ea5f069b7d46674f7aa747fcc6e541189ce7fcf92edcfd7642ff4000000006b4830450221008a5ebfe904c87f21947a44d8418190be5893993a683fde0f96df8a9487923da002205be1bbd8b518ba2f303cae23bc20806e84ffbba6a03f032385b15edb8df107f4012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640fffffffff4fbabd508178f553e676d67ce325796b03aa249b41a23c681c1ad9dedb45ae7000000006a47304402207cea6824abe1ce35e18954b858d45243e2cb57d27d782adc5b6b07ebd21a02d7022007ba0469b298c4b1a7c4a148fa16bae93d28593b34e197c10ac0d1faf9cc1bfa012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff14867aa59932d895be607fb7398f5594e39d9fa2e1c7daaed7b1390dbfdddcab000000006b4830450221009fb6e1885a3658c593809f95ecd2049f8ef9e00379686ac236b17312c9613d4c0220709fc50c9a920a19254389944db366c354708c19885d2479d9968fda0848f9f7012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff75777c692daaa21d216a1a2a7059203becfcdcf6793aa1259cdd7aadec957ab6000000006a47304402202945019860abf9b80e71f340320d114846efa4d2780ce12513e3983fb4d3f15b022019be008fb7368e3f1f022924dc7af1138b94041f46084dd27768bc8cacd1529f012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffffca037b85742e93df4eef8e8ac3b8531321c8a8e21a4a941360866ea57a973665000000006a4730440220760283a7828edcc53671fc73e29c30cdc64d60d300292761d39730f0d09f94c202201e026293e3891a6fe46e40cd21778a41e21641a261a7fbf3bf75b034d9c788d9012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffffa68edd030b4307ad87bfeff96a6db5b3ddd1a0901c488a4fe4d2093531896d75000000006b48304502210091a41e16b2c27d7ef6077e8de9df692b6013e61d72798ff9f7eba7fc983cdb65022034de29a0fb22a339e8044349913915444ab420772ab0ab423e44cfe073cb4e70012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff8c952791181993a7512e48d098a06e6197c993b83241a4bf1330c0e95f2c304d000000006b483045022100fa14b9301feb056f6e6b10446a660525cc1ff3e191b0c45f9e12dcd4f142422c02203f4a94f2a9d3ec0b74fac2156dd9b1addb8fa5b9a1cfc9e34b0802e88b1cbfa3012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff32bc4d014542abf328fecff29b9f4c243c3dd295fe42524b20bf591a3ddc26a1000000006a47304402206f92c4da6651c8959f7aed61608d26b9e46f5c1d69f4fc6e592b1f552b6067f102201c8cc221eac731867d15d483cc83322dba2f14f31d3efb26be937a68ad772394012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffffbb3877248c26b23023d7dbb83a5f8293c65c5bff4ac47935a4a31248cefffd91000000006a47304402205bab19ad082a1918e18ccb6462edc263196fb88c8fdfd6bd07a0cf031a4637810220371a621c1bdc6b957db2447a92dcf87b0309653a2db480c9ed623f34a6e6d8a9012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff6415b7356c94609b9a7a7eb06e4c306896767abbc11399779f952fb9ae197059000000006b483045022100e2d038dbb9a873f5a58ec7901d6a7e79f1b404afea3d852056f4d0746cfb821102207fb274947b10d467cd71aa948e9a50f5f4430b661b27afc347efd9d6cc409d9c012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff1aeefdf80ec8a07d657ca64a2c0aa465f58e6284755c9a263c5a807be43b4b81000000006a47304402206e7ff765ba47a8785008f64f49c8e73232d582b2b2d0a49be0880c2557de8f8602206448423a6a37ad9740eb316513b31f73599ae14f65623709fb5443ae609f3e2e012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff3c091681df17b46f280bc9d8011c1bb31397637ce945b393f70380f8cd0a8b0d010000006a47304402206ca8717000f3086d364318f56d52e2369c40b88a1cb86455a8db262b4816698a02206711caf453bfda6b1b3542e27e68c3180f92f0548326d74e30b3ed18cd2c2353012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff91f32d98b581def165495aff6b69530e1f3de7f68fabfeb93730cf9793bbcd2a000000006a47304402200a8cd5e29ee7ff136772ea1789a39a027eaa1cd92f90f9d57fd8cf77202251f402203dd2bc282a838a5730e840a0d22b4f0edbe3cb2da00466c66bc2b5c66fc8b032012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff854d9226c28a1f5fe440e08f41000f3547f304ecf9cc010d0b5bc845ef1f039a000000006b483045022100fe6cce49975cc78af1c394bc02d995710833ba08cf7f8dd5f99add2cc7db26c40220793491309c215d8314a1c142bef7ec6b9a397249bec1c00a0a5ab47dfc1208b6012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff593bc907aa71f3b0f7aa8c48bb5f650595e65a5a733a9601a8374ed978eec9a7000000006a47304402206362ae3c4cf1a19ba0e43424b03af542077b49761172c1ad26d802f54b1b6ca602206bc7edb655bb0024c0e48c1f4c18c8864f8d1ce59ae55cd81dc0bd1234430691012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff3b9da8c5ab0c0cd6b40f602ea6ed8e36a48034b182b9d1a77ffebd15fe203b94000000006b483045022100f8610eae25899663cb5fa9a4575d937da57cdfd41958794bbb4c02f8bed75da40220262d40e019ec3a57b252f4150d509cce6f8a2dbd83184a9fc2ed56aba8018b15012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff0897c8a57e15e7f3893b195d65cf6c6001b29c8c9734213d7a3131f57b9eca2e000000006b483045022100c485cbd6408cf0759bcf23c4154249882934b522a93c6b49e62412305bf7646902201cc4b668af4bb22fe57c32c4d34e822bceb12f6bd6923afdabf4894752a56ec3012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffffffdc7000f7c45b62960623fa3a277e8a55348a4fe4936fef1224b6953434a249000000006b4830450221008a51a9c26f475d5c0838afe9d51524f95adfb21a9b0a02eae31cb01dc0a31fab022071c5492fbc7270731d4a4947a69398bf99dd28c65bb69d19910bf53a515274c8012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff10ec2af7e31ca28e27177215904d9a59abf80f0652b24e3f749f14fb7b2264ec000000006b483045022100fe4269f8f5ca53ebcff6fb782142a6228f0e50498a531b7a9c0d54768af9854102207cc740a9ea359569b49d69a94215ce3e23aeda5779cebc434ad3d608e1752990012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff5e3830c088dd6ea412d778b0a700ef27c183cf03e19f3d6f71bc5eaf53b2c22e000000006b4830450221009788a7e7f2407ba2f7c504091fbdf8f8498367781e8a357616d68e2a6770b4e70220518c92f5fb21e6bfd7d870a783b2a5572ce003f2dbb237ec59df419c9a148966012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff51630ccb0ad32b24cc7ae1b3602950ba518dca6aa65ef560e57f08c23eed8d80000000006a47304402201aa556153ffeb13aa674353bf88c04a7af15c7eb32e1a835464e4b613c31dc2802200395858c29a46e9108de1f90b401ee26c296388b4073143b63f849b2cce461af012102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ffffffff0200e1f5050000000017a914ab802c4d644be63fd1a72834ff63b650d6b5353987bb7e1e00000000001976a91464ae8510aac9546d5e7704e31ce177451386455588ac680e135d000000000000000000000000000000"
          },
          "type": "TakerPaymentReceived"
        },
        "timestamp": 1561529998938
      },
      {
        "event": {
          "type": "TakerPaymentWaitConfirmStarted"
        },
        "timestamp": 1561529998941
      },
      {
        "event": {
          "type": "TakerPaymentValidatedAndConfirmed"
        },
        "timestamp": 1561530000859
      },
      {
        "event": {
          "data": {
            "block_height": 0,
            "coin": "PIZZA",
            "fee_details": {
              "amount": "0.00001"
            },
            "from": ["bUN5nesdt1xsAjCtAaYUnNbQhGqUWwQT1Q"],
            "internal_id": "235f8e7ab3c9515a17fe8ee721ef971bbee273eb90baf70788edda7b73138c86",
            "my_balance_change": "0.99999",
            "received_by_me": "0.99999",
            "spent_by_me": "0",
            "timestamp": 0,
            "to": ["R9o9xTocqr6CeEDGDH6mEYpwLoMz6jNjMW"],
            "total_amount": "1",
            "tx_hash": "235f8e7ab3c9515a17fe8ee721ef971bbee273eb90baf70788edda7b73138c86",
            "tx_hex": "0400008085202f8901a5464048246f791dca2f8cef2774125992cba7c0b820f32e7980be1de3380e7e00000000d8483045022100beca668a946fcad98da64cc56fa04edd58b4c239aa1362b4453857cc2e0042c90220606afb6272ef0773185ade247775103e715e85797816fbc204ec5128ac10a4b90120ea774bc94dce44c138920c6e9255e31d5645e60c0b64e9a059ab025f1dd2fdc6004c6b6304692c135db1752102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ac6782012088a914eb78e2f0cf001ed7dc69276afd37b25c4d6bb491882102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ac68ffffffff0118ddf505000000001976a91405aab5342166f8594baf17a7d9bef5d56744332788ac8000135d000000000000000000000000000000"
          },
          "type": "TakerPaymentSpent"
        },
        "timestamp": 1561530003429
      },
      {
        "event": {
          "type": "Finished"
        },
        "timestamp": 1561530003433
      }
    ],
    "my_info": {
      "my_amount": "1",
      "my_coin": "BEER",
      "other_amount": "1",
      "other_coin": "PIZZA",
      "started_at": 1561529842
    },
    "maker_coin": "BEER",
    "maker_amount": "1",
    "taker_coin": "PIZZA",
    "taker_amount": "1",
    "gui": "AtomicDEX 1.0",
    "mm_version": "unknown", 
    "recoverable": false,
    "success_events": [
      "Started",
      "Negotiated",
      "TakerFeeValidated",
      "MakerPaymentSent",
      "TakerPaymentReceived",
      "TakerPaymentWaitConfirmStarted",
      "TakerPaymentValidatedAndConfirmed",
      "TakerPaymentSpent",
      "Finished"
    ],
    "type": "Maker",
    "uuid": "6bf6e313-e610-4a9a-ba8c-57fc34a124aa"
  }
}
```

#### Response (error)

```json
{
  "error": "swap data is not found"
}
```

</collapse-text>

</div>

## my\_tx\_history

**(from_id limit=10)**

The `my_tx_history` method returns the blockchain transactions involving the MM2 node's coin address.

#### Arguments

| Structure | Type   | Description                                                                                                                                                                                 |
| --------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| coin      | string | the name of the coin for the history request                                                                                                                                                |
| limit     | number | limits the number of returned transactions                                                                                                                                                  |
| from_id   | string | MM2 will skip records until it reaches this ID, skipping the `from_id` as well; track the `internal_id` of the last displayed transaction to find the value of this field for the next page |

#### Response

| Structure                                     | Type             | Description                                                                                                                                    |
| --------------------------------------------- | ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| transactions                                  | array of objects | transactions data                                                                                                                              |
| from_id                                       | string           | the from_id specified in the request; this value is null if from_id was not set                                                                |
| skipped                                       | number           | the number of skipped records (i.e. the position of `from_id` in the list + 1); this value is 0 if `from_id` was not set                       |
| limit                                         | number           | the limit that was set in the request; note that the actual number of transactions can differ from the specified limit (e.g. on the last page) |
| total                                         | number           | the total number of transactions available                                                                                                     |
| current_block                                 | number           | the number of the latest block of coin blockchain                                                                                              |
| sync_status                                   | object           | provides the information that helps to track the progress of transaction history preloading at background                                      |
| sync_status.state                             | string           | current state of sync; possible values: `NotEnabled`, `NotStarted`, `InProgress`, `Error`, `Finished`                                          |
| sync_status.additional_info                   | object           | additional info that helps to track the progress; present for `InProgress` and `Error` states only                                             |
| sync_status.additional_info.blocks_left       | number           | present for ETH/ERC20 coins only; displays the number of blocks left to be processed for `InProgress` state                                    |
| sync_status.additional_info.transactions_left | number           | present for UTXO coins only; displays the number of transactions left to be processed for `InProgress` state                                   |
| sync_status.additional_info.code              | number           | displays the error code for `Error` state                                                                                                      |
| sync_status.additional_info.message           | number           | displays the error message for `Error` state                                                                                                   |

#### :pushpin: Examples

#### Command

```bash
curl --url "http://127.0.0.1:7783" --data "{\"userpass\":\"$userpass\",\"method\":\"my_tx_history\",\"coin\":\"RICK\",\"limit\":1,\"from_id\":\"1d5c1b67f8ebd3fc480e25a1d60791bece278f5d1245c5f9474c91a142fee8e1\"}"
```

<div style="margin-top: 0.5rem;">

<collapse-text hidden title="Response">

#### Response (success)

```json
{
  "result":{
    "current_block":172418,
    "from_id":null,
    "limit":1,
    "skipped":0,
    "sync_status":{
      "additional_info":{
        "transactions_left":126
      },
      "state":"InProgress"
    },
    "total":5915,
    "transactions":[
      {
        "block_height":172409,
        "coin":"ETOMIC",
        "confirmations":10,
        "fee_details":{
          "amount":"0.00001"
        },
        "from":[
          "R9o9xTocqr6CeEDGDH6mEYpwLoMz6jNjMW"
        ],
        "internal_id":"903e5d71b8717205314a71055fe8bbb868e7b76d001fbe813a34bd71ff131e93",
        "my_balance_change":"-0.10001",
        "received_by_me":"0.8998513",
        "spent_by_me":"0.9998613",
        "timestamp":1566539526,
        "to":[
          "R9o9xTocqr6CeEDGDH6mEYpwLoMz6jNjMW",
          "bJrMTiiRiLHJHc6RKQgesKTg1o9VVuKwT5"
        ],
        "total_amount":"0.9998613",
        "tx_hash":"903e5d71b8717205314a71055fe8bbb868e7b76d001fbe813a34bd71ff131e93",
        "tx_hex":"0400008085202f8901a242dc691de64c732e823ed0a4d8cfa6a230f8e31bc9bd21499009f1a90b855a010000006b483045022100d83113119004ac0504f812a853a831039dfc4b0bc1cb863d2c7a94c0670f07e902206af87b846b18c0d5e38bd874d43918e0400e4b6b838ab0793f5976843daa20cd012102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffff02809698000000000017a9144327a5516b28f66249576c18d15debf6dfbd1124876a105d05000000001976a91405aab5342166f8594baf17a7d9bef5d56744332788ac047f5f5d000000000000000000000000000000"
      }
    ]
  }
}
```

#### Response (error)

```json
{
  "error": "lp_coins:1011] from_id 1d5c1b67f8ebd3fc480e25a1d60791bece278f5d1245c5f9474c91a142fee8e2 is not found"
}
```

#### Response (History too large in electrum mode)

```json
{
  "result": {
    "current_block": 144753,
    "from_id": null,
    "limit": 0,
    "skipped": 0,
    "sync_status": {
      "additional_info": {
        "code": -1,
        "message": "Got `history too large` error from Electrum server. History is not available"
      },
      "state": "Error"
    },
    "total": 0,
    "transactions": []
  }
}
```

#### Response (Sync in progress for UTXO coins)

```json
{
  "result": {
    "current_block": 148300,
    "from_id": null,
    "limit": 0,
    "skipped": 0,
    "sync_status": {
      "additional_info": {
        "transactions_left": 1656
      },
      "state": "InProgress"
    },
    "total": 3956,
    "transactions": []
  }
}
```

#### Response (Sync in progress for ETH/ERC20 coins)

```json
{
  "result": {
    "current_block": 8039935,
    "from_id": null,
    "limit": 0,
    "skipped": 0,
    "sync_status": {
      "additional_info": {
        "blocks_left": 2158991
      },
      "state": "InProgress"
    },
    "total": 0,
    "transactions": []
  }
}
```

</collapse-text>

</div>

## order\_status

**order_status uuid**

The `order_status` method returns the data of the active order with the selected `uuid` created by the MM2 node.

#### Arguments

| Structure | Type   | Description              |
| --------- | ------ | ------------------------ |
| uuid      | string | uuid of order to display |

#### Response

| Structure | Type   | Description                            |
| --------- | ------ | -------------------------------------- |
| type      | string | type of the order ("Maker" or "Taker") |
| order     | object | order data                             |

#### :pushpin: Examples

#### Command

```bash
curl --url "http://127.0.0.1:7783" --data "{\"userpass\":\"$userpass\",\"method\":\"order_status\",\"uuid\":\"c3b3105c-e914-4ed7-9f1c-604783b054a1\"}"
```

<div style="margin-top: 0.5rem;">

<collapse-text hidden title="Response">

#### Response (Maker order)

```json
{
  "order":{
    "available_amount":"1",
    "base":"BEER",
    "cancellable":true,
    "created_at":1568808684710,
    "matches":{
      "60aaacca-ed31-4633-9326-c9757ea4cf78":{
        "connect":{
          "dest_pub_key":"c213230771ebff769c58ade63e8debac1b75062ead66796c8d793594005f3920",
          "maker_order_uuid":"fedd5261-a57e-4cbf-80ac-b3507045e140",
          "method":"connect",
          "sender_pubkey":"5a2f1c468b7083c4f7649bf68a50612ffe7c38b1d62e1ece3829ca88e7e7fd12",
          "taker_order_uuid":"60aaacca-ed31-4633-9326-c9757ea4cf78"
        },
        "connected":{
          "dest_pub_key":"5a2f1c468b7083c4f7649bf68a50612ffe7c38b1d62e1ece3829ca88e7e7fd12",
          "maker_order_uuid":"fedd5261-a57e-4cbf-80ac-b3507045e140",
          "method":"connected",
          "sender_pubkey":"c213230771ebff769c58ade63e8debac1b75062ead66796c8d793594005f3920",
          "taker_order_uuid":"60aaacca-ed31-4633-9326-c9757ea4cf78"
        },
        "last_updated":1560529572571,
        "request":{
          "action":"Buy",
          "base":"BEER",
          "base_amount":"1",
          "dest_pub_key":"0000000000000000000000000000000000000000000000000000000000000000",
          "method":"request",
          "rel":"PIZZA",
          "rel_amount":"1",
          "sender_pubkey":"5a2f1c468b7083c4f7649bf68a50612ffe7c38b1d62e1ece3829ca88e7e7fd12",
          "uuid":"60aaacca-ed31-4633-9326-c9757ea4cf78"
        },
        "reserved":{
          "base":"BEER",
          "base_amount":"1",
          "dest_pub_key":"5a2f1c468b7083c4f7649bf68a50612ffe7c38b1d62e1ece3829ca88e7e7fd12",
          "maker_order_uuid":"fedd5261-a57e-4cbf-80ac-b3507045e140",
          "method":"reserved",
          "rel":"PIZZA",
          "rel_amount":"1",
          "sender_pubkey":"c213230771ebff769c58ade63e8debac1b75062ead66796c8d793594005f3920",
          "taker_order_uuid":"60aaacca-ed31-4633-9326-c9757ea4cf78"
        }
      }
    },
    "max_base_vol":"1",
    "max_base_vol_rat":[[1,[1]],[1,[1]]],
    "min_base_vol":"0",
    "min_base_vol_rat":[[0,[]],[1,[1]]],
    "price":"1",
    "price_rat":[[1,[1]],[1,[1]]],
    "rel":"ETOMIC",
    "started_swaps":[
      "60aaacca-ed31-4633-9326-c9757ea4cf78"
    ],
    "uuid":"ea77dcc3-a711-4c3d-ac36-d45fc5e1ee0c"
  },
  "type":"Maker"
}
```

#### Response (Taker order)

```json
{
  "order":{
    "cancellable":true,
    "created_at":1568811351456,
    "matches":{
      "15922925-cc46-4219-8cbd-613802e17797":{
        "connect":{
          "dest_pub_key":"5a2f1c468b7083c4f7649bf68a50612ffe7c38b1d62e1ece3829ca88e7e7fd12",
          "maker_order_uuid":"15922925-cc46-4219-8cbd-613802e17797",
          "method":"connect",
          "sender_pubkey":"c213230771ebff769c58ade63e8debac1b75062ead66796c8d793594005f3920",
          "taker_order_uuid":"45252de5-ea9f-44ae-8b48-85092a0c99ed"
        },
        "connected":{
          "dest_pub_key":"c213230771ebff769c58ade63e8debac1b75062ead66796c8d793594005f3920",
          "maker_order_uuid":"15922925-cc46-4219-8cbd-613802e17797",
          "method":"connected",
          "sender_pubkey":"5a2f1c468b7083c4f7649bf68a50612ffe7c38b1d62e1ece3829ca88e7e7fd12",
          "taker_order_uuid":"45252de5-ea9f-44ae-8b48-85092a0c99ed"
        },
        "last_updated":1560529049477,
        "reserved":{
          "base":"BEER",
          "base_amount":"1",
          "dest_pub_key":"c213230771ebff769c58ade63e8debac1b75062ead66796c8d793594005f3920",
          "maker_order_uuid":"15922925-cc46-4219-8cbd-613802e17797",
          "method":"reserved",
          "rel":"ETOMIC",
          "rel_amount":"1",
          "sender_pubkey":"5a2f1c468b7083c4f7649bf68a50612ffe7c38b1d62e1ece3829ca88e7e7fd12",
          "taker_order_uuid":"45252de5-ea9f-44ae-8b48-85092a0c99ed"
        }
      }
    },
    "request":{
      "action":"Buy",
      "base":"BEER",
      "base_amount":"1",
      "base_amount_rat":[[1,[1]],[1,[1]]],
      "dest_pub_key":"0000000000000000000000000000000000000000000000000000000000000000",
      "method":"request",
      "rel":"ETOMIC",
      "rel_amount":"1",
      "rel_amount_rat":[[1,[1]],[1,[1]]],
      "sender_pubkey":"031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3",
      "uuid":"ea199ac4-b216-4a04-9f08-ac73aa06ae37"
    }
  },
  "type":"Taker"
}
```

#### Response (No order found)

```json
{ "error": "Order with uuid c3b3105c-e914-4ed7-9f1c-604783b054a1 is not found" }
```

</collapse-text>

</div>

## orderbook

**orderbook base rel (duration=number)**

The `orderbook` method requests from the network the currently available orders for the specified trading pair.

#### Arguments

| Structure | Type   | Description                                                                         |
| --------- | ------ | ----------------------------------------------------------------------------------- |
| base      | string | base currency of a pair                                                             |
| rel       | string | "related" currency, also can be called "quote currency" according to exchange terms |
| duration  | number | `deprecated`                                                                        |

#### Response

| Structure      | Type             | Description                                                                   |
| -------------- | ---------------- | ----------------------------------------------------------------------------- |
| bids           | array            | an array of objects containing outstanding bids                               |
| numbids        | number           | the number of outstanding bids                                                |
| biddepth       | number           | `deprecated`                                                                  |
| asks           | array            | an array of objects containing outstanding asks                               |
| coin           | string           | the name of the `base` coin; the user desires this                            |
| address        | string           | the address offering the trade                                                |
| price          | string (decimal) | the price in `rel` the user is willing to pay per one unit of the `base` coin |
| price_rat      | rational         | the price in rational representation                                          |
| maxvolume      | string (decimal) | the maximum amount of `base` coin the offer provider is willing to sell      |
| max_volume_rat | rational         | the max volume in rational representation                                     |
| pubkey         | string           | the pubkey of the offer provider                                              |
| age            | number           | the age of the offer                                                          |
| zcredits       | number           | the zeroconf deposit amount                                                   |
| numasks        | number           | the total number of asks                                                      |
| askdepth       | number           | the depth of the ask requests                                                 |
| base           | string           | the name of the coin the user desires to receive                              |
| rel            | string           | the name of the coin the user will trade                                      |
| timestamp      | number           | the timestamp of the orderbook request                                        |
| netid          | number           | the id of the network on which the request is made (default is `0`)           |

#### :pushpin: Examples

#### Command

```bash
curl --url "http://127.0.0.1:7783" --data "{\"userpass\":\"$userpass\",\"method\":\"orderbook\",\"base\":\"HELLO\",\"rel\":\"WORLD\"}"
```

<div style="margin-top: 0.5rem;">

<collapse-text hidden title="Response">

#### Response

```json
{
  "askdepth":0,
  "asks":[
    {
      "coin": "HELLO",
      "address": "RJTYiYeJ8eVvJ53n2YbrVmxWNNMVZjDGLh",
      "price": "1.33333333",
      "price_rat":[[1,[4]],[1,[3]]],
      "maxvolume":997.0,
      "max_volume_rat":[[1,[997]],[1,[1]]],
      "pubkey":"631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640",
      "age":1,
      "zcredits":0
    }
  ],
  "base":"HELLO",
  "biddepth":0,
  "bids":[],
  "netid":9999,
  "numasks":1,
  "numbids":0,
  "rel":"WORLD",
  "timestamp":1568807329
}
```

</collapse-text>

</div>

## recover\_funds\_of\_swap

**recover_funds_of_swap uuid**

In certain cases, a swap can finish with an error wherein the user's funds are stuck on the swap-payment address. (This address is the P2SH address when executing on a utxo-based blockchain, or an etomic-swap smart contract when executing on an ETH/ERC20 blockchain.)

This error can occur when one side of the trade does not follow the protocol (for any reason). The error persists as attempts to refund the payment fail due to network connection issues between the MM2 node and the coin's RPC server.

In this scenario, the `recover_funds_of_swap` method instructs the MM2 software to attempt to reclaim the user funds from the swap-payment address, if possible.

#### Arguments

| Structure | Type   | Description                                                                         |
| --------- | ------ | ----------------------------------------------------------------------------------- |
| params.uuid | string | uuid of the swap to recover the funds                                             |

#### Response

| Structure | Type             | Description |
| --------- | ---------------- | ----------- |
| result.action | string       | the action executed to unlock the funds. Can be either `SpentOtherPayment` or `RefundedMyPayment` |
| result.coin   | string       | the balance of this coin will be unstuck by the recovering transaction |
| result.tx_hash| string       | the hash of the recovering transaction |
| result.tx_hex | string       | raw bytes of the recovering transaction in hexadecimal representation |

#### :pushpin: Examples

#### Command

```bash
curl --url "http://127.0.0.1:7783" --data "{\"userpass\":\"$userpass\",\"method\":\"recover_funds_of_swap\",\"params\":{\"uuid\":\"6343b2b1-c896-47d4-b0f2-a11798f654ed\"}}"
```

<div style="margin-top: 0.5rem;">

<collapse-text hidden title="Response">

#### Response (success - SpentOtherPayment)

```json
{
  "result":{
    "action":"SpentOtherPayment",
    "coin":"HELLO",
    "tx_hash":"696571d032976876df94d4b9994ee98faa870b44fbbb4941847e25fb7c49b85d",
    "tx_hex":"0400008085202f890113591b1feb52878f8aea53b658cf9948ba89b0cb27ad0cf30b59b5d3ef6d8ef700000000d8483045022100eda93472c1f6aa18aacb085e456bc47b75ce88527ed01c279ee1a955e85691b702201adf552cfc85cecf588536d5b8257d4969044dde86897f2780e8c122e3a705e40120576fa34d308f39b7a704616656cc124232143565ca7cf1c8c60d95859af8f22d004c6b63042555555db1752102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ac6782012088a9146e602d4affeb86e4ee208802901b8fd43be2e2a4882102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ac68ffffffff0198929800000000001976a91405aab5342166f8594baf17a7d9bef5d56744332788ac0238555d000000000000000000000000000000"
  }
}
```

#### Response (success - RefundedMyPayment)

```json
{
  "result":{
    "action":"RefundedMyPayment",
    "coin":"HELLO",
    "tx_hash":"696571d032976876df94d4b9994ee98faa870b44fbbb4941847e25fb7c49b85d",
    "tx_hex":"0400008085202f890113591b1feb52878f8aea53b658cf9948ba89b0cb27ad0cf30b59b5d3ef6d8ef700000000d8483045022100eda93472c1f6aa18aacb085e456bc47b75ce88527ed01c279ee1a955e85691b702201adf552cfc85cecf588536d5b8257d4969044dde86897f2780e8c122e3a705e40120576fa34d308f39b7a704616656cc124232143565ca7cf1c8c60d95859af8f22d004c6b63042555555db1752102631dcf1d4b1b693aa8c2751afc68e4794b1e5996566cfc701a663f8b7bbbe640ac6782012088a9146e602d4affeb86e4ee208802901b8fd43be2e2a4882102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ac68ffffffff0198929800000000001976a91405aab5342166f8594baf17a7d9bef5d56744332788ac0238555d000000000000000000000000000000"
  }
}
```

#### Response (error - maker payment was already spent)

```json
{"error":"lp_swap:702] lp_swap:412] taker_swap:890] Maker payment is spent, swap is not recoverable"}
```

#### Response (error - swap is not finished yet)

```json
{"error":"lp_swap:702] lp_swap:412] taker_swap:886] Swap must be finished before recover funds attempt"}
```

</collapse-text>

</div>

## sell

**sell base rel price volume**

The `sell` method issues a sell request and attempts to match an order from the orderbook based on the provided arguments.

::: tip

Buy and sell methods always create the `taker` order first. Therefore, you must pay an additional 1/777 fee of the trade amount during the swap when taking liquidity from market. If your order is not matched in 30 seconds, the order is automatically converted to a `maker` request and stays on the orderbook until the request is matched or cancelled. To always act as a maker, please use the [setprice](../../../basic-docs/atomicdex/atomicdex-api.html#setprice) method.

:::

#### Arguments

| Structure | Type                       | Description                                                                       |
| --------- | -------------------------- | --------------------------------------------------------------------------------- |
| base      | string                     | the name of the coin the user desires to sell                                     |
| rel       | string                     | the name of the coin the user desires to receive                                  |
| price     | numeric string or rational | the price in `rel` the user is willing to receive per one unit of the `base` coin |
| volume    | numeric string or rational | the amount of coins the user is willing to sell of the `base` coin                |

#### Response

| Structure              | Type     | Description                                                                                                                                                                                                     |
| ---------------------- | -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| result                 | object   | the resulting order object                                                                                                                                                                                      |
| result.action          | string   | the action of the request (`Sell`)                                                                                                                                                                              |
| result.base            | string   | the base currency of the request                                                                                                                                                                                |
| result.base_amount     | string   | the resulting amount of base currency that will be sold if the order matches (in decimal representation)                                                                                                        |
| result.base_amount_rat | rational | the resulting amount of base currency that will be sold if the order matches (in rational representation)                                                                                                       |
| result.rel             | string   | the rel currency of the request                                                                                                                                                                                 |
| result.rel_amount      | string   | the minimum amount of `rel` coin that will be received to sell the `base_amount` of `base` (according to `price`, in decimal representation)                                                                    |
| result.rel_amount_rat  | rational | the minimum amount of `rel` coin that will be received to sell the `base_amount` of `base` (according to `price`, in rational representation)                                                                   |
| result.method          | string   | this field is used for internal P2P interactions; the value is always equal to "request"                                                                                                                        |
| result.dest_pub_key    | string   | reserved for future use. The `dest_pub_key` will allow the user to choose the P2P node that is be eligible to match with the request. This value defaults to "zero pubkey", which means that `anyone` can match |
| result.sender_pubkey   | string   | the public key of our node                                                                                                                                                                                      |
| result.uuid            | string   | the request uuid                                                                                                                                                                                                |

#### :pushpin: Examples

#### Command (decimal representation)

```bash
curl --url "http://127.0.0.1:7783" --data "{\"userpass\":\"$userpass\",\"method\":\"sell\",\"base\":\"BASE\",\"rel\":\"REL\",\"volume\":"\"1\"",\"price\":"\"1\""}"
```

#### Command (rational representation)

```bash
curl --url "http://127.0.0.1:7783" --data "{\"userpass\":\"$userpass\",\"method\":\"sell\",\"base\":\"BASE\",\"rel\":\"REL\",\"volume\":[[1,[1]],[1,[1]]],\"price\":[[1,[1]],[1,[1]]]}"
```

<div style="margin-top: 0.5rem;">

<collapse-text hidden title="Response">

#### Response (success)

```json
{
  "result": {
    "action": "Sell",
    "base": "BASE",
    "base_amount": "1",
    "base_amount_rat": [[1,[1]],[1,[1]]],
    "dest_pub_key": "0000000000000000000000000000000000000000000000000000000000000000",
    "method": "request",
    "rel": "REL",
    "rel_amount": "1",
    "rel_amount_rat": [[1,[1]],[1,[1]]],
    "sender_pubkey": "c213230771ebff769c58ade63e8debac1b75062ead66796c8d793594005f3920",
    "uuid": "d14452bb-e82d-44a0-86b0-10d4cdcb8b24"
  }
}
```

#### Response (error)

```json
{
  "error": "rpc:278] utxo:884] BASE balance 12.88892991 is too low, required 21.15"
}
```

</collapse-text>

</div>

## send\_raw\_transaction

**send_raw_transaction coin tx_hex**

The `send_raw_transaction` method broadcasts the transaction to the network of selected coin.

#### Arguments

| Structure | Type   | Description                                                                                       |
| --------- | ------ | ------------------------------------------------------------------------------------------------- |
| coin      | string | the name of the coin network on which to broadcast the transaction                                |
| tx_hex    | string | the transaction bytes in hexadecimal format; this is typically generated by the `withdraw` method |

#### Response

| Structure | Type   | Description                           |
| --------- | ------ | ------------------------------------- |
| tx_hash   | string | the hash of the broadcast transaction |

#### :pushpin: Examples

#### Command

```bash
curl --url "http://127.0.0.1:7783" --data "{\"method\":\"send_raw_transaction\",\"coin\":\"KMD\",\"tx_hex\":\"0400008085202f8902d6a5b976db5e5c9e8f9ead50713b25f22cd061edc8ff0ff1049fd2cd775ba087000000006b483045022100bf2073c1ecfef3fc78f272045f46a722591401f61c2d2fac87fc474a17df7c3102200ca1bd0664ba75f3383e5cbbe96127ad534a86238dbea256e000b0fe2067ab8c012102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffffd04d4e07ac5dacd08fb76e08d2a435fc4fe2b16eb0158695c820b44f42f044cb010000006a47304402200a0c21e8c0ae4a740f3663fe08aeff02cea6495157d531045b58d2dd79fb802702202f80dddd264db33f55e49799363997a175d39a91242a95f268c40f7ced97030b012102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffff0200e1f505000000001976a91405aab5342166f8594baf17a7d9bef5d56744332788acc3b3ca27000000001976a91405aab5342166f8594baf17a7d9bef5d56744332788ac00000000000000000000000000000000000000\",\"userpass\":\"$userpass\"}"
```

<div style="margin-top: 0.5rem;">

<collapse-text hidden title="Response">

#### Response (success)

```json
{
  "tx_hash": "0b024ea6997e16387c0931de9f203d534c6b2b8500e4bda2df51a36b52a3ef33"
}
```

</collapse-text>

</div>

## setprice

**setprice base rel price (volume max cancel_previous=true)**

The `setprice` method places an order on the orderbook, and it relies on this node acting as a `maker`, also called a `Bob` node.

The `setprice` order is always considered a `sell`, for internal implementation convenience.

#### Arguments

| Structure       | Type                       | Description                                                                                                              |
| --------------- | -------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| base            | string                     | the name of the coin the user desires to sell                                                                            |
| rel             | string                     | the name of the coin the user desires to receive                                                                         |
| price           | numeric string or rational | the price in `rel` the user is willing to receive per one unit of the `base` coin                                        |
| volume          | numeric string or rational | the maximum amount of `base` coin available for the order, ignored if max is `true`                                      |
| max             | bool                       | MM2 will use the entire coin balance for the order, taking `0.001` coins into reserve to account for fees                |
| cancel_previous | bool                       | MM2 will cancel all existing orders for the selected pair by default; set this value to `false` to prevent this behavior |

#### Response

| Structure               | Type             | Description                                                                                                                                          |
| ----------------------- | ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| result                  | object           | the resulting order object                                                                                                                           |
| result.base             | string           | the base coin of the order                                                                                                                           |
| result.rel              | string           | the rel coin of the order                                                                                                                            |
| result.price            | string (numeric) | the expected amount of `rel` coin to be received per 1 unit of `base` coin; decimal representation                                                   |
| result.price_rat        | rational         | the expected amount of `rel` coin to be received per 1 unit of `base` coin; rational representation                                                  |
| result.max_base_vol     | string (numeric) | the maximum volume of base coin available to trade; decimal representation                                                                           |
| result.max_base_vol_rat | rational         | the maximum volume of base coin available to trade; rational representation                                                                          |
| result.min_base_vol     | string (numeric) | MM2 won't match with other orders that attempt to trade less than `min_base_vol`; decimal representation                                             |
| result.min_base_vol_rat | rational         | MM2 won't match with other orders that attempt to trade less than `min_base_vol`; rational representation                                            |
| result.created_at       | number           | unix timestamp in milliseconds, indicating the order creation time                                                                                   |
| result.matches          | object           | contains the map of ongoing matches with other orders, empty as the order was recently created                                                       |
| result.started_swaps    | array of strings | uuids of swaps that were initiated by the order                                                                                                      |
| result.uuid             | string           | uuid of the created order                                                                                                                            |

#### :pushpin: Examples

#### Command (with volume)

```bash
curl --url "http://127.0.0.1:7783" --data "{\"userpass\":\"$userpass\",\"method\":\"setprice\",\"base\":\"BASE\",\"rel\":\"REL\",\"price\":\"0.9\",\"volume\":\"1\"}"
```

#### Command (max = true)

```bash
curl --url "http://127.0.0.1:7783" --data "{\"userpass\":\"$userpass\",\"method\":\"setprice\",\"base\":\"BASE\",\"rel\":\"REL\",\"price\":\"0.9\",\"max\":true}"
```

<div style="margin-top: 0.5rem;">

<collapse-text hidden title="Response">

#### Response (success)

```json
{
  "result": {
    "base": "BASE",
    "rel": "REL",
    "max_base_vol": "1",
    "max_base_vol_rat": [[1,[1]],[1,[1]]],
    "min_base_vol": "0",
    "min_base_vol": [[0,[]],[1,[1]]],
    "created_at": 1559052299258,
    "matches": {},
    "price": "1",
    "price_rat": [[1,[1]],[1,[1]]],
    "started_swaps": [],
    "uuid": "6a242691-6c05-474a-85c1-5b3f42278f41"
  }
}
```

#### Response (error)

```json
{ "error": "Rel coin REL is not found" }
```

</collapse-text>

</div>

## set\_required\_confirmations

**set_required_confirmations coin confirmations**

The `set_required_confirmations` method sets the number of confirmations for which MM2 will wait for the selected coin.

::: tip Note

Please note that this setting is _**not**_ persistent. The value must be reset in the coins file on restart.

:::

#### Arguments

| Structure       | Type             | Description                                                                                                              |
| --------------- | ---------------- | ------------------------------------------------------------- |
| coin            | string           | the ticker of the selected coin                               |
| confirmations   | number           | the number of confirmations to require                        |

#### Response

| Structure            | Type             | Description                                                                                                                                          |
| -------------------- | ---------------- | -------------------------------------------------------- |
| result.coin          | string           | the coin selected in the request                         |
| result.confirmations | number           | the number of confirmations in the request               |

#### :pushpin: Examples

#### Command

```bash
curl --url "http://127.0.0.1:7783" --data "{\"userpass\":\"$userpass\",\"method\":\"set_required_confirmations\",\"coin\":\"RICK\",\"confirmations\":3}"
```

<div style="margin-top: 0.5rem;">

<collapse-text hidden title="Response">

#### Response (success)

```json
{
  "result": {
    "coin": "ETOMIC",
    "confirmations": 3
  }
}
```

</collapse-text>

</div>

## stop

**stop()**

The `stop` method stops the MM2 software.

#### Arguments

| Structure | Type | Description |
| --------- | ---- | ----------- |
| (none)    |      |             |

#### Response

| Structure | Type | Description |
| --------- | ---- | ----------- |
| (none)    |      |             |

## version

**version()**

The `version` method returns the MM2 version.

#### Arguments

| Structure | Type | Description |
| --------- | ---- | ----------- |
| (none)    |      |             |

#### Response

| Structure | Type   | Description     |
| --------- | ------ | --------------- |
| result    | string | the MM2 version |

#### :pushpin: Examples

#### Command

```bash
curl --url "http://127.0.0.1:7783" --data "{\"method\":\"version\",\"userpass\":\"$userpass\"}"
```

<div style="margin-top: 0.5rem;">

<collapse-text hidden title="Response">

#### Response

```json
{
  "result": "2.0.996_mm2_3bb412578_Linux"
}
```

</collapse-text>

</div>

## withdraw

**withdraw coin to (amount max)**

The `withdraw` method generates, signs, and returns a transaction that transfers the `amount` of `coin` to the address indicated in the `to` argument.

This method generates a raw transaction which should then be broadcast using [send_raw_transaction](../../../basic-docs/atomicdex/atomicdex-api.html#send-raw-transaction).

#### Arguments

| Structure     | Type             | Description                                                      |
| ---------     | ---------------- | ---------------------------------------------------------------- |
| coin          | string           | the name of the coin the user desires to withdraw                |
| to            | string           | coins will be withdrawn to this address                          |
| amount        | string (numeric) | the amount the user desires to withdraw, ignored when `max=true` |
| max           | bool             | withdraw the maximum available amount                            |
| fee.type      | string           | type of transaction fee; possible values: `UtxoFixed`, `UtxoPerKbyte`, `EthGas` |
| fee.amount    | string (numeric) | fee amount in coin units, used only when type is `UtxoFixed` (fixed amount not depending on tx size) or `UtxoPerKbyte` (amount per Kbyte) |
| fee.gas_price | string (numeric) | used only when fee type is EthGas; sets the gas price in `gwei` units |
| fee.gas       | number (integer) | used only when fee type is EthGas; sets the gas limit for transaction |

#### Response

| Structure         | Type             | Description                                                                                                                                                                   |
| ----------------- | ---------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| from              | array of strings | coins will be withdrawn from this address; the array contains a single element, but transactions may be sent from several addresses (UTXO coins)                              |
| to                | array of strings | coins will be withdrawn to this address; this may contain the `my_address` address, where change from UTXO coins is sent                                                      |
| my_balance_change | string (numeric) | the expected balance of change in `my_address` after the transaction broadcasts                                                                                                              |
| received_by_me    | string (numeric) | the amount of coins received by `my_address` after the transaction broadcasts; the value may be above zero when the transaction requires that MM2 send change to `my_address` |
| spent_by_me       | string (numeric) | the amount of coins spent by `my_address`; this value differ from the request amount, as the transaction fee is added here                                                    |
| total_amount      | string (numeric) | the total amount of coins transferred                                                                                                                                         |
| fee_details       | object           | the fee details of the generated transaction; this value differs for utxo and ETH/ERC20 coins, check the examples for more details                                            |
| tx_hash           | string           | the hash of the generated transaction                                                                                                                                         |
| tx_hex            | string           | transaction bytes in hexadecimal format; use this value as input for the `send_raw_transaction` method                                                                        |

#### :pushpin: Examples

#### Command (BTC, KMD, and other BTC-based forks)

```bash
curl --url "http://127.0.0.1:7783" --data "{\"method\":\"withdraw\",\"coin\":\"KMD\",\"to\":\"RJTYiYeJ8eVvJ53n2YbrVmxWNNMVZjDGLh\",\"amount\":\"10\",\"userpass\":\"$userpass\"}"
```

<div style="margin-top: 0.5rem;">

<collapse-text hidden title="Response">

#### Response (success)

```json
{
  "block_height": 0,
  "coin": "ETOMIC",
  "fee_details": {
    "amount": "0.00001"
  },
  "from": ["R9o9xTocqr6CeEDGDH6mEYpwLoMz6jNjMW"],
  "my_balance_change": "-10.00001",
  "received_by_me": "0.34417325",
  "spent_by_me": "10.34418325",
  "to": ["RJTYiYeJ8eVvJ53n2YbrVmxWNNMVZjDGLh"],
  "total_amount": "10.34418325",
  "tx_hash": "3a1c382c50a7d12e4675d12ed7e723ce9f0167693dd75fd772bae8524810e605",
  "tx_hex": "0400008085202f890207a8e96978acfb8f0d002c3e4390142810dc6568b48f8cd6d8c71866ad8743c5010000006a47304402201960a7089f2d93480fff68ce0b7ca7bb7a32a52915753ac7ae780abd6162cb1d02202c9b11d442e5f72a532f44ceb10122898d486b1474a10eb981c60c5538b9c82d012102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffff97f56bf3b0f815bb737b7867e71ddb8198bba3574bb75737ba9c389a4d08edc6000000006a473044022055199d80bd7e2d1b932e54f097c6a15fc4b148d21299dc50067c1da18045f0ed02201d26d85333df65e6daab40a07a0e8a671af9d9b9d92fdf7d7ef97bd868ca545a012102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffff0200ca9a3b000000001976a91464ae8510aac9546d5e7704e31ce177451386455588acad2a0d02000000001976a91405aab5342166f8594baf17a7d9bef5d56744332788ac00000000000000000000000000000000000000"
}
```

</collapse-text>

</div>

#### Command (BTC, KMD, and other BTC-based forks, fixed fee)

```bash
curl --url "http://127.0.0.1:7783" --data "{\"method\":\"withdraw\",\"coin\":\"RICK\",\"to\":\"R9o9xTocqr6CeEDGDH6mEYpwLoMz6jNjMW\",\"amount\":\"1.0\",\"fee\":{\"type\":\"UtxoFixed\",\"amount\":\"0.1\"}}"
```

<div style="margin-top: 0.5rem;">

<collapse-text hidden title="Response">

#### Response (success)

```json
{
  "tx_hex":"0400008085202f8901ef25b1b7417fe7693097918ff90e90bba1351fff1f3a24cb51a9b45c5636e57e010000006b483045022100b05c870fcd149513d07b156e150a22e3e47fab4bb4776b5c2c1b9fc034a80b8f022038b1bf5b6dad923e4fb1c96e2c7345765ff09984de12bbb40b999b88b628c0f9012102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffff0200e1f505000000001976a91405aab5342166f8594baf17a7d9bef5d56744332788ac8cbaae5f010000001976a91405aab5342166f8594baf17a7d9bef5d56744332788ace87a5e5d000000000000000000000000000000",
  "tx_hash":"1ab3bc9308695960bc728fa427ac00d1812c4ae89aaa714c7618cb96d111be58",
  "from":["R9o9xTocqr6CeEDGDH6mEYpwLoMz6jNjMW"],
  "to":["R9o9xTocqr6CeEDGDH6mEYpwLoMz6jNjMW"],
  "total_amount":"60.10253836",
  "spent_by_me":"60.10253836",
  "received_by_me":"60.00253836",
  "my_balance_change":"-0.1",
  "block_height":0,
  "timestamp":1566472936,
  "fee_details":{
    "amount":"0.1"
  },
  "coin":"RICK",
  "internal_id":""
}
```

#### Response (error - attempt to use EthGas for UTXO coin)

```json
{"error":"utxo:1295] Unsupported input fee type"}
```

</collapse-text>

</div>

#### Command (BTC, KMD, and other BTC-based forks, 1 RICK per Kbyte)

```bash
curl --url "http://127.0.0.1:7783" --data "{\"method\":\"withdraw\",\"coin\":\"RICK\",\"to\":\"R9o9xTocqr6CeEDGDH6mEYpwLoMz6jNjMW\",\"amount\":\"1.0\",\"fee\":{\"type\":\"UtxoPerKbyte\",\"amount\":\"1\"}}"
```

<div style="margin-top: 0.5rem;">

<collapse-text hidden title="Response">

#### Response (success)

```json
{
  "tx_hex":"0400008085202f890258be11d196cb18764c71aa9ae84a2c81d100ac27a48f72bc6059690893bcb31a000000006b483045022100ef11280e981be280ca5d24c947842ca6a8689d992b73e3a7eb9ff21070b0442b02203e458a2bbb1f2bf8448fc47c51485015904a5271bb17e14be5afa6625d67b1e8012102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffff58be11d196cb18764c71aa9ae84a2c81d100ac27a48f72bc6059690893bcb31a010000006b483045022100daaa10b09e7abf9d4f596fc5ac1f2542b8ecfab9bb9f2b02201644944ddc0280022067aa1b91ec821aa48f1d06d34cd26fb69a9f27d59d5eecdd451006940d9e83db012102031d4256c4bc9f99ac88bf3dba21773132281f65f9bf23a59928bce08961e2f3ffffffff0200e1f505000000001976a91405aab5342166f8594baf17a7d9bef5d56744332788acf31c655d010000001976a91405aab5342166f8594baf17a7d9bef5d56744332788accd7c5e5d000000000000000000000000000000",
  "tx_hash":"fd115190feec8c0c14df2696969295c59c674886344e5072d64000379101b78c",
  "from":["R9o9xTocqr6CeEDGDH6mEYpwLoMz6jNjMW"],
  "to":["R9o9xTocqr6CeEDGDH6mEYpwLoMz6jNjMW"],
  "total_amount":"60.00253836",
  "spent_by_me":"60.00253836",
  "received_by_me":"59.61874931",
  "my_balance_change":"-0.38378905",
  "block_height":0,
  "timestamp":1566473421,
  "fee_details":{
    "amount":"0.38378905"
  },
  "coin":"RICK",
  "internal_id":""
}
```

#### Response (error - attempt to use EthGas for UTXO coin)

```json
{"error":"utxo:1295] Unsupported input fee type"}
```

</collapse-text>

</div>

#### Command (ETH, ERC20, and other ETH-based forks)

```bash
curl --url "http://127.0.0.1:7783" --data "{\"method\":\"withdraw\",\"coin\":\"ETH\",\"to\":\"0xbab36286672fbdc7b250804bf6d14be0df69fa28\",\"amount\":10,\"userpass\":\"$userpass\"}"
```

<div style="margin-top: 0.5rem;">

<collapse-text hidden title="Response">

#### Response (success)

```json
{
  "block_height": 0,
  "coin": "ETH",
  "fee_details": {
    "coin": "ETH",
    "gas": 21000,
    "gas_price": "0.000000001",
    "total_fee": "0.000021"
  },
  "from": ["0xbab36286672fbdc7b250804bf6d14be0df69fa29"],
  "my_balance_change": ",-10.000021",
  "received_by_me": "0",
  "spent_by_me": "10.000021",
  "to": ["0xbab36286672fbdc7b250804bf6d14be0df69fa28"],
  "total_amount": "10.000021",
  "tx_hash": "8fbc5538679e4c4b78f8b9db0faf9bf78d02410006e8823faadba8e8ae721d60",
  "tx_hex": "f86d820a59843b9aca0082520894bab36286672fbdc7b250804bf6d14be0df69fa28888ac7230489e80000801ba0fee87414a3b40d58043a1ae143f7a75d7f47a24e872b638281c448891fd69452a05b0efcaed9dee1b6d182e3215d91af317d53a627404b0efc5102cfe714c93a28"
}
```

</collapse-text>

</div>

#### Command (ETH, ERC20, and other ETH-based forks, with gas fee)

```bash
curl --url "http://127.0.0.1:7783" --data "{\"userpass\":\"$userpass\",\"method\":\"withdraw\",\"coin\":\"$1\",\"to\":\"$2\",\"amount\":\"$3\",\"fee\":{\"type\":\"EthGas\",\"gas_price\":\"3.5\",\"gas\":55000}}"
```

<div style="margin-top: 0.5rem;">

<collapse-text hidden title="Response">

#### Response (success)

```json
{
  "tx_hex":"f86d820b2884d09dc30082d6d894bab36286672fbdc7b250804bf6d14be0df69fa29888ac7230489e80000801ca0ef0167b0e53ed50d87b6fd630925f2bce6ee72e9b5fdb51c6499a7caaecaed96a062e5cb954e503ff83f2d6ce082649fdcdf8a77c8d37c7d26d46d3f736b228d10",
  "tx_hash":"a26c4dcacf63c04e385dd973ca7e7ca1465a3b904a0893bcadb7e37681d38c95",
  "from":["0xbAB36286672fbdc7B250804bf6D14Be0dF69fa29"],
  "to":["0xbAB36286672fbdc7B250804bf6D14Be0dF69fa29"],
  "total_amount":"10",
  "spent_by_me":"10.0001925",
  "received_by_me":"10",
  "my_balance_change":"-0.0001925",
  "block_height":0,
  "timestamp":1566474670,
  "fee_details":{
    "coin":"ETH",
    "gas":55000,
    "gas_price":"0.0000000035",
    "total_fee":"0.0001925"
  },
  "coin":"ETH",
  "internal_id":""
}
```

#### Response (error - attempt to use UtxoFixed or UtxoPerKbyte for ETH coin)

```json
{"error":"eth:369] Unsupported input fee type"}
```

</collapse-text>

</div>

#### Command (max = true)

```bash
curl --url "http://127.0.0.1:7783" --data "{\"method\":\"withdraw\",\"coin\":\"ETH\",\"to\":\"0xbab36286672fbdc7b250804bf6d14be0df69fa28\",\"max\":true,\"userpass\":\"$userpass\"}"
```

<div style="margin-top: 0.5rem;">

<collapse-text hidden title="Response">

#### Response (success)

```json
{
  "block_height": 0,
  "coin": "ETH",
  "fee_details": {
    "coin": "ETH",
    "gas": 21000,
    "gas_price": "0.000000001",
    "total_fee": "0.000021"
  },
  "from": ["0xbab36286672fbdc7b250804bf6d14be0df69fa29"],
  "my_balance_change": "-10.000021",
  "received_by_me": "0",
  "spent_by_me": "10.000021",
  "to": ["0xbab36286672fbdc7b250804bf6d14be0df69fa28"],
  "total_amount": "10.000021",
  "tx_hash": "8fbc5538679e4c4b78f8b9db0faf9bf78d02410006e8823faadba8e8ae721d60",
  "tx_hex": "f86d820a59843b9aca0082520894bab36286672fbdc7b250804bf6d14be0df69fa28888ac7230489e80000801ba0fee87414a3b40d58043a1ae143f7a75d7f47a24e872b638281c448891fd69452a05b0efcaed9dee1b6d182e3215d91af317d53a627404b0efc5102cfe714c93a28"
}
```

</collapse-text>

</div>
