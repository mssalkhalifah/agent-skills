---
name: surveyjs
description: 'Use this skill whenever the user works with SurveyJS — the JSON-driven form/survey library. Trigger on mentions of SurveyJS, Survey Creator, survey-core, survey-react-ui, survey-angular-ui, survey-creator-* packages, `new Survey.Model`, or survey JSON schemas. Also trigger when authoring or debugging survey JSON (questions, calculatedValues, triggers, visibleIf/enableIf/requiredIf, choicesByUrl, paneldynamic, matrix), customizing Survey Creator (toolbox, property grid, tabs, save hooks), registering custom types or functions via Serializer/ComponentCollection/FunctionFactory, or running survey-core server-side under Microsoft.ClearScript V8 in .NET (schema/data validation, extending surveyjs-validation.js, XMLHttpRequest errors from the bundle). Trigger even on plain-language framings ("multi-page form", "form designer", "calculated field") when SurveyJS or its packages are named. Do NOT trigger for generic form libs (react-hook-form, ajv), other survey vendors (SurveyMonkey, Typeform), or unrelated UI.'
---

# SurveyJS Skill

You are helping the user with **SurveyJS** — the JavaScript library that renders forms (called "surveys") from JSON schemas in Angular and React, plus its visual designer (Survey Creator), and its server-side use under Microsoft.ClearScript V8 in .NET.

## Pinned versions (do not deviate)

- **survey-core**: `2.5.20` (UMD bundle typically vendored at `<dotnet-project>/JavaScript/survey.core.min.js` as an embedded resource)
- **survey-react-ui**: pair with `2.5.20`
- **survey-angular-ui**: pair with `2.5.20` (peer dep `@angular/cdk@^12.0.0`, Angular ≥ 12)
- **survey-creator-core / -react / -angular**: pair with `2.5.20` (commercial license)

If the user is on a newer line and asks about a v3.0 feature (UI Preset Editor, Tailwind/Bootstrap adapters, expression-syntax customization), say so explicitly and offer the closest 2.5.20 equivalent. Don't silently emit code that requires v3.

## Reference implementation (keystone context)

This skill assumes the project contains a .NET module (commonly an ABP module) that hosts `survey-core` inside Microsoft.ClearScript V8 with a sync, JSON-string-in/JSON-string-out bridge. **Every server-side SurveyJS task must extend that baseline, not invent a new one.**

The module's path varies per checkout — find it by searching for these filenames in the working tree (or ask the user):

- `V8EnginePool.cs` — `ConcurrentBag<V8ScriptEngine>` pool, flag `V8ScriptEngineFlags.DisableGlobalMembers`, loads embedded resources in order: `survey.core.min.js` then `surveyjs-validation.js`.
- `JavaScript/surveyjs-validation.js` — the JS glue file. The starting point typically exposes only `validateSchema(schemaJson)` (structural check: page/element non-emptiness). All new server-side capabilities go here, as additional sync functions following the same JSON-string sandwich.
- `JavaScript/survey.core.min.js` — the vendored UMD bundle (target version: 2.5.20).
- `Entities/FormDefinitions/FormDefinitionManager.cs` (or analogous host-side wrapper) — calls `engine.Invoke("<glueFn>", argJsonString)`, deserializes the returned JSON string with `System.Text.Json`. On failure, throw the project's domain exception (typically an ABP `BusinessException` with a code like `FormErrorCodes.InvalidSchema`).
- `Tests/SurveyJs/SurveyJsValidationManager_Tests.cs` — pattern-establishing test, including a 10-task `Task.WhenAll` concurrent-validation case that proves pool safety.

Read those files when in doubt — they are the contract. When generating new C# host methods or new JS glue functions, mirror this style. If you can't find the module in the working tree, ask the user where it lives or whether to scaffold a new one.

## How to route

Identify which domain the user is in and read the matching reference file before drafting code. Do this even if the user's framing is fuzzy — the references contain version-specific detail (deprecated names, exact event payloads, exact value shapes) that you cannot reliably reconstruct from training data alone.

| Domain | When | Read |
|---|---|---|
| **Form Library — React** | Embedding `<Survey/>` in a React app | `references/form-library-react.md` |
| **Form Library — Angular** | Embedding `<survey>` in an Angular app | `references/form-library-angular.md` |
| **Survey Creator (designer)** | Embedding the WYSIWYG builder, customizing toolbox / property grid / tabs / save hooks | `references/creator.md` |
| **Survey JSON schema** | Authoring or hand-editing survey JSON (questions, pages, panels, validators, triggers, calculatedValues) | `references/schema.md` |
| **Survey CORE API** | SurveyModel / Question / Serializer / FunctionFactory / settings — anything that's pure model logic | `references/api.md` |
| **Server-side / V8 ClearScript** | Validating schemas or data on the .NET host, extending `surveyjs-validation.js` | `references/server-side.md` |
| **Custom properties / question types** | `Serializer.addProperty`, `ComponentCollection.Instance.add`, `FunctionFactory.Instance.register` | `references/extension.md` |

