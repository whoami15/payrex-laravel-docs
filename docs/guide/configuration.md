---
title: Configuration
description: Configure API keys, timeouts, retries, webhooks, and default currency for the Laravel PayRex package.
---

# Configuration

The package works out of the box with just environment variables — publishing the config file is optional. If you need to customize beyond what env vars offer, publish it:

```bash
php artisan vendor:publish --tag="payrex-config"
```

This creates `config/payrex.php` where you can customize the package behavior.

## Full Config Reference

```php
return [
    // Your PayRex public API key for client-side integrations (e.g. Elements).
    // This key is safe to expose in frontend code.
    'public_key' => env('PAYREX_PUBLIC_KEY', ''), // [!code focus]

    // Your PayRex secret API key for backend API requests.
    // Find your keys at: Dashboard > Developers > API Keys
    'secret_key' => env('PAYREX_SECRET_KEY', ''), // [!code focus]

    // The base URL for the PayRex API.
    'api_base_url' => env('PAYREX_API_BASE_URL', 'https://api.payrexhq.com'),

    'webhook' => [
        // Whether to register the webhook route.
        // Set to true to enable the built-in webhook route.
        'enabled' => env('PAYREX_WEBHOOK_ENABLED', false), // [!code focus]

        // Your webhook's secret key for signature verification.
        'secret' => env('PAYREX_WEBHOOK_SECRET', ''), // [!code focus]

        // The URI path where PayRex will send webhook events.
        'path' => env('PAYREX_WEBHOOK_PATH', 'payrex/webhook'),

        // Max age in seconds for webhook timestamps (0 = disabled).
        'tolerance' => env('PAYREX_WEBHOOK_TOLERANCE', 300),
    ],

    // Default currency for all API requests.
    // Automatically applied when you don't pass a currency parameter.
    // PayRex currently only supports PHP (Philippine Peso).
    'currency' => env('PAYREX_CURRENCY', 'PHP'), // [!code focus]

    // HTTP request and connection timeouts in seconds.
    'timeout' => env('PAYREX_TIMEOUT', 30),
    'connect_timeout' => env('PAYREX_CONNECT_TIMEOUT', 30),

    // Automatic retry configuration for server errors (5xx).
    // Client errors (4xx) are never retried.
    'retries' => env('PAYREX_RETRIES', 0),
    'retry_delay' => env('PAYREX_RETRY_DELAY', 100),
];
```

## Environment Variables

| Variable | Description | Default |
|---|---|---|
| `PAYREX_PUBLIC_KEY` | Public API key for frontend integrations | `''` |
| `PAYREX_SECRET_KEY` | Secret API key for backend requests | `''` |
| `PAYREX_API_BASE_URL` | PayRex API base URL | `https://api.payrexhq.com` |
| `PAYREX_CURRENCY` | Default currency (auto-applied to requests) | `PHP` |
| `PAYREX_TIMEOUT` | Request timeout in seconds | `30` |
| `PAYREX_CONNECT_TIMEOUT` | Connection timeout in seconds | `30` |
| `PAYREX_RETRIES` | Retry attempts for server errors (0 = disabled) | `0` |
| `PAYREX_RETRY_DELAY` | Delay between retries in milliseconds | `100` |
| `PAYREX_WEBHOOK_ENABLED` | Whether to register the webhook route | `false` |
| `PAYREX_WEBHOOK_SECRET` | Webhook signature verification secret | `''` |
| `PAYREX_WEBHOOK_PATH` | Webhook endpoint path | `payrex/webhook` |
| `PAYREX_WEBHOOK_TOLERANCE` | Max webhook timestamp age in seconds | `300` |

## Default Currency

The `currency` config option is automatically applied to API requests that accept a currency parameter (payment intents, checkout sessions, refunds, customers, billing statements). You don't need to pass `'currency' => 'PHP'` on every call:

```php
// Currency is automatically set from config
$paymentIntent = Payrex::paymentIntents()->create([
    'amount' => 10000,
    'payment_methods' => ['card'],
]);

// You can still override it explicitly
$paymentIntent = Payrex::paymentIntents()->create([
    'amount' => 10000,
    'currency' => 'PHP',
    'payment_methods' => ['card'],
]);

// Access the configured default currency programmatically
$currency = Payrex::defaultCurrency(); // 'PHP'
```

