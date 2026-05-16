# Survey JSON Schema

The runtime is configured entirely from a JSON object (the "schema"). The renderer adds zero keys — every supported key is owned by `survey-core`. The canonical machine-readable schema document is at:

```
https://unpkg.com/survey-core@2.5.20/surveyjs_definition.json
```

You can vendor that file locally for offline validation.

## Top-level shape

```json
{
  "title": "My survey",
  "description": "Optional",
  "logoPosition": "right",
  "showProgressBar": "top",
  "questionsOnPageMode": "standard",
  "completedHtml": "<h3>Thank you!</h3>",
  "completedHtmlOnCondition": [
    { "expression": "{score} >= 7", "html": "<h3>Promoter — thanks!</h3>" }
  ],
  "calculatedValues": [
    { "name": "fullName", "expression": "{firstName} + ' ' + {lastName}", "includeIntoResult": true }
  ],
  "triggers": [
    { "type": "complete", "expression": "{tooBusy} = true" }
  ],
  "pages": [
    {
      "name": "page1",
      "elements": [
        { "type": "text", "name": "firstName", "title": "First name", "isRequired": true }
      ]
    }
  ]
}
```

`questionsOnPageMode` accepts `"standard"` | `"singlePage"` | `"questionPerPage"`. The non-standard modes synthesize a wrapping `PageModel`.

## Built-in question types (21)

Every type listed below is in 2.5.20.

| Type | Value shape | Notes |
|---|---|---|
| `text` | `string` | `inputType: 'email'\|'number'\|'tel'\|'url'\|...` triggers built-in coercion |
| `comment` | `string` | multi-line text |
| `boolean` | `boolean` (or override via `valueTrue`/`valueFalse`) | `booleanValue` exposes underlying boolean |
| `dropdown` | `string` (the chosen `value`) | uses `choices: [{value, text} \| string]` |
| `tagbox` | `string[]` | multi-select dropdown |
| `radiogroup` | `string` | |
| `checkbox` | `string[]` | `storeOthersAsComment` (default `true`) splits "other" into `<name>` + `<name>-Comment` |
| `rating` | `number` | `rateType`, `rateValues` |
| `ranking` | `string[]` (ordered) | drag-to-rank |
| `imagepicker` | `string` or `string[]` | toggled by `multiSelect` |
| `matrix` | `{ row1: 'col1', row2: 'col2', ... }` | rows × columns; per-row choice |
| `matrixdropdown` | `{ row1: { col1: cellValue, col2: ... }, ... }` | rows × typed-cell columns |
| `matrixdynamic` | `[{ col1, col2 }, { col1, col2 }, ...]` | user adds rows |
| `multipletext` | `{ itemName: itemValue, ... }` | grouped string fields |
| `panel` | n/a (container) | no own value; nested `elements` |
| `paneldynamic` | `[{ q1, q2 }, { q1, q2 }, ...]` | user adds panels |
| `expression` | computed via `expression` | read-only display of an expression result |
| `html` | n/a | static HTML, no value |
| `image` | n/a | static image, no value |
| `signaturepad` | data-URL string | render-time DOM (canvas); validation works server-side |
| `file` | `[{ name, type, content }]` or single object | `storeDataAsText: true` → `content` is a base64 data URL; `false` → it's a server-assigned URL/identifier |

## Conditional logic keys (in 2.5.20)

These are present in 2.5.20 — confirmed by grep on the bundled UMD.

**Boolean (ConditionRunner)**:
`visibleIf`, `enableIf`, `requiredIf`, `setValueIf`, `resetValueIf`, `choicesVisibleIf`, `choicesEnableIf`, `rowsVisibleIf`, `columnsVisibleIf`

**Value-producing (ExpressionRunner)**:
`defaultValueExpression`, `setValueExpression`, `runExpression`, `totalExpression`, `expression`, `calculatedValues[].expression`, `triggers[].expression`, `bindings`

**Survey-level**: `navigateToUrlOnCondition`, `completedHtmlOnCondition`

**Not in 2.5.20** (ignore them — they are future/proposed): `defaultValueIf`, `valueExpression`. Don't generate code that references these.

## Validators (built-in)

Question-level `validators: [{ type, ... }]` array. Each validator's JSON shape:

