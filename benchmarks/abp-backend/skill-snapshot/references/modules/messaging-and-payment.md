# Modules — Messaging & Payment (Twilio SMS, Payment, Chat, Text Templates)

> ABP 10.3 communication and commerce modules — Twilio SMS (ISmsSender provider), Payment (multi-gateway one-time and subscription billing), Chat (real-time SignalR messaging), and Text Template Management (UI to edit text/email template contents per culture).

## When to load this reference

Load when the question concerns: sending SMS in ABP / configuring Twilio / implementing ISmsSender; integrating Stripe/PayPal/Iyzico/PayU/2Checkout/Alipay for one-time payments or subscriptions; building user-to-user real-time chat with SignalR over an ABP backend; editing or managing email/text templates in the Administration UI; choosing among ABP's Pro communication modules; or wiring distributed events such as PaymentRequestCompleted, SubscriptionCreated, ChatMessageEto.

**Audience:** ABP developers (Team license or higher) integrating SMS notifications, online payments/subscriptions, in-app chat, or admin-editable text templates into Web/Blazor/Angular/MAUI apps.

## Key concepts

- ISmsSender (Volo.Abp.Sms.ISmsSender) — Framework abstraction for sending SMS. Inject and call SendAsync(phoneNumber, text) or SendAsync(SmsMessage). Reach for it any time you need to send an SMS regardless of provider.
- SmsMessage (Volo.Abp.Sms.SmsMessage) — Value object with PhoneNumber, Text, and Properties dictionary for provider-specific extras.
- NullSmsSender (Volo.Abp.Sms.NullSmsSender) — Default implementation; logs the SMS instead of sending. Replace in production by adding the Twilio module or a custom ITransientDependency implementing ISmsSender.
- AbpTwilioSmsModule (Volo.Abp.Sms.Twilio) — Pro module that registers a Twilio-backed ISmsSender. Add via 'abp add-package Volo.Abp.Sms.Twilio'.
- AbpTwilioSmsOptions (Volo.Abp.Sms.Twilio.AbpTwilioSmsOptions) — Options class with AccountSId, AuthToken, FromNumber. Bind from 'AbpTwilioSms' configuration section or Configure<>() in module.
- IPaymentRequestAppService (Volo.Payment.Requests.IPaymentRequestAppService) — Creates and queries PaymentRequest aggregates; entry point for both one-time payments and subscription initiation (PaymentRequestCreateDto with PaymentType.Subscription + PlanId).
- IGatewayAppService (Volo.Payment.Gateways.IGatewayAppService) — Lists payment gateways available to the current tenant/app for the gateway-selection UI.
- IPlanAppService (Volo.Payment.Plans.IPlanAppService) — CRUD over Plan and PlanGateway entities used for recurring billing.
- PaymentRequest (Volo.Payment.Requests.PaymentRequest) — Aggregate root with Products, State (Waiting/Completed/Failed/Refunded), currency, gateway. Tracked by PaymentRequestId across gateway round-trips.
- Plan / PlanGateway (Volo.Payment.Plans.Plan, Volo.Payment.Plans.PlanGateway) — Subscription plan and per-gateway external-ID mapping (e.g., Stripe price ID).
- PaymentBlazorOptions (Volo.Payment.Blazor.PaymentBlazorOptions) — Sets RootUrl and CallbackUrl for Blazor payment redirection flow.
- Gateway-specific options (StripeOptions, PayPalOptions, IyzicoOptions, PayuOptions, AlipayOptions, plus *WebOptions / *BlazorOptions variants) — Hold API keys, currency, locale, webhook secrets per gateway.
- Distributed events: PaymentRequestCompleted, SubscriptionCreated, SubscriptionUpdated, SubscriptionCanceled — Subscribe via IDistributedEventBus to fulfill orders or update subscription state without coupling to gateway code.
- IConversationAppService (Volo.Chat.Conversations.IConversationAppService) — Sends/receives messages, marks conversations as read, returns paged history.
- IContactAppService (Volo.Chat.Contacts.IContactAppService) — Returns a user's contact list with unread counts.
- ISettingsAppService (Volo.Chat.Settings.ISettingsAppService) — Per-user chat settings (notifications, etc.).
- IRealTimeChatMessageSender (Volo.Chat.Messages.IRealTimeChatMessageSender) — Domain abstraction for delivering messages in real time; concrete implementations use SignalR or distributed event bus when web/API tiers are split.
- MessagingManager (Volo.Chat.MessagingManager) — Domain service that orchestrates message persistence, conversation updates, and event publishing.
- Chat aggregates: Message, ChatUser, UserMessage, Conversation — Messages are immutable; UserMessage stores per-side copies; Conversation tracks unread counts. Message implements IMultiTenant.
- ChatBlazorWebAssemblyOptions / Angular env apis.Chat.signalRUrl — Wire the Chat SignalR hub URL on the client.
- ChatMessageEto — Distributed event published on new messages; useful for cross-service notifications.
- ITemplateDefinitionAppService / ITemplateContentAppService (Volo.Abp.TextTemplateManagement.*) — App services backing the Administration → Text Templates UI (definitions list and per-culture content edits).
- TextTemplateContent (Volo.Abp.TextTemplateManagement.TextTemplates.TextTemplateContent) — Aggregate root storing per-culture content for a template definition; persisted in AbpTextTemplateContents.
- DatabaseTemplateContentContributor — Plugs into the framework's text-templating ITemplateContentContributor pipeline so DB-stored content overrides embedded virtual-file-system templates; cached via IDistributedCache<string, TemplateContentCacheKey>.
- TextTemplateManagementOptions.MinimumCacheDuration — Controls the floor cache lifetime for resolved template contents (default tied to settings).
- ITemplateRenderer + TemplateDefinitionProvider (framework-level, not the module) — Define template names/metadata in code; the Text Template Management module supplies editable bodies that override the in-code defaults at runtime.

