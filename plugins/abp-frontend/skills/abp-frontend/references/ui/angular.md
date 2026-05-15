# UI — Angular

> ABP's Angular UI is a set of @abp/ng.* npm packages that provide a preconfigured Angular SPA shell (auth, HTTP, localization, multi-tenancy, theming, validation, extensions, testing) wired to an ABP backend through generated service proxies and a single environment.ts.

## When to load this reference

- User asks how to bootstrap or configure an ABP Angular front-end (`abp new ... -u angular`, environment.ts, oAuthConfig).
- User asks how to call backend services from Angular (RestService, generated proxies, `abp generate-proxy -t ng`, named apis).
- User asks how to handle authentication/authorization in Angular (OAuth flows, PKCE vs Resource Owner Password, AuthErrorFilterService).
- User asks how to access current user info or config state in Angular (ConfigStateService, getOne$).
- User asks about localization in Angular (LocalizationService, abpLocalization pipe, registerLocale, ResourceName::Key).
- User asks how to validate forms (ngx-validate, withValidationBluePrint, skipValidation, custom error templates).
- User asks how to enable subdomain or header-based multi-tenancy in the Angular UI (`{0}` placeholder, `__tenant` header).
- User asks how to swap default UI components or layouts (ReplaceableComponentsService, replaceable component keys, custom layouts with router-outlet).
- User asks how to extend list pages and forms (entity actions, data-table columns, toolbar, dynamic form extensions, ngx-datatable).
- User asks how to switch or customize themes (Basic vs LeptonX vs LeptonX Lite, RoutesService, NavItemsService, UserMenuService).
- User asks how to write unit tests for ABP Angular components (CoreTestingModule, clearPage, wait, mocking ConfigStateService/RestService).
- User asks about per-feature library wiring (`provide<Feature>Config`, lazy `loadChildren`, e.g. `@abp/ng.identity`, `@abp/ng.tenant-management`).

**Audience:** Front-end developers building or maintaining an Angular SPA against an ABP 10.3 backend — both new template users following Quick Start and senior developers customizing layouts, replacing components, extending tables/forms, configuring multi-tenancy, or writing tests.

## Key concepts

