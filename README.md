# Reessolutions_WorkerModeHyva

Fixes Hyvä CSP state leaking across requests when running Magento 2 under FrankenPHP worker mode.

## The problem

Hyvä's `HyvaCsp` ViewModel caches two values after its first call per request — the collected CSP fetch policies (`$memoizedPolicies`) and the current area code (`$memoizedAreaCode`). Under normal PHP-FPM, this is fine: the process resets on every request. Under FrankenPHP worker mode, the process is long-lived and `HyvaCsp` is a shared singleton, so those cached values persist into the next request.

This causes a subtle but severe bug: if any request runs under a strict CSP (e.g. the checkout page, which doesn't allow `unsafe-eval`), `$memoizedPolicies` is set without `unsafe-eval`. Every *subsequent* request then sees `isEvalAllowed() = false` and loads `alpine3-csp.min.js` (the CSP Alpine build) instead of the standard Alpine. The CSP Alpine build cannot evaluate string expressions, so **every Alpine component on every page throws an error** until the worker process is recycled.

The stale `$memoizedAreaCode` causes a second issue: `isAreaCodeSet()` short-circuits to the wrong area, suppressing nonce/hash handling on pages running under a different area.

## The fix

This module subclasses `HyvaCsp` and implements `ResetAfterRequestInterface`. The `_resetState()` method nullifies both private parent properties via reflection at the end of each request, so the next request starts with a clean slate.

## Requirements

| Dependency | Version |
|---|---|
| PHP | 8.0 – 8.5 |
| `reessolutions/module-worker-mode` | ^1.0 |
| `hyva-themes/magento2-theme-module` | * |

## Installation

```bash
composer require reessolutions/module-worker-mode-hyva
bin/magento module:enable Reessolutions_WorkerModeHyva
bin/magento setup:upgrade
```

## How it works

`etc/di.xml` registers a preference that replaces `Hyva\Theme\ViewModel\HyvaCsp` with `Reessolutions\WorkerModeHyva\ViewModel\HyvaCsp`. The subclass adds only one method:

```php
public function _resetState(): void
{
    // nullifies $memoizedPolicies and $memoizedAreaCode on the parent via reflection
}
```

Magento's object manager calls `_resetState()` on every class that implements `ResetAfterRequestInterface` at the end of each worker-mode request.

## License

OSL-3.0 — see [LICENSE](LICENSE).
