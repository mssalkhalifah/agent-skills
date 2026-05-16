# Survey Creator (the WYSIWYG designer)

Survey Creator is the visual form builder. Two UI bindings exist for our scope: `survey-creator-react` and `survey-creator-angular`. Both wrap the framework-agnostic `survey-creator-core` (commercial license).

## Licensing (heads up)

`survey-creator-*` packages are **commercial**. Production deployments need a paid SurveyJS license. Since v1.9.115 the only banner-removal mechanism is `creator.setLicenseKey('<key>')` — the legacy `haveCommercialLicense` ICreatorOptions flag has been a no-op since that release. Don't add it to new code.

`survey-core` itself remains MIT, so the runtime forms (and the server-side validator) don't need a license.

## React install

```bash
npm install survey-core@2.5.20 survey-creator-core@2.5.20 survey-creator-react@2.5.20
```

```tsx
import "survey-core/survey-core.min.css";
import "survey-creator-core/survey-creator-core.min.css";
import { SurveyCreator, SurveyCreatorComponent } from "survey-creator-react";

const options = { showLogicTab: true, showThemeTab: true, showJSONEditorTab: true };

export function FormDesigner({ initialJson, onSave }: { initialJson: any; onSave: (json: any) => Promise<void> }) {
  const creator = useMemo(() => {
    const c = new SurveyCreator(options);
    c.setLicenseKey(process.env.SURVEYJS_LICENSE_KEY ?? "");
    if (initialJson) c.JSON = initialJson;
    c.saveSurveyFunc = (saveNo, callback) => {
      onSave(c.JSON).then(() => callback(saveNo, true)).catch(() => callback(saveNo, false));
    };
    return c;
  }, [initialJson, onSave]);
  return <SurveyCreatorComponent creator={creator} />;
}
```

Construct the model **once**. Recreating it loses undo history and event subscriptions.

## Angular install

```bash
npm install survey-core@2.5.20 survey-creator-core@2.5.20 survey-angular-ui@2.5.20 survey-creator-angular@2.5.20 @angular/cdk@^12.0.0
```

```ts
import { Component, OnInit } from "@angular/core";
import { SurveyCreatorModule } from "survey-creator-angular";
import { SurveyCreatorModel } from "survey-creator-core";

@Component({
  selector: "form-designer",
  standalone: true,
  imports: [SurveyCreatorModule],
  template: `<survey-creator [model]="creator"></survey-creator>`,
})
export class FormDesignerComponent implements OnInit {
  creator!: SurveyCreatorModel;
  ngOnInit() {
    this.creator = new SurveyCreatorModel({ showLogicTab: true, showThemeTab: true });
    this.creator.setLicenseKey(/* env-supplied key */ "");
    this.creator.saveSurveyFunc = (saveNo, callback) => {
      this.api.saveDraft(this.creator.text).subscribe({
        next: () => callback(saveNo, true),
        error: () => callback(saveNo, false),
      });
    };
  }
}
```

CSS in `angular.json` styles array: `survey-core/survey-core.min.css` + `survey-creator-core/survey-creator-core.min.css`.

## Tabs (visibility flags + name strings)

| Tab | Flag (default) | `creator.switchTab(name)` |
|---|---|---|
| Designer | always on | `"designer"` |
| Preview | `showPreviewTab: true` | `"preview"` |
| JSON Editor | `showJSONEditorTab: true` | `"json"` |
| Logic | `showLogicTab: false` | `"logic"` |
| Translation | `showTranslationTab: false` | `"translation"` |
| Theme | `showThemeTab: false` | `"theme"` |

Custom tabs: `creator.addTab({ name, plugin, title?, iconName?, componentName?, index? })` — implement `ICreatorPlugin` (`activate()`, `deactivate(): boolean`, `dispose()`, optional `model`/`update()`). The legacy `addPluginTab` API is obsolete in v2.

## Persistence hooks

