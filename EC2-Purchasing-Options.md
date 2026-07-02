# EC2 Purchasing Options — On-Demand, Reserved, Spot, Dedicated

**Amazon EC2** rents virtual servers, and the *same instance* can be paid for in radically different ways. The purchasing option doesn't change the hardware or performance — it changes **price, commitment, interruption risk, and tenancy**. Choosing well is one of the highest-leverage cost decisions in AWS: the gap between the most and least expensive way to run the same workload can exceed **90%**.

---

## 1. The Landscape at a Glance

```
                    Commitment ──────────────▶
                    none            1–3 years
                 ┌───────────────┬──────────────────────────┐
 Interruptible?  │  On-Demand    │  Reserved Instances      │
      no         │  (baseline    │  Savings Plans           │
                 │   price)      │  (up to ~72% off)        │
                 ├───────────────┼──────────────────────────┤
      yes        │  Spot         │        —                 │
                 │  (up to ~90%  │                          │
                 │   off)        │                          │
                 └───────────────┴──────────────────────────┘

 Orthogonal knobs:
 ├── Tenancy: shared (default) | Dedicated Instance | Dedicated Host
 └── Capacity Reservations: guarantee capacity exists (billing separate)
```

Two ideas people often conflate, kept separate:

- **Billing discount** (Reserved/Savings Plans/Spot) — *how much you pay*.
- **Tenancy** (Dedicated) — *whose physical hardware you share*. Dedicated options cost *more*, not less; they solve compliance/licensing problems, not cost problems.

---

## 2. On-Demand — the default

Pay per second (Linux, 60 s minimum) with **no commitment and no interruption risk**. Start and stop whenever you like; the price is the published baseline.