## Configuration pattern

Each module is wired in a host AbpModule using DependsOn + Configure<TOptions>. Twilio SMS: depend on AbpSmsTwilioModule and bind AbpTwilioSmsOptions (or use 'AbpTwilioSms' configuration section). Payment: depend on AbpPaymentApplicationModule (+ gateway-specific modules + Web/Blazor variants) and configure each gateway's options class plus PaymentBlazorOptions when on Blazor. Chat: depend on ChatApplicationModule plus the matching SignalR/Blazor/Angular packages, run modelBuilder.ConfigureChat() in EF Core, and configure SignalR hub URL on clients. Text Template Management: depend on TextTemplateManagementModule and optionally Configure<TextTemplateManagementOptions>(o => o.MinimumCacheDuration = TimeSpan.FromHours(1)). Example skeleton:

[DependsOn(
    typeof(AbpSmsTwilioModule),
    typeof(PaymentApplicationModule),
    typeof(PaymentStripeModule),
    typeof(ChatApplicationModule),
    typeof(ChatSignalRModule),
    typeof(TextTemplateManagementApplicationModule)
)]
public class MyAppModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        var configuration = context.Services.GetConfiguration();

        Configure<AbpTwilioSmsOptions>(configuration.GetSection("AbpTwilioSms"));

        Configure<StripeOptions>(o =>
        {
            o.PublishableKey = configuration["Payment:Stripe:PublishableKey"];
            o.SecretKey      = configuration["Payment:Stripe:SecretKey"];
            o.WebhookSecret  = configuration["Payment:Stripe:WebhookSecret"];
        });

        Configure<PaymentBlazorOptions>(o =>
        {
            o.RootUrl     = "https://localhost:44300";
            o.CallbackUrl = "https://localhost:44300/PaymentSucceed";
        });

        Configure<TextTemplateManagementOptions>(o =>
        {
            o.MinimumCacheDuration = TimeSpan.FromHours(1);
        });
    }
}

