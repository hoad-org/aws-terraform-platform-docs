# Security Roadmap — phased, revenue-triggered

This is a pointer, not a spec. [SECURITY_BASELINE.md](SECURITY_BASELINE.md) is maximal for $0;
this is what to turn on, roughly in this order, once the trigger condition is actually true — not
proactively, not "because it'd be nice." Each item costs real money the moment it's enabled.

| Enhancement | Trigger | Why this order |
|---|---|---|
| **GuardDuty** | Any workload starts handling a third party's data (not just Craig's own) | Cheapest, broadest-coverage threat-detection service; the natural first step past the free baseline |
| **AWS Config** | Infrastructure footprint grows past what a human can reasonably eyeball for drift/compliance | Config's value scales with account/resource count — not worth it at today's scale |
| **Security Hub** | Once GuardDuty + Config are both on and producing real findings to aggregate | Aggregates findings from services that don't exist yet at $0 — enabling it earlier has nothing to aggregate |
| **WAF** | A workload starts taking real, unauthenticated public traffic at meaningful volume (not just Craig's own personal use) | Per-request cost only makes sense once there's traffic worth protecting |
| **VPC Flow Logs (persistent)** | A real security incident, or a compliance requirement, needs network-level audit trail | Ongoing CloudWatch Logs cost; enable case-by-case for debugging today, persistent only once justified |
| **Macie** | Hosting real personal data (not synthetic/test data) at meaningful volume | Scanning cost scales with data volume — not worth it until there's real data to worry about |

**A concrete trigger worth naming explicitly, since it's the most likely one for this estate**:
once `personal-ai-cloud` or any successor starts hosting AI access to data beyond Craig's own
account (i.e., "if we start making money from our services and host the kinds of data / AI access
that requires it," per the brief this roadmap was written against) — that's the point to
re-evaluate this whole table, not wait for each row's trigger individually. Revenue or third-party
data access changes the threat model enough to justify moving through several rows at once.
