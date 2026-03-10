---
title: Pagination
description: Cursor-based pagination and auto-pagination with LazyCollection for customers, billing statements, payout transactions, and webhooks.
---

# Pagination

Resources that return lists of items (customers, billing statements, payout transactions, webhooks) use cursor-based pagination. The package provides a `PayrexCollection` class with built-in auto-pagination support.

## Basic Pagination

List methods return a `PayrexCollection` with a `data` array and a `hasMore` flag:

```php
use LegionHQ\LaravelPayrex\Facades\Payrex;

$customers = Payrex::customers()->list(['limit' => 10]);

foreach ($customers->data as $customer) {
    echo $customer->name;
}

// Check if there are more pages
if ($customers->hasMore) {
    // Fetch next page...
}
```

### Pagination Parameters

All list methods accept these cursor-based pagination parameters:

| Parameter | Type | Description |
|---|---|---|
| `limit` | `int` | Number of items to return (default varies by resource) |
| `after` | `string` | Return items after this resource ID (forward pagination) |
| `before` | `string` | Return items before this resource ID (backward pagination) |

```php
// First page
$page1 = Payrex::customers()->list(['limit' => 10]);

// Next page (forward)
$lastId = end($page1->data)->id;
$page2 = Payrex::customers()->list(['limit' => 10, 'after' => $lastId]);

// Previous page (backward)
$firstId = $page2->data[0]->id;
$prevPage = Payrex::customers()->list(['limit' => 10, 'before' => $firstId]);
```

## Auto-Pagination

For iterating over all items across all pages, use `autoPaginate()`. It returns a Laravel `LazyCollection` that fetches pages on demand:

```php
$allCustomers = Payrex::customers()->list(['limit' => 100])->autoPaginate(); // [!code focus]

// Iterates through ALL customers across all pages
foreach ($allCustomers as $customer) {
    echo $customer->name;
}
```

`autoPaginate()` is **lazy** — it only fetches the next page when you've iterated through the current one. This makes it memory-efficient even with thousands of items.

### Page Limit Safety Valve

By default, `autoPaginate()` stops after 100 pages to prevent runaway loops. You can adjust this limit:

```php
// Stop after 50 pages
$customers = Payrex::customers()->list()->autoPaginate(maxPages: 50);

// No limit (use with caution)
$allCustomers = Payrex::customers()->list()->autoPaginate(maxPages: 0);
```

::: warning
Setting `maxPages` to `0` removes the safety limit entirely. Only do this if you're confident the dataset is bounded, or use `take()` to limit results instead.
:::

### Using Laravel Collection Methods

Since `autoPaginate()` returns a `LazyCollection`, you can chain any Laravel collection method:

```php
// Filter and transform
$premiumEmails = Payrex::customers()->list()
    ->autoPaginate()
    ->filter(fn ($customer) => $customer->metadata['tier'] === 'premium') // [!code focus]
    ->map(fn ($customer) => $customer->email) // [!code focus]
    ->all();

// Take the first 50
$first50 = Payrex::customers()->list()
    ->autoPaginate()
    ->take(50) // [!code focus]
    ->all();

// Count all items (fetches all pages)
$total = Payrex::customers()->list()->autoPaginate()->count();
```

::: warning
Methods like `count()`, `all()`, or `toArray()` will fetch **all pages** before returning. For large datasets, prefer streaming with `foreach` or use `take()` to limit results.
:::

### Filtering and Lookup

Some resources support filter parameters alongside pagination — useful both for narrowing results and as a **lookup alternative to `retrieve()`** when you don't have the resource ID.

::: tip Use `list()` as a find/lookup
If you know a customer's email but not their PayRex ID, use `list()` with filters instead of `retrieve()`:
```php
$customers = Payrex::customers()->list([
    'email' => 'juan@example.com',
]);

$customer = $customers->data[0] ?? null; // First match
```
:::

Filters are preserved across auto-paginated pages:

```php
// Auto-paginate through all customers named "Juan"
$juans = Payrex::customers()->list([
    'name' => 'Juan',
    'limit' => 50,
])->autoPaginate();

foreach ($juans as $customer) {
    echo $customer->email;
}

// Filter by metadata
$customers = Payrex::customers()->list([
    'metadata' => ['internal_id' => '12345'],
])->autoPaginate();
```

#### Resources with Filter Support

| Resource | Available Filters |
|---|---|
| [Customers](/api/customers#list-customers) | `name`, `email`, `metadata` |
| [Webhooks](/api/webhooks-resource#list-webhook-endpoints) | `url`, `description` |

Billing Statements and Payout Transactions support `list()` with pagination parameters (`limit`, `before`, `after`) but do not support additional filters.

## PayrexCollection Properties

| Property | Type | Description |
|---|---|---|
| `$collection->data` | `array<PayrexObject>` | Array of items on the current page |
| `$collection->hasMore` | `bool` | Whether more items exist beyond this page |
| `$collection->resource` | `?string` | The resource type (e.g., `list`) |

`PayrexCollection` implements `Countable`, `IteratorAggregate`, `ArrayAccess`, and `JsonSerializable`:

```php
$customers = Payrex::customers()->list();

// Countable
echo count($customers); // Number of items on this page

// IteratorAggregate — iterate directly
foreach ($customers as $customer) {
    echo $customer->name;
}

// ArrayAccess
$typedData = $customers['data']; // Same as $customers->data (typed DTOs)

// JSON serializable
$json = json_encode($customers);
```

## Resources with Pagination

| Resource | Method | Example |
|---|---|---|
| Customers | `customers->list()` | `Payrex::customers()->list(['limit' => 10])` |
| Billing Statements | `billingStatements->list()` | `Payrex::billingStatements()->list()` |
| Payout Transactions | `payoutTransactions->list($payoutId)` | `Payrex::payoutTransactions()->list('po_xxxxx')` |
| Webhooks | `webhooks->list()` | `Payrex::webhooks()->list()` |

::: info Payout Transactions
Note that `payoutTransactions->list()` requires a payout ID as the first argument, unlike other list methods. See [Payout Transactions](/api/payout-transactions) for details.
:::

## Further Reading

- [Customers API](/api/customers) — List and filter customers
- [Billing Statements API](/api/billing-statements) — List billing statements
- [Payout Transactions API](/api/payout-transactions) — List payout transactions
- [Webhooks API](/api/webhooks-resource) — List webhook endpoints
