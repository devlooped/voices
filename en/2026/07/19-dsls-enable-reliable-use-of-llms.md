---
title: "DSLs Enable Reliable Use of LLMs"
summary: "Unmesh Joshi argues that domain-specific languages and shared semantic models give LLMs clear boundaries so they generate reliable code—illustrated with PlantUML presentation tooling and the Tickloom framework for distributed systems."
date: "2026-07-14"
link: "https://martinfowler.com/articles/llm-and-dsls.html"
author: "Unmesh Joshi"
---
DSLs Enable Reliable Use of LLMs
by Unmesh Joshi, posted on July 14th, 2026

Modern large language models possess an incredible capability. They can generate large amounts of code, and sometimes entire systems, from just a high-level natural language description. An important assumption here is that the intent of what needs to be built is well articulated, using precise words that LLMs can map to coding building blocks. However, there are two important points worth noting: the limits of upfront specification, and how design is discovered through implementation.

The Limits of Upfront Specification

Building large systems involves a great many small design decisions, and these cannot all be known in advance or driven entirely from a high-level specification. A specification is at best a starting hypothesis: the real constraints, trade-offs, and edge cases are discovered iteratively, as we proceed with the implementation. We discussed this at length in an earlier article, where we called it Upfront Specification Impossibility. The point is not that specs are worthless, but that the first one is a hypothesis to be revised, never a finished blueprint.

The natural response is to iterate: refine the specification, generate code, review what comes back, and feed what we learn into the next round. That loop works well when each round produces a small, reviewable change.

Design Is Discovered Through Implementation

Reviewing code, particularly while we are still discovering the design, is not the same as writing it. While reviewing the generated code, we review through the chunks validating if it maps to our intent and looking for possible pitfalls. But reviewing rarely forces us to wrestle with the design decisions. Writing code, by contrast, forces us to think through concrete decisions—such as where a responsibility belongs or what boundaries should be exposed so the design can be extended further. It is in making those decisions that a design most fully reveals itself.

Code has two distinct but intertwined purposes. It is a set of instructions for a machine, and it is also a conceptual model of the problem domain. A well designed codebase is a representation of the vocabulary of a domain. These abstractions reveal themselves only as developers build the software. Programming languages act as thinking tools, enabling the construction of a conceptual model that supports later evolution. With LLMs, code acts as essential context: good abstractions, executable behavior, tests, types, and invariants all help constrain the model and make its output more useful.

The programming language and paradigm we code in shapes the design insight we get. A functional design approach or an object-oriented design approach reveals different aspects of the design, along with idioms and patterns that are natural to the paradigm.

So where do LLMs fit in? I see LLMs playing two roles. They are a great help while we shape the design and its vocabulary, acting as brainstorming partners to help us explore the design space and discover the right abstractions. Once the vocabulary is established, LLMs work as an excellent natural language interface to it.

Domain Abstractions and DSLs

A useful way to frame this is through Domain Driven Design. Its core insight is building a shared conceptual model of the domain in code, and then using that model—which DDD calls a Ubiquitous Language—both to evolve the codebase and to give the team a vocabulary to think and communicate in. Often, it is highly effective to build a domain specific language on top of that model: a constrained syntax for expressing the domain's concepts and operations. Seen this way, most development is the process of building a domain model and using it to evolve the system. The LLM plays two distinct roles depending on whether the domain model already exists. In this article, I will focus on how Domain-Specific Languages, or DSLs, work with LLMs.

Why DSLs work so well with LLMs

It is a common experience that DSLs work well with LLMs. PlantUML, Mermaid, and Graphviz are domain specific languages for visual modeling; SQL is a DSL for querying databases; Kubernetes YAML is a DSL for describing cloud infrastructure. These are not general-purpose programming languages—they are deliberately constrained, designed to express a narrow set of concepts in one domain. And it is no surprise that LLMs are remarkably good at generating Mermaid diagrams, SQL queries, or Kubernetes manifests from a plain English description.

My observation is that DSLs make LLMs more reliable because they respond so well to a few in-context examples. A general-purpose language like Java offers lots of valid ways to express the same intent. A DSL strips the variation away. Giving the model a few examples is enough to reliably generate the correct syntax. It's worth noting that frontline models are already heavily exposed to PlantUML or Java fluent interfaces during training, so they aren't starting from scratch. It will be curious to see how smaller, more constrained models perform when tasked with a truly novel DSL.

For an agent—an LLM running in an autonomous generate-and-check loop rather than a single shot generation—there is one more benefit. A DSL almost always ships with a deterministic validator: a parser, a JSON schema, a type checker, or a compiler. The agent can generate a candidate, run it past the validator, and repair it from the error, all without a human in the loop. Crucially, the errors are phrased at the level of the domain—for example, you cannot select an action before choosing a client—rather than as a stack trace buried deep in generated code. A DSL's toolset itself acts as an excellent harness.

