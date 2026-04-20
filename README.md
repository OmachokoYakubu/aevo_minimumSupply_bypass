# Aevo minimumSupply Bypass — Submission Package

**Vulnerability**: `withdrawInstantly()` has no `minimumSupply` guard, allowing inflation attack  
**Severity**: High (Theft of User Funds)  
**Target**: RibbonThetaVault (AAVE) — `0xe63151A0Ed4e5fafdc951D877102cf0977Abd365`  
**Network**: Ethereum Mainnet  
**Fork Block**: 24,900,000  
**Test Status**: ✅ 5/5 PASS on live mainnet fork (Infura RPC)

---

## Structure

```
aevo_submission/
├── IMMUNEFI_SUBMISSION.md        # Full bug report (ready to submit)
├── README.md                     # This file
├── foundry.toml                  # Forge config (uses ETH_RPC_URL env var)
├── exploit_results_fork.txt      # Raw terminal output from mainnet fork run
└── test/
    └── AevoForkExploit.t.sol     # 5-step PoC against deployed vault bytecode
```

## Running the PoC

```bash
export ETH_RPC_URL="https://mainnet.infura.io/v3/<YOUR_KEY>"

forge test --match-contract AevoForkExploit \
  --fork-url "$ETH_RPC_URL" \
  --fork-block-number 24900000 \
  -vvv
```

Expected: `Suite result: ok. 5 passed; 0 failed; 0 skipped`

## Test Coverage

| Test | What It Proves |
|------|----------------|
| `test_step0_baseline_vault_state` | Vault is live with real TVL at fork block |
| `test_step1_deposit_guard_present_withdrawal_guard_absent` | Guard exists in deposit code, absent in withdrawal code |
| `test_step2_withdrawInstantly_bypasses_minimumSupply` | withdrawInstantly() succeeds leaving 1 wei (below minimumSupply) |
| `test_step3_inflation_attack_sharemath_proof` | Math proof: inflated PPS causes 0-share minting for victim |
| `test_step4_endToEnd_sameRound_theft` | Full attack chain: deposit → drain → donate → victim loses funds |
