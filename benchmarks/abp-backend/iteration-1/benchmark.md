# Skill Benchmark: abp-backend

**Model**: claude-opus-4-7[1m]  
**Date**: 2026-05-10T12:51:01Z  
**Evals**: 7 (1 run per configuration)

## Summary

| Metric | With Skill | Without Skill | Delta |
|--------|-----------:|--------------:|------:|
| Pass rate | 100% +/- 0% | 70% +/- 40% | +0.30 |
| Time | 77.8s +/- 18.3s | 73.2s +/- 10.6s | +4.6s |
| Tokens | 45,462 +/- 8,940 | 22,341 +/- 1,066 | +23120 |

## Per-eval breakdown

| Eval | With | Without | d pass | d time | d tokens |
|------|-----:|--------:|-------:|-------:|---------:|
| manager-injects-tracking-repo | 5/5 | 0/5 | +1.00 | -5.7s | +12,373 |
| appservice-uses-iqueryable | 4/4 | 4/4 | +0.00 | -18.6s | +29,001 |
| cross-module-read | 6/6 | 6/6 | +0.00 | +10.7s | +15,675 |
| controller-on-integration-service | 4/4 | 2/4 | +0.50 | +11.2s | +32,853 |
| add-module-dbcontext | 5/5 | 2/5 | +0.60 | +26.6s | +32,026 |
| controller-injecting-repo | 4/4 | 4/4 | +0.00 | -2.6s | +16,951 |
| businessexception-in-appservice | 5/5 | 5/5 | +0.00 | +10.5s | +22,963 |

## Analyst notes

- Overall pass rate +30 pp (100% with-skill vs 70% baseline) — but the lift is concentrated on 3 of 7 evals; the other 4 are tied at 100/100.
- Strongly differentiating evals (skill clearly wins): manager-injects-tracking-repo (5/5 vs 0/5) — baseline keeps the Manager persisting; controller-on-integration-service (4/4 vs 2/4) — baseline misnames the wiring knobs; add-module-dbcontext (5/5 vs 2/5) — baseline recommends a *shared* host DbContext via [ReplaceDbContext] and explicitly denies that per-module migrations-history tables matter.
- Non-differentiating evals (both pass 100%): appservice-uses-iqueryable, cross-module-read, controller-injecting-repo, businessexception-in-appservice. Baseline LLM knowledge of ABP DDD patterns is sufficient here — the skill adds polish (correct API names, focused answers) but no new correctness. These four prompts are good for regression detection but weak as discriminators of skill value.
- Time cost is small (+4.6s mean, ~6% slower) — reading SKILL.md and references is cheap relative to the core reasoning latency.
- Token cost is significant: with-skill consumes ~2x baseline tokens (45.5k vs 22.3k mean). The skill adds ~23k tokens regardless of whether the eval needed help. On the 4 tied evals, this is pure overhead — the skill loaded for nothing.
- Assertion-quality concern (manager-injects-tracking-repo): the 'CreateAsync returns the new Order without InsertAsync' check would also pass for a lazy fix that doesn't pre-mint the Id in the AppService. Tightening to require the Id-parameter pattern would make this more discriminating.
- Critical baseline failure mode worth flagging: in add-module-dbcontext, the baseline architectural recommendation is the *opposite* of what ABP best-practices say — it would lead a user to share schema across modules and silently breaks if the project later splits. This is exactly the sort of high-blast-radius mistake the skill is designed to catch.
