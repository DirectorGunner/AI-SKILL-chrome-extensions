# Identity & Authentication

> Manifest V3 reference. Original prose; identifiers preserved verbatim.

## Overview
APIs for authentication and credentials: OAuth2 / web auth flows (`chrome.identity`), ChromeOS session/profile state (`chrome.loginState`), smart-card / certificate provisioning and signing (`chrome.certificateProvider`, `chrome.platformKeys`), and proxying WebAuthn to a remote authenticator (`chrome.webAuthenticationProxy`).

## chrome.identity
- **Purpose:** obtain OAuth2 access tokens and run interactive web auth flows.
- **Permission:** `"identity"`.
- **Key methods:**
  - `getAuthToken(details?)` resolves to a `GetAuthTokenResult`. Uses the `client_id` and `scopes` from the manifest `oauth2` key. Set `interactive: true` to allow Chrome to prompt; otherwise it fails silently when interaction is required. Google accounts only.
  - `getRedirectURL(path?)` returns a string redirect URL for `launchWebAuthFlow`, of the form `https://EXTENSION_ID.chromiumapp.org/`.
  - `launchWebAuthFlow(details)` resolves to the redirect URL string. Runs an OAuth2 flow against any provider in a browser window — use this for non-Google providers.
  - `removeCachedAuthToken(details)` removes one token from the in-memory cache.
  - `clearAllCachedAuthTokens()` (Chrome 87+) clears all cached tokens and authorizations.
  - `getProfileUserInfo(details?)` resolves to a `ProfileUserInfo`; `getAccounts()` (Dev channel) resolves to `AccountInfo[]`.
- **Key events:** `onSignInChanged` → `(account, signedIn)`.
- **Key types:** `TokenDetails` (`account`, `scopes`, `interactive`, `enableGranularPermissions` (Chrome 87+)); `GetAuthTokenResult` (Chrome 105+: `token`, `grantedScopes` (Chrome 87+)); `WebAuthFlowDetails` (`url`, `interactive`, `abortOnLoadForNonInteractive` (Chrome 113+), `timeoutMsForNonInteractive` (Chrome 113+)); `InvalidTokenDetails` (`token`); `ProfileUserInfo` (`email`, `id`); `AccountStatus` (`"SYNC"`, `"ANY"`).
- **Manifest `oauth2` key** (required for `getAuthToken`):

```json
"oauth2": {
  "client_id": "yourExtensionOAuthClientID.apps.googleusercontent.com",
  "scopes": ["https://www.googleapis.com/auth/contacts.readonly"]
}
```

- **MV3 notes:** Promise-capable in MV3 (callbacks still work). Use `removeCachedAuthToken`/`clearAllCachedAuthTokens` to recover from revoked tokens. A `"key"` manifest field keeps a stable extension id (and OAuth `client_id`) during development.

```js
chrome.identity.getAuthToken({ interactive: true }, (token) => {
  if (chrome.runtime.lastError || !token) {
    console.error(chrome.runtime.lastError?.message);
    return;
  }
  // If the API later rejects the token: chrome.identity.removeCachedAuthToken({ token }, () => {});
});
```

## chrome.loginState
- **Purpose:** read the ChromeOS profile type and session state. ChromeOS only (Chrome 78+; Promises Chrome 96+).
- **Permission:** `"loginState"`.
- **Key methods:** `getProfileType()` resolves to a `ProfileType`; `getSessionState()` resolves to a `SessionState`.
- **Key events:** `onSessionStateChanged` → `(sessionState)`.
- **Key types:** `ProfileType` (`"SIGNIN_PROFILE"`, `"USER_PROFILE"`, `"LOCK_PROFILE"`); `SessionState` (`"UNKNOWN"`, `"IN_OOBE_SCREEN"`, `"IN_LOGIN_SCREEN"`, `"IN_SESSION"`, `"IN_LOCK_SCREEN"`, `"IN_RMA_SCREEN"`).

```js
const state = await chrome.loginState.getSessionState();
if (state === "IN_SESSION") console.log("active user session");
```