- `Config.Environment` (from `@abp/ng.core`): TypeScript shape of `environment.ts` — `{ production, application, oAuthConfig, apis, remoteEnv? }`. Reach for it whenever editing environment.ts or typing a custom environment loader.
- `EnvironmentService` (`@abp/ng.core`): root-singleton accessor for the merged environment. Methods: `getEnvironment()`, `getApiUrl(apiName?)`, `setState(env)`. Reach for it from app code that needs the API URL or to mutate config at runtime.
- `ConfigStateService` (`@abp/ng.core`): holds the `application-configuration` payload (current user, granted policies, settings, features, localizations). Methods: `getOne(key)`, `getOne$(key)`, `getDeep(path)`, `getDeep$(path)`. Canonical way to read current user, settings, and granted permissions on the client.
- `RestService` (`@abp/ng.core`): HTTP wrapper over `HttpClient` that auto-attaches auth + tenant headers and routes errors through ThemeShared error handler. `request<TBody, TResp>(req: Rest.Request<TBody>, config?: Rest.Config)`. Reach for it in hand-written service code (proxies use it under the hood).
- `Rest.Request` / `Rest.Config` / `Rest.Observe` (`@abp/ng.core`): types describing an HTTP call (method, url, body, params) and per-call options (`apiName`, `skipHandleError`, `observe: Rest.Observe.Response | Body | Events`).
- `ExternalHttpClient` (`@abp/ng.core`): drop-in HttpClient for calling non-ABP/external APIs without leaking ABP auth/tenant headers.
- Generated service proxies (`@proxy/...`): TypeScript services produced by `abp generate-proxy -t ng`, one per backend controller, with `apiName` matching the module's `RemoteServiceName`. Reach for these instead of hand-rolling RestService calls when the backend exposes app services.
- `LocalizationService` (`@abp/ng.core`): `instant(key, ...args)` (sync) and `get(key, ...args)` (Observable). Supports `{ key, defaultValue }` fallback object. Use in TS code; use `abpLocalization` pipe in templates.
- `abpLocalization` pipe (`@abp/ng.core`): template binding for `'Resource::Key' | abpLocalization` with positional interpolation arguments and default-value object syntax.
- `registerLocale(options)` (`@abp/ng.core/locale`): higher-order function used as `registerLocaleFn` inside `provideAbpCore(withOptions({ ... }))`. Maps .NET culture names (e.g. `pt-BR`) to Angular locale files (e.g. `pt`) with optional `errorHandlerFn`.
- OAuth via angular-oauth2-oidc: configured by `oAuthConfig` in `environment.ts`. PKCE (`responseType: 'code'`) is the default and recommended flow; Resource Owner Password Flow uses `dummyClientSecret` and is supported but discouraged.
- `AuthErrorFilterService` (`@abp/ng.oauth` / OAuth feature lib): override point for selectively suppressing default OAuth error handling.
- `ReplaceableComponentsService` (`@abp/ng.core`): `add({ component, key }, refreshLayout?)`. Swaps default ABP-shipped components (Identity grids, theme layouts, etc.) for custom ones at runtime. Component keys come from enums like `eIdentityComponents`, `eThemeBasicComponents`, `eThemeSharedComponents`.
- `RoutesService` (`@abp/ng.core`): adds/edits/sorts main-menu routes. Used by themes to render the side/top menu and by application code to register feature menu items.
- `NavItemsService` (`@abp/ng.theme.shared`): registers items in the top toolbar / nav area (e.g. language switcher, custom actions).
- `UserMenuService` (`@abp/ng.theme.shared`): registers entries in the user dropdown menu (profile, security logs, logout, custom links).
- `PageAlertService` (`@abp/ng.theme.shared`): pushes page-level alert banners.
- `SessionStateService` (`@abp/ng.core`): holds session-scoped state — current language, current tenant. Used by language selector and tenant box.
- Tenant resolution: `application.baseUrl: 'https://{0}.mydomain.com/'` plus `/api/abp/multi-tenancy/tenants/by-name/{name}` lookup; resolved tenant id is sent as `__tenant` HTTP header. `TENANT_NOT_FOUND_BY_NAME` injection token customizes the not-found path.
- ngx-validate integration: `provideAbpThemeShared(withValidationBluePrint({...}))` registers per-validator messages; `VALIDATION_BLUEPRINTS` and `DEFAULT_VALIDATION_BLUEPRINTS` tokens; `VALIDATION_ERROR_TEMPLATE` overrides the visual component; `skipValidation` attribute disables auto-rendered errors at form or control level.
- Extension system: `EntityActionExtensions`, `DataTableColumnExtensions`, `PageToolbarExtensions`, `DynamicFormExtensions` — provider-configured maps keyed by component name (e.g. `eIdentityComponents.Users`) with `propList`, `ePropType`, validators, visibility predicates.
- ngx-datatable extensible table: `actionsText`, `data`, `recordsTotal`, `actionsTemplate`, `selectionType` (single/multi), `infiniteScroll` + `loadMore` event.
- Testing modules: `CoreTestingModule.withConfig()`, `ThemeSharedTestingModule.withConfig()`, `ThemeBasicTestingModule.withConfig()` from `@abp/ng.<pkg>/testing`. Helpers `clearPage(fixture)` and `wait(fixture, timeout?)` from `@abp/ng.core/testing`.

## Configuration pattern

ABP Angular 10.3 uses Angular's standalone-component bootstrap (no NgModule). Wiring lives in `app.config.ts` via `provide*` functions and per-feature `provide<Feature>Config()` calls; feature pages are lazy-loaded in `app.routes.ts`. The single source of truth for backend coordinates is `src/environments/environment.ts` (typed as `Config.Environment`). Example `app.config.ts`:

```ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideAbpCore, withOptions } from '@abp/ng.core';
import { provideAbpOAuth } from '@abp/ng.oauth';
import { provideAbpThemeShared, withValidationBluePrint } from '@abp/ng.theme.shared';
import { provideSettingManagementConfig } from '@abp/ng.setting-management/config';
import { provideFeatureManagementConfig } from '@abp/ng.feature-management/config';
import { provideAccountConfig } from '@abp/ng.account/config';
import { provideIdentityConfig } from '@abp/ng.identity/config';
import { provideTenantManagementConfig } from '@abp/ng.tenant-management/config';
import { registerLocale } from '@abp/ng.core/locale';
import { APP_ROUTES } from './app.routes';
import { environment } from '../environments/environment';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(APP_ROUTES),
    provideAbpCore(
      withOptions({
        environment,
        registerLocaleFn: registerLocale({ cultureNameLocaleFileMap: { 'pt-BR': 'pt' } }),
        // optional UI-defined fallbacks; backend localizations win
        localizations: [{ culture: 'en', resources: [{ resourceName: 'MyProjectName', texts: { HomePage: 'Home' } }] }],
      }),
    ),
    provideAbpOAuth(),
    provideAbpThemeShared(
      withValidationBluePrint({ uniqueUsername: '::AlreadyExists[{{ username }}]' }),
    ),
    provideAccountConfig({ redirectUrl: '/' }),
    provideIdentityConfig(),
    provideTenantManagementConfig(),
    provideFeatureManagementConfig(),
    provideSettingManagementConfig(),
  ],
};
```

