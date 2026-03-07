# Git Branching Strategies

A branching strategy defines **how your team uses Git branches** to develop features, fix bugs, and release code. The two most common strategies are **Trunk-Based Development** and **Git Flow**.

---

## Strategy 1: Trunk-Based Development (TBD)

All developers work on a **single main branch** (the "trunk"). Changes are small, frequent, and merged directly into main.

```
Trunk-Based Development:
════════════════════════════════════════════════════════════════

  main ─────●────●────●────●────●────●────●────●────●──→ (always deployable)
             │         │              │        │
             └─ feat ──┘              └─ fix ──┘
            (1-2 days max)            (hours)
              
  Short-lived branches only - merge back to main within 1-2 days
```

**How it works:**
- `main` is the **single source of truth** and is always deployable
- Developers create **short-lived feature branches** (hours to 1-2 days max)
- Changes are small and merge back into main frequently (at least daily)
- Feature flags are used to hide incomplete features in production
- CI/CD pipeline runs on every merge to main

**When to use:**
- Teams practicing continuous delivery / deployment
- Small to medium, experienced teams
- When you want fast iteration and rapid feedback
- SaaS products, cloud-native apps, startups

**Advantages:**
- Fewer merge conflicts (small, frequent merges)
- Code is always in a deployable state
- Aligns naturally with CI/CD
- Simpler branch management

**Challenges:**
- Requires strong automated testing (bad merge = broken main)
- Requires developer discipline (small commits, no long-lived branches)
- Feature flags add complexity
- Harder with large, less experienced teams

---

## Strategy 2: Git Flow

Uses **multiple long-lived branches** with a strict structure for features, releases, and hotfixes.

```
Git Flow:
════════════════════════════════════════════════════════════════

  main     ─────────────────────●─────────────────────●──→ (production releases only)
                                ↑                     ↑
  release  ──────────── release/v1.0 ──┘    release/v2.0 ──┘
                         ↑                     ↑
  develop  ───●────●────●────●────●────●────●────●────●──→ (integration branch)
               │         │    │              │
               └─ feat/A─┘    └─ feat/B ─────┘
               (days-weeks)    (days-weeks)

  hotfix   ──────────────────────── hotfix/fix-crash ──→ (merges to main + develop)
```

**The branches:**

```
Branch         │ Purpose                              │ Lifetime
───────────────┼──────────────────────────────────────┼─────────────
main           │ Production-ready code only            │ Permanent
develop        │ Integration branch for all features   │ Permanent
feature/*      │ Individual feature development        │ Days to weeks
release/*      │ Prepare and stabilize a release       │ Days
hotfix/*       │ Emergency production fixes            │ Hours to days
```

**How it works:**
- Developers branch off `develop` to create `feature/*` branches
- Completed features merge back into `develop`
- When ready for release, a `release/*` branch is cut from `develop`
- Release branch is tested, stabilized, and bug-fixed
- Once approved, release merges into both `main` (tagged) and `develop`
- Hotfixes branch off `main` and merge back into both `main` and `develop`

**When to use:**
- Teams with scheduled, versioned releases (v1.0, v2.0, etc.)
- Larger teams where strict control over code changes is needed
- Products that support multiple versions simultaneously
- Regulated industries (finance, healthcare) where audit trails matter

**Advantages:**
- Clear separation between in-progress work and production code
- Structured release process with dedicated stabilization
- Parallel development of multiple features without affecting main
- Good for managing multiple production versions

**Challenges:**
- Complex branch management (many long-lived branches)
- Merge conflicts accumulate on long-lived feature branches
- Slower feedback loop (features sit in develop for weeks)
- Pull requests can pile up, slowing velocity
- Does not align well with continuous delivery

---

### Comparison

```
┌──────────────────────┬─────────────────────────┬─────────────────────────┐
│                      │ Trunk-Based Development │ Git Flow                │
├──────────────────────┼─────────────────────────┼─────────────────────────┤
│ Main branch          │ Always deployable       │ Only release-ready code │
│ Feature branches     │ Short-lived (hours-days)│ Long-lived (days-weeks) │
│ Number of branches   │ Minimal (1 + temp)      │ Many (5+ types)         │
│ Merge frequency      │ Multiple times per day  │ Per feature completion  │
│ Merge conflicts      │ Rare (small changes)    │ Common (large changes)  │
│ Release process      │ Continuous deployment   │ Scheduled releases      │
│ CI/CD fit            │ Excellent               │ Moderate                │
│ Team size            │ Small-medium            │ Medium-large            │
│ Developer discipline │ High required           │ Process enforced        │
│ Feature flags needed │ Often                   │ Rarely                  │
│ Used by              │ Google, Meta, Netflix   │ Enterprise / regulated  │
└──────────────────────┴─────────────────────────┴─────────────────────────┘
```

---

### Quick Decision Guide

```
Question                                          → Strategy
────────────────────────────────────────────────────────────────
Do you deploy multiple times per day?             → Trunk-Based
Do you have scheduled releases (v1.0, v2.0)?      → Git Flow
Small, senior team that ships fast?               → Trunk-Based
Large team, multiple features in parallel?        → Git Flow
Continuous delivery / SaaS product?               → Trunk-Based
Regulated industry with audit requirements?       → Git Flow
Early-stage startup / MVP?                        → Trunk-Based
Enterprise with multiple supported versions?      → Git Flow
────────────────────────────────────────────────────────────────
```

> **Real-world trend:** Most modern, high-performing engineering teams (as measured by DORA metrics) are moving toward trunk-based development. Google, Meta, and Netflix all use trunk-based workflows. Git Flow is still widely used in enterprise and regulated environments where scheduled releases and strict controls are required.

---