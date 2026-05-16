# Server-side / V8 ClearScript

This is the keystone domain. Treat the project's existing V8 ClearScript module as the contract; do not invent a new architecture. Find it by searching the working tree for `V8EnginePool.cs` and `surveyjs-validation.js`. If those files don't exist, the project hasn't set up server-side SurveyJS yet — ask the user how they want to scaffold one before generating code.

## Contract recap

- **Engine**: `V8ScriptEngine` constructed with `V8ScriptEngineFlags.DisableGlobalMembers` (no host-CLR globals leak into JS).
- **Pool**: `ConcurrentBag<V8ScriptEngine>` — rent per request, return on completion. Pool grows on demand; only drains on `Dispose`.
- **Bootstrap**: each engine `engine.Execute(...)` loads in fixed order: console-stub (4 lines, see below) → `survey.core.min.js` (UMD) → `surveyjs-validation.js` (glue). Both are embedded resources of the .NET project (declared via `EmbeddedResource` in the csproj).
- **Bridge**: every glue function takes JSON strings, returns a JSON string. Sync only. No host objects, no callbacks, no Promises, no async.
- **Cold-start**: ~50–150 ms to parse the 1.4 MB UMD on first engine creation. Pool reuse amortizes this.

## Console stub (mandatory)

`survey-core` 2.5.20 has 3 unguarded `console.warn` calls in deprecation/parse-warning paths. Without a stub, those throw `ReferenceError: console is not defined` and crash subsequent calls. Prepend this before loading `survey.core.min.js`:

```javascript
// console-stub.js — embedded resource, loaded BEFORE survey.core.min.js
(function () {
    var noop = function () {};
    globalThis.console = globalThis.console || {
        log: noop, warn: noop, error: noop, info: noop, debug: noop, trace: noop
    };
})();
```

In `V8EnginePool.CreateEngine`:

```csharp
var engine = new V8ScriptEngine(V8ScriptEngineFlags.DisableGlobalMembers);
engine.Execute("console-stub.js", _consoleStubSource);  // NEW
engine.Execute("survey.core.min.js", _surveyJsSource);
engine.Execute("surveyjs-validation.js", _glueSource);
return engine;
```

If the user's `V8EnginePool.cs` doesn't already do this, add it — without it, the moment a deprecated property is set on a Question, every subsequent call fails.

## Canonical replacement glue file

This is the complete `surveyjs-validation.js` the user should arrive at. It preserves the existing `validateSchema` verbatim and adds the functions that lift the validator from "schema structure ok?" to "does data pass validators + conditional logic + computed values?".

