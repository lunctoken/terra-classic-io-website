This guide explains how to install the `terra_sdk` python SDK, connect to the recommended endpoints, and submit common transactions with typed examples.

## Requirements

- Python 3.8 or later
- `pip` 23+ (or `uv`)
- Optional: `make` and Docker for running the local Terra Classic network

## Set up a virtual environment

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
```

Install Terra.py:

```bash
pip install --upgrade terra_sdk
```

To work on the library itself, clone the repository and install extras with Poetry:

```bash
git clone --depth 1 https://github.com/terra-money/terra.py.git
cd terra.py
pip install poetry
poetry install
```

## Network endpoints

> **Warning**
> Public endpoints are rate-limited. Run your own node or purchase dedicated access for production workloads.

| Network | Chain ID | LCD | RPC | Gas prices |
| --- | --- | --- | --- | --- |
| LocalTerra | `localterra` | `http://localhost:1317` | `http://localhost:26657` | `http://localhost:1317/terra/tx/v1beta1/gas_prices` |
| rebel-2 (testnet) | `rebel-2` | [https://lcd.luncblaze.com](https://lcd.luncblaze.com) | [https://rpc.luncblaze.com](https://rpc.luncblaze.com) | [https://fcd.luncblaze.com/v1/txs/gas_prices](https://fcd.luncblaze.com/v1/txs/gas_prices) |
| columbus-5 (mainnet) | `columbus-5` | [https://terra-classic-lcd.publicnode.com](https://terra-classic-lcd.publicnode.com) | [https://terra-classic-rpc.publicnode.com](https://terra-classic-rpc.publicnode.com) | [https://terra-classic-fcd.publicnode.com/v1/txs/gas_prices](https://terra-classic-fcd.publicnode.com/v1/txs/gas_prices) |

Alternative mainnet LCD/RPC mirrors include [https://api-lunc-lcd.binodes.com](https://api-lunc-lcd.binodes.com) and [https://api-lunc-rpc.binodes.com](https://api-lunc-rpc.binodes.com).

## Connect to a wallet

The LCD client is the primary entry point for queries and transaction submission.

### LocalTerra

```python
from terra_sdk.client.localterra import LocalTerra
from terra_sdk.client.lcd.wallet import Wallet

terra: LocalTerra = LocalTerra()
wallet: Wallet = terra.wallets["test1"]
print("test wallet", wallet.key.acc_address)
```

### rebel-2 testnet

```python
from terra_sdk.client.lcd import LCDClient
from terra_sdk.client.lcd.wallet import Wallet
from terra_sdk.key.mnemonic import MnemonicKey

LCD_URL: str = "https://lcd.luncblaze.com"
CHAIN_ID: str = "rebel-2"
MNEMONIC: str = "<INSERT TESTNET MNEMONIC>"

terra: LCDClient = LCDClient(url=LCD_URL, chain_id=CHAIN_ID)
mnemonic_key: MnemonicKey = MnemonicKey(mnemonic=MNEMONIC)
testnet_wallet: Wallet = terra.wallet(mnemonic_key)
print("testnet", testnet_wallet.key.acc_address)
```

Request rebel-2 test funds from the [LUNC Blaze faucet](https://faucet.luncblaze.com/) before broadcasting transactions.

### columbus-5 mainnet

```python
from terra_sdk.client.lcd import LCDClient
from terra_sdk.client.lcd.wallet import Wallet
from terra_sdk.key.mnemonic import MnemonicKey

LCD_URL: str = "https://terra-classic-lcd.publicnode.com"
CHAIN_ID: str = "columbus-5"
MNEMONIC: str = "<STORE THIS SECURELY>"

terra: LCDClient = LCDClient(url=LCD_URL, chain_id=CHAIN_ID)
mnemonic_key: MnemonicKey = MnemonicKey(mnemonic=MNEMONIC)
mainnet_wallet: Wallet = terra.wallet(mnemonic_key)
print("mainnet", mainnet_wallet.key.acc_address)
```

> **Danger**
> Never commit or log mnemonics. Use environment variables or secure secret stores for production deployments.

## Fetch gas prices

```python
from typing import Dict
import requests

GAS_PRICE_ENDPOINT: str = "https://terra-classic-fcd.publicnode.com/v1/txs/gas_prices"

def fetch_gas_prices(endpoint: str = GAS_PRICE_ENDPOINT) -> Dict[str, str]:
    response = requests.get(endpoint, timeout=10)
    response.raise_for_status()
    return response.json()

current_prices: Dict[str, str] = fetch_gas_prices()
print(current_prices["uluna"])
```

Gas prices are quoted in micro-denominations (`uluna`, `uusd`, etc.). Adjust fees based on current burn tax policy and validator recommendations.

## Send LUNC

```python
from typing import Dict
from terra_sdk.client.lcd.wallet import Wallet
from terra_sdk.client.lcd import LCDClient
from terra_sdk.client.lcd.api.tx import CreateTxOptions
from terra_sdk.core import Coins, Coin
from terra_sdk.core.bank import MsgSend

AMOUNT_ULUNA: int = 1_000_000
GAS_ADJUSTMENT: float = 1.4
FEE_DENOMS: str = "uluna"

lcd: LCDClient = LCDClient(
    url="https://terra-classic-lcd.publicnode.com",
    chain_id="columbus-5",
)
wallet: Wallet = lcd.wallet(MnemonicKey(mnemonic="<SECURE MNEMONIC>"))
gas_prices: Dict[str, str] = fetch_gas_prices()

def send_lunc(recipient: str) -> str:
    tx_options: CreateTxOptions = CreateTxOptions(
        msgs=[
            MsgSend(
                from_address=wallet.key.acc_address,
                to_address=recipient,
                amount=Coins([Coin("uluna", AMOUNT_ULUNA)]),
            )
        ],
        gas="auto",
        gas_prices=Coins(gas_prices),
        fee_denoms=FEE_DENOMS,
        gas_adjustment=GAS_ADJUSTMENT,
    )
    tx = wallet.create_and_sign_tx(options=tx_options)
    result = lcd.tx.broadcast(tx)
    return result.txhash

recipient_address: str = "terra1..."
tx_hash: str = send_lunc(recipient_address)
print("broadcasted", tx_hash)
```

`lcd.tx.broadcast` returns a result object with execution details, logs, and confirmation status. Query the transaction hash to confirm inclusion in a block.

## Swap assets

```python
from terra_sdk.core.market import MsgSwap

AMOUNT_OFFER: int = 1_000_000
ASK_DENOM: str = "uusd"

swap_options: CreateTxOptions = CreateTxOptions(
    msgs=[
        MsgSwap(
            trader=wallet.key.acc_address,
            offer_coin=Coin("uluna", AMOUNT_OFFER),
            ask_denom=ASK_DENOM,
        )
    ],
    gas="auto",
    gas_prices=Coins(gas_prices),
    fee_denoms=FEE_DENOMS,
    gas_adjustment=GAS_ADJUSTMENT,
)
swap_tx = wallet.create_and_sign_tx(options=swap_options)
swap_result = lcd.tx.broadcast(swap_tx)
print("swap", swap_result.txhash)
```

## Execute a smart contract

```python
from typing import Any, Mapping
from terra_sdk.core.wasm import MsgExecuteContract

CONTRACT_ADDRESS: str = "terra1..."
EXECUTE_MSG: Mapping[str, Any] = {"ping": {}}
EXECUTE_AMOUNT: int = 500_000

execute_options: CreateTxOptions = CreateTxOptions(
    msgs=[
        MsgExecuteContract(
            sender=wallet.key.acc_address,
            contract=CONTRACT_ADDRESS,
            execute_msg=EXECUTE_MSG,
            coins=Coins([Coin("uluna", EXECUTE_AMOUNT)]),
        )
    ],
    gas="auto",
    gas_prices=Coins(gas_prices),
    fee_denoms=FEE_DENOMS,
    gas_adjustment=GAS_ADJUSTMENT,
)
execute_tx = wallet.create_and_sign_tx(options=execute_options)
execute_result = lcd.tx.broadcast(execute_tx)
print("execute", execute_result.txhash)
```

Ensure the contract accepts the execute message and attached funds; otherwise, the transaction fails with a non-zero code.

## Next steps

- Review the [Terra.py reference](https://terra-money.github.io/terra.py/) for detailed APIs.
- Track validator gas recommendations via governance channels.
- Combine Terra.py with notebook environments or backend frameworks to build analytics, bots, and scheduled jobs.
