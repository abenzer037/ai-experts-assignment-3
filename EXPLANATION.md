# Bug Fix Explanation

## What was the bug?

The `Client.request()` method failed to refresh expired OAuth2 tokens when the token was stored as a dict-shaped tokens instead of an `OAuth2Token` object. This caused API requests to proceed without the required `Authorization` header when a dict token had an expired `expires_at` timestamp.

## Why did it happen?

The refresh condition only checked for `OAuth2Token` instances:
```python
if not self.oauth2_token or (
    isinstance(self.oauth2_token, OAuth2Token) and self.oauth2_token.expired
):
```

When `oauth2_token` was a dict, the condition `isinstance(self.oauth2_token, OAuth2Token)` returned `False`, so the expiration check was skipped entirely. The token was never refreshed, and since the authorization header is only set for `OAuth2Token` objects, the request failed.

## Why does your fix actually solve it?

The fix adds an explicit check for dict-type tokens:
```python
or (
    isinstance(self.oauth2_token, dict)
    and int(time.time()) >= int(self.oauth2_token.get("expires_at", 0))
)
```

Now when a dict token's `expires_at` is in the past (timestamp <= now), it triggers a refresh. After refresh, `oauth2_token` becomes an `OAuth2Token` object, so the authorization header is properly set in subsequent code.

## One realistic edge case not covered

**Type mismatch or invalid string in dict tokens:** If `expires_at` in the dict is a non-integer or a non-numeric string (e.g., `"expires_at": "never"` or `"expires_at": ""`), the code will raise a `ValueError` when trying to convert it to `int`. This will cause the request to fail with an exception, and the token will not be refreshed. The fix assumes `expires_at` is always a valid integer or numeric string, but does not handle invalid or unexpected string values.
