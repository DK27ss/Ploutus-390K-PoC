# Ploutus-390K-PoC

>| Item | Detail |
>|---|---|
>| **Protocol** | Aave v3 (Ethereum mainnet) |
>| **Type** | Oracle Misconfiguration |
>| **Misconfiguration block** | 24,538,894 |
>| **Attack block** | 24,538,895 |
>| **Total loss** | 187.37 ETH (~$390,000) |
>| **Attacker cost** | ~8.88 USDC + gas + 0.004 WETH (flash swap fee) |

## Summary

An `AssetListingAdmin` on Aave v3 (Ethereum) misconfigured the **USDC** price oracle to point to the Chainlink **BTC/USD** feed instead of **USDC/USD**, this inflated the USDC value by a factor of **~68,558x** within the Aave system, an attacker immediately exploited this by depositing **~8.88 USDC** as collateral (valued at $610K by the corrupted oracle) and borrowing **~187.37 WETH** (~$390K) that was never repaid.

---

## Exploit

// Oracle Misconfiguration

Address `0xfb33...02Cc`, holding the `AssetListingAdmin` role in Aave's ACL Manager, executed the following call

```solidity
AaveOracle.setAssetSources(
    [0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48],  // USDC
    [0xF4030086522a5bEEa4988F8cA5B36dbC97BeE88c]   // Chainlink BTC/USD (WRONG)
);
```

<img width="1098" height="149" alt="image" src="https://github.com/user-attachments/assets/aee4dde1-59a3-4393-9124-9dafd25472c3" />

**Before:**
- `getAssetPrice(USDC)` -> `99,993,076` (8 decimals) = **~$1.00**
- Feed: Chainlink USDC/USD (`0x8fFf...18f6`)

**After:**
- `getAssetPrice(USDC)` -> `6,855,405,329,514` (8 decimals) = **~$68,554**
- Feed: Chainlink BTC/USD (`0xF403...E88c`)

<img width="1090" height="241" alt="image" src="https://github.com/user-attachments/assets/a4a831b5-be1a-4a8b-b1d1-dcd49ceb5df4" />

The Aave system now considers **1 USDC = 1 BTC in value**.

Misconfiguration TX: https://app.blocksec.com/phalcon/explorer/tx/eth/0xcfedf63b37a6cd45b21bc94e3de5412fee0765e7dad6b7c8561a01cebd193ab6

Attack TX: https://app.blocksec.com/phalcon/explorer/tx/eth/0xa17dc37e1b65c65d20042212fb834974f7faaa961442e3fc05393778705f8474

// Exploitation

The attacker deployed a contract and executed the exploit in a single atomic transaction

```
Flash swap 8.88 USDC (Uniswap V2)
        |
        v
Deposit 8.88 USDC into Aave (collateral)
        |  Oracle: 8.88 * $68,554 = ~$608,840
        v
Borrow 187.37 WETH (variable rate, mode 2)
        |  Value: 187.37 * $2,080 = ~$389,730
        |  LTV check passes thanks to inflated collateral
        v
Repay flash swap: 0.004 WETH
        |
        v
Unwrap WETH -> ETH
        |
        v
Distribute profit:
  - 5.61 ETH  -> 0x4838...5f97 (deployer/fee)
  - 181.75 ETH -> 0x3885...cEe  (main profit)
```

<img width="1098" height="170" alt="image" src="https://github.com/user-attachments/assets/e16f9690-d306-4117-bb31-b8dc13896e86" />

---

## Root cause

The Aave oracle is a simple `asset -> priceFeed` mapping

The Aave v3 oracle (`AaveOracle`) maintains an internal `assetsSources` mapping that associates each token to a Chainlink aggregator, the `setAssetSources` function allows admins to update this mapping **without any coherence validation** (no sanity check, no price bounds, no timelock).

// AssetListingAdmin role has no safeguards

The only check is a `require(isAssetListingAdmin(msg.sender))` in the ACL Manager, there is :
- No **timelock** on oracle changes
- No **multi-sig** requirement
- No **sanity check** on prices (e.g., verifying the new price is within a reasonable range of the old one)
- No **automatic circuit breaker**

// Collateral inflation is multiplicative

Aave health factor calculation directly uses the price returned by the oracle

```
collateralValue = depositedAmount * oraclePrice * LTV
borrowCapacity  = collateralValue / borrowedAssetPrice
```

With the BTC/USD feed
```
collateralValue = 8.88 USDC * $68,554 * 0.80 (LTV) = ~$487,071
borrowCapacity  = $487,071 / $2,080 = ~234 WETH max
```

The attacker borrowed 187.37 WETH, well below the theoretical maximum, to ensure the transaction would succeed.

// Flash swap enables zero upfront cost

The attacker needed no initial capital, the Uniswap V2 flash swap provides the USDC, which is deposited as collateral, enabling the WETH borrow needed to repay the flash swap and keep the profit, everything happens in a single atomic transaction.

// Debt remains unpaid

The 187.37 WETH borrow on Aave will never be repaid, the real collateral (8.88 USDC = ~$8.88) is negligible, this is a net loss for WETH depositors in the Aave pool.

---

## Fund Flow

```
Initial state
  Attacker: 0 USDC, 0 WETH, 0 ETH

1. Uniswap V2 flash swap
   Uniswap -> Attacker:  +8,879,192 USDC  (8.88 USDC)

2. Aave deposit
   Attacker -> Aave Pool: -8,879,192 USDC
   Aave -> Attacker:      +aUSDC (deposit receipt token)

3. Aave borrow
   Aave Pool -> Attacker: +187,366,746,326,704,993,556 wei WETH (187.37 WETH)
   Aave:                   variable debt token minted to attacker

4. Flash swap repayment
   Attacker -> Uniswap:   -4,289,216,474,598,283 wei WETH (0.004 WETH)

5. Unwrap and distribute:
   Attacker              187.37 - 0.004 = 187.362 WETH -> ETH
   -> 0x4838...5f97:       5.613 ETH
   -> 0x3885...cEe:       181.749 ETH

Final state:
  Attacker:     0 USDC, 0 WETH, 0 ETH (all distributed)
  Beneficiaries: 187.362 ETH total
  Aave:         -187.37 WETH (bad debt), +8.88 USDC (worthless collateral)
```

---

## Output

<img width="552" height="299" alt="image" src="https://github.com/user-attachments/assets/9a7f30d7-308a-4d24-a402-d42dba73c90f" />

```
USDC price BEFORE misconfiguration (8 decimals): 99993076         (~$1.00)
USDC price AFTER misconfiguration (8 decimals):  6855405329514    (~$68,554)
Price inflation factor: 68558

USDC price at attack block (8 decimals): 6855405329514
WETH price at attack block (8 decimals): 208047000000             (~$2,080)
Attacker ETH balance BEFORE: 0.000000000000000000
Attacker ETH balance AFTER:  187.362457110230395273

=== RESULTS ===
ETH stolen: 187.362457110230395273 ETH
```
