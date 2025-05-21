# Custom Results

## InjectIdToken

`com.tremolosecurity.proxy.results.InjectIdToken`

This custom result is designed to use an OpenID Connect session when interacting with Kubernetes through a web browser. For instance when using the dashboard this result will inject an id_token that will be validated by the api server providing authenticated access.

## PersistentCookieResult

`com.tremolosecurity.proxy.auth.persistentCookie.PersistentCookieResult`

Works with the [`com.tremolosecurity.proxy.auth.persistentCookie.PersistentCookie`](/documentation/reference/authentication-mechanisms/#persistentcookie) authentication mechanism to generate a cookie that can be used later to skip authentication.