It is important to note that this is not a one-size-fits-all solution. The advantage holds while the DSL stays small and constrained enough that a few in-context examples can convey its usage. There is also a real upfront cost in designing and maintaining the language and its semantic model. The payoff is therefore concentrated in well-factored, genuinely constrained DSLs backed by a validator.

Example: Using LLMs to generate diagram-rich PowerPoint presentations

LLMs make it really easy to build custom tools. While teaching distributed systems, I frequently need to create presentations which mostly have diagrams explaining distributed operations in a cluster. UML sequence diagrams have been great for that, but showing a full sequence diagram while explaining the flow of messages through the cluster is not very useful. I needed a tool to show a sequence diagram step by step in a PowerPoint presentation. With the help of LLMs I was able to build a tool which processes a YAML description of the presentation structure with references to PlantUML diagrams and generates a PowerPoint presentation. The PlantUML diagrams are marked with steps, and the tool generates a separate slide for each step. This made it really easy to create diagram-rich presentations.

A natural language prompt can generate PlantUML sequence diagrams with step markers for a three-node cluster—Athens, Byzantium, and Cyrene—showing message flow, failures, and quorum checks. A second, smaller YAML DSL then describes which diagram belongs on which slide. Because the tool and the YAML specification understood by the tool are used as context in the prompt, the LLM generates the correct YAML that the tool can consume directly.

Notice that the LLM played two different parts in this single example. First it was a co-designer—helping shape the step-marked PlantUML extension and the slide YAML on top of existing PlantUML tooling. Then, once that small DSL existed, it became the natural-language interface that turns an English request into a valid specification.

Building the Semantic Model

In the presentation example, the YAML was used as a carrier syntax, and I process its parsed syntax tree directly, effectively using the syntax tree itself as my Semantic Model—though this couples the syntax to the execution semantics. But in more complex domains, like distributed systems, we need more complex semantic models to represent the concepts in the domain and the design decisions we have made in the codebase.

Example: Tickloom—a semantic model for distributed systems

Implementing distributed systems such as quorum-based key-value stores or consensus protocols like Raft and Paxos is a daunting task. Even if implementation is guided incrementally through prompts, specifications, or carefully constructed skill files, the asynchronous runtimes still expose an overwhelming space of possible implementation decisions. Threading models, networking patterns, storage coordination, retry behavior, and timing semantics all remain entangled within the generated code. The problem is not merely code generation complexity, but verification complexity. The resulting state space created by all possible interleavings across thread scheduling, network delays, process pauses, and clock skew becomes so large that systematically reviewing and validating correctness across all interacting behaviors is nearly impossible. This is the reason Jepsen tests find bugs even in the most battle-tested distributed systems.

This is exactly where a semantic model is beneficial. Tickloom is a small framework I built to construct and test distributed algorithms. Its abstractions are not a generic runtime; they are a set of design decisions about how a distributed process behaves. Every node runs in a single-threaded tick loop: each call to tick advances a logical clock by one and processes pending work in a fixed, deterministic order—network, then message bus, then process, then storage. Time is measured in ticks, not milliseconds. Messages are plain Java records. Coordination across replicas is expressed through a Replica base class that already knows about peers, broadcasts, and quorums.

Threading, timing, and network delivery are no longer open questions to be re-decided in every prompt. What remains for the algorithm author is the actual protocol logic. A quorum replica, for instance, is just a set of message handlers expressed in the framework's vocabulary.

Because the framework supplies the vocabulary—Replica, quorum request, count response if, message type, handler—a prompt can stay at the level of the protocol rather than the plumbing. For example: using the Tickloom Replica abstraction, implement a quorum-based key-value store where a client get collects values from a majority and returns the one with the highest timestamp, last-writer-wins, and a write is applied locally only if its timestamp is newer than the stored one. The generated code fills in protocol handlers against those fixed types. The semantic model itself acts as context. The prompt names concepts that exist as concrete types in the codebase, so the LLM is not inventing a threading model or a networking layer—it is filling in protocol logic against a fixed, well-understood substrate.

Even good abstractions help—without a DSL

A DSL is one end of the spectrum, and it is not easy to build. Before reaching for a language of your own it is worth noticing that a clean set of abstractions is already a lighter version of the same idea—and, just as the framework supplied the vocabulary for the quorum store above, a library's named types and methods are themselves a vocabulary the model can be grounded in. Tickloom's semantic model is really just four such seams—Process and Replica for compute and message handling, Network for communication, Storage for persistence, and a logical Clock for time—and that decomposition does most of the work with no new syntax at all.

This is why abstractions, not just DSLs, pair well with LLMs. A prompt to implement Raft as a Tickloom Replica has limited state space to explore. The existing QuorumReplica can be used in context as a worked example.

