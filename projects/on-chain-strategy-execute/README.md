# RFC: On-Chain Indicator Oracle & Ephemeral Strategy Executor Architecture

**Authors:** Replicats  
**Date:** 31/08/2025  
**Status:** Draft  
**Target:** Internal discussion / external contributors  

---

## 1. Summary
This RFC proposes an architecture for on-chain strategy automation leveraging a modular **Indicator Oracle** and an **ephemeral Strategy Executor**.  
The design goal is to allow DeFi vaults to execute trades based on transparent, verifiable indicators, while minimizing alpha leakage and protecting strategies.

---

## 2. Motivation
Current vault designs rely on off-chain agents to monitor indicators and push transactions. This creates two challenges:

- **Transparency vs. secrecy tradeoff:** Indicators may be public, but strategies often need to remain proprietary.  
- **Trust assumptions:** Off-chain execution introduces risks of manipulation, front-running, or downtime.  

By deploying an on-chain oracle of indicators and ephemeral Strategy Executors, we aim to:

- Ensure verifiable on-chain strategies.  
- Decouple indicator computation from trade execution.  
- Obfuscate proprietary strategies by self-destructing executors.  

---

## 3. Architecture Overview

### 3.1 Components

**Indicator Oracle**  
- Stores or computes standardized technical indicators:
  - Moving Averages (EMA, SMA, VWAP)  
  - RSI  
  - Bollinger Bands  
  - Volatility measures  
  - Price forecasts  
- **Data sources:**
  - Primary: Chainlink Price Feeds (ETH/USD, BTC/USD, token/USD), Pyth, Redstone  
  - Optional: TWAPs from DEXs (Uniswap v3, Balancer, Aerodrome, Curve)  
- Provides normalized values in `1e18` format.  

**Strategy Executor**  
- Smart contract created per strategy execution.  
- Reads indicators from the Oracle.  
- Encodes trading logic (e.g., EMA crossover, RSI oversold `< 30`).  
- Calls the Vault’s `trade()` function.  
- Self-destructs after execution to reduce transparency of strategy logic.  

**Vault**  
- Holds user funds.  
- Validates allowed routers, assets, and exposures.  
- Executes trades as instructed by Strategy Executors.  

---

## 4. Execution Flow
1. Off-chain keeper or autonomous agent deploys a Strategy Executor contract.  
2. Executor queries Indicator Oracle for conditions.  
3. If conditions are met → Executor calls `Vault.trade(...)`.  
4. Executor self-destructs immediately after execution.  

---

## 5. Design Options

### 5.1 Indicator Computation
- **On-chain (deterministic):** Indicators derived from oracle price feeds + stored historical values.  
- **Hybrid:** Off-chain keepers compute indicators and push them into the Oracle.  
- **Fully off-chain:** Only raw feeds on-chain; indicator computation remains external.  

### 5.2 Strategy Confidentiality
- **Ephemeral Executors (self-destruct):** Prevents public mempool & chain explorers from long-term storing strategy logic.  
- **Obfuscation via factories:** Strategies deployed via a factory with bytecode hashing, no source mapping.  
- **Private mempool (e.g., Flashbots):** Executors submitted privately to avoid sandwiching.  

### 5.3 Security
**Vault enforces:**  
- Allowed router list  
- Exposure caps  
- Max slippage  

**Oracle must:**  
- Use robust, tamper-proof price feeds  
- Provide circuit-breakers on stale data  

---

## 6. Trade-offs
- **Gas costs:** On-chain computation of indicators can be expensive; off-chain submission is cheaper but requires additional trust.  
- **Transparency vs secrecy:** Indicator Oracle is public; Executor obfuscation mitigates but doesn’t fully hide strategies.  
- **Complexity:** Adds new contracts (Oracle, Executor factory) and off-chain keeper requirements.  

---

## 7. Alternatives
- Keep everything off-chain and only submit signed trades.  
- Use zk-proofs for private strategy validation (long-term research).  
- Integrate with Chainlink Functions / DECO for secure off-chain computation.  