## HTTP Timeouts

Customize the request and connection timeouts (in seconds) for all API calls:

```ini
PAYREX_TIMEOUT=60
PAYREX_CONNECT_TIMEOUT=10
```

- **`timeout`** — Maximum time to wait for a response from the PayRex API.
- **`connect_timeout`** — Maximum time to wait when establishing a connection.

Both default to 30 seconds.

## Retry on Server Errors

The package can automatically retry failed requests when the PayRex API returns a server error (5xx status codes). Client errors (4xx) are never retried.

Retries are disabled by default. To enable:

```ini
PAYREX_RETRIES=3
PAYREX_RETRY_DELAY=200
```

- **`retries`** — Number of retry attempts. Set to `0` to disable.
- **`retry_delay`** — Delay between retries in milliseconds.

::: warning
Be cautious when enabling retries for write operations (creating payments, refunds, etc.) — a retried request may create a duplicate resource if the first request actually succeeded but the response was lost due to a network error. Use [idempotency keys](/guide/error-handling#idempotency-keys) to prevent this.
:::

## Webhook Configuration

### Enabling the Webhook Route

The webhook route is disabled by default. To use the built-in webhook route, enable it in your `.env`:

```ini
PAYREX_WEBHOOK_ENABLED=true
```

If you need extra middleware (e.g., rate limiting, logging), keep the route disabled and register your own using the package's built-in controller. Events are dispatched automatically just like with the built-in route:

```php
use LegionHQ\LaravelPayrex\Http\Controllers\WebhookController;
use LegionHQ\LaravelPayrex\Middleware\VerifyWebhookSignature;

Route::post('my/custom/webhook', WebhookController::class) // [!code focus:3]
    ->middleware(VerifyWebhookSignature::class)
    ->name('payrex.webhook');
```

::: tip Need full control?
If you need to build your own controller entirely (e.g., for multi-tenant setups with different webhook secrets per tenant), use [`constructEvent()`](/guide/webhooks#construct-event) to verify signatures and construct typed events in your own controller.
:::

::: tip Ready to handle webhook events?
Once the webhook route is enabled, head to the [Webhook Handling](/guide/webhooks) guide to set up event listeners for payment notifications, refund updates, and more.
:::

### Webhook Path

The package automatically registers a POST route at the configured webhook path. By default, this is `/payrex/webhook`.

If your application URL is `https://example.com`, enter the following URL in your [PayRex Dashboard](https://dashboard.payrexhq.com/t/developers/webhooks) under **Developers > Webhooks**:

```
https://example.com/payrex/webhook
```

Customize the path:

```ini
PAYREX_WEBHOOK_PATH=api/webhooks/payrex
```

This would change the webhook URL to `https://example.com/api/webhooks/payrex`.

::: tip
The webhook endpoint must be publicly accessible and use HTTPS. During local development, use a tunneling tool like [ngrok](https://ngrok.com), [Expose](https://expose.dev), or [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/) to expose your local server.
:::

### Timestamp Tolerance

The webhook signature verification includes a timestamp check to prevent replay attacks. By default, webhooks older than 300 seconds (5 minutes) are rejected.

Set to `0` to disable the timestamp check:

```ini
PAYREX_WEBHOOK_TOLERANCE=0
```

### CSRF Exclusion

The webhook route is registered outside of the `web` middleware group, so CSRF protection does not apply by default. If you register the webhook route manually within the `web` group, you must exclude it from CSRF verification.

**bootstrap/app.php:**

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->validateCsrfTokens(except: [ // [!code focus:3]
        'my/custom/webhook',
    ]);
})
```

---

**Next up:** [Choosing an Integration](/guide/choosing-an-integration) — decide between Checkout Sessions and Payment Intents + Elements.

## Further Reading

- [Webhook Handling](/guide/webhooks) — Set up event listeners for payment notifications
- [Error Handling](/guide/error-handling) — Handle API errors gracefully
- [Testing](/guide/testing) — Test your integration with `Http::fake()` and webhook test commands