```javascript
// surveyjs-validation.js — V8 ClearScript glue for survey-core 2.5.20
// Bridge contract: sync, JSON-string in / JSON-string out, no host objects.

function _serializeError(e) {
    return JSON.stringify({
        ok: false,
        errors: [(e && e.message) ? e.message : String(e)]
    });
}

// PRESERVED — existing structural validator. Do not change behavior.
function validateSchema(schemaJson) {
    try {
        var schema = JSON.parse(schemaJson);
        var survey = new Survey.Model(schema);
        var pages = survey.pages;
        if (!pages || pages.length === 0) {
            return JSON.stringify({ isValid: false, errors: ["Schema must contain at least one page with elements."] });
        }
        var hasElements = false;
        for (var i = 0; i < pages.length; i++) {
            if (pages[i].elements && pages[i].elements.length > 0) { hasElements = true; break; }
        }
        if (!hasElements) {
            return JSON.stringify({ isValid: false, errors: ["Schema must contain at least one element."] });
        }
        return JSON.stringify({ isValid: true, errors: [] });
    } catch (e) {
        return JSON.stringify({ isValid: false, errors: [(e && e.message) ? e.message : String(e)] });
    }
}

// NEW — type-system validation via Serializer.
// Detects unknown question types, unknown properties, and surfaces survey.jsonErrors.
function validateSchemaTypes(schemaJson) {
    try {
        var schema = JSON.parse(schemaJson);
        var survey = new Survey.Model({});
        survey.fromJSON(schema, { validatePropertyValues: true });
        var jsonErrors = (survey.jsonErrors || []).map(function (e) {
            return {
                type: e.type || (e.getType ? e.getType() : "unknown"),
                message: e.message || (e.getText ? e.getText() : String(e)),
                element: e.element ? (e.element.name || e.element.getType()) : null,
            };
        });
        return JSON.stringify({ isValid: jsonErrors.length === 0, errors: jsonErrors });
    } catch (e) { return _serializeError(e); }
}

// NEW — full data validation. Walks every visible question and runs every validator + isRequired check.
// Returns per-question, per-validator entries.
function validateData(schemaJson, dataJson) {
    try {
        var schema = JSON.parse(schemaJson);
        var data = JSON.parse(dataJson);
        var survey = new Survey.Model(schema);
        survey.mergeData(data);  // overlay; preserves defaults
        var ok = survey.validate(/*fireCallback*/ false, /*focusOnFirstError*/ false);
        var errors = [];
        var questions = survey.getAllQuestions();
        for (var i = 0; i < questions.length; i++) {
            var q = questions[i];
            if (!q.errors || q.errors.length === 0) continue;
            for (var j = 0; j < q.errors.length; j++) {
                var err = q.errors[j];
                errors.push({
                    question: q.name,
                    type: q.getType(),
                    notificationType: err.notificationType || "error",
                    message: err.getText ? err.getText() : String(err)
                });
            }
        }
        return JSON.stringify({ isValid: ok, errors: errors });
    } catch (e) { return _serializeError(e); }
}

// NEW — evaluate a single expression against a data context.
function evaluateExpression(expr, contextJson) {
    try {
        var ctx = contextJson ? JSON.parse(contextJson) : {};
        var runner = new Survey.ExpressionRunner(expr);
        var value = runner.run(ctx);
        return JSON.stringify({ ok: true, value: value });
    } catch (e) { return _serializeError(e); }
}

// NEW — evaluate a boolean condition.
function evaluateCondition(expr, contextJson) {
    try {
        var ctx = contextJson ? JSON.parse(contextJson) : {};
        var runner = new Survey.ConditionRunner(expr);
        var value = runner.run(ctx);
        return JSON.stringify({ ok: true, value: !!value });
    } catch (e) { return _serializeError(e); }
}

// NEW — compute survey-level calculatedValues against a data set.
// Honors includeIntoResult per CalculatedValue entry.
function computeCalculatedValues(schemaJson, dataJson) {
    try {
        var schema = JSON.parse(schemaJson);
        var data = JSON.parse(dataJson);
        var survey = new Survey.Model(schema);
        survey.mergeData(data);
        var out = {};
        var calc = survey.calculatedValues || [];
        for (var i = 0; i < calc.length; i++) {
            var cv = calc[i];
            try { out[cv.name] = cv.value; } catch (_) { /* ignore */ }
        }
        return JSON.stringify({ ok: true, values: out });
    } catch (e) { return _serializeError(e); }
}

// NEW — quiz scoring. Returns per-question correct/incorrect against question.correctAnswer.
function scoreQuiz(schemaJson, dataJson) {
    try {
        var schema = JSON.parse(schemaJson);
        var data = JSON.parse(dataJson);
        var survey = new Survey.Model(schema);
        survey.mergeData(data);
        var total = survey.getQuizQuestionCount();
        var correct = survey.getCorrectAnswerCount();
        var per = [];
        var questions = survey.getAllQuestions();
        for (var i = 0; i < questions.length; i++) {
            var q = questions[i];
            if (q.correctAnswer === undefined || q.correctAnswer === null) continue;
            per.push({
                name: q.name,
                expected: q.correctAnswer,
                actual: q.value,
                correct: q.isAnswerCorrect()
            });
        }
        return JSON.stringify({ ok: true, correctCount: correct, totalCount: total, perQuestion: per });
    } catch (e) { return _serializeError(e); }
}

// NEW — strip choicesByUrl / lazyLoad before validation, since those need network.
// Returns a normalized schema JSON string the host can pass to validateData/validateSchema/etc.
function stripChoicesByUrl(schemaJson) {
    try {
        var schema = JSON.parse(schemaJson);
        function walk(node) {
            if (!node || typeof node !== "object") return;
            if (Array.isArray(node)) { for (var k = 0; k < node.length; k++) walk(node[k]); return; }
            if (node.choicesByUrl) { delete node.choicesByUrl; if (!node.choices) node.choices = []; }
            if (node.choicesLazyLoadEnabled) node.choicesLazyLoadEnabled = false;
            for (var key in node) if (Object.prototype.hasOwnProperty.call(node, key)) walk(node[key]);
        }
        walk(schema);
        return JSON.stringify({ ok: true, schema: schema });
    } catch (e) { return _serializeError(e); }
}
```

