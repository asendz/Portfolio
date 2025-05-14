## About me

I’m Asen, a blockchain security researcher focused on identifying vulnerabilities in smart contracts.

I actively participate in audit contests on platforms like [Code4rena](https://code4rena.com/@0xAsen), [Sherlock](https://audits.sherlock.xyz/watson/0xAsen), [CodeHawks](https://www.codehawks.com/profile/clk3vjbfh000kkx08mg4x5ug0), and Immunefi, where I’ve contributed to securing a  wide range of DeFi protocols, bridges, and infrastructure projects.


## Highlights
Here's a quick overview of my track record in Web3 security research:
- 🧠 **60+ accepted findings** accross Code4rena, Sherlock, Codehawks, Immunefi
- 💰 **$10,000+ earned** in public contests
- 🥇 **Top 5 and Top 10 placements** in contests on Code4rena, Sherlock, and Immunefi
- 🧪 Audited protocols including **bridges, launchpads, DEXes, and infrastructure**
- ⚒️ Languages: **Solidity**, **Cairo**

## Top 5 Personal Favorite Findings
These are five findings I particularly like - not solely because of their impact, but because they reflect different aspects: subtle logic, economic insight, griefing vectors, and cross-chain reasoning. Each one required different thinking to find. 

Note: Some findings link to Solodit report, but the full submissions — including my original writeups - are available in the next section.

- [**A malicious actor can stuck other unsuspecting users' tokens forever - ArkProject | Codehawks**](https://solodit.cyfrin.io/issues/reentrancy-attack-to-make-an-nft-unbridgeable-codehawks-arkproject-nft-bridge-git)  
  
  A complex reentrancy attack in ArkProject where a malicious actor could trap NFTs forever by manipulating escrow logic mid-transfer and then selling the compromised NFT. This finding stands out because it required connecting on-chain mechanics with user flows to create a griefing vector.
- [**Harvest timing exploit enables theft of unclaimed yield - Yeet | Immunefi**](https://github.com/asendz/Portfolio/blob/main/Security%20Reports/Immunefi/Yeet.md)
  
  A subtle economic exploit in a DeFi vault where share price manipulation allows an attacker to exploit unclaimed rewards. I like this finding because it required thinking beyond code — into how time, incentives, and accounting interact.
- [**Liquidity migration DOS via token injection - IQ AI | Code4rena**](https://solodit.cyfrin.io/issues/m-02-attacker-can-dos-liquidity-migration-in-liquiditymanagersol-code4rena-iq-ai-iq-ai-git)
  
  In IQ AI, an attacker could brick a key migration function by artificially inflating token balances. I like this one because it involved a very "innocent" root cause - simply using raw address balances to cause a denial of service.
- [**SALT emissions can't be fully emitted if performUpkeep() is called too often due to a precision loss - Salty | Code4rena**](https://solodit.cyfrin.io/issues/m-23-stakingrewards-pools-are-not-given-their-promised-share-of-rewards-due-to-incorrect-calculation-code4rena-saltyio-saltyio-git)
  
  This was a subtle but high-impact issue where repeated calls to performUpkeep() could lead to rounding down rewards to zero — permanently locking emissions in the contract. I like this one because it's not an obvious security bug — it’s an economic degradation that compounds over time, especially on fast L2s. It shows how low-level arithmetic and execution timing can quietly break tokenomics.
- [**Cross-chain message always marked as SUCCESS due to weak handler return logic - Chakra | Code4rena**](https://solodit.cyfrin.io/issues/h-02-in-starknet-already-processed-messages-can-be-re-submitted-and-by-anyone-code4rena-chakra-chakra-git)
  
  A message was always marked as successful - even if it failed silently - because the handler function returned true as long as it didn’t revert. I like this one because it reveals how protocols can mistakenly equate “no revert” with “success,” breaking critical invariants around transaction status. It also needed the ability to spot differences across implementations (Cairo vs Solidity) and reason about silent logic failures, which are often overlooked.

## Audit Contests
Below is a curated list of my public audit contest contributions.

Each entry reflects findings that were accepted and rewarded for identifying vulnerabilities across real-world protocols.

For a complete view of my audit history, see my full profile [here](https://audits.sherlock.xyz/watson/0xAsen).

| Contest                                                                                                         | Year | Platform  | Type of protocol                                      | # of findings and severity | My findings                                                                                      | Language         | Rank |
| :-------------------------------------------------------------------------------------------------------------- | :--- | :-------- | :---------------------------------------------------- | :------------------------- | :------------------------------------------------------------------------------------------------ | :--------------- | :--- |
| [Starknet Perps](https://code4rena.com/audits/2025-03-starknet-perpetual)                                                       | 2025 | Code4rena  | Perpetuals DEX                                                   | 1 High          | [Link](https://github.com/asendz/Portfolio/blob/main/Security%20Reports/Code4rena/StarknetPerpetuals.md)         | Cairo         | -   |
| [Yeet](https://immunefi.com/audit-competition/audit-comp-yeet/leaderboard/#top)                                                       | 2025 | Immunefi  | Gamified DeFi protocol                                                   | 1 Medium           | [Link](https://github.com/asendz/Portfolio/blob/main/Security%20Reports/Immunefi/Yeet.md)         | Solidity         | 7   |
| [Recall](https://code4rena.com/audits/2025-02-recall)                                                       | 2025 | Code4rena  | Intelligence network for AI agents                                                   | 1 High, 1 Medium           | [Link](https://github.com/asendz/Portfolio/blob/main/Security%20Reports/Code4rena/Recall.md)         | Solidity & Rust        | 5   |
| [THORWallet](https://code4rena.com/audits/2025-02-thorwallet)                                                       | 2025 | Code4rena  | All-in-one DeFi solution                                                   | 1 High, 1 Medium           | [Link](https://github.com/asendz/Portfolio/blob/main/Security%20Reports/Code4rena/THORWallet.md)         | Solidity         | 8  |
| [Rova](https://audits.sherlock.xyz/contests/498)                                                       | 2025 | Sherlock  | Onchain launchpad                                                   | 1 Medium           | [Link](https://github.com/asendz/Portfolio/blob/main/Security%20Reports/Sherlock/Rova.md)         | Solidity         | 3   |
| [IQ AI](https://code4rena.com/audits/2025-01-iq-ai)                                                       | 2025 | Code4rena  | Tokenized AI agents                                                   | 1 Medium           | [Link](https://github.com/asendz/Portfolio/blob/main/Security%20Reports/Code4rena/IQ-AI.md)         | Solidity         | 4   |
| [Chakra](https://code4rena.com/audits/2024-08-chakra)                                                           | 2024 | Code4rena  | Bridge infrastructure                                 | 6 High, 2 Medium           | [Link](https://github.com/asendz/Portfolio/blob/main/Security%20Reports/Code4rena/Chakra.md)                                                                                           | Solidity & Cairo | 4    |
| [ArkProject](https://codehawks.cyfrin.io/c/2024-07-ark-project)                                                 | 2024 | Codehawks  | NFT Bridge                                            | 3 High, 2 Medium           | [Link](https://github.com/asendz/Portfolio/blob/main/Security%20Reports/CodeHawks/ArkProject.md) | Solidity & Cairo | 12   |
| [MagicSea](https://audits.sherlock.xyz/contests/437)                                                       | 2024 | Sherlock  | DEX                                                   | 2 High, 2 Medium           | [Link](https://github.com/asendz/Portfolio/blob/main/Security%20Reports/Sherlock/MagicSea.md)         | Solidity         | -   |
| [AI Arena](https://code4rena.com/audits/2024-02-ai-arena)                                                       | 2024 | Code4rena  | AI game                                                  | 4 High           | [Link](https://github.com/asendz/Portfolio/blob/main/Security%20Reports/Code4rena/AIArena.md)         | Solidity         | -   |
| [Salty.io](https://code4rena.com/audits/2024-01-saltyio)                                                       | 2024 | Code4rena  | DEX                                                   | 1 High, 3 Medium           | [Link](https://github.com/asendz/Portfolio/blob/main/Security%20Reports/Code4rena/Salty.md)         | Solidity         | 14   |
| [NextGen](https://code4rena.com/audits/2023-10-nextgen)                                                         | 2023 | Code4rena  | Generative art projects | 2 High                     | [Link](https://github.com/asendz/Portfolio/blob/main/Security%20Reports/Code4rena/NextGen.md)       | Solidity         | -    |
| [SteadeFi](https://www.codehawks.com/contests/clo38mm260001la08daw5cbuf)                                       | 2023 | CodeHawks  | Leveraged yield vault w/ GMX                          | 1 High, 2 Medium           | [Link](https://github.com/asendz/Portfolio/blob/main/Security%20Reports/CodeHawks/SteadeFi.md)      | Solidity         | -    |
| [Ondo](https://code4rena.com/contests/2023-09-ondo-finance)                                                    | 2023 | Code4rena  | RWA rebasing stablecoin                               | 1 Medium                   | [Link](https://github.com/asendz/Portfolio/blob/main/Security%20Reports/Code4rena/Ondo.md)          | Solidity         | 9   |
| [BeedleFi](https://www.codehawks.com/contests/clkbo1fa20009jr08nyyf9wbx)                                       | 2023 | CodeHawks | Oracle-free perpetual lending                         | 5 High, 5 Medium           | [Link](https://github.com/asendz/Portfolio/blob/main/Security%20Reports/CodeHawks/BeedleFi.md)      | Solidity         | -    |
| [Tangible](https://code4rena.com/contests/2023-08-tangible-caviar)                                             | 2023 | Code4rena  | RWA liquid wrapper                                    | 1 High, 2 Medium           | [Link](https://github.com/asendz/Portfolio/blob/main/Security%20Reports/Code4rena/Tangible.md)      | Solidity         | -    |
| [Footium](https://audits.sherlock.xyz/contests/71)                                                              | 2023 | Sherlock   | NFT Football game                                     | 1 High, 2 Medium           | [Link](https://github.com/asendz/Portfolio/blob/main/Security%20Reports/Sherlock/Footium.md)        | Solidity         | 11   |


## Private Audits

| Name                                                      | Type of protocol | # of findings and severity | Report                                                                                                 |
| :-------------------------------------------------------- | :--------------- | :------------------------- | :----------------------------------------------------------------------------------------------------- |
| [ArtStakes](https://github.com/owl11/ArtStakes/tree/main) | NFT, Bridge      | 3 High, 3 Low              | [Link](https://github.com/asendz/Portfolio/blob/main/Security%20Reports/Private%20audits/ArtStakes.md) |
| [EventShares]() | Unlimited upside sports betting     | 1 High, 2 Medium, 4 Low              | [Link](https://github.com/asendz/Portfolio/blob/main/Security%20Reports/Private%20audits/EventShares.md) |

## Contacts

You can find me on Twitter [@asen_sec](https://twitter.com/asen_sec).
