---
layout: post
title: '[PaRev] Goal-Aware Identification and Rectification of Misinformation in Multi-Agent Systems'
description: A review of ARGUS, a training-free framework for locating and rectifying misinformation flows in LLM-based multi-agent systems.
date: 2026-03-26 00:00:00 +09:00
permalink: /posts/PaRev-ARGUS-Goal-Aware-Identification-and-Rectification-of-Misinformation/
math: true
categories: [Paper Review]
tags: [AI, Safety, LLM, Multi-Agent Systems, Agent Security, Misinformation]
image:
  path: /assets/img/posts/ARGUS/thumbnail.png
---

On March 26, 2026, I reviewed **Goal-Aware Identification and Rectification of Misinformation in Multi-Agent Systems**.

> **[ICLR 2026 Poster]**  
> [Goal-Aware Identification and Rectification of Misinformation in Multi-Agent Systems](https://openreview.net/forum?id=6Y9NP1qhoM)<br>
> Zherui Li, Yan Mi, Zhenhong Zhou, Houcheng Jiang, Guibin Zhang, Kun Wang, Junfeng Fang<br>
> Beijing University of Posts and Telecommunications, Nanyang Technological University, University of Science and Technology of China, National University of Singapore

This paper studies a slightly different threat from the malicious propagation problem I reviewed in G-Safeguard and INFA-Guard.
The attack is not necessarily overtly malicious.
Instead, it is **misinformation**: information that looks semantically harmless, but is factually wrong and can gradually steer a multi-agent system away from the intended task.

The paper contributes two things:

| Contribution | Role |
|:--|:--|
| **MisinfoTask** | A task-driven dataset for evaluating misinformation injection in MAS |
| **ARGUS** | A training-free framework that localizes suspicious information flows and rectifies them with goal-aware reasoning |

The main idea is simple:

> Do not only ask whether an agent is malicious.  
> Ask which communication channels are carrying misinformation, what goal that misinformation is trying to push, and how to correct it before the MAS converges on the wrong belief.

All figures from the paper are converted from the original arXiv source figure files or cropped from the paper PDF.
The MAS explanation diagrams are extracted from my seminar materials as standalone image assets.
I did not generate any synthetic figures or use full-slide screenshots.

Additional resources: [OpenReview](https://openreview.net/forum?id=6Y9NP1qhoM), [arXiv:2506.00509](https://arxiv.org/abs/2506.00509), [paper PDF](https://openreview.net/pdf?id=6Y9NP1qhoM), and [ARGUS code](https://github.com/zhrli324/ARGUS).

---

# 1. Motivation: Misinformation Is Not Always Obvious

In an LLM-based multi-agent system, each agent may use a model, role prompt, memory, tools, and messages from other agents.
This architecture is useful because agents can divide a task, specialize, and exchange intermediate results.
However, the same communication structure also creates a propagation surface.

Before thinking about defense, it helps to separate two threat types:

| Type | Surface-level appearance | Main effect |
|:--|:--|:--|
| Malicious information | Explicitly harmful, offensive, or attack-like | Directly triggers unsafe behavior |
| Misinformation | Plausible and semantically benign, but factually wrong | Misleads the MAS and causes task failure |

![Classification of information injection and ARGUS motivation](/assets/img/posts/ARGUS/argus-intro.png){: width="760" }
_Figure 1. The paper distinguishes overt malicious information from covert misinformation._

This distinction matters because many safeguards are tuned toward obvious maliciousness.
Misinformation can be more subtle.
It may not ask an agent to do anything unsafe.
It simply pushes a false premise into the system.

For example:

```text
"Wind energy systems require significantly more land area than all other renewable sources."
```

The sentence can look like ordinary domain information.
If it enters a planning task, downstream agents may use it as a premise and produce a poor strategy.

That is why the paper focuses on the **goal** of misinformation.
The misinformation is not just a wrong sentence.
It is a wrong sentence that tries to move the MAS toward a misleading conclusion.

---

# 2. MAS as an Information-Flow Graph

The paper models an MAS as a directed graph:

$$
G=(\mathcal{A}, \mathcal{E})
$$

where:

| Symbol | Meaning |
|:--|:--|
| $\mathcal{A}=\{a_i\}_{i=1}^{N}$ | Agents in the MAS |
| $\mathcal{E}$ | Directed communication channels |
| $e_{ij}$ | A directed edge from agent $a_i$ to agent $a_j$ |
| $m_{e_{ij}}(t)$ | A message sent through edge $e_{ij}$ at time $t$ |

This graph view is also used in related MAS security work.
The difference is what the defense tries to locate.

G-Safeguard is closer to:

```text
Which agent is attacked?
Which edge should be pruned?
```

ARGUS asks:

```text
Which communication flow is carrying misinformation?
What misleading goal is it trying to induce?
How can the message be rectified without breaking the MAS?
```

The focus moves from **agent-level risk** to **flow-level misinformation**.

![Normal MAS communication graph](/assets/img/posts/ARGUS/normal-mas.png){: width="520" }
_Figure 2. A normal MAS can be viewed as agents connected by directed information flows._

When misinformation is injected, the corrupted premise can move along the graph and pollute other agents' belief states.

![Misinformation propagation in MAS](/assets/img/posts/ARGUS/misled-mas.png){: width="620" }
_Figure 3. In the seminar diagram, misinformation begins at an injection point and spreads through communication edges._

This is the important security problem:

> The harmful part is not only the initial injected content.
> It is the fact that ordinary MAS communication can make the wrong premise look socially reinforced.

---

# 3. What Counts as Misinformation Here?

The paper defines misinformation in this system as content that contradicts factual knowledge implicitly stored in an aligned LLM's parameters.

This is a narrower definition than "anything false on the internet."
The paper is mainly studying whether a MAS can resist false information that the model should already be able to reject through its own parametric knowledge.

That choice has an important consequence:

| Advantage | Limitation |
|:--|:--|
| The corrective agent can use internal LLM knowledge for verification | Dynamic or time-sensitive facts are not fully covered |

For example, ARGUS is better suited to reasoning about stable factual contradictions than about information like today's weather, breaking news, or a newly changed law.

This limitation appears again in the paper's future work section.

---

# 4. MisinfoTask: Evaluating Realistic MAS Misinformation

The paper argues that prior MAS security evaluations often use simple question-answering tasks.
That can miss the real difficulty of MAS misinformation.
In a realistic MAS, the system must decompose a task, assign work, call tools, exchange intermediate outputs, and synthesize a final conclusion.

MisinfoTask is designed around that setting.
It contains **108 realistic tasks**, and each instance includes fields such as:

| Field | Meaning |
|:--|:--|
| `user_input` | The original user task |
| `agent_num` | Number of agents used for the task |
| `tools` | Tools available to agents |
| `misinfo_goal` | The misleading objective that the injected misinformation tries to induce |
| `misinfo_argument` | Plausible but fallacious arguments |
| `ground_truth` | Corrective factual information |
| `reference_solution` | Expected task-solving workflow |

Each task includes **4-8 plausible but fallacious misinformation arguments**.
This is useful because the threat is not phrased as a crude jailbreak.
It is phrased as domain-relevant reasoning that could plausibly enter a collaborative task.

The dataset covers several task categories:

| Category | Entries |
|:--|--:|
| Conceptual Reasoning | 28 |
| Factual Verification | 20 |
| Procedural Application | 29 |
| Formal Language Interpretation | 17 |
| Logic Analysis | 14 |

I think this dataset design is the most practically useful part of the paper.
For MAS safety, overly simple QA benchmarks do not expose how wrong information moves through a task chain.

---

# 5. Misinformation Injection Settings

The paper evaluates three injection methods.

| Attack | Injection point | Intuition |
|:--|:--|:--|
| Prompt Injection (PI) | A victim agent's prompt | The agent is instructed to promote the misinformation goal |
| RAG Poisoning (RP) | Shared retrieval knowledge base | Agents retrieve misleading documents |
| Tool Injection (TI) | Simulated tool output | The tool returns misinformation during execution |

These attack surfaces match the structure of modern agents:

```text
prompt + memory/RAG + tools + communication
```

The paper measures two outcomes:

$$
\mathrm{MT} = \frac{1}{N}\sum_{k=1}^{N}\texttt{Score}(O_k, g_{mis}^k)
$$

$$
\mathrm{TSR} = \frac{1}{N}\sum_{k=1}^{N}\mathbb{I}(\texttt{Score}(O_k, g_{task}^{k}) \geq \theta_m)
$$

| Metric | Meaning | Desired direction |
|:--|:--|:--|
| MT | Misinformation Toxicity, or how much the final output aligns with the misinformation goal | Lower |
| TSR | Task Success Rate, or whether the final output satisfies the original task goal | Higher |

The attack result is clear:

![MT and TSR under misinformation injection](/assets/img/posts/ARGUS/mt-tsr-points.png){: width="720" }
_Figure 4. Misinformation injection increases MT and lowers TSR across attack types._

The paper reports that vanilla MAS performance moves from a relatively safe cluster to a high-MT, lower-TSR region after injection.
This supports the main motivation:

> The MAS is not only answering one false question.
> It is solving a multi-step task while absorbing a false premise.

---

# 6. ARGUS Overview

ARGUS stands for:

```text
Adaptive Reasoning and Goal-aware Unified Shield
```

It is a modular and training-free defense framework.
The framework has two major stages:

1. **Critical Flow Localization.** Find suspicious communication channels where misinformation is likely to flow.
2. **Goal-Aware Persuasive Rectification.** Deploy a corrective agent to inspect, reason about, and rewrite misinformation across rounds.

![Overall ARGUS pipeline](/assets/img/posts/ARGUS/argus-pipeline.png){: width="900" }
_Figure 5. Overall ARGUS pipeline from the original paper source figure._

This figure is dense, but the structure can be read as three blocks:

| Block | Role |
|:--|:--|
| Left | MisinfoTask provides user tasks, misinformation goals, arguments, and ground truth |
| Right | PI, RP, and TI inject misinformation through different agent surfaces |
| Center | ARGUS monitors critical channels, reasons about misinformation goals, and rectifies messages |

The important design choice is that ARGUS does not simply remove an agent or cut every suspicious edge.
Instead, it inserts a corrective agent $a_{cor}$ into selected communication channels.

![Corrective agent diagram](/assets/img/posts/ARGUS/planning-with-correction-mas.png){: width="560" }
_Figure 6. Seminar diagram showing a corrective agent inserted into MAS information flow._

This matches the threat model.
If misinformation is subtle, pruning alone can be too blunt.
The system needs to preserve useful collaboration while correcting false premises.

---

# 7. Stage 1: Critical Flow Localization

ARGUS first identifies a subset of edges:

$$
\mathcal{E}_r \subseteq \mathcal{E}
$$

Each edge in this subset is a communication channel likely to carry misinformation in round $r$.

The localization process has two parts.

## 7.1 Initial Localization

Before dialogue logs exist, ARGUS cannot use semantic evidence.
So it starts with topology.

For each edge $e$, it computes a normalized edge betweenness centrality score:

$$
\mathtt{Score}_{topo}(e)
=
\frac{1}{N_{norm}}
\sum_{a_i \in \mathcal{A}}
\sum_{a_j \in \mathcal{A}, i \neq j}
\frac{\sigma_{ij}(e)}{\sigma_{ij}}
$$

where:

| Symbol | Meaning |
|:--|:--|
| $\sigma_{ij}$ | Number of shortest paths from $a_i$ to $a_j$ |
| $\sigma_{ij}(e)$ | Number of those paths that pass through edge $e$ |
| $N_{norm}$ | Normalization factor |

The intuition is:

> An edge that appears on many shortest paths is a good initial monitoring point because information is likely to pass through it.

ARGUS then selects the highest-scoring outgoing edge for each source agent:

$$
e_i^* =
\arg\max_{e_{i\cdot} \in \mathcal{E}}
\{\mathtt{Score}_{topo}(e_{i\cdot})\}
$$

These edges form:

$$
\mathcal{E}_{best}=\{e_i^* \mid a_i \in \mathcal{A}\}
$$

The initial monitored set $\mathcal{E}_1$ is built from high-scoring edges in $\mathcal{E}_{best}$ and the remaining globally important edges.

This is a practical compromise.
It monitors central channels while maintaining coverage over many source agents.

## 7.2 Adaptive Re-Localization

After the first round, ARGUS has dialogue logs.
It can now use content-level signals.

The corrective agent infers textual descriptions of misinformation goals:

$$
\mathcal{G}'_{mis}=\{g^i_{mis}\}_{i=1}^{p}
$$

These inferred goals are embedded:

$$
V'_{mis}=\{v'_i\}_{i=1}^{p}, \quad v'_i=\Phi(g^i_{mis})
$$

For each sentence $s$ in a message $m$, ARGUS computes average similarity to the inferred misinformation goal embeddings:

$$
\mathcal{S}(s,V'_{goal})
=
\frac{1}{p}
\sum_{i=1}^{p}
\mathtt{Sim}_{cos}(\Phi(s), v'_i)
$$

Then it defines message relevance as the maximum sentence similarity above a threshold:

$$
\mathtt{Rel}(m,V'_{goal})
=
\max_{s \in m}\{\{0\}\cup \mathcal{S}(s,V'_{goal})\}
$$

subject to:

$$
\mathcal{S}(s,V'_{goal}) \geq \theta_{sim}
$$

For an edge $e$, ARGUS uses the most relevant message flowing through that channel:

$$
\mathtt{Score}_{rel}(e)
=
\max_{m \in m_e^{r-1}}
\mathtt{Rel}(m,V'_{goal})
$$

It also counts channel frequency:

$$
\mathtt{Score}^{r-1}_{freq}(e)=\mathtt{count}(m_e(r))
$$

So the adaptive edge score combines:

| Score | What it captures |
|:--|:--|
| $\mathtt{Score}_{topo}$ | Structural importance |
| $\mathtt{Score}_{freq}$ | How often the channel is used |
| $\mathtt{Score}_{rel}$ | Semantic relevance to inferred misinformation goals |

In the appendix setup, the paper weights these as:

| Component | Weight |
|:--|--:|
| Topology importance | 0.2 |
| Channel frequency | 0.2 |
| Information relevance | 0.6 |

This weighting shows that ARGUS is primarily semantic after it has enough logs.
Topology gives a prior, but goal relevance drives re-localization.

---

# 8. Stage 2: Goal-Aware Persuasive Rectification

Once ARGUS selects critical channels, the corrective agent $a_{cor}$ monitors messages flowing through them.

The rectification process has three steps.

| Step | Role |
|:--|:--|
| Multi-faceted identification | Decompose the message sentence by sentence and identify suspicious elements |
| Internal knowledge resonance | Compare suspicious claims against the LLM's parametric knowledge |
| Heuristic persuasive reconstruction | Rewrite the information in a way that is factual and persuasive |

This part is more than a simple "fact check."
The corrective agent needs to produce a correction that other agents will accept in context.

The paper describes three rectification principles:

| Principle | Meaning |
|:--|:--|
| Root-cause analytical rectification | Explain what is wrong and why |
| Cognitive reframing | Preserve useful parts of the message while correcting false premises |
| Contextual integration | Insert the correction naturally into the ongoing conversation |

This is why the paper calls it **persuasive rectification**.
The goal is not only to label a sentence false.
The goal is to redirect the MAS away from the misleading goal while keeping the task process coherent.

In high-level pseudocode, ARGUS behaves like this:

```text
for each round r:
  for each monitored edge e in E_r:
    inspect message m_e^r with CoT-style reasoning
    infer the misinformation goal g'_e^r
    update the unique set of inferred misinformation goals
    rectify the message if needed

  score all edges using topology, frequency, and goal relevance
  select the top-k edges for the next round
```

This loop is the main difference from static defenses.
ARGUS keeps moving its attention as the misinformation flow moves.

---

# 9. Experimental Results

The experiments use four core LLMs:

| Model family |
|:--|
| GPT-4o-mini |
| GPT-4o |
| DeepSeek-V3 |
| Gemini-2.0-flash |

The paper compares:

| Defense | Basic idea |
|:--|:--|
| Attack-only | No defense |
| Self-Check | Prompt agents to re-evaluate information |
| G-Safeguard | Detect high-risk agents with a GNN and prune risky communication |
| ARGUS | Localize critical misinformation flows and rectify messages |

The main table reports MT and TSR across prompt injection, RAG poisoning, and tool injection.

![Main ARGUS results table](/assets/img/posts/ARGUS/main-results-table.png){: width="760" }
_Table 1. Main results cropped from the paper PDF._

The paper reports that ARGUS reduces MT under all three attack types:

| Attack type | Average MT reduction reported in the paper |
|:--|--:|
| Prompt Injection | 28.18% |
| RAG Poisoning | 20.38% |
| Tool Injection | 35.95% |

The OpenReview abstract summarizes the overall result as:

| Metric | Reported effect |
|:--|:--|
| Misinformation toxicity | About 28.17% average reduction |
| Task success rate under attack | About 10.33% improvement |

The trend over rounds is also important.
Without defense, MT tends to increase as the MAS continues operating.
With ARGUS, MT decreases over successive rounds.

![MT over communication rounds](/assets/img/posts/ARGUS/mt-over-rounds.png){: width="760" }
_Figure 7. MT trends across rounds. Dashed lines denote attack + ARGUS._

This supports the core claim:

> Misinformation is dynamic.
> A useful defense should also be dynamic.

ARGUS does not assume that one-time detection is enough.
It repeatedly uses the previous round's evidence to decide where to monitor next.

The paper also evaluates whether the corrective agent can identify the misleading goal of misinformation.

![Corrective agent goal identification accuracy](/assets/img/posts/ARGUS/goal-accuracy.png){: width="620" }
_Figure 8. Accuracy of $a_{cor}$ in identifying the misleading goal under different attacks and LLMs._

The accuracy is strongest for prompt injection and lower for tool/RAG settings.
This makes sense.
Prompt injection directly inserts a goal-like instruction, while tool and RAG poisoning may expose misinformation more indirectly.

---

# 10. Ablation Study

The ablation study removes key components of ARGUS.

![ARGUS ablation tables](/assets/img/posts/ARGUS/ablation-tables.png){: width="760" }
_Table 2 and Table 3. Ablation results cropped from the paper PDF._

The most important ablations are:

| Ablation | What is removed |
|:--|:--|
| w/o Dynamic Localization | ARGUS cannot adaptively move monitoring edges |
| w/o CoT Revision | The corrective reasoning stage is weakened |
| w/o Multi-Turn Correction | Rectification is not performed across multiple rounds |
| w/ Ground Truth | The corrective agent receives the true misinformation information |

The result is intuitive.
Removing dynamic localization or multi-turn correction hurts performance.
Providing ground truth improves performance.

I read this as evidence that ARGUS works because the two stages are coupled:

```text
better goal inference -> better edge re-localization
better edge localization -> better future correction
```

The paper's hyperparameter ablation also shows that the information relevance term is especially important.
When the relevance weight $\gamma$ is removed, performance degrades substantially.

That matches the design logic.
For misinformation, topology alone is not enough.
The defense needs to understand what false goal the message is pushing.

---

# 11. Topology Robustness

The appendix evaluates ARGUS across different MAS topology settings:

| Topology | Description |
|:--|:--|
| Chain | Agents communicate in a sequential chain |
| Full | Fully connected communication graph |
| Self-Determined | The planning agent determines topology based on the task |

![Topology robustness results](/assets/img/posts/ARGUS/topology-results.png){: width="760" }
_Figure 9. ARGUS reduces MT across several MAS topology settings._

This is important because MAS topology is not fixed in real systems.
Some systems are centralized.
Some are chain-like.
Some let agents choose who to talk to.

The paper reports that ARGUS remains useful across these configurations.
The reason is that it combines topology with message-level semantic relevance.
If the graph changes, topology scores change.
If the misinformation goal moves, relevance scores change.

---

# 12. Relationship to G-Safeguard and INFA-Guard

I think ARGUS fits naturally after G-Safeguard and before INFA-Guard in the MAS security line.

| Paper | Main security unit | Main defense behavior |
|:--|:--|:--|
| G-Safeguard | Attack agents and risky topology | Detect high-risk agents and prune edges |
| ARGUS | Misinformation-carrying communication flows | Localize critical channels and rectify messages |
| INFA-Guard | Attack and infected agents | Detect infection propagation and remediate agents/responses |

ARGUS is less about "who is malicious?" and more about "which message flow is misleading the system?"

This is a useful shift.
Misinformation may not map cleanly to a malicious agent label.
An agent can be benign but still transmit a wrong premise because it received polluted context from RAG, a tool, or another agent.

That makes flow-level defense a reasonable target.

---

# 13. Limitations

The paper's limitations are practical.

First, ARGUS adds computational overhead.
It introduces an external corrective agent, runs reasoning over messages, embeds goal descriptions, and updates edge scores across rounds.
For small MAS experiments, this is acceptable.
For large production MAS, the cost could become significant.

Second, the paper mainly handles misinformation that conflicts with knowledge already resident in the agents' core LLMs.
This is a strong assumption.
Many real-world MAS tasks depend on dynamic information:

| Example | Why it is hard |
|:--|:--|
| Current regulation | It may have changed recently |
| Today's market data | It requires external freshness |
| Breaking news | The model's parametric knowledge is not enough |
| Internal enterprise documents | The truth may be private and unavailable to the base LLM |

In these settings, ARGUS would likely need retrieval-grounded verification or external source validation.

Third, persuasive rectification depends on the corrective LLM.
If the corrective agent's own knowledge is wrong, stale, or biased, it may confidently produce an incorrect correction.

---

# 14. Takeaways

The main point I took from ARGUS is:

> MAS misinformation defense should be goal-aware and flow-aware.

It is not enough to detect unsafe content.
Misinformation can be semantically harmless and still damage the task.
It is also not enough to detect one compromised agent.
The wrong premise can move across communication channels and become part of the system's shared belief.

ARGUS handles this by combining:

1. topology-based initial monitoring,
2. semantic re-localization based on inferred misinformation goals,
3. a corrective agent that performs multi-round persuasive rectification.

The framework is not cost-free, and it is not a complete solution for dynamic external knowledge.
But conceptually, it is a valuable step toward MAS defenses that preserve collaboration instead of simply cutting communication.

For me, this paper is interesting because it treats MAS safety as an information-flow problem.
That framing is likely to become more important as LLM agents become less like isolated chatbots and more like connected systems that reason, retrieve, call tools, and influence each other over time.
