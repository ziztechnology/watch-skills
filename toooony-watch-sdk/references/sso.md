# One-Time SSO Tokens

## Enable the Runtime Handler

Enable the `requestSSOToken` Handler in the watch-face template's `customFields` before calling the SDK. Use only the package-root export; do not call a Runtime-injected method on `window`.

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

Both forms return `Promise<SSOToken | null>`. Request a fresh token immediately before every protected HTTP request.

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

With SDK 0.1.4 or later, `requestSSOToken()` transparently waits for the managed Runtime Bridge-ready signal for up to 3 seconds. For an older Runtime that initially rejects with `BRIDGE_DISABLED` and `bridge page inactive`, it waits for the existing `toooony-sso-ready` authentication context and then retries the Runtime call when necessary. Do not add application-level Runtime-readiness polling.

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

A missing Runtime or missing `requestSSOToken` method returns `null`. A `null` or malformed Runtime response also returns `null`. Passing an empty `clientID` or `scope` throws `TypeError`. Failures from an available Runtime method, including permission and network failures, reject the Promise and preserve the original error.

## Protect the One-Time Token

- Send the token only in the `X-Toooony-SSO-Token` request header required by the protected API.
- Do not put the token in a URL, query string, log, analytics event, persistent cache, browser storage, or React/Vue state.
- Do not reuse one token across multiple protected requests.
- Do not retry a failed HTTP request with the same token. Request a new token before a deliberate retry.
- Do not copy a token into error objects or diagnostics. Record only non-secret context needed to investigate the failure.
