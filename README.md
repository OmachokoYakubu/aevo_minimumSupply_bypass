# Aevo minimumSupply Bypass — Critical Vulnerability PoC

**Researcher**: Hackerdemy  
**Date**: 20 April 2026  
**Program**: Aevo / Ribbon Finance (Immunefi)  
**Severity**: Critical — Direct Theft of User Funds  

---

## Overview

This repository contains a complete proof of concept demonstrating a critical logic asymmetry in Aevo's legacy Ribbon Finance Vaults. The `RibbonVault` base contract relies on a `minimumSupply` guard to prevent ERC-4626 first-depositor share inflation attacks. However, this guard is only enforced during deposits. 

By executing a deposit followed immediately by `withdrawInstantly()`, an attacker can legally drain the vault's pending supply to exactly 1 wei—bypassing the `minimumSupply` floor. Subsequent direct donations to the vault artificially inflate `pricePerShare` to extreme values, causing all future user deposits in that round to round down to 0 shares, permanently capturing their capital.

The bug is systemic and affects multiple vaults in scope, representing over **$3.3 Million** in Total Value Locked (TVL).

**Target Contract**: [RibbonThetaVault (AAVE)](https://etherscan.io/address/0xe63151A0Ed4e5fafdc951D877102cf0977Abd365) — `0xe63151A0Ed4e5fafdc951D877102cf0977Abd365` (Ethereum Mainnet)

---

## Repository Contents

| File | Description |
|------|-------------|
| `IMMUNEFI_SUBMISSION.md` | Main bug report covering the vulnerability, impact analysis, novelty defense vs prior audits, recommendation, and PoC details. |
| `test/AevoForkExploit.t.sol` | Foundry PoC — 5 tests demonstrating the minimumSupply bypass and inflation attack on forked Ethereum Mainnet. |
| `exploit_results.txt` | Full `-vvvv` verbosity output from the final test run. |
| `foundry.toml` | Foundry configuration file. |

---

## Setup & Reproduction

### Prerequisites

- [Foundry](https://book.getfoundry.sh/getting-started/installation) installed. If you don't have it:
  ```bash
  curl -L https://foundry.paradigm.xyz | bash
  foundryup
  ```
- Internet connection (the tests fork live Ethereum state)

### Clone and Run

```bash
# 1. Clone this repository
git clone https://github.com/OmachokoYakubu/aevo_minimumSupply_bypass.git

# 2. Navigate into the project directory
cd aevo_minimumSupply_bypass

# 3. Install dependencies (forge-std)
forge install

# 4. Set your Ethereum mainnet RPC
export ETH_RPC_URL="https://mainnet.infura.io/v3/<YOUR_KEY>"

# 5. Run the full PoC (5 tests)
forge test --match-contract AevoForkExploit --fork-url "$ETH_RPC_URL" --fork-block-number 24900000 -vvvv
```

### Expected Output

```
[PASS] test_step0_baseline_vault_state() (gas: 51299)
[PASS] test_step1_deposit_guard_present_withdrawal_guard_absent() (gas: 183132)
[PASS] test_step2_withdrawInstantly_bypasses_minimumSupply() (gas: 184302)
[PASS] test_step3_inflation_attack_sharemath_proof() (gas: 28739)
[PASS] test_step4_endToEnd_sameRound_theft() (gas: 294377)

Suite result: ok. 5 passed; 0 failed; 0 skipped
```

---

## Test Breakdown

| Test | What It Demonstrates |
|------|---------------------|
| **test_step0** | Confirms the vault is live with real TVL at the fork block (baseline state). |
| **test_step1** | Proves the asymmetry: `_depositFor()` enforces the guard, but `withdrawInstantly()` completely lacks it. |
| **test_step2** | Executes the bypass: successfully withdraws funds leaving exactly 1 wei pending (far below the 1e10 `minimumSupply`). |
| **test_step3** | Math proof: Shows that an inflated `pricePerShare` causes even large victim deposits to round down to 0 shares. |
| **test_step4** | End-to-end full attack chain: attacker deposits, drains to 1 wei, donates to inflate, and perfectly captures the victim's deposit. |

---

## Important Notes

- The PoC forks Ethereum Mainnet at block **24,900,000** — it does **not** test on live mainnet.
- Token acquisition uses Foundry's `deal()` cheatcode to simulate purchasing AAVE. The actual exploit uses only legitimate, permissionless on-chain calls (`deposit` and `withdrawInstantly`).
- All test output includes clearly labeled `[CONFIRMED]` markers to highlight successful attack steps.

---

*Omachoko Yakubu, Security Researcher*