For any task that touches the project's V8 ClearScript validator module (the one that owns `V8EnginePool.cs` and `surveyjs-validation.js`), also read `references/server-side.md` regardless of framing. Bridge-contract alignment is non-negotiable.

## Output contract for code generation

When generating SurveyJS code:

1. **State the framework and package version** at the top of the snippet (`// React + survey-react-ui 2.5.20`).
2. **For React surveys**: always memoize the model with `useMemo` or `useState(() => new Model(json))`. Recreating it on every render leaks subscriptions and resets user input.
3. **For Angular surveys**: import `SurveyModule` from `survey-angular-ui`; CSS comes from `survey-core/survey-core.min.css`. Both NgModule and standalone-component setups work — pick based on the user's app.
4. **For schemas**: validate against the canonical JSON Schema at `https://unpkg.com/survey-core@2.5.20/surveyjs_definition.json`. Don't invent property names — confirm via `references/schema.md` or `references/api.md`.
5. **For server-side glue**: every new function in `surveyjs-validation.js` must be sync, accept JSON strings only, return `JSON.stringify({ok|isValid, errors, ...})`, wrap in `try/catch`, and not depend on any host object beyond what `Survey.*` provides. See `references/server-side.md` for the canonical template.

## Anti-patterns to refuse (server-side specifically)

These break the V8 ClearScript bridge contract. If the user asks for them server-side, push back and offer the sync alternative from `references/server-side.md`.

- **`onServerValidateQuestions`** — its callback model needs an event-loop pump that ClearScript's sync bridge does not provide. Use a sync `SurveyValidator` subclass or sync `FunctionFactory.Instance.register(name, fn, /*isAsync*/ false)` instead.
- **`choicesByUrl` / `choicesLazyLoad`** — these issue `XMLHttpRequest`/`fetch` at runtime; both are unguarded in the bundle (will throw `ReferenceError`). Either pre-resolve choices on the .NET side and inject them, or run the `stripChoicesByUrl(schemaJson)` preprocessor before `new Survey.Model(schema)`.
- **Async `FunctionFactory` entries** — `FunctionFactory.Instance.register(name, fn, /*isAsync*/ true)` returns via `this.returnResult(value)`, which never resolves under ClearScript. Convert async rules to sync by prefetching dependencies into `dataJson` before calling the glue.
- **`survey.startTimer()`** — uses `setInterval`. SurveyTimer's interval calls are inside `start()`/`stop()`, not at module load, so the bundle imports cleanly — but never call `startTimer` server-side.
- **`onAfterRender*` events / popup / focus / scroll APIs** — DOM-only, will throw.
- **`storeDataAsText: false` for `file` questions** — the .NET host can't resolve the URL identifier from a file question; either accept the data-URL form or pre-resolve.
- **`survey.data = X` on a partial host payload** — wholesale replace, drops defaults. Use `survey.mergeData(patch)` instead.

## Conditional logic keys reference (cite from memory only when confident)

These keys exist in 2.5.20 (confirmed by grep on the bundle). **`defaultValueIf` and `valueExpression` do NOT exist** in 2.5.20 — they are future/proposed.

- Boolean (ConditionRunner): `visibleIf`, `enableIf`, `requiredIf`, `setValueIf`, `resetValueIf`, `choicesVisibleIf`, `choicesEnableIf`, `rowsVisibleIf`, `columnsVisibleIf`
- Value-producing (ExpressionRunner): `defaultValueExpression`, `setValueExpression`, `runExpression`, `totalExpression`, `expression`
- Survey-level: `navigateToUrlOnCondition`, `completedHtmlOnCondition`, `triggers[].expression`, `calculatedValues[].expression`, `bindings`

## Built-in question types (21 total)

`boolean`, `checkbox`, `comment`, `dropdown`, `tagbox`, `expression`, `file`, `html`, `image`, `imagepicker`, `matrix`, `matrixdropdown`, `matrixdynamic`, `multipletext`, `panel`, `paneldynamic`, `radiogroup`, `rating`, `ranking`, `signaturepad`, `text`. **All 21 are model-load DOM-clean** in 2.5.20 (validators, expressions, value shapes all run server-side under V8). At render time, `signaturepad` needs canvas and `file` needs File/FileReader — neither blocks headless validation.

Value shapes and per-type config detail: see `references/schema.md`.

## Built-in validators

`numeric`, `text`, `email`, `regex`, `expression`, `answercount`. Each validator's JSON shape, options, and `validate(value, owner)` signature are in `references/api.md`. `notificationType` (`error`|`warning`|`info`) added in 2.3.8 and present in 2.5.20 — `warning`/`info` don't block submission.

## When you finish a task

If you generated server-side glue, remind the user that:

1. The new function must be added to the `surveyjs-validation.js` glue file in the project's V8 ClearScript module.
2. A C# wrapper in the form-domain manager should rent an engine, `Invoke` the function, deserialize the JSON string, and return the engine to the pool.
3. The integration tests should grow a 10-task concurrent test for the new function, mirroring the existing concurrent-validation case.
4. The engine pool already loads survey-core once per engine — no rebuild needed.

If the project does not yet contain a V8 ClearScript module of this shape, surface that explicitly and ask whether to scaffold a new one or stop.
