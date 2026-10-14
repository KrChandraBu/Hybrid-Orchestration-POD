# ADR [NNNN]: [Short, decision-focused title]

**Status**: [Proposed | Accepted | Deprecated | Superseded]
**Date**: [YYYY-MM-DD]
**Authors**: [Team or author names] ([contact email or channel])

**Supersedes**: [ADR NNNN](NNNN-title.md) <!-- optional; delete if not applicable -->
**Superseded in part by**: [ADR NNNN](NNNN-title.md) <!-- optional; delete if not applicable -->

---

## Context

[Describe the situation that forces this decision. What problem exists, what is the current state, and what constraints apply (technical, organizational, timeline)?]

- [Component or system A]: [how it works today / what it does relevant to this decision]
- [Component or system B]: [how it works today / what it does relevant to this decision]
- [Upcoming or planned changes that influence the decision]

[State the core question this ADR answers, e.g. "The question is **how X should do Y given Z**."]

[Number of options] options were considered.

## Decision

**[One-sentence statement of the chosen approach.]**

[Explain how the decision works in practice: the main components, their responsibilities, and how data or control flows between them. Be specific about boundaries, i.e. what each component owns and what it must not touch.]

[Optional: key schema, interface, or contract details. Link to a full design doc rather than duplicating it.]
See [Design doc title](path/to/design-doc.md#section) for full details.

## Consequences

**Positive**
- [Benefit 1: what becomes easier or simpler]
- [Benefit 2: what contract or boundary becomes clearer]
- [Benefit 3: what risk is reduced]
- [Benefit 4: how existing infrastructure or patterns are reused]

**Negative**
- [Cost 1: added complexity, components, or operational burden]
- [Cost 2: new risk introduced]. Mitigation: [how the risk is reduced or monitored].
- [Cost 3: performance, latency, or other trade-off]. [Why it is acceptable.]

## Alternatives considered

### A. [Name of alternative]

[Brief description of the alternative approach.]

**Rejected because**: [Concrete reasons: what breaks down, what coupling or cost it introduces.]

### B. [Name of alternative]

[Brief description of the alternative approach.]

**Rejected because**: [Concrete reasons.]

[Add more alternatives as needed. Optionally note hybrids that were also considered and why they were rejected.]

## References

- [Design doc or spec](path/to/doc.md): [one-line description]
- `[path/to/relevant/file]`: [why it is relevant, e.g. existing pattern reused or example of the current state]
- [Related ADR, ticket, or external link]
