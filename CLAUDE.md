# Claude - Context & Goals

## What This Repository Really Is

This is my private prep ground for crushing SRE/DevOps/Platform Engineering interviews. Not a public-facing tutorial site. Not a collection of tool commands. This is where I build the deep mental models that separate seniors from juniors in technical discussions.

## The Interview Gap I'm Closing

Most candidates can recite how to write a Kubernetes deployment or explain what a load balancer does. That's table stakes. What distinguishes someone in a senior+ interview is the ability to:

- Explain *why* systems are designed the way they are
- Articulate trade-offs between architectural approaches
- Connect operational decisions to business impact
- Reason from first principles when encountering novel problems
- Discuss the theoretical foundations that underpin the tools we use

## Content Philosophy

**Theory Over Tools**: I don't need another kubectl cheat sheet. I need to understand consensus algorithms, distributed systems failure modes, and the economics of release velocity. When I discuss Kubernetes in an interview, I should be able to talk about the Raft consensus in etcd, not just "it's a distributed database."

**First Principles Thinking**: Break everything down to its fundamentals. Why do we need container orchestration? What problem does it actually solve? What happens without it? What are the fundamental constraints of distributed systems that inform design choices?

**Academic Depth, Practical Grounding**: Think graduate-level computer science lecture meets battle-tested operational knowledge. The theory is pointless if I can't connect it to real production scenarios. The practical is shallow if I don't understand the underlying principles.

**Natural, Not Textbook**: Write like an experienced engineer explaining concepts over coffee, not like a formal academic paper. Use clear examples. Call out common misconceptions. Be direct about what actually matters versus what's marketing fluff.

## Current Focus: Release Management

Release management is fascinating because it sits at the intersection of:
- Distributed systems (how do we safely propagate state changes?)
- Economics (cost of delay vs. cost of failure)
- Risk management (blast radius, error budgets, progressive delivery)
- Automation theory (what *should* be automated vs. what *can* be automated)

I'm building a mental model that lets me discuss:
- Why trunk-based development mathematically reduces integration risk
- How feature flags change the deployment/release contract
- The relationship between CI maturity and release velocity
- Why reproducible builds matter for security and debugging
- How observability enables automated release decisions

## Interview Readiness Metrics

I'll know this content is ready when I can:

1. **Whiteboard system designs** that incorporate these concepts naturally
2. **Explain trade-offs** between different approaches (GitFlow vs. Trunk-Based) with real numbers and scenarios
3. **Debug theoretical problems** ("Your canary deployment is showing a 2% error rate increase. Walk me through your decision process.")
4. **Connect to business value** ("How does improving build reproducibility impact our security posture and audit compliance?")
5. **Go deep on request** ("Tell me about the CAP theorem and how it relates to your monitoring system choice")

## What Success Looks Like

In an interview, when asked "Tell me about your deployment strategy," I shouldn't just describe what we did. I should:
- Explain the risk model we optimized for
- Compare it to alternatives we considered
- Discuss the trade-offs we accepted
- Mention the theoretical foundations (error budgets, blast radius reduction)
- Connect to measurable business outcomes

This repository is how I get there.

## Domains Covered

- **Behavioral**: How SREs think, operational decision-making frameworks, incident response mental models
- **Databases**: Theory of persistence, consistency models, replication strategies
- **Kubernetes**: Container orchestration theory, control loop patterns, declarative vs. imperative
- **Linux**: Systems fundamentals, process models, resource isolation
- **Release Management**: Deployment theory, risk quantification, progressive delivery
- **Terminology**: Precise definitions for concepts that get misused (idempotency, observability, etc.)

## Anti-Goals

**Not a certification study guide**: Those teach to the test. I need deeper understanding.

**Not a tutorial site**: I'm not writing "How to set up CI/CD in 10 steps." I'm building frameworks for thinking about CI/CD strategy.

**Not comprehensive breadth**: I don't need to cover every possible topic. I need depth in the areas that matter for the roles I'm targeting.

**Not public-ready**: This is rough notes, half-formed thoughts, and work in progress. That's fine. It's for me.