```json
{ "type": "numeric",     "minValue": 0, "maxValue": 100, "text": "Score must be 0–100" }
{ "type": "text",        "minLength": 2, "maxLength": 50, "allowDigits": false }
{ "type": "email" }
{ "type": "regex",       "regex": "^[12]\\d{9}$", "caseInsensitive": false, "text": "Saudi National ID required" }
{ "type": "expression",  "expression": "{age} >= 18", "text": "Must be 18+" }
{ "type": "answercount", "minCount": 1, "maxCount": 3 }
```

`notificationType` (added 2.3.8): `"error"` (default, blocks submit), `"warning"` or `"info"` (don't block).

## Triggers (top-level `triggers`)

| Type | Effect |
|---|---|
| `setvalue` | set `setToName` to `setValue` (literal) |
| `copyvalue` | copy `fromName` to `setToName` |
| `runexpression` | evaluate `runExpression`, optionally store in `setToName` |
| `skip` | jump to `gotoName` |
| `complete` | finalize survey |
| `visible` | show/hide questions in `selectedItems`; **note**: docs prefer per-question `visibleIf` |

All trigger types take an `expression`. `survey.runTriggers()` is the public sync API for re-evaluating on the server side.

## Calculated values

Top-level `calculatedValues: [{ name, expression, includeIntoResult }]`. `CalculatedValue` is a separate class. `includeIntoResult: true` makes the value appear in `survey.data` and `getPlainData()`. Calculated values recompute automatically when `survey.data` changes.

## Choice-based questions: REST loading vs static

Static (always works server-side):

```json
{ "type": "dropdown", "name": "country", "choices": [
  { "value": "sa", "text": "Saudi Arabia" },
  { "value": "ae", "text": "United Arab Emirates" }
]}
```

REST (works in the browser; **does NOT work under ClearScript** — `XMLHttpRequest`/`fetch` are unguarded):

```json
{ "type": "dropdown", "name": "country", "choicesByUrl": {
  "url": "/api/countries",
  "valueName": "code",
  "titleName": "name",
  "path": "data"
}}
```

Lazy:

```json
{ "type": "dropdown", "name": "city", "choicesLazyLoadEnabled": true, "choicesLazyLoadPageSize": 25 }
```

For headless validation, either pre-resolve choices on the .NET side and inject them, or run `stripChoicesByUrl(schemaJson)` (see `references/server-side.md`).

## Multilingual schema

Replace any string property (`title`, `description`, `placeholder`, `requiredErrorText`, `choices[].text`) with a per-locale object:

```json
{ "type": "text", "name": "firstName",
  "title": { "default": "First name", "ar": "الاسم الأول", "fr": "Prénom" },
  "requiredErrorText": { "default": "Required", "ar": "مطلوب" }
}
```

`survey.locale = "ar"` switches active locale. `survey.getUsedLocales()` walks every `LocalizableString` in the tree.

## Quiz mode

Per-question `correctAnswer` plus survey-level `getCorrectAnswerCount()`, `getQuizQuestionCount()`. Use synthetic placeholders in `completedHtml` / `completedHtmlOnCondition`: `{correctAnswers}`, `{incorrectAnswers}`, `{questionCount}`.

Modern timer fields (since v1.12.5): `showTimer`, `timerLocation: 'top'|'bottom'`, `timeLimit`, `timeLimitPerPage`, `timerInfoMode`, `onTimerTick`. Legacy aliases (`showTimerPanel`, `maxTimeToFinish`, `maxTimeToFinishPage`, `onTimer`) are still accepted as input but new schemas should emit the modern names.

## Quick authoring checklist

- [ ] Every question has a unique `name`.
- [ ] Required questions use `isRequired: true` (not validators).
- [ ] Conditional keys reference `{questionName}` or `{panel.fieldName}` (in `paneldynamic`) or `{composite.fieldName}` (in composite types).
- [ ] No `defaultValueIf` / `valueExpression` (not in 2.5.20).
- [ ] If using REST choices, decide whether the schema needs a server-side validation path; if yes, plan to strip or pre-resolve.
- [ ] Quiz scoring uses `correctAnswer`, NOT custom validators.

## Validate by hand

```ts
import { Model } from "survey-core";
const survey = new Model({});
survey.fromJSON(yourJson, { validatePropertyValues: true });
// survey.jsonErrors[] now contains JsonError subclasses for unknown types/properties/required-missing.
```

This is the official type-system check. The same flow exists server-side as `validateSchemaTypes` glue (see `references/server-side.md`).