Key options: `withOptions({ environment, registerLocaleFn, localizations, skipGetAppConfiguration? })` controls bootstrap of ConfigStateService and the locale-loading pipeline. `withValidationBluePrint(map)` registers ngx-validate error messages; provide the `VALIDATION_BLUEPRINTS` token directly when you need to spread `DEFAULT_VALIDATION_BLUEPRINTS`. Each `provide<Feature>Config()` accepts feature-specific options (e.g. `redirectUrl` for the account module). Routes use lazy `loadChildren`:

```ts
// app.routes.ts
import { Routes } from '@angular/router';
export const APP_ROUTES: Routes = [
  { path: '', loadChildren: () => import('./home/home.routes').then(m => m.HOME_ROUTES) },
  { path: 'account', loadChildren: () => import('@abp/ng.account') },
  { path: 'identity', loadChildren: () => import('@abp/ng.identity') },
  { path: 'tenant-management', loadChildren: () => import('@abp/ng.tenant-management') },
  { path: 'setting-management', loadChildren: () => import('@abp/ng.setting-management') },
];
```

Themes are wired by adding the corresponding theme package's `provideTheme<Name>` (or by importing its layout components and registering them via `ReplaceableComponentsService` for layout slots). Service proxies are generated once per build with `abp generate-proxy -t ng` and consumed via `inject(SomeService)` from the `@proxy/...` namespace.

## Code examples

### environment.ts for a standard Angular template

_Wire OAuth (Authorization Code + PKCE), backend API URL, and proxy root namespace._

```csharp
// src/environments/environment.ts
import { Config } from '@abp/ng.core';

const baseUrl = 'http://localhost:4200';

export const environment = {
  production: false,
  application: {
    baseUrl,
    name: 'MyProjectName',
    logoUrl: '',
  },
  oAuthConfig: {
    issuer: 'https://localhost:44381',
    redirectUri: baseUrl,
    clientId: 'MyProjectName_App',
    responseType: 'code',
    scope: 'offline_access MyProjectName',
  },
  apis: {
    default: {
      url: 'https://localhost:44381',
      rootNamespace: 'MyCompanyName.MyProjectName',
    },
    AbpAccountPublic: {
      url: 'https://localhost:44322',
      rootNamespace: 'AbpAccountPublic',
    },
  },
} as Config.Environment;
```

**Key lines:** `oAuthConfig.responseType: 'code'` selects PKCE; `apis.default.url` is the fallback target for RestService calls; `rootNamespace` drives proxy generation namespacing; named API entries (`AbpAccountPublic`) let modules target a different backend by passing `apiName` to RestService.

### Calling a generated service proxy

_Fetch a paginated list using the auto-generated TypeScript proxy._

```csharp
import { Component, OnInit, inject } from '@angular/core';
import { BookService, BookDto } from '@proxy/books';

@Component({ selector: 'app-book-list', standalone: true, template: `...` })
export class BookListComponent implements OnInit {
  private books = inject(BookService);
  data: BookDto[] = [];

  ngOnInit() {
    this.books.getList({ skipCount: 0, maxResultCount: 20 }).subscribe(res => {
      this.data = res.items;
    });
  }
}
```

**Key lines:** The proxy method name (`getList`) and DTO (`BookDto`) come straight from the backend `IBookAppService`; the proxy internally calls `RestService.request` with `apiName` set to the controller's `RemoteServiceName`, so auth + tenant headers and error handling are automatic.

### Custom RestService call with named API and skipHandleError

_Hit a non-default backend endpoint and handle errors locally._

```csharp
import { inject, Injectable } from '@angular/core';
import { RestService, Rest } from '@abp/ng.core';
import { Observable } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class ReportApiService {
  private rest = inject(RestService);

  download(id: string): Observable<Blob> {
    const request: Rest.Request<null> = {
      method: 'GET',
      url: `/api/reports/${id}/download`,
      responseType: 'blob' as 'json',
    };
    return this.rest.request<null, Blob>(request, {
      apiName: 'Reports',
      skipHandleError: true,
      observe: Rest.Observe.Body,
    });
  }
}
```

**Key lines:** `apiName: 'Reports'` routes through `apis.Reports.url` from environment.ts; `skipHandleError: true` opts out of the global ThemeShared error toast so the caller can display its own error; `observe` selects body vs full response.

