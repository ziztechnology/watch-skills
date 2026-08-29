# One-Time SSO Tokens

## Select the Standard and Compatibility Paths

Use only the package-root SDK export in an SDK integration. SDK 0.2.0 first calls the public `t4ony.requestSSOToken()` method when it exists. It uses the old `window.requestSSOToken()` compatibility Bridge only when the standard method is missing.

The standard t4ony path is available to supported packaged H5 watch faces without an old Handler. Enable the `requestSSOToken` Handler in the watch-face template's `customFields` only when the application must support an older Runtime through the compatibility fallback. Do not call the old `window.requestSSOToken()` method directly.

## Request and Use a Token

Call `requestSSOToken()` without arguments when the deployed watch face already has its SSO client and scope binding:

```ts
import { requestSSOToken } from '@ziztechnology/dial-library';

const ssoToken = await requestSSOToken();
```

Use the no-argument form with SDK 0.1.3 or later. Follow the locked package's declarations when maintaining an older project.

Pass `clientID` and `scope` when the request must specify that binding explicitly:

```ts
const ssoToken = await requestSSOToken({
  clientID: 'finance-watch-face',
  scope: 'finance.read',
});
```

Both forms return `Promise<SSOToken | null>`. The SDK normalizes standard and compatibility responses to the same `SSOToken` contract. Request a fresh token immediately before every protected HTTP request.

Use the exported `RequestSSOTokenOptions` and `SSOToken` types when a project wraps the request or stores the result in a local variable for the duration of one HTTP call.

```ts
import { requestSSOToken } from '@ziztechnology/dial-library';

const loadFinanceData = async (): Promise<Response | null> => {
  const ssoToken = await requestSSOToken({
    clientID: 'finance-watch-face',
    scope: 'finance.read',
  });

  if (!ssoToken) return null;

  return fetch(new URL('/api/v1/finance/a-stock-app', ssoToken.issuer), {
    headers: {
      'X-Toooony-SSO-Token': ssoToken.token,
    },
  });
};
```

Use `issuer` as the HTTPS API origin supplied with the token. The returned `SSOToken` also exposes `expiresInSeconds`, `expiresAtMs`, `deviceID`, `userID`, and `authorizeURL` when the application needs those values.

The SDK handles compatibility-path Runtime readiness. Do not add application-level Runtime-readiness polling.

## Handle Absence and Failure

Treat a `null` result as an unavailable or invalid token and show the application's signed-out, unavailable, or retry entry state. Keep rejected calls in a separate error path:

```ts
try {
  const ssoToken = await requestSSOToken();

  if (!ssoToken) {
    showAuthenticationUnavailable();
  } else {
    await callProtectedAPI(ssoToken);
  }
} catch (error) {
  reportAuthenticationFailure(error);
}
```

Treat a missing Runtime, missing methods on both paths, or an invalid selected-path response as `null`. Treat empty option values as `TypeError` when a Runtime path is available. Preserve rejections from the selected method as failures, and do not retry an invalid or rejected standard call through the legacy Bridge.

## Protect the One-Time Token

- Send the token only in the `X-Toooony-SSO-Token` request header required by the protected API.
- Do not put the token in a URL, query string, log, analytics event, persistent cache, browser storage, or React/Vue state.
- Do not reuse one token across multiple protected requests.
- Do not retry a failed HTTP request with the same token. Request a new token before a deliberate retry.
- Do not copy a token into error objects or diagnostics. Record only non-secret context needed to investigate the failure.
