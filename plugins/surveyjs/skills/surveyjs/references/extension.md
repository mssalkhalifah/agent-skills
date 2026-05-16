# Custom properties, validators, expression functions, composite types

The four extension points that cover ~95% of authoring needs. All of them are in `survey-core` (MIT) — they work both in the browser and under V8 ClearScript.

**Order matters.** Register extensions BEFORE constructing any `SurveyModel`. In React: do it once at module top or in `useEffect(() => { ... }, [])`. In Angular: do it in your `APP_INITIALIZER` or service constructor. In the V8 ClearScript glue: register at the top of `surveyjs-validation.js` (the IIFE pattern in `references/server-side.md`).

---

## 1) Custom property on a built-in class

Show in Property Grid + serialize to JSON + readable via `getPropertyValue`.

```ts
import { Serializer } from "survey-core";

Serializer.addProperty("question", {
  name: "score:number",        // type-suffix lets the editor pick the right widget
  default: 0,
  category: "general",
  visibleIndex: 9,
  isSerializable: true,
});

// Serializer.addProperty("text", { name: "phoneFormat", choices: ["E.164", "national"], default: "E.164" });
```

`propMeta` keys: `name` (with optional `:type` suffix), `default` / `defaultValue`, `choices`, `isRequired`, `isUnique`, `dependsOn`, `onSetValue`, `onGetValue`, `onExecuteExpression`, `category`, `visibleIndex`, `serializationProperty`, `isCustom`, `isLocalizable`, `isSerializable` (set false to hide from JSON output but keep at runtime), `baseClassName`, `minValue`/`maxValue`/`maxLength`.

`onSetValue(obj, val)` warning: do **not** assign to the same property name inside it (infinite recursion). Use a different storage property or a local cache.

**Survey Creator integration**: nothing extra needed — Property Grid is built off `Serializer` metadata, so the new property appears in the right `category` page automatically.

## 2) Custom validator

```ts
import { SurveyValidator, ValidatorResult, CustomError, Serializer } from "survey-core";

class CrossFieldValidator extends SurveyValidator {
  getType() { return "crossfield"; }
  validate(value: any, name?: string, values?: any) {
    if (values?.startDate && values?.endDate && new Date(values.startDate) > new Date(values.endDate)) {
      return new ValidatorResult(value, new CustomError("End date must be after start date", this.errorOwner));
    }
    return null;  // null === pass
  }
}

Serializer.addClass("crossfield", [], () => new CrossFieldValidator(), "surveyvalidator");
```

Now any question can declare `"validators": [{ "type": "crossfield" }]`. The 4th argument to `addClass` (`"surveyvalidator"`) is what makes it discoverable as a validator.

For server-side use under V8 ClearScript, register it inside `surveyjs-validation.js` using the `Survey.*` globals (no `import`). See the `// Custom validator class via Serializer.addClass` snippet in `references/server-side.md`.

## 3) Custom expression function (for `visibleIf` / `expression` / etc.)

```ts
import { FunctionFactory } from "survey-core";

FunctionFactory.Instance.register("isSaudiId", function (params: any[]) {
  return /^[12]\d{9}$/.test(String((params && params[0]) || ""));
}, /*isAsync*/ false);

// Survey JSON: "visibleIf": "isSaudiId({nationalId})"
```

Inside the function body, `this.survey` is the SurveyModel. **Async functions** (`isAsync: true`) use `this.returnResult(value)` and don't work under the V8 ClearScript sync bridge — register sync and prefetch any data you need into `dataJson` before invoking the glue.

## 4) Composite (custom) question type — no class needed

`ComponentCollection.Instance.add(componentSpec)` lets you define a "type" purely from JSON. Two flavors:

### Specialized — extend a single built-in type with locked properties (FLAT data shape)

```ts
import { ComponentCollection } from "survey-core";

ComponentCollection.Instance.add({
  name: "saudi-id",
  title: "Saudi National ID",
  iconName: "icon-text",
  questionJSON: {
    type: "text",
    inputType: "tel",
    maxLength: 10,
    validators: [{ type: "regex", regex: "^[12]\\d{9}$", text: "Saudi National ID required" }]
  },
  inheritBaseProps: true   // expose `text` props in Property Grid (or pass an array to whitelist)
});

// Survey JSON: { "type": "saudi-id", "name": "natId" }
// survey.data: { natId: "1234567890" }
```

### Composite — wrap a panel of nested questions (NESTED data shape)

```ts
ComponentCollection.Instance.add({
  name: "fullname",
  title: "Full name",
  elementsJSON: [
    { type: "text", name: "firstName", title: "First", isRequired: true },
    { type: "text", name: "lastName",  title: "Last",  isRequired: true }
  ]
});

// Survey JSON: { "type": "fullname", "name": "person" }
// survey.data: { person: { firstName: "...", lastName: "..." } }
// Conditional refs use {composite.firstName}: e.g. "visibleIf": "{person.firstName} notempty"
```

Lifecycle callbacks you can attach to the spec:

| Callback | When |
|---|---|
| `onInit()` | type-level; ideal place for `Serializer.addProperty` calls related to this type |
| `onCreated(question)` | per-instance, after construction |
| `onLoaded(question)` | after JSON load |
| `onPropertyChanged(question, name, newValue)` | property-level change |
| `onValueChanging(question, name, newValue)` | mutate `newValue` if needed (sync only) |
| `onValueChanged(question, name, newValue)` | post-change side effects |
| `onItemValuePropertyChanged(...)` | choice-array mutations |
| `onUpdateQuestionCssClasses(...)` | browser-only; do not include in glue |
| `onAfterRender(...)` / `onAfterRenderContentElement(...)` | browser-only; do not include in glue |

Do **not** include `onAfterRender*` callbacks in registrations made server-side under ClearScript — they expect DOM and will throw at render time. Server-side, omit them.

## 5) Custom Question subclass (the rare alternative)

Only needed when composite/specialized via `ComponentCollection` is insufficient (e.g., you need new rendering logic, not just a different schema).

```ts
import { Question, Serializer, ElementFactory } from "survey-core";

class QuestionColorPicker extends Question {
  getType() { return "colorpicker"; }
  // override input shape, value coercion, etc.
}

Serializer.addClass(
  "colorpicker",
  [{ name: "format", default: "hex", choices: ["hex", "rgba", "hsl"] }],
  () => new QuestionColorPicker(""),
  "question"
);
ElementFactory.Instance.registerElement("colorpicker", (name) => new QuestionColorPicker(name));
```

You then register a renderer per framework:

- React: `ReactQuestionFactory.Instance.registerQuestion("colorpicker", (props) => <MyColorRenderer {...props} />)`
- Angular: `AngularComponentFactory.Instance.registerComponent("colorpicker", MyColorComponent)`

Server-side under ClearScript, the model class is enough — no renderer needed; `survey.validate()` operates on the model.

## Where to put the registrations

| Surface | Where |
|---|---|
| Browser-only (React) | top-level module side-effect, or `useEffect(() => { ... }, [])` once |
| Browser-only (Angular) | `APP_INITIALIZER` factory, or a service constructor with `providedIn: 'root'` |
| Browser AND server (V8 ClearScript) | duplicate the registrations: in your bootstrapping module *and* in `surveyjs-validation.js` glue, keeping the JS bodies identical so behavior is consistent across the bridge |

The "duplicate the same registration in both places" is the friction the .NET host's JSON-string bridge buys you back: there's no shared module system between Node-style imports (browser) and the embedded glue (server). Keep them in sync by sourcing both from a single canonical JS file when possible.