## C# host extension pattern

For each new glue function, add a manager method that rents an engine, invokes, deserializes, and returns. Keep the rent/return symmetric in `try/finally`.

```csharp
public sealed record DataValidationResult(bool IsValid, DataValidationError[] Errors);
public sealed record DataValidationError(string Question, string Type, string NotificationType, string Message);

public DataValidationResult ValidateData(JsonElement schema, JsonElement data)
{
    var schemaJson = schema.GetRawText();
    var dataJson = data.GetRawText();
    var engine = enginePool.Rent();
    try
    {
        var resultJson = (string)engine.Invoke("validateData", schemaJson, dataJson);
        return JsonSerializer.Deserialize<DataValidationResult>(resultJson, JsonOptions)
               ?? throw new InvalidOperationException("Failed to deserialize ValidateData result.");
    }
    finally
    {
        enginePool.Return(engine);
    }
}
```

Mirror `EnsureSchemaIsValid` for the validator pattern: throw `BusinessException(FormErrorCodes.InvalidSchema)` (or a new code like `InvalidData`) with `.WithData("errors", string.Join("; ", result.Errors.Select(e => $"{e.Question}: {e.Message}")))`.

## Settings preset (apply once at engine bootstrap)

These mutations make survey-core safer in headless mode. They're optional but recommended — apply inside `surveyjs-validation.js` IIFE at module top:

```javascript
(function configureSurveyForServer() {
    if (typeof Survey === "undefined" || !Survey.settings) return;
    var s = Survey.settings;
    s.animationEnabled = false;
    s.autoAdvanceDelay = 0;
    s.confirmActionAsync = undefined;
    s.confirmActionFunc = function () { return true; };
    if (s.web) s.web.cacheLoadedChoices = false;
    if (s.lazyRender) s.lazyRender.enabled = false;
    if (s.notifications) s.notifications.lifetime = 0;
    // Defensive: settings.environment getters touch document on read.
    Object.defineProperty(s, "environment", {
        configurable: true, enumerable: true,
        get: function () { return { root: null, rootElement: null, popupMountContainer: null, svgMountContainer: null, stylesSheetsMountContainer: null }; }
    });
})();
```

## Concurrent-validation test pattern

Mirror the existing `Should_Handle_Concurrent_Validations` for any new glue function. The test proves the pool is safe for that function's call shape:

```csharp
[Fact]
public async Task ValidateData_Should_Handle_Concurrent_Calls()
{
    var schema = JsonDocument.Parse("""
    {"pages":[{"elements":[{"type":"text","name":"name","isRequired":true}]}]}
    """).RootElement;
    var data = JsonDocument.Parse("""{"name":"Jane"}""").RootElement;

    var tasks = new Task<DataValidationResult>[10];
    for (var i = 0; i < tasks.Length; i++)
        tasks[i] = Task.Run(() => _manager.ValidateData(schema, data));
    await Task.WhenAll(tasks);
    foreach (var t in tasks) (await t).IsValid.ShouldBeTrue();
}
```

