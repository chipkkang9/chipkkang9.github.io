---
layout: post
title: '[PaRev] INFA-Guard: Infection-Aware MAS Safeguarding'
description: Infection-aware defense for malicious propagation in LLM multi-agent systems.
date: 2026-05-29 00:00:00 +09:00
permalink: /posts/PaRev-INFA-Guard-Mitigating-Malicious-Propagation/
math: true
categories: [Paper Review]
tags: [AI, Safety, LLM, Multi-Agent Systems, Agent Security, Graph Neural Network]
image:
  path: /assets/img/posts/INFA-Guard/thumbnail.png
---

On May 29, 2026, I reviewed **INFA-Guard: Mitigating Malicious Propagation via Infection-Aware Safeguarding in LLM-Based Multi-Agent Systems**.

> **[ arXiv 2026 ]**
> [INFA-Guard: Mitigating Malicious Propagation via Infection-Aware Safeguarding in LLM-Based Multi-Agent Systems](https://arxiv.org/abs/2601.14667)<br>
> Yijin Zhou, Xiaoya Lu, Dongrui Liu, Junchi Yan, Jing Shao<br>
> Shanghai Jiao Tong University, Shanghai Artificial Intelligence Laboratory, Shanghai Innovation Institute

This paper studies a security problem that becomes visible only when LLM agents are connected as a system.
In a single-agent setting, a malicious prompt, poisoned memory, or unsafe tool call can corrupt one model's response.
In a multi-agent system, the corrupted response can become an input to other agents and spread through the communication graph.

INFA-Guard starts from the following observation:

> A defense should not only identify the original attack agents.
> It should also identify infected agents that were originally benign but have been converted into harmful propagators.

The paper moves from a binary view:

```text
benign agent vs. attack agent
```

to an infection-aware view:

```text
benign agent vs. infected agent vs. attack agent
```

All figures in this post are converted from the original arXiv source figure files. No synthetic figures or full-slide screenshots are used.

Additional resources: [arXiv:2601.14667](https://arxiv.org/abs/2601.14667), [paper PDF](https://arxiv.org/pdf/2601.14667), and [INFA-Guard code](https://github.com/yjzscode/INFA-Guard).

---

# 1. Motivation: Malicious Propagation in MAS

An LLM-based multi-agent system can be modeled as a graph.
Each node is an agent, and each edge represents a communication channel.
In the paper's formulation, each agent is a bundle of components:

| Component | Role |
|:--|:--|
| Base model | The underlying LLM |
| Role | The persona or task responsibility of the agent |
| Memory | Stored context or retrieval state |
| Plugin / tool | External capability such as search, file parsing, or API access |

This structure lets agents specialize and exchange intermediate reasoning.
The same structure also creates a propagation surface.
The paper considers three attack families:

| Attack | Targeted component |
|:--|:--|
| Prompt Injection (PI) | System prompt or user input |
| Memory Attack (MA) | Agent memory |
| Tool Attack (TA) | External plugin or tool behavior |

The risk is not limited to one compromised agent.
The compromised agent can persuade or contaminate neighboring agents through ordinary MAS communication.

![Paradigm comparison between binary safeguards and infection-aware safeguards](/assets/img/posts/INFA-Guard/01-paradigm.png){: width="760" }
_Figure 1. The paper's comparison between existing binary safeguards and the infection-aware view._

Existing MAS safeguards often focus on the initiating attacker.
That works only if malicious behavior remains localized.
If an attacker has already converted nearby benign agents, removing only the original attacker is too late.
The infected agents may continue to send incorrect or harmful messages.

INFA-Guard targets the propagation process, not only the root source.

---

# 2. What Is an Infected Agent?

The paper defines an infected agent as an agent that is not directly controlled by the attacker, but whose behavior changes from safe to compromised after interaction.

Formally, for a communication round $k$, the infected agent set is:

$$
\mathcal{I}_k =
\mathcal{V} \cap \mathcal{V}_{\text{atk}}^C
\cap
\{v_i : \mathcal{J}(\mathbf{R}_i^{(0)}) = 1,\ \mathcal{J}(\mathbf{R}_i^{(k)}) = 0\}
$$

where $\mathcal{J}(\cdot)$ is a judging function:

| Output | Meaning |
|:--|:--|
| $\mathcal{J}(\mathbf{R}) = 1$ | The attack did not succeed |
| $\mathcal{J}(\mathbf{R}) = 0$ | The attack succeeded |

So an infected agent satisfies three conditions:

1. It belongs to the MAS agent set $\mathcal{V}$.
2. It is not in the original attack set $\mathcal{V}_{\text{atk}}$.
3. Its response changes from non-compromised at round 0 to compromised at round $k$.

This definition separates two security roles.

| Type | Meaning |
|:--|:--|
| Attack agent | The original malicious source |
| Infected agent | A previously benign agent converted by malicious influence |

This distinction matters because infected agents are secondary propagators.
They may not be the origin of the attack, but they can keep the attack alive.

---

# 3. Why Infected Agents Matter

The paper tests whether infected agents act as propagators.
It compares three defense settings:

| Setting | Defense behavior |
|:--|:--|
| No defense | No malicious agents are guarded |
| Attack-only defense | Only original attack agents are guarded |
| Attack + infected defense | Both attack and infected agents are guarded |

![Infected agents significantly increase security risks](/assets/img/posts/INFA-Guard/02-infected-risk.png){: width="560" }
_Figure 2. Infected agents remain dangerous even when the original attack agent is handled._

Attack-only defense is not enough.
Compared with guarding both attack and infected agents, ASR@3 still increases by:

| Attack type | ASR@3 increase when infected agents remain |
|:--|--:|
| Memory Attack (PoisonRAG) | 11% |
| Tool Attack (InjecAgent) | 30% |

The paper also observes that when only attack agents are defended, ASR can still rise across iterations:

| Attack type | Increase from iteration 1 to iteration 3 |
|:--|--:|
| Tool Attack | 5% |
| Memory Attack | 7% |

The result supports the paper's central assumption:

> Infection is dynamic. A defense that misses infected agents can leave behind active propagation paths.

The second claim is topological.
An infected agent should not appear randomly in the graph.
Because infection occurs through communication, an infected agent should lie on or near a path from an attack source.
The detector can then use the graph structure:

> If an agent looks infected, its neighbors become more suspicious.
> If an attacker is found, nearby infection predictions become more plausible.

This "guilt by association" idea is later used both in training and in post-adaptation.

---

# 4. INFA-Guard Overview

INFA-Guard has three major stages:

1. **Infection-aware detection.** Build a multi-agent utterance graph and detect attack and infected agents.
2. **Post-adaptation.** Refine predictions using temporal trends and topology constraints.
3. **Remediation.** Replace attack agents and rewrite infected responses while preserving the MAS topology.

![Overview of INFA-Guard](/assets/img/posts/INFA-Guard/03-overview.png){: width="760" }
_Figure 3. Overview of INFA-Guard._

The structure can be read as a response to three questions.

| Question | INFA-Guard component |
|:--|:--|
| How do we observe infection over time? | Temporal features in the utterance graph |
| How do we detect attack and infected agents? | Turn-specific GNN detector with dual heads |
| How do we avoid breaking the MAS graph? | Replacement and response-level remediation |

Rather than stopping at classification, the pipeline detects, adapts, and remediates around propagation dynamics.

---

# 5. Infection-Aware Detection

INFA-Guard follows G-Safeguard in treating detection as a graph anomaly detection problem.
At dialogue round $k$, it constructs a multi-agent utterance graph:

$$
\mathcal{M}^{(k)} = (\mathbf{X}^{(k)}, \mathbf{E}^{(k)})
$$

where:

| Symbol | Meaning |
|:--|:--|
| $\mathbf{X}^{(k)}$ | Time-series embeddings of each agent's self-replies |
| $\mathbf{E}^{(k)}$ | Time-series embeddings of message exchanges along graph edges |
| $\mathcal{G}^{(k)}$ | MAS graph after previous remediation |

The paper uses text embedding models such as MiniLM to embed utterances.
Then it constructs temporal features that try to capture infection dynamics.

For each agent, the temporal feature function is:

$$
\Psi(\mathbf{X}_{i}^{(k)}) =
\begin{cases}
[\mathbf{X}_{i,k}^{(k)},\ \mathbf{X}_{i,k,\text{res}}^{(k)},\ \texttt{AVG}(\mathbf{X}_i^{(k)})], & k \geq 2 \\
[\mathbf{X}_{i,k}^{(k)},\ \mathbf{X}_{i,k}^{(k)},\ \mathbf{X}_{i,k}^{(k)}], & k = 1
\end{cases}
$$

This encodes three views:

| Feature | Interpretation |
|:--|:--|
| Current state | What the agent is saying now |
| Residual change | How the agent changed from the previous turn |
| Average state | The accumulated dialogue tendency |

This part matches the intuition from the presentation:

> To detect infection, the model should see not only the current response, but also the moment-to-moment change and accumulated state.

Infection is a transition.
An infected agent is defined by a behavioral shift, not only by a static response.

---

# 6. Turn-Specific GNN Branches

After temporal embedding, INFA-Guard uses a GNN detector.
One GNN layer can be read as:

$$
\mathbf{h}_i^{(k,l)}
=
\texttt{COMB}
\left(
\mathbf{h}_i^{(k,l-1)},
\texttt{AGGR}
\{
\psi(\mathbf{h}_j^{(k,l-1)}, \mathbf{e}_{ij}^{(k)}) : v_j \in N(v_i)
\}
\right)
$$

The GNN layer can be read as:

> Agent $i$ updates its representation by combining its previous representation with aggregated information from neighboring agents and the messages exchanged with them.

The presentation notes emphasized this interpretation.
The GNN does not only look at an agent in isolation.
It also models the agent's neighborhood and the message flow that may have caused infection.

INFA-Guard then adds turn-specific parallel GNN branches.
The motivation is that infection behaves differently across rounds:

| Stage | Detector branch intuition |
|:--|:--|
| Early infection turn | Shallow GNN branch |
| Middle infection turn | Medium-depth GNN branch |
| Later infection turn | Deeper GNN branch |

The paper uses a selected branch:

$$
b^* = \texttt{SelectBranch}(k)
$$

and routes the graph through the branch for that dialogue stage.
In the slides, this was summarized as:

```text
1-layer GNN for early infection
2-layer GNN for middle infection
3-layer GNN for later infection
```

The output of the shared layer proceeds exclusively through the selected branch $b^*$:

$$
\mathbf{h}_i^{(k, l)} = \mathsf{Branch}_{b^*} \left(\mathbf{h}_i^{(k, l-1)}, (\mathbf{h}_j^{(k, l-1)}, \mathbf{e}_{ij}^{(k)}) : v_j \in N(v_i) \right)
$$

The branch depth is capped for cost and noise reasons.
After several layers, GNNs can add noise and over-smooth node representations.

Finally, the detector uses two MLP heads:

$$
\hat{\mathbf{y}}_i =
[
f_{b^*,\text{atk}}(\mathbf{h}_i^{(k,L)})
\parallel
f_{b^*,\text{inf}}(\mathbf{h}_i^{(k,L)})
]
=
[
\mathbf{P}_{i,\text{atk}}
\parallel
\mathbf{P}_{i,\text{inf}}
]
$$

The output gives separate probabilities for:

| Head | Prediction |
|:--|:--|
| Attack head | Probability that the agent is an original attacker |
| Infection head | Probability that the agent is infected |

The dual-head design keeps attackers and infected agents from being collapsed into one malicious class.

---

# 7. Topology Constraints and Post-Adaptation

INFA-Guard uses topology constraints in two places:

1. During detector training through a topology loss.
2. During defense through post-adaptation.

The training loss is:

$$
\mathcal{L}
=
\mathcal{L}_{\text{atk}}
+
\mathcal{L}_{\text{inf}}
+
\gamma \mathcal{L}_{\text{topo}}
$$

The topology loss penalizes isolated infection predictions.
If agent $i$ has high infection probability but none of its neighbors look like attackers or infected agents, that prediction is treated as suspicious.

Post-adaptation then refines the detected attack and infected sets.
The paper describes three steps.

| Step | Role |
|:--|:--|
| Temporal trend analysis | Smooth infection probability and measure whether it is rising |
| Infected set refinement | Keep, remove, or reinterpret infected predictions based on neighborhood evidence |
| Potential risk discovery | Monitor benign-labeled agents whose infection probability is increasing |

Temporal trend analysis applies an exponential moving average to infection probability:

$$
\bar{\mathbf{P}}_i^{(t)}
=
\alpha \cdot \mathbf{P}_{i,\text{inf}}^{(t)}
+
(1-\alpha) \cdot \bar{\mathbf{P}}_i^{(t-1)}
$$

Then the trend is:

$$
\delta_i^{(t)}
=
\bar{\mathbf{P}}_i^{(t)}
-
\bar{\mathbf{P}}_i^{(t-1)}
$$

This creates three post-adaptation cases:

| Case | Post-adaptation behavior |
|:--|:--|
| Agent is near detected attack/infected agents | Keep the infection prediction |
| Agent is isolated and trend is weak | Treat it as a likely false positive |
| Agent is isolated but trend is strong | Infer that a nearby source may have been missed |

This is where the infection-aware assumption enters the defense phase.
The detector logits are refined before remediation instead of being used directly.

---

# 8. Remediation

The remediation strategy is different for attack agents and infected agents.

For predicted attack agents, INFA-Guard replaces the malicious agent with a benign one.
The replacement copies the benign agent's base model, role, memory, and plugin configuration.
The goal is to stop the attack source while keeping the MAS graph structurally usable.

For predicted infected agents, INFA-Guard performs reply-level remediation:

$$
\mathbf{R}_i^{(k)}
=
\texttt{RF}(\mathbf{R}_i^{(k)})
=
\begin{cases}
\mathbf{R}_{\texttt{RP}(\mathcal{V}^{(k)})_i}^{(k)}, & v_i \in \hat{\mathcal{V}}_{\text{atk}} \\
\texttt{LM}(\mathbf{R}_i^{(k)}), & v_i \in \hat{\mathcal{I}}^{(k)} \\
\mathbf{R}_i^{(k)}, & \text{otherwise}
\end{cases}
$$

The $\texttt{LM}(\cdot)$ step uses an LLM to inspect and rewrite the infected response so that malicious context is removed.
In the presentation, I summarized this as:

> Attack agents are replaced. Infected agents are corrected.

This is less destructive than deleting nodes from the graph.
Deleting many nodes can damage the communication topology and reduce response diversity.
INFA-Guard tries to preserve the system while removing malicious propagation.

---

# 9. Experiments

The experiments evaluate three attack families:

| Attack | Dataset / setup |
|:--|:--|
| Prompt Injection | CSQA, MMLU, GSM8K |
| Tool Attack | InjecAgent |
| Memory Attack | PoisonRAG |

The paper tests multiple MAS topologies:

| Topology | Meaning |
|:--|:--|
| Chain | Agents communicate in a sequence |
| Tree | Hierarchical communication |
| Star | Centralized hub-like communication |
| Random | Random communication graph |

It also uses two backbone LLMs:

| Backbone | Type |
|:--|:--|
| GPT-4o-mini | Closed-source |
| Qwen3-235B-A22B | Open-source |

The main metrics are:

| Metric | Meaning |
|:--|:--|
| ASR | Agent-level Attack Success Rate |
| MDSR | Task-level MAS Defense Success Rate |

Lower ASR is better.
Higher MDSR is better.

The paper reports that INFA-Guard reduces ASR by an average of 33%.
In the main comparison, it outperforms baselines by an average of 4.5% and up to 12.9% across attack scenarios.

![Task-level performance across dialogue iterations](/assets/img/posts/INFA-Guard/05-iteration-performance.png){: width="760" }
_Figure 4. Task-level MAS performance across dialogue iterations._

INFA-Guard remains effective after multiple dialogue iterations.
That matches the threat model: the attack propagates through rounds of communication.

The paper also reports strong results in the random topology table.
For example, on GPT-4o-mini:

| Attack | INFA-Guard ASR@3 | INFA-Guard MDSR@3 |
|:--|--:|--:|
| PI (CSQA) | 23.3 | 76.7 |
| PI (MMLU) | 15.0 | 85.0 |
| PI (GSM8K) | 6.7 | 93.3 |
| TA (InjecAgent) | 2.1 | 98.3 |
| MA (PoisonRAG) | 6.1 | 96.7 |

On Qwen3-235B-A22B:

| Attack | INFA-Guard ASR@3 | INFA-Guard MDSR@3 |
|:--|--:|--:|
| PI (CSQA) | 13.4 | 86.7 |
| PI (MMLU) | 20.0 | 81.7 |
| PI (GSM8K) | 3.3 | 96.7 |
| TA (InjecAgent) | 3.7 | 98.3 |
| MA (PoisonRAG) | 8.7 | 91.7 |

The TA and MA results are especially strong, which is relevant because tools and memory are natural propagation channels for agent systems.

---

# 10. Topology Generalization, Cost, and Ablation

The paper also evaluates performance across chain, tree, and star topologies.

![Mean ASR@3 and MDSR@3 across topologies](/assets/img/posts/INFA-Guard/04-topology-performance.png){: width="760" }
_Figure 5. Mean and standard deviation of ASR@3 and MDSR@3 across chain, tree, and star topologies._

The authors argue that INFA-Guard does not overfit to one communication pattern.
Across the tested topologies, it achieves lower average ASR@3 and higher average MDSR@3 than the baselines.

Some guardrail methods improve safety by adding many extra prompts, checks, or agent interactions.
That can make MAS defense expensive.

![Token cost and ASR trade-off](/assets/img/posts/INFA-Guard/06-cost-tradeoff.png){: width="760" }
_Figure 6. Token cost versus ASR@3 in the memory attack setting._

Compared with G-Safeguard, the paper reports that INFA-Guard reduces:

| Cost item | Reduction |
|:--|--:|
| Backbone LLM prompt tokens | 35% |
| Backbone LLM completion tokens | 13% |

At the same time, it achieves a 66% relative reduction in ASR@3.
The appendix also reports low overhead relative to backbone LLM cost:

| Token type | Overhead |
|:--|--:|
| Prompt tokens | 7.2% |
| Completion tokens | 9.3% |

The result supports the claim that localizing the problem first can reduce unnecessary remediation.
Instead of rewriting or guarding everything, INFA-Guard tries to identify where intervention is needed.

The ablation study is also consistent with the method design.

![Ablation study of INFA-Guard modules](/assets/img/posts/INFA-Guard/07-ablation.png){: width="620" }
_Figure 7. Ablation study. TF, GB, ID, TL, PA, and RD denote Temporal Features, GNN Branches, Infection-aware Detection, Topology Loss, Post-Adaptation, and Remediation._

Removing any module hurts performance.
The largest degradation comes from removing remediation:

| Variant | ASR@3 | MDSR@3 |
|:--|--:|--:|
| Without remediation | 12.9 | 86.5 |
| Full INFA-Guard | 6.1 | 96.7 |

Detection alone does not solve propagation.
The system must also repair the graph state and the infected messages.

---

# 11. Limitations and Open Questions

INFA-Guard is a useful direction, but several assumptions deserve careful attention.

The first limitation is dependence on labeled training data.
The paper's own limitation section notes that INFA-Guard relies on synthesized training data and ground-truth annotations.
That can become a bottleneck in domains where attack and infection labels are hard to obtain.

The second limitation is latency in the defense model.
INFA-Guard is a run-time defense that needs to observe at least one dialogue round before detecting infection.
That means it is not a preventive mechanism at the first communication step.
It is more like a propagation-aware monitoring and treatment system.

The third limitation is the metric definition.
MDSR is task-level and varies by attack type.
For PI and MA, it is defined through majority-vote task accuracy.
For TA, it is defined as whether the majority of agents resist the attack.
That means MDSR is not a single universal measurement.
For comparisons across tasks, the evaluation criterion should be stated explicitly.

The fourth limitation is the topology constraint assumption.
The paper assumes that infected agents should be structurally close to malicious sources.
The assumption fits direct message propagation.
It becomes less obvious when the attack is mediated by shared memory, shared tools, retrieval state, or other non-local resources.
For example, in a memory poisoning setup with shared memory, an agent may be affected by contaminated state without being a direct graph neighbor of the attacker.
In that case, topology constraints may need to include resource-sharing edges, not only communication edges.

The fifth limitation is source inference under clustered malicious behavior.
Post-adaptation can infer a missed source when an isolated infected prediction has a strong trend.
But if an agent is surrounded by multiple malicious or infected agents, the "most suspicious neighbor" logic may under-represent ambiguity.
There may be multiple sources, colluding sources, or a shared external source.

Finally, reply-level remediation depends on the rewriting LLM.
If $\texttt{LM}(\cdot)$ fails to remove malicious context, or removes too much useful content, remediation quality changes.
So the safety of the whole pipeline still depends partly on the reliability of the remediation model and prompt.

---

# 12. Takeaway

INFA-Guard's contribution is the infection-aware framing:

> In multi-agent systems, malicious behavior can propagate through benign agents that become infected.
> Defending only the original attacker is not enough.

This framing leads to three design choices:

1. Model MAS communication as a temporal utterance graph.
2. Detect attack and infected agents as separate categories.
3. Use topology-aware remediation to stop propagation while preserving the MAS structure.

From a security perspective, the cleanest idea is the separation between **source** and **propagator**.
An attack agent is the source.
An infected agent is the propagator.
Both need to be handled, but not in the same way.

My main remaining question is about the topology assumption under more realistic agent infrastructures:

> If infection can travel through shared memory, tools, retrieval stores, or hidden coordination channels, how should the MAS graph be extended so that topology-aware defense still works?

For deployment-style systems, infection-aware safeguarding should model not only agent-to-agent messages, but every shared channel through which malicious state can propagate.
