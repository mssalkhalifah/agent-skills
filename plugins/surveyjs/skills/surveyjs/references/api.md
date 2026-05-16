# Survey CORE API surface (the parts you'll actually call)

`survey-core` is MIT and framework-agnostic. Everything below is sync, DOM-free, and works under V8 ClearScript unless explicitly noted.

## SurveyModel

```ts
import { Model } from "survey-core";
const survey = new Model(json);
```

| Member | Description |
|---|---|
| `data` | the master answer object. **Wholesale assignment replaces everything (drops defaults)** — prefer `mergeData()`. |
| `mergeData(patch)` | overlay; preserves keys not in `patch`. |
| `getValue(name)` / `setValue(name, value)` | single-question access. |
| `getPlainData(options?)` | array of `{ name, title, value, displayValue, isNode, data: nested[] }`. Pass `{ includeQuestionTypes: true }` to also get `questionType`. |
| `getFilteredValues()` | merged values + variables + calculated, used by the expression engine. |
| `validate(fireCallback?, focusOnFirstError?, onAsyncValidation?)` | walks every visible question. Sync when no async function is registered. **Use `validate(false, false)` server-side** — `true` invokes browser-only focus/scroll. |
| `hasErrors(...)` | inverse of `validate`. |
| `validateCurrentPage()` / `hasErrorsOnCurrentPage()` | scoped to current page. |
| `tryComplete()` | v2-recommended completion: validate then complete. |
| `doComplete(isCompleteOnTrigger?)` | bypass validation. |
| `completeLastPage()` | obsolete alias of `tryComplete()` (still accepted). |
| `runTriggers()` | force trigger re-evaluation. |
| `getAllQuestions()` | every Question instance, including in panels/dynamic panels. |
| `getCorrectAnswerCount()` / `getQuizQuestionCount()` | quiz scoring. |
| `pages` / `currentPage` | array of `PageModel` / current page. |
| `state` | `'empty' \| 'loading' \| 'starting' \| 'running' \| 'completed' \| 'completedbefore' \| 'preview' \| 'restoring'`. |
| `locale` | active locale code; setting it switches messages. |
| `getUsedLocales()` | every locale code referenced in any LocalizableString. |

## Class hierarchy

```
Base (root: getPropertyValue/setPropertyValue, registerPropertyChangedHandlers, addOnPropertyValueChangedCallback, addOnArrayChangedCallback, dispose, isDescendantOf)
└── SurveyElementCore (title, description, cssClasses)
    └── SurveyElement (name, parent, survey, errors, visible, isReadOnly, focus*)
        ├── Question (and all 21 type-specific subclasses)
        ├── PanelModelBase
        │   ├── PageModel
        │   └── PanelModel
        └── (others)
ItemValue extends Base directly (not a SurveyElement) — primitive used by choices/rows/columns.
```

`Base.registerPropertyChangedHandlers([propertyNames], handler, key)` is the modern observability API. The legacy `registerFunctionOnPropertyValueChanged` still works but isn't preferred.

## Serializer (the JSON metadata registry)

```ts
import { Serializer } from "survey-core";
```

| Method | Use |
|---|---|
| `addClass(name, propMeta, factory, baseClassName)` | register a new class (e.g., a custom validator) |
| `addProperty(className, propMeta)` | add a custom property to an existing class so it shows in Property Grid + serializes |
| `removeProperty(className, name)` | remove a property |
| `findClass(name)` | class metadata or null (use to detect unknown question types) |
| `findProperty(className, name)` | property metadata or null |
| `getProperties(className)` / `getAllPropertiesByName(name)` | introspect |

`JsonObjectProperty` (the property-meta record): `name`, `type`, `default`/`defaultValue`, `choices`, `isRequired`, `isUnique`, `isSerializable`, `dependsOn`, `onSetValue`, `onGetValue`, `onExecuteExpression`, `category`, `visibleIndex`, `serializationProperty`, `isCustom`, `isLocalizable`, `baseClassName`, `minValue`/`maxValue`/`maxLength`.

