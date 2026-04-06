# Autonomous Research Agent for Drone Flight Control

**How I used Claude Code as an autonomous research agent to improve a neural flight controller from 40% to 73.3% hit rate — while I worked on other things.**

[![Claude Code](https://img.shields.io/badge/Agent-Claude%20Code-blueviolet.svg)](https://claude.ai/claude-code)
[![Results](https://img.shields.io/badge/Hit%20Rate-73.3%25-brightgreen.svg)](#results)
[![Hardware](https://img.shields.io/badge/GPU-RTX%204000%20Ada-76b900.svg)](#hardware)

---

## The Problem

I was training a neural network to fly a drone — specifically, to intercept a fast-moving aerial target using only a camera and gyroscope. No GPS, no range sensor. Just bounding box detections from YOLO and angular rates from an IMU.

After weeks of manual experimentation with PPO (reinforcement learning from scratch), I had plateaued at **40.5% hit rate**. The classical expert controller (proportional navigation) achieved 83.3%. I was stuck, and I had other things to work on.

So I built a system that let me deploy a baseline, provide research guidelines, and **let an LLM agent autonomously run experiments 24/7** — hypothesizing, implementing, training, evaluating, and iterating without any human in the loop.

**The result: 73.3% hit rate. A +33 percentage point improvement, achieved autonomously.**

---

## How It Works

<p align="center">
  <img src="assets/loop_diagram.png" alt="Autonomous Research Loop" width="100%">
</p>

The system runs a continuous loop where each iteration is a complete scientific experiment:

### 1. Read Context & History
The agent reads a persistent `research_context.md` file that accumulates all prior findings — what worked, what failed, and why. It also reads `experiment_history.tsv` for quantitative results across all prior experiments. This gives each new agent instance the full institutional memory of the research program.

### 2. Hypothesize & Implement
Based on the accumulated context and a `CLAUDE.md` file specifying constraints and the search space, the agent forms a hypothesis ("2-layer LSTM should help because the single-layer can't capture temporal dependencies at this complexity") and implements exactly **one change** to isolate its effect.

### 3. Deploy to GPU
The agent SSHs into a remote GPU server, SCPs the modified source files, rebuilds the C physics simulator, and runs a verification test suite (40 physics + projection tests) to ensure nothing broke.

### 4. Train
Training runs on the remote GPU with a fixed budget — either 500K PPO steps (~8 min) or 60 DAgger rounds (~45 min). Fixed budgets ensure fair comparison across experiments and enable ~7 experiments per hour.

### 5. Extract Metrics & Compare
The agent extracts metrics from TensorBoard events (or training logs), computes the primary metric (hit rate on randomized evaluation scenarios), and compares against the current best.

### 6. KEEP or DISCARD
Binary decision. If the hit rate improved: **KEEP** — merge the change into the main branch (git as ratchet). If not: **DISCARD** — revert all changes. No fuzzy "looks promising." Numbers only.

### 7. Update Context & Repeat Forever
The agent writes what it learned to the persistent context file, logs the result to `experiment_history.tsv`, and immediately starts the next experiment. **No pause. No human approval needed.**

---

## Results

<p align="center">
  <img src="assets/progression.png" alt="Progression from Human to Autonomous" width="100%">
</p>

### The Numbers

| Phase | Approach | Best Hit Rate | Who Did the Work |
|---|---|---|---|
| Manual PPO | RL from scratch, hand-tuned | 40.5% | Me, over weeks |
| DAgger Baseline | Imitation learning, initial setup | 40.0% | Me, 1 day setup |
| Autonomous Research | Agent-driven iteration | **73.3%** | Claude Code, autonomously |
| Expert Ceiling | Classical proportional navigation | 83.3% | Optimal control theory |

**The agent closed 69% of the gap between the baseline and the expert ceiling — autonomously.**

### Key Breakthroughs Found by the Agent

The agent discovered several improvements that I had not tried or had overlooked:

1. **2-layer LSTM** (+3.3pp) — architecture depth matters for temporal reasoning over bounding box sequences
2. **Cosine learning rate annealing** — smoother convergence than flat LR schedule
3. **Dataset buffer size** (maxlen=25 vs 5) — the single biggest lever, letting the model train on more diverse past experience
4. **3-phase learning rate schedule** (7e-5 → 1.5e-5 → 5e-6) — progressive refinement across 200 DAgger rounds
5. **Loss weighting** [15, 15, 0.1, 1] — the agent discovered that roll/pitch matter far more than yaw for interception

### What the Agent Learned NOT to Do

Equally valuable — the agent saved me time by ruling out dead ends:

- Episode boundary splitting in BPTT (29.3% — catastrophic regression)
- Inter-layer dropout between LSTM layers (40.7% — hurt performance)
- Zero entropy coefficient (premature convergence, policy gets stuck)
- Quadratic terminal reward bonus (made optimization landscape harder)
- Aggressive curriculum transitions (caused catastrophic forgetting)

---

## Parallel Execution on Dual GPUs

<p align="center">
  <img src="assets/timeline.png" alt="Dual GPU Timeline" width="100%">
</p>

A single orchestrator manages two independent GPU servers running experiments in parallel:

- **GPU 1** (Hetzner cloud, NVIDIA RTX 4000 Ada, 20GB VRAM) — primary research track
- **GPU 2** (university lab server) — parallel exploration track

Each GPU runs a full experiment pipeline independently. The orchestrator:
- Assigns experiments to idle GPU slots
- Creates isolated git worktrees for each experiment (no code contamination)
- Monitors training progress via SSH (detects stalls, kills hung workers)
- Collects results and updates the shared experiment history
- Merges improvements into the main branch

**6 complete experiments in 4 hours** with full parallel utilization. At the PPO experiment rate (~7/hour), this system can run **~170 experiments overnight**.

---

## The Infrastructure

### Orchestrator (`auto_research_v5.sh`)

The orchestrator evolved through 5 iterations:

| Version | Key Innovation |
|---|---|
| v1 | Basic single-GPU loop |
| v2 | Persistent `research_context.md` for accumulated knowledge |
| v3 | Adaptive modes: incremental → pivot → exploration (based on failure count) |
| v4 | macOS compatibility, stall detection, better error handling |
| v5 | **Dual GPU orchestration**, per-GPU slot tracking, code contamination prevention |

### Adaptive Research Modes

The system adapts its strategy based on consecutive failures:

- **Incremental mode** (0-2 failures): Small changes building on current best
- **Pivot mode** (3-4 failures): Switch to a completely different approach category
- **Exploration mode** (5+ failures): Use web search to find novel techniques from papers

### Constraint Enforcement

Hard constraints prevent the agent from inflating results:

```
Physics constants (mass, inertia, thrust)     → IMMUTABLE
Domain randomization ranges                    → Can only INCREASE
Scenario difficulty (target speed, distance)   → FIXED
Camera FOV limits                              → FIXED
Hit detection radius                           → FIXED (0.3m)
```

Any constraint violation → automatic **REJECT**, even if hit rate improved.

### Communication via File System

Rather than complex APIs, the system uses files for agent communication:

- `CLAUDE.md` → Instructions, constraints, search space, recent history (written by orchestrator, read by agent)
- `experiment_result.json` → Results (written by agent, read by orchestrator)
- `research_context.md` → Accumulated knowledge (append-only, persists across all agents)
- `experiment_history.tsv` → Quantitative log of every experiment ever run

Each Claude Code instance starts fresh but inherits the full research history through these files.

---

## What Makes This Work

### 1. Git as Ratchet
Only improvements are merged. The codebase can only get better. Every commit message is an experiment summary, so `git log` reads like a research journal.

### 2. Fixed-Budget Experiments
Every experiment gets exactly the same compute budget. No "let it run longer to see if it catches up." This enables fair comparison and predictable throughput.

### 3. Persistent Research Memory
Each agent writes what it learned — both successes and failures — to a shared context file. The next agent reads this and builds on it. Knowledge compounds across agents.

### 4. One Change Per Experiment
Strict isolation. If something improves, you know exactly what caused it. If it regresses, you know exactly what to revert.

### 5. The "Never Stop" Philosophy
Inspired by [Karpathy's autoresearch](https://x.com/karpathy/status/1886192184808149383) concept. The agent doesn't ask for permission, doesn't wait for review, doesn't pause between experiments. It just keeps going. 7 experiments per hour, 170 overnight, indefinitely.

---

## Hardware

| Component | Spec |
|---|---|
| GPU 1 (training) | NVIDIA RTX 4000 Ada, 20GB VRAM |
| GPU 2 (training) | University lab GPU server |
| Orchestrator | MacBook Pro (Apple Silicon) |
| Physics Sim | Custom C, ~200K steps/sec per env |
| Training Time | ~8 min/experiment (PPO) or ~45 min (DAgger) |
| Experiment Rate | ~7/hour (PPO) or ~1.3/hour (DAgger) |

---

## Why This Matters

### For me personally
I deployed the baseline on a Sunday evening and went to bed. By Monday morning, the agent had run multiple experiments and found a +3.3pp improvement. Over the following days, it continued iterating — finding the dataset buffer size insight, the learning rate schedule, the loss weighting — while I worked on hardware integration, paper writing, and other projects.

**Autonomous research doesn't replace the researcher. It multiplies them.** I still designed the system, chose the approach, set the constraints, and validated the results. But the tedious cycle of "change one thing → train → wait → evaluate → repeat" was fully automated.

### For the field
This is a proof of concept that LLM agents can perform meaningful scientific iteration — not just code generation, but the full loop of hypothesis formation, implementation, execution, analysis, and knowledge accumulation. The agent discovered non-obvious insights (loss weighting, buffer size effects) that I hadn't considered.

The key ingredients are:
1. **A well-defined metric** (hit rate on randomized evaluation)
2. **Clear constraints** (what can and cannot be changed)
3. **Fast feedback** (8-45 min per experiment)
4. **Persistent memory** (research context that compounds)
5. **A ratchet mechanism** (git merge only on improvement)

---

## Related Work

- [Karpathy's Autoresearch](https://x.com/karpathy/status/1886192184808149383) — the philosophical inspiration for fixed-budget, never-stop autonomous research
- [The AI Scientist](https://arxiv.org/abs/2408.06292) — Sakana AI's system for autonomous paper writing and review
- [FunSearch](https://www.nature.com/articles/s41586-023-06924-6) — DeepMind's LLM-driven mathematical discovery
- [Claude Code](https://claude.ai/claude-code) — the LLM agent used as the research executor

---

## Repository Structure

This repository documents the autonomous research methodology. The flight controller itself is in a [separate repository](https://github.com/J3ykob/rl-drone-flight-controller).

---

## License

MIT — the methodology is open. The trained models and flight controller code are proprietary (see the [flight controller repo](https://github.com/J3ykob/rl-drone-flight-controller)).