For Chat over a split Web/API topology, also register a Distributed Event Bus (RabbitMQ recommended) so the IRealTimeChatMessageSender on the API tier can fan out to SignalR clients connected to the Web tier. For Angular, environment.apis.Chat.url and environment.apis.Chat.signalRUrl drive the proxy + hub addresses, and provideChatConfig + provideTextTemplateManagementConfig must be registered in app.config.ts.

## Code examples

### Send an SMS via Twilio (after adding Volo.Abp.Sms.Twilio)

_Two-factor code or transactional SMS — production-grade Twilio delivery while keeping ISmsSender abstraction._

```csharp
// In your AbpModule:
Configure<AbpTwilioSmsOptions>(options =>
{
    options.AccountSId  = configuration["AbpTwilioSms:AccountSId"];
    options.AuthToken   = configuration["AbpTwilioSms:AuthToken"];
    options.FromNumber  = configuration["AbpTwilioSms:FromNumber"];
});

// In an application service:
public class NotificationAppService : ApplicationService
{
    private readonly ISmsSender _smsSender;
    public NotificationAppService(ISmsSender smsSender) => _smsSender = smsSender;

    public async Task NotifyAsync(string phone, string code)
    {
        await _smsSender.SendAsync(new SmsMessage(
            phoneNumber: phone,
            text: $"Your code is {code}"));
    }
}
```

**Key lines:** AbpTwilioSmsOptions binds the three Twilio credentials. ISmsSender is the framework abstraction — your code does not reference any Twilio type, so swapping providers (custom, NullSmsSender for tests) requires no consumer changes.

### One-time payment with Stripe — create request and redirect

_Customer checks out a basket; create a PaymentRequest, send them through the gateway-selection page, and act on success via distributed event._

```csharp
public class CheckoutAppService : ApplicationService
{
    private readonly IPaymentRequestAppService _paymentRequests;
    public CheckoutAppService(IPaymentRequestAppService paymentRequests)
        => _paymentRequests = paymentRequests;

    public async Task<Guid> StartAsync(BasketDto basket)
    {
        var request = await _paymentRequests.CreateAsync(new PaymentRequestCreateDto
        {
            Currency    = "USD",
            Products    = basket.Items.Select(i => new PaymentRequestProductCreateDto
            {
                Code        = i.Sku,
                Name        = i.Name,
                Count       = i.Quantity,
                UnitPrice   = i.UnitPrice,
                TotalPrice  = i.UnitPrice * i.Quantity,
                PaymentType = PaymentType.OneTime
            }).ToList()
        });
        return request.Id; // POST-redirect to /Payment/GatewaySelection?paymentRequestId={id}
    }
}

// Fulfillment via distributed event (decoupled from gateway):
public class FulfillmentHandler
    : IDistributedEventHandler<PaymentRequestCompletedEto>, ITransientDependency
{
    public Task HandleEventAsync(PaymentRequestCompletedEto eventData)
    {
        // Idempotently mark eventData.PaymentRequestId as fulfilled.
        return Task.CompletedTask;
    }
}
```

**Key lines:** CreateAsync returns the PaymentRequest id; redirection to /Payment/GatewaySelection MUST be POST (use LocalRedirectPreserveMethod). Fulfillment subscribes to PaymentRequestCompletedEto so the same code works for any gateway and tolerates webhook retries — guard with a 'this PaymentRequestId already fulfilled' check to avoid duplicate delivery.

### Stripe subscription plan and webhook wiring

_Recurring SaaS billing — define a Plan, link it to a Stripe price, and start a subscription._

```csharp
// 1. In Stripe dashboard, create Product + recurring Price (e.g., price_123).
// 2. In ABP, create Plan + PlanGateway via IPlanAppService:
await _planAppService.CreateAsync(new PlanCreateDto
{
    Name = "Pro Monthly",
    Gateways = new List<PlanGatewayCreateDto>
    {
        new() { Gateway = "Stripe", ExternalGatewayPlanId = "price_123" }
    }
});

// 3. Initiate subscription:
var request = await _paymentRequests.CreateAsync(new PaymentRequestCreateDto
{
    Currency    = "USD",
    PaymentType = PaymentType.Subscription,
    PlanId      = planId
});

// 4. Configure Stripe webhook endpoint at:
//    https://yourdomain.com/api/payment/stripe/webhook
//    Events: customer.subscription.created/updated/deleted, checkout.session.completed
//    Set StripeOptions.WebhookSecret to the signing secret.
```

