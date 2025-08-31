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

---

## 8. Price Forecasts with Machine Learning

### 8.1 Motivation
Beyond technical indicators (EMA, RSI, volatility), strategies often benefit from forward-looking signals such as short-term or long-term price forecasts. Off-chain machine learning (ML) models trained on historical market data, order book depth, and sentiment analysis can provide predictive insights. Integrating these forecasts into the Oracle expands strategy design space, enabling adaptive trading and risk management.

### 8.2 Architecture

**Off-chain ML Models**
- Models trained and updated regularly (e.g., LSTM, Transformer, Gradient Boosted Trees).
- **Inputs:**
  - Historical OHLCV data
  - On-chain liquidity & order flow (Uniswap v3 swaps, CEX futures funding rates)
  - Market sentiment (optional, off-chain signals like Twitter, news feeds)
- **Outputs:**
  - Price and/or direction forecast at different horizons (e.g., 1h, 1d, 7d).
  - Confidence score (0–1).

**Forecast Oracle**
- Off-chain agent pushes ML forecasts to the on-chain Oracle at regular intervals.
- Forecasts stored in standardized format:
```solidity
struct PriceForecast {
    int256 forecastedPrice;  // 1e18 precision
    uint256 horizon;         // in seconds (e.g., 3600 = 1h)
    uint256 confidence;      // 0–1e18 scale
    uint256 timestamp;       // block.timestamp of update
}
```
- Oracle maintains the latest forecasts for supported assets.

**Strategy Executor Integration**
- Strategies can combine indicators + forecasts (e.g., EMA cross AND 1h forecast > current price).
- Vault execution remains guarded by slippage/exposure checks.

### 8.3 Design Options

**Deterministic vs Probabilistic**
- Forecasts are inherently probabilistic — Executors must account for confidence.
- Example: only trade if confidence > 0.7e18.

**Source of Trust**
- Forecasts cannot be fully trustless. Options:
  - **Centralized publisher:** Single trusted ML operator.
  - **Multi-publisher consensus:** Several providers push forecasts, Oracle aggregates.
  - **Token-curated registry:** Stakers vote on forecast providers.

**Data Freshness**
- Forecasts must expire after horizon + grace period.
- Prevents stale signals from triggering trades.

### 8.4 Risks
- **Manipulation:** Forecast submitters may bias signals to influence trades.
- **Front-running:** Adversaries may exploit publicly available forecasts.
- **Overfitting:** ML models trained on historical data may fail in regime shifts.
- **Gas overhead:** Storing multiple horizons and assets increases state size.

### 8.5 Alternatives
- Keep forecasts entirely off-chain, use Executors to check them privately before sending trades.
- Use zkML (zero-knowledge proofs of ML inference) for future-proof private forecast validation.
- Integrate Oracle Functions to fetch ML forecasts from external APIs securely.