Example: Building a DSL for testing distributed system scenarios

Implementing the algorithm is one thing; exercising it is another. The subtle bugs in distributed systems live in specific orderings: a write that replicates to one node before a reader's quorum shifts, a partition that heals at just the wrong moment, two coordinators whose clocks have drifted apart. Writing such a scenario directly against the test kit involves juggling futures and manual tick loops. The intent—Bob writes through Byzantium, Alice writes through Athens, a reader sees Bob's value because Byzantium's clock was ahead—is buried under mechanics. That style of code is also hard to verify: there are dozens of incidental decisions for an LLM to get subtly wrong, and a reviewer must check every one of them.

So on top of the semantic model I built an internal DSL whose vocabulary is the vocabulary of the scenario itself—servers, clients, who is connected to whom, what each client does, and which faults are in effect while it does it. The same clock-skew scenario becomes a short declarative description: servers Athens, Byzantium, and Cyrene; clients Alice, Bob, and a reader; server times set differently; Bob writes B, Alice writes A, and the reader expects B.

The DSL is a thin, declarative surface that compiles down to a pure intermediate representation—a Scenario made of Steps, where each step carries an Action, such as a read or write, and optional cluster events, such as partitions and message delays. Faults read as English too: partition Byzantium from Cyrene, reconnect Byzantium, delay an internal set request from Athens to the other nodes by a hundred ticks. The grammar is enforced by the type system through progressive interfaces—you cannot declare a step before the topology, or an action before selecting a client—so whole classes of malformed scenarios simply do not compile. Because the DSL is an internal DSL built in Java, the host compiler validates the grammar for free, and a malformed generation comes back as a compile error pinned to exactly the illegal step rather than a runtime surprise.

Once the DSL exists, a natural language description of a failure scenario maps almost directly onto it. A prompt can ask for a scenario reproducing a non-linearizable quorum read from Designing Data-Intensive Applications section 10.6: a writer connected to Athens sets a key, then updates it while replication is delayed; Alice reading through Byzantium is forced onto a quorum that includes Athens and sees the new value; Bob reading later through a quorum of Byzantium and Cyrene still sees the old value. The LLM yields a scenario that stays entirely within the DSL's constrained vocabulary.

Because the surface is so small—and the space of valid code it can generate is so much smaller than the space of valid Java programs—the LLM has very little room to hallucinate, and a reviewer can read the result as a description of an experiment rather than as code to be audited line by line. Even if the LLM hallucinates, the internal DSL will fail to compile, allowing the LLM to correct the errors.

Two phases working with LLMs

A pattern emerges from the examples discussed above: in every one of them the LLM was useful in two quite different ways.

The first phase is designing the abstraction or DSL itself. Here the LLM is best treated as a brainstorming partner rather than a code generator. As argued at the start of this article, the design decisions that make up a semantic model cannot all be specified upfront—we discover the constraints, trade-offs, and edge cases as we implement them. So this phase is inherently iterative and feedback-driven: you propose a structure, try it against a real case, see where it turns out awkward, and feed what you learned back into the next round. The LLM speeds that loop up—it sketches alternatives, critiques a design, ports an idea from one language to another—but you stay firmly in the driver's seat, because these are exactly the decisions you need to understand and own. The structures that make a DSL pleasant to use, like progressive interfaces that make an illegal scenario fail to compile or a semantic model kept separate from the builder that produces it, are the kind of thing you converge on by iterating, not by writing a specification and generating code.

The second phase begins once the abstraction or DSL is in place. Now what the LLM does changes: it becomes a natural-language interface to what you have built. The prompts in this article are examples—implement a quorum store as a Tickloom Replica, write a scenario reproducing the non-linearizable read, create a slide YAML for this diagram. In each case the English description maps almost directly onto the vocabulary you defined, and the LLM is a dependable generator precisely because the abstraction supplies both the context that grounds the prompt and the harness that checks the result.

The DSL as the Source of Truth

There is a growing trend of treating prompts as the primary source of truth. A well-designed DSL fundamentally changes this dynamic. One of the key advantages I observe of working with DSLs is that the generated program itself often becomes the artifact that humans maintain. Because a DSL is dense, expressive, and largely free of incidental boilerplate, it captures the essential intent of the solution in a form that remains readable long after generation. If an LLM generates a Tickloom failure scenario from a natural language request, the resulting scenario is already expressed in the vocabulary of the domain. If the scenario needs to change next month, there is no need to recover the original prompt and regenerate everything. The DSL has enough context for the LLM to know the intent and work with it. The enduring asset is not the prompt, but the DSL and the semantic model.

Acknowledgments

I would like to thank Martin Fowler and Rebecca Parsons for their valuable feedback and suggestions.