### Reading the current user from ConfigStateService

_Show the current user's name and react to login/logout._

```csharp
import { Component, inject } from '@angular/core';
import { ConfigStateService, CurrentUserDto } from '@abp/ng.core';
import { AsyncPipe } from '@angular/common';
import { Observable } from 'rxjs';

@Component({
  selector: 'app-user-badge',
  standalone: true,
  imports: [AsyncPipe],
  template: `<span *ngIf="user$ | async as u">{{ u.userName }}</span>`,
})
export class UserBadgeComponent {
  private config = inject(ConfigStateService);
  user$: Observable<CurrentUserDto> = this.config.getOne$('currentUser');
}
```

**Key lines:** `ConfigStateService` is the single source of truth for the application-configuration payload; `getOne$('currentUser')` emits whenever `ConfigStateService.refreshAppState()` runs (e.g. after login). Use `getOne(...)` for one-shot synchronous reads.

### Replacing a default ABP component

_Provide a custom Roles list to override the Identity module's default._

```csharp
import { Component, OnInit, inject } from '@angular/core';
import { ReplaceableComponentsService } from '@abp/ng.core';
import { eIdentityComponents } from '@abp/ng.identity';
import { CustomRolesComponent } from './custom-roles.component';

@Component({ selector: 'app-root', standalone: true, template: `<router-outlet />` })
export class AppComponent implements OnInit {
  private replaceable = inject(ReplaceableComponentsService);

  ngOnInit() {
    this.replaceable.add({
      component: CustomRolesComponent,
      key: eIdentityComponents.Roles,
    });
  }
}
```

**Key lines:** `ReplaceableComponentsService.add({...})` registers a runtime substitution; `eIdentityComponents.Roles` is the well-known key the Identity module looks up via `<abp-replaceable-template [key]="...">`. Layout components (e.g. `eThemeBasicComponents.ApplicationLayout`) follow the same pattern but the replacement template must contain `<router-outlet>`.

### Subdomain-based multi-tenancy

_Resolve tenant from `mytenant1.mydomain.com` and route per-tenant API and identity URLs._

```csharp
// environment.ts (excerpt)
export const environment = {
  production: true,
  application: {
    baseUrl: 'https://{0}.mydomain.com/',
    name: 'MyProjectName',
    logoUrl: '',
  },
  oAuthConfig: {
    issuer: 'https://{0}.ids.mydomain.com',
    redirectUri: 'https://{0}.mydomain.com',
    clientId: 'MyProjectName_App',
    responseType: 'code',
    scope: 'offline_access MyProjectName',
  },
  apis: {
    default: {
      url: 'https://{0}.api.mydomain.com',
      rootNamespace: 'MyCompanyName.MyProjectName',
    },
  },
} as Config.Environment;
```

**Key lines:** The `{0}` placeholder is replaced by the subdomain at runtime. ABP first calls `/api/abp/multi-tenancy/tenants/by-name/{tenantName}` against the resolved API; on success it stores the tenant and sends `__tenant: <id>` on every subsequent request. Customize the not-found path by providing `TENANT_NOT_FOUND_BY_NAME`.

### Form validation with custom blueprint

_Show a localized error for an async `uniqueUsername` validator._

```csharp
// app.config.ts
import { provideAbpThemeShared, withValidationBluePrint } from '@abp/ng.theme.shared';

export const appConfig: ApplicationConfig = {
  providers: [
    provideAbpThemeShared(
      withValidationBluePrint({
        uniqueUsername: '::AlreadyExists[{{ username }}]',
      }),
    ),
  ],
};

// signup.component.ts
import { Component, inject } from '@angular/core';
import { FormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';

@Component({
  selector: 'app-signup', standalone: true, imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="form">
      <input formControlName="userName" />
      <input formControlName="email" skipValidation />
    </form>`,
})
export class SignupComponent {
  private fb = inject(FormBuilder);
  form = this.fb.group({
    userName: ['', [Validators.required, Validators.minLength(3)]],
    email: [''],
  });
}
```

**Key lines:** Blueprint keys map to validator names; the `::AlreadyExists` localization key is resolved automatically and `{{ username }}` interpolates the validator's error payload. ngx-validate auto-renders the message under the input — `skipValidation` disables that auto-render for a single control or the whole form.

### Unit-testing a component that uses ABP services

_Render a component, mock RestService and ConfigStateService, and assert behavior._

```csharp
import { TestBed, ComponentFixture } from '@angular/core/testing';
import { CoreTestingModule, clearPage, wait } from '@abp/ng.core/testing';
import { ThemeSharedTestingModule } from '@abp/ng.theme.shared/testing';
import { ConfigStateService } from '@abp/ng.core';
import { of } from 'rxjs';
import { UserBadgeComponent } from './user-badge.component';