Round-trip: `survey.fromJSON(json, { validatePropertyValues: true })` populates `survey.jsonErrors[]` with `JsonError` subclasses (`incorrectvalue`, `unknownproperty`, `unknowntype`, `requiredproperty`). This is the canonical type-system validator. (Note: duplicate-name detection is **not** exposed in 2.5.20 per issue #10741.)

## ExpressionRunner / ConditionRunner / FunctionFactory

```ts
import { ExpressionRunner, ConditionRunner, FunctionFactory } from "survey-core";

const er = new ExpressionRunner("{age} + 1");
er.run({ age: 17 }); // → 18

const cr = new ConditionRunner("{age} >= 18 and {country} = 'KSA'");
cr.run({ age: 20, country: "KSA" }); // → true
```

The parser is **Peggy-generated PEG**, not `eval` / `new Function`. CSP-safe and ClearScript-safe.

Operators: `+ - * / % ^`, `== = != < > <= >=`, `&& and ||or !not negate`, `contains notcontains anyof allof noneof`, `empty notempty`.

### Built-in functions (catalog)

`iif(cond, then, else)`, `sum`, `min`, `max`, `count`, `avg`, `round`, `trunc`, `sumInArray`, `maxInArray`, `minInArray`, `avgInArray`, `countInArray`, `getDate`, `age`, `dateDiff`, `dateAdd`, `currentDate`, `today`, `year`, `month`, `day`, `weekday`, `getYear`, `currentYear`, `diffDays`, `isContainerReady`, `isDisplayMode`, `displayValue` (async, useCache=false), `propertyValue`, `substring`, `getComment`.

### Register a custom function

```ts
FunctionFactory.Instance.register("isSaudiId", function (params) {
  return /^[12]\d{9}$/.test(String((params && params[0]) || ""));
}, /*isAsync*/ false);
```

Inside the function, `this.survey` is the survey instance, and `this.returnResult(value)` is for async (callback-based) functions. **Async functions don't work under the V8 ClearScript sync bridge** — register them sync and prefetch dependencies into `dataJson`.

## Validators

`SurveyValidator` is the abstract base. Built-in concrete classes:

`NumericValidator`, `TextValidator`, `EmailValidator`, `RegexValidator`, `ExpressionValidator`, `AnswerCountValidator`. Each implements `validate(value, name?, values?, properties?)` returning `ValidatorResult` (with optional `SurveyError`) or `null` for success.

`notificationType` (added 2.3.8): `"error"` (default — blocks submit), `"warning"`, `"info"` (don't block).

Custom validator:

```ts
import { SurveyValidator, ValidatorResult, CustomError, Serializer } from "survey-core";

class CrossFieldValidator extends SurveyValidator {
  getType() { return "crossfield"; }
  validate(value: any, name?: string, values?: any) {
    if (values?.startDate && values?.endDate && values.startDate > values.endDate) {
      return new ValidatorResult(value, new CustomError("End date must be after start date", this.errorOwner));
    }
    return null;
  }
}
Serializer.addClass("crossfield", [], () => new CrossFieldValidator(), "surveyvalidator");
```

## settings (the global config singleton)

```ts
import { settings } from "survey-core";
settings.animationEnabled = false;
settings.confirmActionFunc = () => true;
```

`settings` is one of the few survey-core modules that's import-safe under V8 ClearScript because the only DOM references are LAZY getters on `settings.environment` (root, rootElement, popupMountContainer, svgMountContainer, stylesSheetsMountContainer) that dereference `document` only when *read*. Mutating settings is safe; reading `settings.environment` is not.

For server-side config, see the `configureSurveyForServer()` IIFE in `references/server-side.md`.

## Events on SurveyModel (most-used)

`onValueChanged`, `onValueChanging`, `onValidateQuestion`, `onValidatedErrorsOnCurrentPage`, `onComplete`, `onCompleting`, `onCurrentPageChanging`, `onCurrentPageChanged`, `onTextMarkdown`, `onMatrixCellValueChanged`, `onDynamicPanelItemValueChanged`, `onAfterRenderQuestion` (DOM, browser-only), `onUpdateQuestionCssClasses` (browser-only), `onUploadFiles` / `onDownloadFile` / `onClearFiles`, `onBeforeRequestChoices`, `onChoicesLazyLoadCallback`, `onPopupVisibleChanged`.

The async-by-design events (`onServerValidateQuestions`, file upload lifecycle, async REST choices) **do not work under the sync V8 ClearScript bridge** — see `references/server-side.md` for sync alternatives.

## Localization

```ts
import { surveyLocalization } from "survey-core";
surveyLocalization.defaultLocale = "ar";
surveyLocalization.setupLocale({ localeCode: "ar-eg", strings: { /* ... */ }, nativeName: "العربية (مصر)", rtl: true });
```

`surveyLocalization` is sync and DOM-free — fits the ClearScript bridge. The Creator UI's localization (`editorLocalization` from `survey-creator-core`) is browser-only.

## Themes

```ts
import { ContrastLight } from "survey-core/themes";
survey.applyTheme(ContrastLight);
```

ITheme: `{ themeName, colorPalette: 'light'|'dark', isPanelless, cssVariables: Record<string,string>, backgroundImage?, backgroundOpacity?, backgroundImageAttachment?, backgroundImageFit?, headerView?, header?: IHeader }`. All CSS variables prefixed `--sjs-*`; you can add custom `--myorg-*` variables and they round-trip through JSON unchanged.

## Data shapes worth memorizing

```js
// boolean: true | false (or valueTrue/valueFalse override)
// dropdown / radiogroup: <choice value>
// checkbox / tagbox: <choice value>[]
// rating: number
// ranking: <choice value>[] (ordered)
// matrix:        { row1: 'col1', row2: 'col2' }
// matrixdropdown:{ row1: { col1: cellValue, col2: ... }, ... }
// matrixdynamic: [ { col1, col2 }, { col1, col2 } ]
// multipletext:  { itemName: itemValue, ... }
// paneldynamic:  [ { q1, q2 }, { q1, q2 } ]
// signaturepad:  "data:image/png;base64,..."
// file (storeDataAsText:true):  [{ name, type, content: "data:.." }]
// file (storeDataAsText:false): [{ name, type, content: "<server-id>" }]
```