**Key lines:** PaymentType.Subscription + PlanId is what differentiates a recurring flow. The webhook secret in StripeOptions verifies signatures; without it ABP rejects webhook callbacks. Subscribe to SubscriptionCreated / SubscriptionCanceled events to toggle entitlements server-side.

### Chat SignalR hub authentication for Angular/Blazor WASM

_JWT bearer doesn't ride on WebSocket headers in browsers; you must promote ?access_token=... to an Authorization header on the SignalR path._

```csharp
// In the host module (e.g., the API host) OnApplicationInitialization:
app.Use(async (httpContext, next) =>
{
    if (httpContext.Request.Path.StartsWithSegments("/signalr-hubs/chat") &&
        httpContext.Request.Query.TryGetValue("access_token", out var token))
    {
        httpContext.Request.Headers.Authorization = $"Bearer {token}";
    }
    await next();
});

// Angular: environment.ts
apis: {
    Chat: {
        url:        'https://api.mycompany.com',
        signalRUrl: 'https://api.mycompany.com'
    }
}
// app.config.ts
import { provideChatConfig } from '@volo/abp.ng.chat/config';
export const appConfig: ApplicationConfig = {
    providers: [provideChatConfig()]
};
```

**Key lines:** Without the query-string-to-header rewrite, browsers cannot authenticate the SignalR connection and the chat hub silently rejects. provideChatConfig() registers proxy clients; routes are added separately using createRoutes() from @volo/abp.ng.chat.

### Define a text template in code and override its body via Text Template Management UI

_Email body authored in code (Razor/Scriban), but admins can edit the text in the Administration → Text Templates page per culture._

```csharp
public class WelcomeEmailTemplateProvider : TemplateDefinitionProvider
{
    public override void Define(ITemplateDefinitionContext context)
    {
        context.Add(
            new TemplateDefinition("WelcomeEmail", displayName: LocalizableString.Create<MyResource>("WelcomeEmail"))
                .WithVirtualFilePath("/EmailTemplates/WelcomeEmail.tpl", isInlineLocalized: true));
    }
}

// Send rendered template:
public class WelcomeMailer : ITransientDependency
{
    private readonly ITemplateRenderer _renderer;
    private readonly IEmailSender _email;
    public WelcomeMailer(ITemplateRenderer renderer, IEmailSender email)
    { _renderer = renderer; _email = email; }

    public async Task SendAsync(string to, string name)
    {
        var body = await _renderer.RenderAsync("WelcomeEmail", new { name });
        await _email.SendAsync(to, "Welcome!", body, isBodyHtml: true);
    }
}

// Module:
Configure<TextTemplateManagementOptions>(o =>
    o.MinimumCacheDuration = TimeSpan.FromMinutes(30));
```

**Key lines:** TemplateDefinitionProvider names and registers the template; the Text Template Management module's DatabaseTemplateContentContributor returns DB-stored bodies first (cached for MinimumCacheDuration), falling back to the virtual file. Admins editing in the UI immediately override the embedded version once cache expires.

## Common mistakes

### Leaving NullSmsSender in production after installing the Twilio module but failing to configure AccountSId/AuthToken/FromNumber.

**Why wrong:** Without valid credentials the Twilio sender throws on every send; in some setups the framework still resolves NullSmsSender if AbpSmsTwilioModule is not in the module's DependsOn, silently logging instead of sending.

```csharp
Verify AbpSmsTwilioModule appears in DependsOn, bind AbpTwilioSmsOptions from configuration, and write an integration test that asserts ISmsSender resolves to the Twilio implementation in non-Test environments.
```

### Redirecting to /Payment/GatewaySelection with HTTP GET.

**Why wrong:** The endpoint requires POST so that PaymentRequestId travels in form data and CSRF tokens are honored; GET will 404/405 or break antiforgery.

