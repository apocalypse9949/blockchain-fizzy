# Metamorphic Blockchain — MWP Technical Architecture

---

## 0) Scope (What This Demo Proves)
- Live chain can detect a threat → agree to switch → hot-swap consensus without losing state or halting liveness.
- Two modes for the MWP (keep it lean): **PoS ↔ BFT**. (Add PoW later.)
- Threat signal is real (latency/fork rate), not mocked.

---

## 1) Assumptions
- Permissioned testnet (20–30 nodes) to simplify coordination and observability.
- Finality available in both modes (BFT has instant finality; PoS uses checkpoints).
- Nodes run the same runtime with pluggable consensus engines.

---

## 2) Stack (Pick 1 and Commit)
- **Substrate (Rust)** — recommended. Clean consensus abstraction, on-chain pallets, off-chain workers.
- **Pallets:** 
  - `pallet-consensus-manager`
  - `pallet-threat-oracle`
  - `pallet-checkpoint`
- **Engines:** 
  - BABE/GRANDPA (PoS-ish) 
  - HotStuff/PBFT-like custom engine (BFT)
- **Metrics:** Prometheus + Grafana
- **Attack Simulation:** Mininet + Scapy + custom Python scripts

---

## 3) High-Level Architecture (Modules)

### Threat Detection Engine (TDE)
- Runs on each node (agent) + an on-chain Threat Oracle pallet to aggregate.
- **Signals:**
  - `net_latency_ms` (EWMA)
  - `fork_rate`
  - `missed_blocks`
  - `validator_join_rate`
  - Optional ML score `anomaly_score ∈ [0,1]`
- Emits **ThreatLevel** ∈ {LOW, ELEVATED, HIGH, CRITICAL}.

### Consensus Manager (CM) Pallet
- Owns a finite state machine (FSM) of consensus modes.
- Exposes extrinsics:
  - `propose_switch(mode, at_block)`
  - `vote_switch(proposal_id)`
- Enforces safe-switch conditions (min finality, quorum, cooldown).

### Checkpoint & State Continuity
- Checkpoint pallet commits `{state_root, block_number, validator_set, mode}` under GRANDPA finality.
- Switch only armed at Checkpoint K to guarantee replay consistency.

### Consensus Engines
- **Mode A:** PoS (BABE+GRANDPA) — normal operations.
- **Mode B:** BFT (HotStuff-lite) — smaller committee, fast finality, Sybil/DDoS hardening.
- Engines implement a common trait:
```javascript
trait ConsensusEngine {
fn start(from_checkpoint: Checkpoint);
fn stop() -> Result<(), Error>;
fn health() -> EngineHealth;
}
```

### Switch Coordinator (Meta-Consensus)
- Implemented on-chain via CM pallet + off-chain worker to schedule activation at block N+Δ.
- Prevents split-brain: all nodes read the same proposal & activation block.

---

## 4) Data Models (On-chain)

```javascript
struct ThreatReport {
node_id: H160,
ts: u64,
net_latency_ms: u32,
fork_rate: u16,
missed_blocks: u16,
anomaly_score: u16 // 0..1000
}

enum ThreatLevel { LOW, ELEVATED, HIGH, CRITICAL }

struct SwitchProposal {
id: u64,
target_mode: Mode, // PoS or BFT
activation_block: u64, // must be >= current + guard_window
quorum_needed: u32, // e.g., 2/3 validators
yes_votes: BTreeSet<AccountId>,
status: Pending | Committed | Expired
}

struct Checkpoint {
block: u64,
state_root: H256,
validator_set_hash: H256,
mode: Mode
}
```

---

## 5) Protocol Flow (DDSPR Applied)

### Detect
- TDE agents push `ThreatReport` to `pallet-threat-oracle`.
- Oracle computes rolling **ThreatLevel** (median across reports + sanity rules).

### Decide
- If `ThreatLevel ≥ HIGH` and mode == PoS, CM auto-creates `SwitchProposal(BFT)` with activation_block = now + guard_window.
- Validators vote on-chain; require ≥ 2/3 quorum.

### Switch (Commit & Activate)
- On quorum: CM sets `proposal.status = Committed`.
- Checkpoint at block C (≥ last finalized).
- At `activation_block`, all nodes:
  - `current_engine.stop()`
  - Load Checkpoint C
  - `target_engine.start(C)`

### Protect
- In BFT mode, only committee signs blocks; fork window collapses; DDoS/Sybil impact drops.

### Recover
- When `ThreatLevel ≤ ELEVATED` for T consecutive blocks, CM proposes switch back to PoS with same process + checkpoint.

---

## 6) Safety & Liveness Guards
- **Single Source of Truth:** Switch params on-chain, not in config files.
- **Guard Windows:** activation_block = now + Δ to ensure all nodes receive the plan.
- **Finality First:** Never switch without a finalized checkpoint.
- **Cooldown:** Minimum N blocks between switches to avoid flapping.
- **Health Fencing:** If target engine fails `health.startup == OK` within M seconds, auto-roll back to previous mode using checkpoint.