- **Advantages:** total flexibility, zero planning, no upfront money, capacity available immediately (usually).
- **Disadvantages:** the most expensive way to run anything long-term.
- **Use when:**
  - New or unpredictable workloads whose shape you don't know yet.
  - Short-lived, spiky, or experimental work (dev/test that can't be interrupted).
  - The *first* phase of anything — measure with On-Demand, then commit (§3) once the baseline is clear.

> **Rule of thumb:** On-Demand is a discovery tool, not a destination. A steady workload still on On-Demand after 3–6 months is money left on the table.

---

## 3. Commitment Discounts — Reserved Instances & Savings Plans

Both trade a **1- or 3-year commitment** for discounts up to ~72%. You commit to *spend*, and the discount is applied automatically to matching usage; nothing changes about the instances themselves.

### 3.1 Reserved Instances (RIs)

A commitment to a specific **instance family + region** (and optionally OS, tenancy, AZ).

| Flavor | What's fixed | Notes |
|---|---|---|
| **Standard RI** | Family, region; size flexible within family (Linux, shared tenancy) | Biggest discount (~up to 72% for 3-yr all-upfront); can be *sold* on the RI Marketplace if plans change |
| **Convertible RI** | Can be *exchanged* for different family/OS/tenancy | ~Up to 66%; flexibility at a slightly lower discount |
| **Zonal RI** | Pinned to one AZ | Slightly different role: also acts as a **capacity reservation** in that AZ |

Payment options (applies to Savings Plans too): **All Upfront** (max discount) > **Partial Upfront** > **No Upfront** (smallest discount, no cash outlay).

### 3.2 Savings Plans — the modern default

A commitment to **$X per hour of compute spend** for 1 or 3 years, applied automatically to whatever matches:

| Type | Covers | Flexibility | Max discount |
|---|---|---|---|
| **Compute Savings Plan** | EC2 (any family/region/OS), **Fargate, Lambda** | Highest — survives migrations between families, regions, even to containers/serverless | ~66% |
| **EC2 Instance Savings Plan** | One instance family in one region | Lower — like a Standard RI, but size/OS/tenancy flexible | ~72% |

- **Advantages over RIs:** far simpler (no instance-matching bookkeeping), follows your architecture as it evolves, covers Fargate and Lambda.
- **RIs still win when:** you need the marketplace resale option, zonal capacity reservation, or you're covering RDS/ElastiCache/etc. (those services use RIs, not Savings Plans).

- **Use when:** any **steady, predictable baseline** — the web fleet that always runs, the database tier, the minimum size of an Auto Scaling group.

> **Rule of thumb:** cover the *floor* of your usage with a Compute Savings Plan, serve the *variable middle* with On-Demand/Auto Scaling, and push the *interruptible edge* to Spot.

---

## 4. Spot Instances — the deep discount with a catch

Spot sells AWS's **spare capacity** at up to ~90% off On-Demand. The catch: AWS can reclaim the instance with a **2-minute interruption notice** whenever it needs the capacity back.

- **Advantages:** by far the cheapest compute on AWS; same hardware, full performance; scales to enormous fleets.
- **Disadvantages:** can disappear at any moment; capacity varies by instance type/AZ/time; not for anything that can't tolerate interruption.
- **Interruption handling:** listen for the 2-minute notice (instance metadata / EventBridge), checkpoint work, drain connections. **Spot Fleet / EC2 Fleet** and Auto Scaling groups with **mixed instances policy** automatically replace interrupted instances and diversify across instance pools to lower interruption rates.
- **Use when:** workloads that are **fault-tolerant, stateless, or checkpointable**:
  - Batch processing, big data (EMR/Spark), CI/CD build agents
  - ML training with checkpoints, rendering, simulations
  - Containerized stateless services as *part* of a fleet (e.g., 70% Spot / 30% On-Demand behind a load balancer)
- **Avoid for:** databases, single-instance stateful apps, anything with strict SLAs on its own.

> **Rule of thumb:** if a node vanishing mid-work costs you only *time already saved by the discount*, use Spot. If it costs you *correctness or an outage*, don't.

---

## 5. Dedicated Options — tenancy, not price

By default, EC2 is **shared tenancy**: your instances run on physical hosts shared with other AWS customers (fully isolated by the hypervisor). Dedicated options give you hardware that runs *only your* instances — for **compliance and licensing**, at a premium.

### 5.1 Dedicated Instances

Instances that run on hardware dedicated to your account, but **you don't see or control the host**. AWS places them; the host may change on stop/start.

- **Use when:** a compliance/regulatory requirement says "no shared hardware" and that's the *entire* requirement.
- **Advantage:** simplest way to satisfy physical-isolation mandates.
- **Disadvantage:** costs more (per-instance premium + per-region fee); no host visibility, so no socket/core-based licensing.

### 5.2 Dedicated Hosts

An **entire physical server** allocated to you, with visibility into its **sockets, cores, and host ID**, and control over instance placement on it.

- **Use when:**
  - **BYOL (Bring Your Own License)** software licensed per-socket/per-core/per-VM — Windows Server, SQL Server, Oracle — where license terms require binding to specific physical hardware.
  - Compliance regimes demanding host-level visibility/affinity.
- **Advantages:** enables license reuse that can dwarf the price premium; **host affinity** keeps an instance on the same physical machine across restarts; can be combined with RIs/Savings Plans for discounts.
- **Disadvantages:** most expensive tenancy; you pay for the whole host regardless of utilization; capacity planning is on you.

| | **Dedicated Instance** | **Dedicated Host** |
|---|---|---|
| Isolation | Hardware dedicated to your account | Entire physical server yours |
| Host visibility/control | None | Sockets, cores, host ID, placement |
| BYOL per-socket/core licensing | No | **Yes — the main reason it exists** |
| Billing | Per instance (+ premium) | Per host |

---

## 6. Capacity Reservations — guaranteeing the capacity exists

**On-Demand Capacity Reservations** reserve capacity in a **specific AZ** so instances can *always* launch there — independent of any billing discount.

- **No commitment required** (create/cancel anytime), but you **pay for the reserved capacity whether or not you use it**.
- Combine with Savings Plans/RIs to get both the discount *and* the guarantee.
- **Use when:** disaster-recovery targets that must be launchable during a regional rush, known traffic events (product launch, Black Friday), or regulated workloads that cannot tolerate `InsufficientInstanceCapacity` errors.

> Zonal RIs, Capacity Blocks (short-term reserved GPU capacity for ML), and On-Demand Capacity Reservations all address *capacity assurance*; only RIs/Savings Plans address *price*. Know which problem you're solving.

---

## 7. Choosing — Decision Summary

| Situation | Choose |
|---|---|
| Unknown/new workload, still measuring | **On-Demand** |
| Steady 24/7 baseline (web fleet, DB tier) | **Savings Plan** (Compute SP for flexibility, EC2 Instance SP / Standard RI for max discount) |
| Fault-tolerant batch, CI, ML training, rendering | **Spot** (with fleet diversification + interruption handling) |
| Mixed steady + variable + interruptible | **Layer them:** SP floor + On-Demand middle + Spot edge |
| "No shared hardware" compliance requirement | **Dedicated Instances** |
| Per-socket/core BYOL licenses (SQL Server, Oracle) | **Dedicated Hosts** |
| Must be able to launch N instances in AZ X during a crisis/event | **Capacity Reservation** (± Savings Plan for the price) |
| Short-term guaranteed GPU capacity for ML | **Capacity Blocks** |

A realistic production fleet uses several at once:

```
Auto Scaling group (100 instances at peak, 40 at floor)
├── 40 × baseline    → covered by Compute Savings Plan   (~66% off)
├── 30 × daily wave  → On-Demand                          (full price)
└── 30 × burst       → Spot, diversified across 6 pools   (~70–90% off)
```

---

## 8. Pitfalls & Best Practices

- **Measure before committing.** Buy Savings Plans/RIs based on observed usage (Cost Explorer's recommendations), not forecasts of hoped-for growth. Undercommit slightly — you can always add more later; you can't shrink a commitment.
- **Prefer Savings Plans over RIs for EC2** unless you specifically need resale, zonal capacity, or are covering RDS/ElastiCache (which only have RIs).
- **Don't buy 3-year commitments for architectures in flux** — or if you must, use Compute Savings Plans / Convertible RIs, which survive re-platforming.
- **RIs/SPs are "use it or lose it"** — the hourly commitment bills whether matching instances run or not. Unused commitment shows up in coverage/utilization reports; watch them.
- **Never treat Spot as discounted On-Demand.** Without interruption handling and pool diversification, Spot *will* eventually bite. Design for the 2-minute notice from day one.
- **Dedicated ≠ cheaper.** It's a compliance/licensing tool that costs more. Choose it only when a rule or license forces you.
- **A discount ≠ guaranteed capacity** (except zonal RIs). If launchability during a crunch matters, add a Capacity Reservation.
- **Right-size first, then commit.** Committing to an oversized instance family locks in the waste at a discount.