```csharp
Use LocalRedirectPreserveMethod from a POST-handling endpoint, or build a self-submitting form that POSTs paymentRequestId.
```

### Treating gateway callbacks/webhooks as exactly-once.

**Why wrong:** Stripe, PayPal, and Iyzico all retry callbacks. Without idempotency, the same PaymentRequestId can trigger duplicate fulfillment.

```csharp
Persist a 'fulfilled' flag keyed by PaymentRequestId before delivering goods; subscribe to PaymentRequestCompletedEto and short-circuit if already fulfilled.
```

### Skipping StripeOptions.WebhookSecret.

**Why wrong:** Subscription state events (created/updated/deleted) won't be processed because ABP refuses unsigned webhooks; subscriptions appear to never activate or cancel server-side.

```csharp
Create the webhook in Stripe pointing at /api/payment/stripe/webhook, copy the signing secret into StripeOptions.WebhookSecret, and select the four required events.
```

### Using Alipay with USD or another non-CNY currency.

**Why wrong:** Alipay only accepts CNY; PaymentRequest creation will be rejected by the gateway.

```csharp
Either restrict Alipay to CNY-priced products or hide it from IGatewayAppService for incompatible carts.
```

### Running Chat with split Web and API tiers but no distributed event bus configured.

**Why wrong:** IRealTimeChatMessageSender on the API can't reach SignalR clients connected to the Web tier; messages persist but never appear in real time.

```csharp
Configure RabbitMQ (or another distributed event bus) on both tiers and ensure ChatMessageEto crosses the bus.
```

### Not promoting ?access_token=... to an Authorization header for the chat SignalR path.

**Why wrong:** Browsers can't set Authorization headers on WebSocket transports, so the JWT never reaches the hub and the connection is silently unauthorized.

```csharp
Add the Use(...) middleware that copies the access_token query string to httpContext.Request.Headers.Authorization for paths beginning with '/signalr-hubs/chat'.
```

### Forgetting to enable the 'Chat' feature for the tenant.

**Why wrong:** Even with packages installed, Chat services throw or hide the navigation icon when the feature flag is off — this is by design for tiered SaaS.

```csharp
Enable the Chat feature at host or per-tenant level via the Feature Management UI / IFeatureChecker programmatically.
```

### Editing email body directly in the virtual file system after the Text Template Management module is installed.

**Why wrong:** DatabaseTemplateContentContributor returns DB content first (cached for MinimumCacheDuration); your file edits appear ineffective until DB content is cleared.

```csharp
Edit via Administration → Text Templates, or, for code-managed templates, delete the corresponding TextTemplateContent rows so the virtual file falls through.
```

### Setting MinimumCacheDuration too high during template authoring.

**Why wrong:** Edits in the UI seem to take many minutes to appear; developers blame the UI when it's the cache.

```csharp
Lower MinimumCacheDuration (e.g., TimeSpan.FromSeconds(30)) in Development environments; raise it in Production.
```

### Sending SMS without using SmsMessage when you need provider-specific options.

**Why wrong:** The (phone, text) overload doesn't accept Properties, so Twilio-specific extras (e.g., status callback) aren't passed through.

```csharp
Use SendAsync(new SmsMessage(phone, text) { Properties = { ... } }) and read the keys in a custom ISmsSender if you bypass the standard module.
```

## Version pins (ABP 10.3)

- ABP 10.3 — All four modules require an ABP Team license or higher (Pro). [uncertain] whether 10.4 preview alters licensing tiers.
- Payment table prefix is 'Pay' (PayPaymentRequests, PayPlans, PayGatewayPlans); Chat tables use 'Chat' prefix (ChatUsers, ChatMessages, ChatUserMessages, ChatConversations); Text Template Management uses 'AbpTextTemplateContents'. Prefixes/connection-string keys are configurable via the respective DbProperties classes.
- Default connection-string keys: 'AbpPayment', 'Chat', 'TextTemplateManagement' — each falls back to 'Default' if absent.
- Stripe webhook path: /api/payment/stripe/webhook with required events customer.subscription.created/updated/deleted plus checkout.session.completed (optional). Path/event list is current as of 10.3.
- Iyzico in Blazor requires the additional Volo.Payment.Iyzico.HttpApi package because Blazor cannot directly receive the gateway's POST callback.
- Alipay supports CNY only in 10.3.
- Chat module is not pre-installed in startup templates; Text Template Management IS pre-installed in startup templates.
- Twilio module is the only first-party ISmsSender provider shipped by ABP; other providers must be implemented as ITransientDependency.

