# Executive Case Study: Reducing the Risk of Persistent Privileged Access

## Case Study Scope

This case study examines how an organization can reduce the risks associated with persistent administrative access by moving toward centrally controlled, time-bound privileged access.

I implemented a reference architecture that demonstrates the core access pattern. The purpose was to evaluate how administrative access could be granted for a limited period rather than relying on authorization that remains in place until someone manually removes it.

The reference implementation demonstrates the core control model. A production deployment would require additional decisions around enterprise identity, approval processes, emergency access, monitoring, resilience, and operational ownership.

## Business Problem

Privileged administrators need elevated access to maintain infrastructure, troubleshoot problems, respond to incidents, and make authorized changes. That access is necessary, but it also creates significant risk.

If administrative access remains available indefinitely, the organization has to continuously manage questions such as:

- Does this person still need access?
- Has an old credential been removed?
- What happens when someone changes roles or leaves the organization?
- How quickly can access be limited during a security incident?
- Can the organization determine when privileged access was granted and subsequently used?

The business problem is therefore not simply protecting administrator passwords or keys. It is controlling the **lifecycle of privileged access**.

For this scenario, I evaluated a different model: instead of treating administrative access as something that remains available until manually removed, privileged access is centrally authorized and issued for a limited period.

That changes the fundamental question from:

**"Who has permanent administrator access?"**

to:

**"Who is authorized for privileged access right now, under what conditions, and for how long?"**

That shift provides the foundation for a more controlled and auditable privileged-access model.