---

## 7) Minimal Algorithms (Pseudo-code)

### Threat Oracle (per block)
```javascript
reports = get_reports(last_window)
lat = median(reports.net_latency_ms)
fork = median(reports.fork_rate)
miss = median(reports.missed_blocks)
anom = median(reports.anomaly_score)

score = w1norm(lat) + w2norm(fork) + w3norm(miss) + w4anom

if score < τ1 -> LOW
else if score < τ2 -> ELEVATED
else if score < τ3 -> HIGH
else -> CRITICAL

emit ThreatLevel
```

### Consensus Manager (Transition)
```javascript
on ThreatLevel change:
if mode == PoS and level >= HIGH and !cooldown:
create SwitchProposal(target = BFT, activation = now + Δ)

if mode == BFT and level <= ELEVATED and stable_for >= T and !cooldown:
create SwitchProposal(target = PoS, activation = now + Δ)

on ProposalCommitted:
checkpoint = finalize_and_record()
schedule Activation(activation_block, checkpoint)

on Activation:
old.stop()
new.start(checkpoint)
record ModeChanged
```

---

## 8) Test Plan (What to Demo)

### Scenarios
- Sybil / stake spike → Threat=HIGH → PoS→BFT in ≤ X seconds → TPS dips briefly then stabilizes; forks → ~0.
- DDoS latency surge → PoS→BFT; block finality remains < target.
- Threat clears for T blocks → BFT→PoS; throughput returns to baseline.
- Engine failure injection during activation → auto-rollback works.

### Metrics to Capture
- Switch decision latency (Detect → Commit)
- Activation downtime (target: zero-halt)
- Finality time before/after
- Fork/orphan rate
- TPS and p95 latency
- Missed blocks during switch (target: 0)

---

## 9) Milestones (Aggressive but Realistic)

| Week | Tasks                                                                                 |
|-------|---------------------------------------------------------------------------------------|
| 1–2   | Substrate node scaffold; pallets for Threat Oracle & CM; PoS mode running; metrics wired. |
| 3–4   | BFT engine (committee selection, quorum, HotStuff-lite); Checkpoint pallet; one manual switch (CLI-triggered). |
| 5     | Full auto switch (oracle → proposal → vote → checkpoint → activation); DDoS & Sybil simulations; dashboards. |
| 6     | Hardening (cooldown, rollback, health checks), paper figures, demo video.             |

---

## 10) Known Risks (And How You’re Covering Them)
- **Split-brain risk:** Eliminated by on-chain switch plan + single activation block + finalized checkpoints.
- **Oracle manipulation:** Median across many nodes + rate limits + stake-weighted reports (optional).
- **Switch thrashing:** Cooldowns + hysteresis (need sustained HIGH to switch, sustained LOW to revert).
- **Engine ABI drift:** Common ConsensusEngine trait and a shared state schema.

---

## 11) Stretch Goals (If Time Permits)
- Add PoW engine and tri-mode routing (PoS ↔ BFT ↔ PoW).
- Lightweight ML model (Isolation Forest) embedded as an off-chain worker.
- Dynamic committee sizing in BFT based on threat intensity.

# Adaptive Consensus Protocol Flow

---

## Transaction Entry
- Users submit transactions to the network as usual.
- Transactions enter a pending pool.

---

## Normal Consensus
- The blockchain operates with its default consensus (e.g., Multi-Weight PoS) under stable conditions.
- Validator influence is determined by a combination of:
  - Stake weight
  - Reputation score
  - Computational contribution
  - Historical reliability

---

## Continuous Threat Monitoring
- The Threat Detection Layer continuously analyzes:
  - Unusual block times
  - Vote inconsistencies
  - Sudden validator concentration
  - Double-spend attempts
  - DDoS patterns
- Detection methods may include:
  - Statistical anomaly detection
  - ML-based node behavior profiling
  - Real-time network telemetry

---

## Trigger Event
- If abnormal activity is detected:
  - A **Threat Severity Score** is calculated.
  - If the score crosses a threshold, **Adaptive Consensus Shifting** is triggered.

---

## Consensus Shape-Shift
- The protocol switches to a more secure mode, for example:
  - Multi-Weight PoS → Hybrid PoS + BFT
- Validators’ weights are rebalanced:
  - Suspicious nodes lose weight.
  - Trusted nodes gain temporary influence.
- If the attack vector suggests, a temporary Proof-of-Work shield can be applied for block finalization.

---

## Survivability Mode
- Block times may slow down.
- Confirmation requirements may increase.
- Priority is on **Integrity over throughput**.

---

## Post-Threat Recovery
- After stability returns:
  - The system gradually reverts validator weights to normal.
  - Default consensus resumes.
  - Threat report is logged on-chain for audit and future model training.