## Cross-references

**Phase 1 references:**
- [references/infrastructure/eventing.md](../infrastructure/eventing.md) — Triggered for PaymentRequestCompleted/Subscription* and ChatMessageEto handling, and for the RabbitMQ relay required when Chat's web and API tiers are split.
- [references/framework/multi-tenancy.md](../framework/multi-tenancy.md) — Triggered because Message implements IMultiTenant and PaymentRequest/Plan are tenant-aware; tenant resolution and per-tenant gateway settings matter.
- [references/modules/operations.md](../modules/operations.md) — Triggered for the Chat feature flag (must be enabled to use messaging) and module permission setup for Payment / Text Template Management UIs.
- [references/modules/identity-and-auth.md](../modules/identity-and-auth.md) — Triggered when Twilio SMS is used as a 2FA channel and when Chat consumes IUser/UpdateUserData abstractions.
- [references/infrastructure/blob-storing.md](../infrastructure/blob-storing.md) — Triggered when chat or payment receipts attach files (chat attachments, invoice PDFs).
- [references/infrastructure/background-processing.md](../infrastructure/background-processing.md) — Triggered for retrying SMS/email sends and for asynchronous fulfillment after PaymentRequestCompleted.

**External docs:**
- [Twilio Programmable Messaging API](https://www.twilio.com/docs/sms) — Source of AccountSid, AuthToken, sender numbers, and quotas consumed by Volo.Abp.Sms.Twilio.
- [Stripe API & Webhooks](https://stripe.com/docs/api) — PaymentIntents, Prices, Subscriptions, and webhook signing used by ABP's Stripe gateway and StripeOptions.WebhookSecret.
- [PayPal REST APIs](https://developer.paypal.com/api/rest/) — ClientId/Secret and Sandbox vs Live environment used by PayPalOptions.
- [Iyzico API](https://docs.iyzico.com/) — Iyzipay credentials, customer-data fields, and installment counts mapped onto IyzicoOptions.
- [Alipay Open Platform](https://opendocs.alipay.com/) — AppId, gateway host, RSA keys, and notify URL plumbed through AlipayOptions; CNY-only constraint comes from this provider.
- [ASP.NET Core SignalR](https://learn.microsoft.com/aspnet/core/signalr/introduction) — Underlies Volo.Chat.SignalR hub; query-string token transport and CORS guidance apply directly.
- [RabbitMQ](https://www.rabbitmq.com/documentation.html) — Recommended distributed event bus for relaying chat messages between separated Web and API tiers.
- [Scriban](https://github.com/scriban/scriban) — Alternative templating engine supported by ABP's text-templating system that the Text Template Management UI edits content for.

## Sources (ABP 10.3)

- [https://abp.io/docs/10.3/modules/twilio-sms](https://abp.io/docs/10.3/modules/twilio-sms)
- [https://abp.io/docs/10.3/modules/payment](https://abp.io/docs/10.3/modules/payment)
- [https://abp.io/docs/10.3/modules/chat](https://abp.io/docs/10.3/modules/chat)
- [https://abp.io/docs/10.3/modules/text-template-management](https://abp.io/docs/10.3/modules/text-template-management)
- [https://abp.io/docs/10.3/framework/infrastructure/sms-sending](https://abp.io/docs/10.3/framework/infrastructure/sms-sending)
- [https://abp.io/docs/10.3/framework/infrastructure/text-templating](https://abp.io/docs/10.3/framework/infrastructure/text-templating)

Last verified: 2026-05-10
