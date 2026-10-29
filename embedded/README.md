# Embedded Harness Anatomy

Top-down dissections of agent harnesses that run on milliwatts — a companion
series to [Agent Harness Anatomy](../README.md). Same discipline (pinned
source, every behavioral claim verifiable), adapted for event-driven systems
under physical constraints: power budgets, KB-scale memory, bare metal,
no guaranteed connectivity.

Where the main series asks "what happens between your prompt and the tool
call?", this one asks "what happens between the sensor interrupt and the
actuation?" — and what each world teaches the other.

## Teardowns

| # | Harness | Article | Trace |
|---|---------|---------|-------|
| E1 | ESP-Claw (v0.1.0) | [esp-claw/article.md](esp-claw/article.md) | [Follow one sensor event](esp-claw/trace.html) |

## Dimensions

Embedded teardowns are compared across ten adapted dimensions: compute
placement, power envelope, connectivity dependence, event model,
sensor/actuator interface, model footprint, memory architecture, update
mechanism, real-time constraints, and failure & safety.