## chrome.certificateProvider
- **Purpose:** expose client certificates (e.g. smart card) to ChromeOS and perform TLS client-auth signing, including PIN dialogs. ChromeOS only (Chrome 46+).
- **Permission:** `"certificateProvider"`.
- **Key methods:** `setCertificates(details)`; `reportSignature(details)`; `requestPin(details)` resolves to a `PinResponseDetails`; `stopPinRequest(details)`.
- **Key events:** `onCertificatesUpdateRequested` (respond with `setCertificates`); `onSignatureRequested` (respond with `reportSignature`).
- **Key types/enums:** `Algorithm` (`"RSASSA_PKCS1_v1_5_SHA1"`, `"RSASSA_PKCS1_v1_5_SHA256"`, `"RSASSA_PKCS1_v1_5_SHA384"`, `"RSASSA_PKCS1_v1_5_SHA512"`, `"RSASSA_PSS_SHA256"`, `"RSASSA_PSS_SHA384"`, `"RSASSA_PSS_SHA512"`); `Error` (`"GENERAL_ERROR"`); `PinRequestType` (`"PIN"`, `"PUK"`); `PinRequestErrorType` (`"INVALID_PIN"`, `"INVALID_PUK"`, `"MAX_ATTEMPTS_EXCEEDED"`, `"UNKNOWN_ERROR"`); `ClientCertificateInfo` (`certificateChain`, `supportedAlgorithms`); `SignatureRequest` (`signRequestId`, `input`, `algorithm`, `certificate`).
- **MV3 notes:** request-driven — register the events in the service worker and answer asynchronously, correlating by `certificatesRequestId` / `signRequestId`.

```js
chrome.certificateProvider.onSignatureRequested.addListener(async (req) => {
  try {
    const signature = await mySmartCardSign(req.input, req.algorithm, req.certificate);
    await chrome.certificateProvider.reportSignature({ signRequestId: req.signRequestId, signature });
  } catch {
    await chrome.certificateProvider.reportSignature({ signRequestId: req.signRequestId, error: "GENERAL_ERROR" });
  }
});
```

## chrome.platformKeys
- **Purpose:** access ChromeOS-managed client certificates and use their keys for signing/decryption via a WebCrypto-style interface. ChromeOS only (Chrome 45+).
- **Permission:** `"platformKeys"`.
- **Key methods:** `selectClientCertificates(details)` resolves to `Match[]`; `getKeyPair(certificate, parameters, callback)`; `getKeyPairBySpki(publicKeySpkiDer, parameters, callback)` (Chrome 85+); `subtleCrypto()`; `verifyTLSServerCertificate(details)` (Chrome 121+).
- **Key types/enums:** `ClientCertificateType` (`"rsaSign"`, `"ecdsaSign"`); `SelectDetails` (`clientCerts?`, `interactive`, `request`); `Match` (`certificate`, `keyAlgorithm`); `VerificationResult` (`trusted`, `debug_errors`).

```js
const matches = await chrome.platformKeys.selectClientCertificates({
  interactive: true,
  request: { certificateAuthorities: [], certificateTypes: ["rsaSign"] },
});
```

## chrome.webAuthenticationProxy
- **Purpose:** act as the proxy for WebAuthn requests, forwarding `navigator.credentials.create()` / `.get()` and platform-authenticator checks to a remote authenticator. Chrome 115+.
- **Permission:** `"webAuthenticationProxy"`.
- **Key methods:** `attach()`; `detach()`; `completeCreateRequest(details)`; `completeGetRequest(details)`; `completeIsUvpaaRequest(details)`.
- **Key events:** `onCreateRequest` → `(CreateRequest)`; `onGetRequest` → `(GetRequest)`; `onIsUvpaaRequest` → `(IsUvpaaRequest)`; `onRemoteSessionStateChange`; `onRequestCanceled` → `(requestId)`.
- **Key types:** `CreateRequest`/`GetRequest` (`requestId`, `requestDetailsJson`); `IsUvpaaResponseDetails` (`requestId`, `isUvpaa`); `DOMExceptionDetails` (`name`, `message`).
- **MV3 notes:** register request events in the service worker; respond with the matching `complete*Request`, echoing the same `requestId`; surface failures via the `error` (`DOMExceptionDetails`) field.

```js
chrome.webAuthenticationProxy.onIsUvpaaRequest.addListener((req) => {
  chrome.webAuthenticationProxy.completeIsUvpaaRequest({ requestId: req.requestId, isUvpaa: false });
});
await chrome.webAuthenticationProxy.attach();
```