- `saveSurveyFunc(saveNo, callback)` — fires on autosave (or manual `creator.saveSurvey()`). `saveNo` is monotonic; treat it as an optimistic-concurrency token — drop a `callback(saveNo, true)` if a later `saveNo` already succeeded.
- `saveThemeFunc(saveNo, callback)` — same shape, for theme saves.
- `creator.JSON` (object) and `creator.text` (raw string) — getter/setter pair, round-trips through the survey-core `Serializer`.
- `autoSaveEnabled` (legacy alias `isAutoSave`), `autoSaveDelay` (default 500ms).

## Important events (V2 — strongly typed in `creator-events-api.ts`)

| Event | Use |
|---|---|
| `onModified` | something in the survey JSON changed |
| `onActiveTabChanging` / `onActiveTabChanged` | tab navigation |
| `onSurveyInstanceCreated` | new survey shown (Designer / Preview / Property Grid). Discriminate via `options.area` |
| `onPropertyShowing` (replaces deprecated `onShowingProperty`) | hide/show properties in Property Grid |
| `onCanShowProperty` | gate visibility per element |
| `onElementAllowOperations` | restrict per-element drag/copy/delete |
| `onConditionGetQuestionList` (replaces deprecated `onConditionQuestionsGetList`) | scope the variable picker in conditional-logic UIs |
| `onLogicItemSaved` | a Logic-tab rule was saved |
| `onPropertyEditorCreated` | inject custom UI into the property editor |
| `onBeforePropertyChanged` (replaces deprecated `onPropertyValueChanging`) | veto a property change |

## Toolbox

`creator.toolbox` exposes `QuestionToolbox`. **`ICreatorOptions.questionTypes` is consumed once at construction** — runtime changes do not propagate. To add/remove items at runtime use:

```ts
creator.toolbox.addItem({ name: "fullname", title: "Full name", iconName: "icon-text", json: { type: "fullname" } });
creator.toolbox.removeItem("file");
creator.toolbox.changeCategory("rating", "Feedback");
```

Subitems API (added v1.11.13): `addSubitem`, `removeSubitem`, `clearSubitems`. Toolbox properties like `searchEnabled` (default true), `showCategoryTitles` (default true), `allowExpandMultipleCategories` (default false), `forceCompact`, `overflowBehavior` ('scroll' | 'dropdown') tune layout.

## Property Grid

The Property Grid IS a one-page SurveyModel — `creator.propertyGrid.survey` exposes it. Each `JsonObjectProperty` becomes a question; each `category` becomes a page (or accordion panel when `propertyGridNavigationMode === 'accordion'`). This means the entire Form Library API works for validating/hiding/decorating editors.

Custom property editors: `PropertyGridEditorCollection.register({ fit, getJSON, onCreated, onPropertyChanged })`.

Custom *property* (showing in the grid + serialized to JSON) is `Survey.Serializer.addProperty(...)` — see `references/extension.md`.

## Server-side note

**Don't load `survey-creator-*` under V8 ClearScript.** The Creator constructor unconditionally touches `window`, `document`, `HTMLElement`, `ResizeObserver`. Architecture: Creator runs only in the browser; on save, it POSTs `creator.text` to the .NET host, which validates via the V8 ClearScript module's `validateSchema` glue (and any new sibling glue functions you've added — see `references/server-side.md`).

## Common gotchas

| Symptom | Cause | Fix |
|---|---|---|
| Watermark/banner shows in production | `setLicenseKey` not called or wrong key | call `creator.setLicenseKey('<key>')` immediately after `new SurveyCreatorModel(...)` |
| Adding to `creator.questionTypes` array at runtime has no effect | array is consumed once at construction | use `creator.toolbox.addItem(...)` / `removeItem(...)` |
| Logic Tab not visible | `showLogicTab` defaults to `false` | set `showLogicTab: true` in options |
| `saveSurveyFunc` fires repeatedly on every keystroke | `autoSaveDelay` defaults to 500ms; if your save is slow it queues | raise `autoSaveDelay` or use manual save via a button calling `creator.saveSurvey()` |
| Property Grid hides a custom property | property's `category` doesn't match an existing one and was added to a hidden category | set `category: 'general'` or define a new category via `Serializer.addProperties` |
| Subscriptions disappear after re-render | new `SurveyCreatorModel` per render in React | wrap in `useMemo` or `useState(() => new SurveyCreatorModel(opts))` |