## Allowed registrations at glue load time

The glue file is the right place to register custom expression functions, custom validators, and composite question types — they need to exist before `new Survey.Model(schema)` runs. Patterns:

```javascript
// Sync custom function (Survey.FunctionFactory.Instance.register; isAsync = false)
Survey.FunctionFactory.Instance.register("isSaudiId", function (params) {
    var v = (params && params[0]) || "";
    return /^[12]\d{9}$/.test(String(v));
}, /*isAsync*/ false);

// Custom validator class via Serializer.addClass
function CrossFieldValidator() { Survey.SurveyValidator.call(this); }
CrossFieldValidator.prototype = Object.create(Survey.SurveyValidator.prototype);
CrossFieldValidator.prototype.getType = function () { return "crossfield"; };
CrossFieldValidator.prototype.validate = function (value, name, values, properties) {
    if (values && values.startDate && values.endDate && values.startDate > values.endDate) {
        return new Survey.ValidatorResult(value, new Survey.CustomError("End date must be after start date", this.errorOwner));
    }
    return null;
};
Survey.Serializer.addClass("crossfield", [], function () { return new CrossFieldValidator(); }, "surveyvalidator");

// Composite type via ComponentCollection (use elementsJSON for nested data shape, questionJSON for flat)
Survey.ComponentCollection.Instance.add({
    name: "fullname",
    title: "Full name",
    elementsJSON: [
        { type: "text", name: "firstName", title: "First", isRequired: true },
        { type: "text", name: "lastName",  title: "Last",  isRequired: true }
    ]
    // NEVER include onAfterRender / onAfterRenderContentElement under ClearScript — they need DOM.
});
```

## Anti-patterns (refuse server-side)

| API | Why it breaks | Sync alternative |
|---|---|---|
| `survey.onServerValidateQuestions.add(...)` | callback model, needs event-loop pump | sync custom `SurveyValidator` or sync `FunctionFactory` |
| `choicesByUrl` / `choicesLazyLoad` (runtime) | uses `XMLHttpRequest`/`fetch`, unguarded → `ReferenceError` | `stripChoicesByUrl(schemaJson)` preprocessor or pre-resolve on .NET side |
| `FunctionFactory.Instance.register(name, fn, true)` | async path resolves via `this.returnResult`, never fires under sync bridge | prefetch dependency data into `dataJson`, register sync |
| `survey.startTimer()` | `setInterval`, ticks never fire | don't call server-side; quiz-mode timer is browser-only |
| `survey.applyTheme(...)` | DOM styles | apply on the browser side; ITheme can still be JSON-validated via a `validateThemeJson` glue |
| `onAfterRender*`, popup, focus, scroll | DOM-only | don't use server-side |
| `survey.data = X` for partial host data | wholesale replace, drops defaults | `survey.mergeData(patch)` |
| `file` question with `storeDataAsText: false` | host can't resolve URL identifier | accept data-URL form (`storeDataAsText: true`) or pre-resolve to data URL |

## Quick reference

- **Canonical schema**: `https://unpkg.com/survey-core@2.5.20/surveyjs_definition.json`
- **Sync validate entry**: `survey.validate(false, false)` (fireCallback=false, focusOnFirstError=false)
- **Sync data overlay**: `survey.mergeData(data)` (vs assignment which replaces wholesale)
- **Quiz scoring (sync)**: `survey.getCorrectAnswerCount()`, `survey.getQuizQuestionCount()`, per-question `q.isAnswerCorrect()`
- **Trigger replay (sync)**: `survey.runTriggers()`
- **Plain output**: `survey.getPlainData({ includeQuestionTypes: true })`
- **Locale switch**: `survey.locale = 'fr'` (sync, DOM-free) — error messages then localize via `SurveyError.getText()`
