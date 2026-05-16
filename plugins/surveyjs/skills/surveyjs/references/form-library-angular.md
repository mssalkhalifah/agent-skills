# Form Library — Angular (`survey-angular-ui` 2.5.20)

MIT-licensed. Peer deps in 2.5.20: `survey-core@2.5.20`, `@angular/core *`, `@angular/forms *`, `@angular/cdk *`. Requires Angular ≥ 12. Pure renderer — adds **zero keys** to the survey JSON schema.

## Install

```bash
npm install survey-core@2.5.20 survey-angular-ui@2.5.20
```

CSS — add to `angular.json` styles or import in your app entry:
```
"styles": [ "node_modules/survey-core/survey-core.min.css", "src/styles.css" ]
```

## NgModule setup

```ts
import { NgModule } from "@angular/core";
import { BrowserModule } from "@angular/platform-browser";
import { SurveyModule } from "survey-angular-ui";
import { AppComponent } from "./app.component";

@NgModule({
  imports: [BrowserModule, SurveyModule],
  declarations: [AppComponent],
  bootstrap: [AppComponent],
})
export class AppModule {}
```

## Standalone-component setup

```ts
import { Component } from "@angular/core";
import { SurveyModule } from "survey-angular-ui";
import { Model } from "survey-core";

@Component({
  selector: "app-my-survey",
  standalone: true,
  imports: [SurveyModule],
  template: `<survey [model]="survey"></survey>`,
})
export class MySurveyComponent {
  json = { pages: [{ elements: [{ type: "text", name: "name", title: "Your name", isRequired: true }] }] };
  survey = new Model(this.json);

  constructor() {
    this.survey.onComplete.add((sender) => console.log(sender.data));
  }
}
```

## Available selectors

- `<survey [model]="...">` — full survey
- `<popup-survey [model]="..." [isExpanded]="..." [allowClose]="..." [closeOnCompleteTimeout]="..." [allowFullScreen]="..." [onClose]="...">` — popup variant

## Replacing a built-in question renderer

```ts
import { AngularComponentFactory } from "survey-angular-ui";
import { MyFancyTextComponent } from "./my-fancy-text.component";
AngularComponentFactory.Instance.registerComponent("text", MyFancyTextComponent);
```

`MyFancyTextComponent` should extend `BaseAngular<QuestionTextModel>` (re-exported by `survey-angular-ui`).

## Change detection model

`SurveyComponent` detaches its own `ChangeDetectorRef` in the constructor and drives updates via survey-core's `addOnPropertyValueChangedCallback` / `addOnArrayChangedCallback`, batching most updates through `queueMicrotask`. Implications:

- **Replacing the SurveyModel reference** on the component is the supported way to swap surveys.
- **OnPush parents** that bind to live survey state (e.g., `survey.data`) need manual `markForCheck()` after value changes — easiest done via `survey.onValueChanged` calling `cdr.markForCheck()`.

## Pre-fill data from server (use mergeData)

```ts
ngOnInit() {
  this.api.getDraft(this.formId).subscribe(draft => this.survey.mergeData(draft));
}
```

Wholesale assignment (`survey.data = draft`) replaces and drops defaults.

## Programmatic validation (browser side)

```ts
const ok = this.survey.validate(/*fireCallback*/ true, /*focusOnFirstError*/ true);
```

## SSR / Angular Universal

**Officially unsupported.** survey-angular-ui needs a real DOM at construction. If the host app uses Universal, exclude this component from server rendering or hydrate it client-only.

## What this package does NOT do

- Does not add JSON schema keys.
- Does not include the Creator UI — that is `survey-creator-angular`, see `references/creator.md`.
- Does not run server-side; only `survey-core` is loadable under V8 ClearScript.

## Common gotchas

| Symptom | Cause | Fix |
|---|---|---|
| "No provider for ..." at runtime | Forgot to `import SurveyModule` | add to NgModule `imports` (or standalone component `imports`) |
| Survey doesn't respond to property changes from outside | Custom code mutating survey under OnPush parent | `survey.onValueChanged.add(() => cdr.markForCheck())` |
| Theme not applied | Missing CSS in `angular.json` | add `survey-core/survey-core.min.css` to `styles` |
| ResizeObserver-related console errors | Angular zone catching the survey-core observer | usually safe to ignore; can be suppressed at `NgZone` boundary |
| `Unable to find component for type 'foo'` | A custom question type wasn't registered with `AngularComponentFactory` | register before constructing the SurveyModel |
