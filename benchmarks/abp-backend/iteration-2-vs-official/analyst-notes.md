# Analyst notes — abp-backend vs official ABP skills

**Workspace:** `iteration-2-vs-official/`
**Date:** 2026-05-11
**Setup:** 7 evals × 3 runs × 2 configs = 42 runs. Baseline = all 18 official skills from `github.com/abpframework/abp/tree/dev/.claude/skills` (read in parallel; subagent picked the relevant ones per task).

## Headline

| Config | Total expectations passed | Pass rate | Mean time / run | Mean tokens / run |
|---|---|---|---|---|
| `with_user_skill` (abp-backend) | **111 / 111** | **100.0%** | 131.6 s | 46,838 |
| `with_official_skill` (18 ABP skills) | **102 / 111** | **91.9%** | 133.5 s | 42,497 |
| **Delta** | **+9** | **+8.1 pp** | -1.9 s (effectively tied) | +4,341 (~10% heavier) |

## Where the gap lives

All 9 failures are on **eval-0 `manager-injects-tracking-repo`**. The other 6 evals are dead even at 100%.

On eval-0 the official-skill runs consistently miss three rules the user's skill codifies as Rule 3:

| Assertion (user-skill rule) | User | Official | Why official misses |
|---|---|---|---|
| Manager injects `IReadOnlyRepository<T, Guid>`, never the tracking `IRepository` | 3/3 | 0/3 | The official `abp-ddd` skill describes the domain-service pattern but doesn't enforce the read-only-vs-tracking repository distinction in Manager DI. |
| `ChangePriceAsync` takes `Order` by parameter, doesn't reload via `GetAsync(id)` inside the Manager | 3/3 | 0/3 | Two of three official runs delete `ChangePriceAsync` from the Manager outright ("it has no cross-aggregate logic") — defensible but doesn't match the assertion. |
| AppService mints the `Id` via inherited `GuidGenerator`, then calls the Manager | 3/3 | 0/3 | Official runs leave `GuidGenerator.Create()` inside the Manager. The official `abp-ddd` example does mention "don't generate GUIDs in entity constructors" but stops short of locating the mint in the AppService. |

These are real architectural opinions, not noise: every official run failed the same three.

## Honest caveats

1. **Assertion provenance.** The 7 evals and their assertions were written *from the user's skill's perspective* (Rule numbers cited verbatim in the assertions, e.g. "places `[Authorize]` at the AppService layer per Rule 4"). A skill that codified those rules was always favoured. The eval-0 result reflects a genuine extra rule in the user skill, but a fully unbiased benchmark would also need eval prompts authored from the official-skill perspective.

2. **Cost difference is mild.** User-skill runs spend ~10% more tokens on average. Worth it given the win on eval-0; cheaper if you only ever ask the 6 evals that are already ties.

3. **Two evals showed minor official-skill upsides** that the assertions didn't reward:
   - Several official runs flagged the *cross-aggregate `IRepository<Customer>` injection* as a "reference other aggregates by Id only" violation. The user's skill doesn't have this rule and the assertion doesn't ask for it.
   - Official runs frequently default to **Mapperly** for object mapping (the ABP 10.3 default), while user-skill runs stick with `IObjectMapper`. Neither was scored.

4. **Variance is misleading at this sample size.** With 3 runs per cell, the pass-rate stddev is dominated by all-or-nothing per-eval outcomes. Bumping to 5 runs would tighten the official-skill stddev but is unlikely to change the headline.

## Recommended next steps

- Add 2-3 eval prompts written from the **official skill's angle** (cross-aggregate repo injection; Mapperly vs AutoMapper; auto-API controllers vs hand-written) — re-benchmark to remove the assertion-bias asymmetry.
- Keep the user's Rule 3 wording (`IReadOnlyRepository<T, Guid>` in Managers; mint Id in AppService). It's the only place this benchmark surfaces a *substantive* content advantage.
- The 10% token premium is mostly the consolidated SKILL.md plus its references file. If size matters, a description-optimization pass on the user skill could trim it.