describe('UserBadgeComponent', () => {
  let fixture: ComponentFixture<UserBadgeComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [
        CoreTestingModule.withConfig(),
        ThemeSharedTestingModule.withConfig(),
        UserBadgeComponent,
      ],
      providers: [
        { provide: ConfigStateService, useValue: { getOne$: () => of({ userName: 'alice' }) } },
      ],
    }).compileComponents();

    fixture = TestBed.createComponent(UserBadgeComponent);
    fixture.detectChanges();
    await wait(fixture);
  });

  afterEach(() => clearPage(fixture));

  it('renders the current user name', () => {
    expect(fixture.nativeElement.textContent).toContain('alice');
  });
});
```

**Key lines:** `CoreTestingModule.withConfig()` and `ThemeSharedTestingModule.withConfig()` install the doubles ABP needs (LocalizationService, EnvironmentService, etc.); `wait(fixture)` flushes off-detection-cycle work like modals/dialogs; `clearPage(fixture)` removes leftover DOM between tests so Karma doesn't fail on dialog leaks; service mocks plug in through standard Angular `providers`.

## Common mistakes

- - Mistake: Treating `environment.ts` as static at deploy time. Why wrong: hard-coded URLs force a rebuild per environment. Correct pattern: use `remoteEnv: { url, mergeStrategy }` so the SPA fetches a `/assets/env.json` (or any URL) at startup and merges it over the bundled defaults; serve it with the right content-type from IIS/Nginx.
- - Mistake: Calling backend endpoints with raw `HttpClient` for ABP-secured APIs. Why wrong: tokens, `__tenant`, culture, and the global error handler are all wired into `RestService`/`ApiInterceptor`; bypassing them silently drops auth and breaks tenant scoping. Correct pattern: use generated proxies (`@proxy/...`) or `RestService.request(...)`. For genuinely external APIs, use `ExternalHttpClient`.
- - Mistake: Hand-writing TypeScript DTOs to mirror C# DTOs. Why wrong: drift is guaranteed and `apiName` plumbing is missed. Correct pattern: run `abp generate-proxy -t ng` after backend changes and import from `@proxy/...`. Make sure `@abp/ng.schematics` is installed and `apis.default.rootNamespace` is set.
- - Mistake: Calling `LocalizationService.instant(key)` before localizations are loaded (e.g. inside a constructor that runs before `provideAbpCore` finishes). Why wrong: returns the raw key. Correct pattern: bind in templates with `abpLocalization` (which subscribes), or use `LocalizationService.get(key, ...)` and subscribe; for default-fallback use `{ key, defaultValue }`.
- - Mistake: Putting business logic into a replaced layout component without keeping `<router-outlet>`. Why wrong: routing breaks and child pages stop rendering with no obvious error. Correct pattern: layout replacements (`ApplicationLayoutComponent`, `AccountLayoutComponent`, `EmptyLayoutComponent`) must include `<router-outlet></router-outlet>`; pass `true` as the second arg of `replaceableComponents.add({...}, true)` only when you actually need a runtime layout swap (it refreshes routes and clears component state).
- - Mistake: Modifying menu items by editing theme source. Why wrong: theme upgrades blow the changes away. Correct pattern: register/modify routes through `RoutesService`, toolbar items through `NavItemsService`, and user-menu entries through `UserMenuService`. Use `abp add-package @abp/ng.theme.basic --with-source-code` only when a deep visual fork is unavoidable.
- - Mistake: Using Resource Owner Password Flow with `dummyClientSecret` for new apps. Why wrong: it requires shipping a 'secret' to the browser and can't satisfy modern OIDC requirements. Correct pattern: stick with `responseType: 'code'` (Authorization Code + PKCE), which the templates ship with.
- - Mistake: Forgetting to clear DOM between specs that open modals. Why wrong: leftover dialogs cause Karma to fail downstream tests with confusing errors. Correct pattern: call `clearPage(fixture)` in `afterEach`, and `await wait(fixture)` after triggering modal/dialog logic so off-detection-cycle work flushes before assertions.
- - Mistake: Subscribing to `ConfigStateService.getOne$('currentUser')` and never unsubscribing. Why wrong: ConfigStateService is application-scoped, so subscriptions leak for the lifetime of the app. Correct pattern: use the `async` pipe in templates, or `takeUntilDestroyed()` (Angular 16+) in components.
- - Mistake: Mixing different validation-message keys without spreading defaults. Why wrong: providing `VALIDATION_BLUEPRINTS` directly without spreading `DEFAULT_VALIDATION_BLUEPRINTS` strips built-in messages (required, minlength, etc.). Correct pattern: prefer `withValidationBluePrint({...})` (it merges), or when providing the token directly do `{ provide: VALIDATION_BLUEPRINTS, useValue: { ...DEFAULT_VALIDATION_BLUEPRINTS, ...mine } }`.
- - Mistake: Forgetting to add the tenant placeholder to `oAuthConfig.issuer` when using subdomain tenancy. Why wrong: only the API gets per-tenant routing while OIDC stays on a shared issuer, causing CORS/issuer-mismatch errors. Correct pattern: use `{0}` consistently across `application.baseUrl`, `oAuthConfig.issuer`, `oAuthConfig.redirectUri`, and `apis.default.url`.

## Version pins (ABP 10.3)

- ABP 10.3 Angular templates pin Angular 21.0.x; consequently they assume standalone-component bootstrap (`bootstrapApplication` + `app.config.ts`) and `provideRouter` rather than the legacy `AppModule` style. Older ABP versions (<= 8.x) used NgModule + `forRoot()` patterns and may not be drop-in compatible with the current `provide<Feature>Config()` API.
- Node.js 20.19+ is the documented minimum; older Node versions will not boot the dev server.
- Angular CLI is not installed globally; all CLI commands run via local `npm`/`yarn` scripts. Reaching for a globally installed `ng` may pull in a different version than the project's local one.
- OAuth flow defaults: `responseType: 'code'` (PKCE). Resource Owner Password Flow is documented but considered legacy and is expected to be removed/discouraged in future versions.
- Service-proxy generator: `abp generate-proxy -t ng` requires `@abp/ng.schematics` installed and a populated `apis.default.rootNamespace`; the older `nswag`-based generator is not the path used by ABP.
- Theme packages are sold/licensed independently — `@abp/ng.theme.lepton-x` (full) is commercial, `@abp/ng.theme.lepton-x.lite` and `@abp/ng.theme.basic` are open source. Switching themes is a package swap plus app.config wiring; theme APIs may change between minor versions.
- [uncertain] Exact list of feature-library packages in 10.3 (e.g. `@abp/ng.audit-logging`, `@abp/ng.openiddictpro`) — the feature-libraries doc only enumerates `@abp/ng.identity` as the worked example and references the others by navigation only.
- [uncertain] Whether `provideAbpOAuth()` is named exactly that in 10.3 vs. an alternative export from `@abp/ng.oauth`; the authorization doc covers configuration but does not show the exact provider import line.
- [uncertain] Public API surface of `ConfigStateService.getDeep$()` — only `getOne`/`getOne$` are illustrated in the current-user doc; `getDeep$` is mentioned by convention but not by the 10.3 page text fetched.
- ngx-validate, ngx-datatable, ng-bootstrap and angular-oauth2-oidc are external libraries; their breaking changes can ripple into ABP minor releases even when ABP's own API is stable. Pin them via the template's lockfile.

## Cross-references

**Sibling UI references (in this skill):**
- [references/ui/themes.md](./themes.md) — Triggered when the user asks about LeptonX/LeptonX Lite styling, layout slots, or CSS variable overrides; the theming sections of this reference point at it.
- [references/ui/overview.md](./overview.md) — Triggered when comparing Angular against MVC/Blazor/RN/MAUI before drilling in.
- [references/ui/blazor.md](./blazor.md) — Triggered when the user is comparing Angular's component-replacement model against Blazor's `[ExposeServices]` + `[Dependency(ReplaceServices=true)]` override pattern.
- [references/ui/mobile.md](./mobile.md) — Triggered when the user pairs an Angular SPA with a React Native mobile companion (the Layered template ships both side-by-side).

**Cross-skill references (in the abp-backend skill):**
- See **authorization** in the abp-backend skill (`framework/authorization.md`) — Triggered when the user asks how Angular permission checks line up with backend `IAuthorizationService.IsGrantedAsync` and policy definitions.
- See **multi-tenancy** in the abp-backend skill (`framework/multi-tenancy.md`) — Triggered when the user asks how the Angular tenant box / `__tenant` header coordinates with the backend's `ICurrentTenant`, tenant resolvers, and connection-string per tenant.
- See **localization / IStringLocalizer** in the abp-backend skill (`framework/fundamentals.md`) — Triggered when the user asks how Angular localization resources are produced and contributed by the backend (LocalizationResource classes, virtual files).
- See **dynamic / static C# clients** in the abp-backend skill (`framework/api-development.md`) — Triggered when the user compares Angular service proxies (`abp generate-proxy -t ng`) with the dynamic C#/HttpApi.Client clients used in MVC/Blazor templates.
- See **template selection** in the abp-backend skill (`templates/single-layer.md`, `templates/layered.md`) — Triggered when the user wants the full solution layout for an Angular SPA + .NET backend produced by `abp new ... -u angular`.

**External docs:**
- Angular - https://angular.dev/ - Underlying framework; ABP 10.3 ships Angular 21.0.x and uses standalone components plus modern provider APIs.
- angular-oauth2-oidc - https://github.com/manfredsteyer/angular-oauth2-oidc - Library that backs ABP's OAuth/OIDC flows; the `oAuthConfig` shape mirrors its options.
- ngx-validate - https://github.com/ng-turkey/ngx-validate - Reactive-form validation engine used by ABP's auto-rendered error messages and `withValidationBluePrint`.
- ngx-datatable - https://github.com/swimlane/ngx-datatable - Powers the `abp-extensible-table`; its inputs (`selectionType`, virtualization, etc.) bleed through ABP's wrapper.
- ng-bootstrap - https://ng-bootstrap.github.io/ - Base component library used by Basic and LeptonX themes (modals, dropdowns, etc.).
- Bootstrap - https://getbootstrap.com/ - HTML/CSS framework underlying all official ABP themes.
- Chart.js - https://www.chartjs.org/ - Charting library used by the prebuilt chart components.
- FontAwesome - https://fontawesome.com/ - Icon set used by the default themes.
- Karma - https://karma-runner.github.io/ - Default test runner shipped with the ABP Angular template.
- Jasmine - https://jasmine.github.io/ - Default assertion/spec framework paired with Karma in the template.
- Angular Testing Library - https://testing-library.com/docs/angular-testing-library/intro/ - Recommended TestBed alternative for component tests; the testing docs explicitly call it out as a good fit.
- OpenIddict - https://documentation.openiddict.com/ - The default token server on the ABP backend that issues tokens consumed by the Angular SPA's OAuth flow.

## Sources (ABP 10.3)

- [https://abp.io/docs/10.3/framework/ui/angular/overview](https://abp.io/docs/10.3/framework/ui/angular/overview) — Top-level overview of the Angular UI: organization (Development, Core Functionality, Utilities, Customization, Components), supported themes (LeptonX/LeptonX Lite), and the role of @abp/ng.* libraries in connecting an Angular SPA to an ABP backend.
- [https://abp.io/docs/10.3/framework/ui/angular/quick-start](https://abp.io/docs/10.3/framework/ui/angular/quick-start) — End-to-end onboarding: prerequisites (Node 20.19+, optional Yarn), `abp new ... -u angular` CLI command, project layout under `angular/`, environment.ts shape (oAuthConfig, apis, application), `yarn start` / `yarn build:prod`, and deployment of `dist/<App>/browser` static output.
- [https://abp.io/docs/10.3/framework/ui/angular/environment](https://abp.io/docs/10.3/framework/ui/angular/environment) — The `Config.Environment` interface (apis, application, oAuthConfig, production, remoteEnv), `EnvironmentService` (getEnvironment, getApiUrl, setState), named API endpoints, `rootNamespace`, and the remoteEnv runtime override mechanism with mergeStrategy options.
- [https://abp.io/docs/10.3/framework/ui/angular/feature-libraries](https://abp.io/docs/10.3/framework/ui/angular/feature-libraries) — Pattern for ABP feature libraries: each module ships a feature module + a `provide<Feature>Config()` configuration provider (e.g. `provideIdentityConfig` from `@abp/ng.identity/config`) plus lazy-loaded routes via `loadChildren: () => import('@abp/ng.identity')`.
- [https://abp.io/docs/10.3/framework/ui/angular/service-proxies](https://abp.io/docs/10.3/framework/ui/angular/service-proxies) — `abp generate-proxy -t ng` workflow: requires `@abp/ng.schematics`, hits `/api/abp/api-definition`, generates one TypeScript service per backend controller plus DTO interfaces and enums under a namespace driven by `apis.default.rootNamespace`.
- [https://abp.io/docs/10.3/framework/ui/angular/authorization](https://abp.io/docs/10.3/framework/ui/angular/authorization) — OAuth/OIDC integration via angular-oauth2-oidc: oAuthConfig flows (Authorization Code with PKCE using responseType: 'code', and Resource Owner Password with dummyClientSecret), and `AuthErrorFilterService` for selectively skipping default OAuth error handling.
- [https://abp.io/docs/10.3/framework/ui/angular/current-user](https://abp.io/docs/10.3/framework/ui/angular/current-user) — Current user is held in Config State; access via `ConfigStateService.getOne('currentUser')` (sync) or `getOne$('currentUser')` (Observable). No standalone CurrentUserService — the canonical entry point is ConfigStateService.
- [https://abp.io/docs/10.3/framework/ui/angular/http-requests](https://abp.io/docs/10.3/framework/ui/angular/http-requests) — `RestService.request<TBody, TResponse>(req, options)` from `@abp/ng.core`: accepts a `Rest.Request` config, supports `apiName` for non-default endpoints, `skipHandleError` to bypass the centralized error handler, `observe: Rest.Observe.Response` for full HttpResponse, plus `ExternalHttpClient` for non-ABP APIs.
- [https://abp.io/docs/10.3/framework/ui/angular/localization](https://abp.io/docs/10.3/framework/ui/angular/localization) — `LocalizationService.instant(key, ...args)` and `get(key, ...args)`, the `abpLocalization` pipe, the `ResourceName::Key` format with `defaultResourceName` fallback, default-value object syntax, `registerLocale` for mapping .NET cultures to Angular locale files, and UI-side `localizations` config.
- [https://abp.io/docs/10.3/framework/ui/angular/form-validation](https://abp.io/docs/10.3/framework/ui/angular/form-validation) — ngx-validate-based reactive-form validation, `provideAbpThemeShared(withValidationBluePrint({ ... }))` to register custom error message keys, `VALIDATION_BLUEPRINTS` and `DEFAULT_VALIDATION_BLUEPRINTS`, `VALIDATION_ERROR_TEMPLATE` for custom error component, and `skipValidation` form/control attribute.
- [https://abp.io/docs/10.3/framework/ui/angular/multi-tenancy](https://abp.io/docs/10.3/framework/ui/angular/multi-tenancy) — Tenant box switcher on account pages, `__tenant` HTTP header, domain/subdomain tenant resolution via `application.baseUrl: 'https://{0}.mydomain.com/'` (also valid in `oAuthConfig.issuer` and `apis.default.url`), `/api/abp/multi-tenancy/tenants/by-name/{name}` lookup, and the `TENANT_NOT_FOUND_BY_NAME` injection token.
- [https://abp.io/docs/10.3/framework/ui/angular/customization-user-interface](https://abp.io/docs/10.3/framework/ui/angular/customization-user-interface) — Index of customization paths: component replacement, modifying the menu, sorting navigation, profile-page tabs, layout components, and the four extensions (entity actions, data-table columns, toolbar, dynamic forms).
- [https://abp.io/docs/10.3/framework/ui/angular/theming](https://abp.io/docs/10.3/framework/ui/angular/theming) — Theme model: themes are NPM packages (`@abp/ng.theme.basic`, `@abp/ng.theme.lepton-x[/lite]`) on top of `@abp/ng.theme.shared`; layout slots managed via `RoutesService`, `NavItemsService`, `UserMenuService`, `PageAlertService`, `SessionStateService`; `abp add-package @abp/ng.theme.basic --with-source-code` for full source.
- [https://abp.io/docs/10.3/framework/ui/angular/component-replacement](https://abp.io/docs/10.3/framework/ui/angular/component-replacement) — `ReplaceableComponentsService.add({ component, key })` from `@abp/ng.core`, component-key enums (`eIdentityComponents`, `eThemeBasicComponents`, etc.), layout replacement requires `<router-outlet>` in the replacement template, and the second-arg flag for runtime layout swap.
- [https://abp.io/docs/10.3/framework/ui/angular/extensions-overall](https://abp.io/docs/10.3/framework/ui/angular/extensions-overall) — Four extension surfaces — Entity Action, Data Table Column, Page Toolbar, Dynamic Form Extensions — wired through provider configs, plus the ngx-datatable-based extensible table with `actionsText`, `data`, `recordsTotal`, `actionsTemplate`, `selectionType`, and `infiniteScroll` / `loadMore` features.
- [https://abp.io/docs/10.3/framework/ui/angular/testing](https://abp.io/docs/10.3/framework/ui/angular/testing) — Karma+Jasmine preconfigured; `CoreTestingModule.withConfig()`, `ThemeSharedTestingModule.withConfig()`, `ThemeBasicTestingModule.withConfig()`; helpers `clearPage(fixture)` and `wait(fixture)` from `@abp/ng.core/testing`; mock ABP services via `providers: [{ provide: X, useValue: ... }]`; Angular Testing Library is a recommended alternative.

Last verified: 2026-05-10
