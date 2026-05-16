# Form Library — React (`survey-react-ui` 2.5.20)

MIT-licensed. Peer deps: `survey-core@2.5.20` (exact pin), `react ^16.5 || ^17 || ^18 || ^19`. Pure renderer — adds **zero keys** to the survey JSON schema.

## Install

```bash
npm install survey-core@2.5.20 survey-react-ui@2.5.20
```

CSS (one of):
```ts
import "survey-core/survey-core.min.css";       // V2 default theme
// or apply a built-in theme programmatically (see below)
```

## Minimal example

```tsx
import { useMemo } from "react";
import { Model } from "survey-core";
import { Survey } from "survey-react-ui";
import "survey-core/survey-core.min.css";

const json = {
  pages: [{
    elements: [
      { type: "text",    name: "name",  title: "Your name", isRequired: true },
      { type: "rating",  name: "score", title: "How likely are you to recommend us?" },
    ]
  }]
};

export function MySurvey() {
  // CRITICAL: memoize the model. Recreating it every render leaks subscriptions and resets input.
  const survey = useMemo(() => new Model(json), []);
  survey.onComplete.add((sender) => {
    console.log("data", sender.data);
  });
  return <Survey model={survey} />;
}
```

The `<Survey/>` component is a class component with `componentDidMount`-driven subscription. **Do not** inline `new Model(json)` in JSX — every render creates a fresh model.

## Common patterns

### Apply a built-in theme

```tsx
import { ContrastLight } from "survey-core/themes";
const survey = useMemo(() => {
    const m = new Model(json);
    m.applyTheme(ContrastLight);
    return m;
}, []);
```

### Subscribe to events

```tsx
useMemo(() => {
    const m = new Model(json);
    m.onValueChanged.add((sender, options) => { /* options.name, options.value */ });
    m.onValidateQuestion.add((sender, options) => {
        if (options.name === "age" && options.value < 18) options.error = "Must be 18+";
    });
    m.onComplete.add(async (sender, options) => {
        const res = await fetch("/api/forms/submit", {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify(sender.data)
        });
        if (!res.ok) options.showSaveError("Submission failed");
        else options.showSaveSuccess("Saved!");
    });
    return m;
}, []);
```

### Pre-fill data from server (use mergeData, NOT assignment)

```tsx
useEffect(() => {
    fetch(`/api/forms/${id}/data`).then(r => r.json()).then((draft) => {
        survey.mergeData(draft);  // overlay, preserves defaults
    });
}, [survey, id]);
```

`survey.data = draft` would replace wholesale and drop any defaults populated by the schema. Use `mergeData`.

### Custom React renderer for one question type

```tsx
import { ReactQuestionFactory } from "survey-react-ui";
import { SurveyQuestionElementBase } from "survey-react-ui";

class FancyTextRenderer extends SurveyQuestionElementBase {
    get question() { return this.questionBase; }
    renderElement() {
        return <input type="text" value={this.question.value || ""} onChange={(e) => this.question.value = e.target.value} />;
    }
}
ReactQuestionFactory.Instance.registerQuestion("text", (props) => <FancyTextRenderer {...props}/>);
```

## Next.js / SSR

The survey-react-ui module is browser-only at render time. For Next.js App Router:

```tsx
"use client";
import dynamic from "next/dynamic";
const Survey = dynamic(() => import("survey-react-ui").then(m => m.Survey), { ssr: false });
```

`'use client'` alone isn't sufficient — `dynamic(..., { ssr: false })` is required.

## Programmatic validation (browser side)

```ts
const ok = survey.validate(/*fireCallback*/ true, /*focusOnFirstError*/ true);
if (!ok) {
    // survey.getAllErrors() → SurveyError[] across all questions
    // each question: q.errors[].getText()
}
```

## What this package does NOT do

- Does not add JSON schema keys (all schema authoring is via `survey-core`).
- Does not run server-side under ClearScript — see `references/server-side.md` for the headless path. Only `survey-core` loads under V8.
- Does not include the Creator UI — that is `survey-creator-react`, see `references/creator.md`.

## Common gotchas

| Symptom | Cause | Fix |
|---|---|---|
| Survey resets when parent re-renders | `new Model(json)` inlined in JSX | wrap in `useMemo`/`useState` |
| `window is not defined` in Next.js | SSR rendering survey-react-ui | `dynamic(..., { ssr:false })` |
| Theme not visible | Forgot to `import "survey-core/survey-core.min.css"` | add the import once at app entry |
| Survey doesn't update when external `data` prop changes | Setting prop directly on a memoized model | call `survey.mergeData(newData)` in a `useEffect` |
| `onComplete` runs but data is empty | Question marked `clearInvisibleValues: 'onComplete'` and was hidden via `visibleIf` | snapshot data BEFORE complete or set `clearInvisibleValues: 'none'` |
