# basesddimport time
import math
from web3 import Web3

RPC_URL = "https://mainnet.base.org"

# Replace with actual Base pool
POOL_ADDRESS = Web3.to_checksum_address(
    "0x88e6A0c2dDD26FEEb64F039a2c41296FcB3f5640"
)

# keccak256("Swap(address,address,int256,int256,uint160,uint128,int24)")
SWAP_TOPIC = "0xc42079f94a6350d7e6235f29174924f928cc2ac818eb64fed8004e115fbcca67"

BLOCK_WINDOW = 50

POOL_ABI_MIN = [
    {
        "name": "slot0",
        "type": "function",
        "stateMutability": "view",
        "inputs": [],
        "outputs": [
            {"name": "sqrtPriceX96", "type": "uint160"},
            {"name": "tick", "type": "int24"},
            {"name": "observationIndex", "type": "uint16"},
            {"name": "observationCardinality", "type": "uint16"},
            {"name": "observationCardinalityNext", "type": "uint16"},
            {"name": "feeProtocol", "type": "uint8"},
            {"name": "unlocked", "type": "bool"},
        ],
    }
]


def sqrtPriceX96_to_price(sqrt_price_x96):
    return (sqrt_price_x96 / (2 ** 96)) ** 2


def main():
    w3 = Web3(Web3.HTTPProvider(RPC_URL))
    if not w3.is_connected():
        raise RuntimeError("Cannot connect to Base RPC")

    print("Connected to Base")
    print("Calculating realized volatility...")

    last_block = w3.eth.block_number

    while True:
        latest = w3.eth.block_number

        if latest >= last_block + BLOCK_WINDOW:

            from_block = latest - BLOCK_WINDOW
            to_block = latest

            logs = w3.eth.get_logs(
                {
                    "fromBlock": from_block,
                    "toBlock": to_block,
                    "address": POOL_ADDRESS,
                    "topics": [SWAP_TOPIC],
                }
            )

            prices = []

            for log in logs:
                data = bytes.fromhex(log["data"][2:])
                sqrt_price_x96 = int.from_bytes(data[64:96], "big")
                price = sqrtPriceX96_to_price(sqrt_price_x96)
                prices.append(price)

            if len(prices) > 2:
                log_returns = [
                    math.log(prices[i] / prices[i - 1])
                    for i in range(1, len(prices))
                ]

                mean = sum(log_returns) / len(log_returns)
                variance = sum(
                    (r - mean) ** 2 for r in log_returns
                ) / len(log_returns)

                realized_vol = math.sqrt(variance)

                # Rough annualization assuming ~12s block time
                blocks_per_year = (60 / 12) * 60 * 24 * 365
                annualized_vol = realized_vol * math.sqrt(blocks_per_year)

                print("\n--- Realized Volatility ---")
                print(f"Swaps analyzed: {len(prices)}")
                print(f"Annualized Vol (approx): {annualized_vol}")

            last_block = latest

        time.sleep(10)


if __name__ == "__main__":
    main()
