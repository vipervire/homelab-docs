# Authentik Forward Auth with Nginx Proxy Manager

A practical guide for deploying Authentik forward authentication in front of applications proxied by Nginx Proxy Manager (NPM), with emphasis on the most common failure mode:

```text
auth request unexpected status: 404 while sending to client
```

---

## What this setup does

With forward auth, NPM sits in front of your application and asks Authentik whether a request should be allowed.

Request flow:

1. A user visits `https://app.example.com`
2. NPM receives the request
3. NPM sends an internal auth subrequest to Authentik at `/outpost.goauthentik.io/auth/nginx`
4. Authentik responds:
   - `2xx` → allow
   - `401` / `403` → deny or redirect to sign-in
   - anything else, especially `404` → Nginx treats it as an upstream error and returns `500`

---

## The key rule

For **single-application forward auth**, Authentik must see the **protected app hostname**.

Example:

- Protected app: `cloud.example.com`
- Authentik UI: `auth.example.com`

When NPM asks Authentik to validate access for `cloud.example.com`, Authentik must still receive:

- `Host: cloud.example.com`
- `X-Original-URL: https://cloud.example.com/...`

If Authentik instead sees:

- `Host: auth.example.com`

it may fail to match the request to the correct provider and return `404`.

That `404` becomes a `500` in Nginx.

---

# Architecture options

## Recommended topology

The most reliable design is to let the NPM instance protecting the app reach an Authentik outpost directly over an internal network path.

```text
+--------+        +------------------+        +------------------+
| Client | -----> | NPM for app      | -----> | Protected app    |
|        |        | app.example.com  |        | backend service  |
+--------+        +------------------+        +------------------+
                         |
                         | auth_request to
                         v
                  +------------------+
                  | Authentik outpost|
                  | internal IP:9000 |
                  +------------------+
                         |
                         v
                  +------------------+
                  | Authentik server |
                  +------------------+
```

This avoids host/header confusion.

---

## Common broken topology

This is where many `404 -> 500` problems happen:

```text
+--------+        +-------------------+
| Client | -----> | NPM for app       |
+--------+        | app.example.com   |
                  +-------------------+
                           |
                           | auth_request to public Authentik URL
                           v
                  +-------------------+
                  | Second NPM        |
                  | auth.example.com  |
                  +-------------------+
                           |
                           v
                  +-------------------+
                  | Authentik server  |
                  +-------------------+
```

What often breaks here:

- The first NPM sends the auth subrequest to `auth.example.com`
- The second NPM forwards to Authentik
- The original app hostname is lost
- Authentik no longer knows the request belongs to `app.example.com`
- `/auth/nginx` returns `404`

---

# How the auth flow works

## Healthy flow

```text
Client -> NPM -> /outpost.goauthentik.io/auth/nginx
                 Host: app.example.com
                 X-Original-URL: https://app.example.com/

Authentik -> 401 if not logged in
NPM -> redirect to /outpost.goauthentik.io/start?rd=...
```

Later:

```text
Client logs in via Authentik
Client returns to app
NPM -> /auth/nginx again
Authentik -> 200
NPM -> request forwarded to backend app
```

## Broken flow

```text
Client -> NPM -> /outpost.goauthentik.io/auth/nginx
                 Host: auth.example.com
                 X-Original-URL: https://app.example.com/

Authentik -> 404 (wrong app/host context)
NPM -> 500 to client
```

---

# Required Authentik configuration

## Provider
Use:

- **Forward auth (single application)**

for each protected app.

## External host
Set the provider external host to the protected application URL, for example:

```text
https://app.example.com
```

Not the Authentik URL.

## Outpost assignment
Ensure the application/provider is assigned to the outpost you are actually using.

---

# NPM configuration

## Copy/paste working template

Use this in the **Custom Nginx Configuration** field for the protected host in NPM.

Replace:

- `AUTHENTIK_OR_OUTPOST_INTERNAL_IP`
- backend host settings if needed

```nginx
# Buffer tuning for larger auth headers/cookies
proxy_buffers 8 16k;
proxy_buffer_size 32k;

# Avoid redirecting to wrong ports
port_in_redirect off;

location / {
    proxy_pass $forward_scheme://$server:$port;

    # WebSocket support
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $http_connection;
    proxy_http_version 1.1;

    # Authentik forward auth
    auth_request /outpost.goauthentik.io/auth/nginx;
    error_page 401 = @goauthentik_proxy_signin;

    # Pass session cookies back from outpost
    auth_request_set $auth_cookie $upstream_http_set_cookie;
    add_header Set-Cookie $auth_cookie;

    # Map Authentik headers into variables
    auth_request_set $authentik_username $upstream_http_x_authentik_username;
    auth_request_set $authentik_groups $upstream_http_x_authentik_groups;
    auth_request_set $authentik_entitlements $upstream_http_x_authentik_entitlements;
    auth_request_set $authentik_email $upstream_http_x_authentik_email;
    auth_request_set $authentik_name $upstream_http_x_authentik_name;
    auth_request_set $authentik_uid $upstream_http_x_authentik_uid;

    # Forward identity headers to the app
    proxy_set_header X-authentik-username $authentik_username;
    proxy_set_header X-authentik-groups $authentik_groups;
    proxy_set_header X-authentik-entitlements $authentik_entitlements;
    proxy_set_header X-authentik-email $authentik_email;
    proxy_set_header X-authentik-name $authentik_name;
    proxy_set_header X-authentik-uid $authentik_uid;

    # Uncomment only if using HTTP Basic auth passthrough in Authentik
    # auth_request_set $authentik_auth $upstream_http_authorization;
    # proxy_set_header Authorization $authentik_auth;
}

location /outpost.goauthentik.io {
    proxy_pass http://AUTHENTIK_OR_OUTPOST_INTERNAL_IP:9000/outpost.goauthentik.io;

    # Critical: preserve the original protected app hostname
    proxy_set_header Host $host;

    # Critical: preserve the original requested URL
    proxy_set_header X-Original-URL $scheme://$http_host$request_uri;

    # Cookie passthrough from Authentik
    add_header Set-Cookie $auth_cookie;
    auth_request_set $auth_cookie $upstream_http_set_cookie;

    # No body required for auth subrequests
    proxy_pass_request_body off;
    proxy_set_header Content-Length "";
}

location @goauthentik_proxy_signin {
    internal;
    add_header Set-Cookie $auth_cookie;
    return 302 /outpost.goauthentik.io/start?rd=$scheme://$http_host$request_uri;
}
```

---

# The two most important lines

These are the lines that usually determine whether the setup works:

```nginx
proxy_set_header Host $host;
proxy_set_header X-Original-URL $scheme://$http_host$request_uri;
```

## Why `Host $host` matters

This preserves the protected application hostname, such as:

```text
cloud.example.com
```

instead of rewriting it to:

```text
auth.example.com
```

## Why `X-Original-URL` matters

This tells Authentik what the user actually requested, so it can issue the correct redirect and return path.

---

# Embedded outpost vs standalone outpost

## Option 1: Embedded outpost

Use the Authentik server’s built-in outpost listener.

Typical ports:

- `9000` for HTTP
- `9443` for HTTPS

Example:

```nginx
location /outpost.goauthentik.io {
    proxy_pass http://AUTHENTIK_IP:9000/outpost.goauthentik.io;
    proxy_set_header Host $host;
    proxy_set_header X-Original-URL $scheme://$http_host$request_uri;
}
```

### Best when
- NPM can directly reach the Authentik server over the LAN or Docker network
- You want the simplest deployment

---

## Option 2: Standalone proxy outpost

Deploy a separate Authentik proxy outpost closer to the NPM instance protecting the app.

Example:

```nginx
location /outpost.goauthentik.io {
    proxy_pass http://OUTPOST_IP:9000;
    proxy_set_header Host $host;
    proxy_set_header X-Original-URL $scheme://$http_host$request_uri;
}
```

### Best when
- NPM cannot directly reach the Authentik server
- Your network is segmented
- You want to avoid double-proxying auth traffic through another reverse proxy

### Often the best homelab design
Keep the Authentik UI on one reverse proxy, but place a standalone outpost near the reverse proxy protecting your apps.

---

## Option 3: Relay through a second reverse proxy

This can work when the app-facing NPM cannot directly reach the Authentik server or an outpost, but it requires extra care.

### When to use it
- The Authentik server is only reachable behind a different proxy
- You cannot expose the embedded outpost on an internal address the app-facing NPM can reach
- You want to keep Authentik isolated behind a dedicated edge proxy

### Main caveat
A relay is more fragile than direct outpost access. The relay must preserve or reconstruct the original protected app context correctly. If it does not, Authentik will often return `404` for `/auth/nginx`.

### Recommended relay design
Use a **dedicated relay hostname** such as:

```text
ak-relay.example.com
```

Do not reuse the public Authentik UI hostname for relay traffic.

### Relay flow

```text
+--------+        +-------------------+        +-------------------+
| Client | -----> | App-facing NPM    | -----> | Protected app     |
+--------+        | app.example.com   |        +-------------------+
                  +-------------------+
                           |
                           | auth_request to relay hostname
                           v
                  +-------------------+
                  | Relay proxy       |
                  | ak-relay.example  |
                  +-------------------+
                           |
                           | forwards to embedded or standalone outpost
                           v
                  +-------------------+
                  | Authentik/outpost |
                  +-------------------+
```

### Required relay settings
The relay path must preserve these values all the way to Authentik:

- original protected app host
- original requested URL
- correct scheme where needed

At minimum, make sure these headers survive the full path:

```text
Host: app.example.com
X-Original-URL: https://app.example.com/...
X-Forwarded-Host: app.example.com
X-Forwarded-Proto: https
```

---

# Relay configuration example

## App-facing NPM config

This goes in the protected app host on the first NPM instance.

```nginx
# Buffer tuning for larger auth headers/cookies
proxy_buffers 8 16k;
proxy_buffer_size 32k;
port_in_redirect off;

location / {
    proxy_pass $forward_scheme://$server:$port;

    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $http_connection;
    proxy_http_version 1.1;

    auth_request /outpost.goauthentik.io/auth/nginx;
    error_page 401 = @goauthentik_proxy_signin;

    auth_request_set $auth_cookie $upstream_http_set_cookie;
    add_header Set-Cookie $auth_cookie;

    auth_request_set $authentik_username $upstream_http_x_authentik_username;
    auth_request_set $authentik_groups $upstream_http_x_authentik_groups;
    auth_request_set $authentik_entitlements $upstream_http_x_authentik_entitlements;
    auth_request_set $authentik_email $upstream_http_x_authentik_email;
    auth_request_set $authentik_name $upstream_http_x_authentik_name;
    auth_request_set $authentik_uid $upstream_http_x_authentik_uid;

    proxy_set_header X-authentik-username $authentik_username;
    proxy_set_header X-authentik-groups $authentik_groups;
    proxy_set_header X-authentik-entitlements $authentik_entitlements;
    proxy_set_header X-authentik-email $authentik_email;
    proxy_set_header X-authentik-name $authentik_name;
    proxy_set_header X-authentik-uid $authentik_uid;
}

location /outpost.goauthentik.io {
    proxy_pass https://ak-relay.example.com/outpost.goauthentik.io;
    proxy_ssl_server_name on;
    proxy_ssl_name ak-relay.example.com;

    # Route to the relay hostname
    proxy_set_header Host ak-relay.example.com;

    # Preserve original app context for the relay
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Original-URL $scheme://$http_host$request_uri;

    add_header Set-Cookie $auth_cookie;
    auth_request_set $auth_cookie $upstream_http_set_cookie;

    proxy_pass_request_body off;
    proxy_set_header Content-Length "";
}

location @goauthentik_proxy_signin {
    internal;
    add_header Set-Cookie $auth_cookie;
    return 302 /outpost.goauthentik.io/start?rd=$scheme://$http_host$request_uri;
}
```

## Relay proxy config

This goes on the second reverse proxy serving `ak-relay.example.com`.

### Relay behavior goals
- accept requests for `/outpost.goauthentik.io`
- forward them to the embedded outpost or standalone outpost
- restore the protected app host before the request reaches Authentik
- preserve the original URL

### Example relay location block

```nginx
location /outpost.goauthentik.io {
    proxy_pass http://AUTHENTIK_IP:9000/outpost.goauthentik.io;

    # Restore the original protected app hostname
    proxy_set_header Host $http_x_forwarded_host;
    proxy_set_header X-Forwarded-Host $http_x_forwarded_host;

    # Preserve original URL and scheme
    proxy_set_header X-Original-URL $http_x_original_url;
    proxy_set_header X-Forwarded-Proto $http_x_forwarded_proto;

    proxy_pass_request_body off;
    proxy_set_header Content-Length "";
}
```

### Relay host requirements
The relay host should:
- forward only the outpost path, not the full Authentik UI unless intended
- avoid rewriting the path
- avoid forcing `Host auth.example.com` toward the outpost
- ideally have its own dedicated DNS name

### Important warning
This relay pattern depends on your reverse proxy stack preserving non-standard headers exactly as expected. If the relay strips or rewrites `X-Forwarded-Host` or `X-Original-URL`, the setup may still fail.

---

# Example topologies

## Example A: Simple single-NPM deployment

```text
Internet
   |
   v
+----------------------+
| NPM                  |
| app.example.com      |
| auth.example.com     |
+----------------------+
   |               |
   |               +----------------------+
   |                                      |
   v                                      v
+------------------+             +------------------+
| Protected app    |             | Authentik server |
+------------------+             +------------------+
```

This is the easiest case.

---

## Example B: App NPM + direct internal outpost access

```text
+--------+        +-------------------+        +------------------+
| Client | -----> | App-facing NPM    | -----> | Protected app    |
+--------+        +-------------------+        +------------------+
                          |
                          v
                  +-------------------+
                  | Authentik outpost |
                  | direct internal   |
                  +-------------------+
                          |
                          v
                  +-------------------+
                  | Authentik server  |
                  +-------------------+
```

This is preferred for split environments.

---

## Example C: Double-proxy layout to avoid when possible

```text
+--------+        +-------------------+        +-------------------+
| Client | -----> | App-facing NPM    | -----> | Protected app     |
+--------+        +-------------------+        +-------------------+
                          |
                          | auth_request to public Authentik hostname
                          v
                  +-------------------+
                  | Auth NPM          |
                  | auth.example.com  |
                  +-------------------+
                          |
                          v
                  +-------------------+
                  | Authentik server  |
                  +-------------------+
```

This can work, but it is easier to mis-handle `Host` and break provider matching.

---

## Example D: Relay topology

```text
+--------+        +-------------------+        +-------------------+
| Client | -----> | App-facing NPM    | -----> | Protected app     |
+--------+        | app.example.com   |        +-------------------+
                  +-------------------+
                           |
                           | auth_request to relay host
                           v
                  +-------------------+
                  | Relay proxy       |
                  | ak-relay.example  |
                  +-------------------+
                           |
                           v
                  +-------------------+
                  | Authentik/outpost |
                  +-------------------+
```

This is the fallback pattern when direct internal outpost access is not possible.

---

# Example broken configs

## Broken pattern: forwarding auth to public Authentik host

```nginx
location /outpost.goauthentik.io {
    proxy_pass https://auth.example.com/outpost.goauthentik.io;
    proxy_set_header Host auth.example.com;
}
```

## Why it breaks

Authentik sees the request as belonging to `auth.example.com`, not the protected app domain.

## Typical result

```text
/outpost.goauthentik.io/ping      -> 204
/outpost.goauthentik.io/auth/nginx -> 404
NPM auth_request                   -> 500
```

This is the classic symptom chain.

---

# Example working configs

## Working example: embedded outpost over internal IP

```nginx
location /outpost.goauthentik.io {
    proxy_pass http://192.168.1.50:9000/outpost.goauthentik.io;
    proxy_set_header Host $host;
    proxy_set_header X-Original-URL $scheme://$http_host$request_uri;
    proxy_pass_request_body off;
    proxy_set_header Content-Length "";
}
```

## Working example: standalone outpost

```nginx
location /outpost.goauthentik.io {
    proxy_pass http://192.168.1.60:9000;
    proxy_set_header Host $host;
    proxy_set_header X-Original-URL $scheme://$http_host$request_uri;
    proxy_pass_request_body off;
    proxy_set_header Content-Length "";
}
```

## Working example: relay pattern

```nginx
# App-facing NPM
location /outpost.goauthentik.io {
    proxy_pass https://ak-relay.example.com/outpost.goauthentik.io;
    proxy_set_header Host ak-relay.example.com;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Original-URL $scheme://$http_host$request_uri;
    proxy_pass_request_body off;
    proxy_set_header Content-Length "";
}
```

```nginx
# Relay proxy
location /outpost.goauthentik.io {
    proxy_pass http://AUTHENTIK_IP:9000/outpost.goauthentik.io;
    proxy_set_header Host $http_x_forwarded_host;
    proxy_set_header X-Forwarded-Host $http_x_forwarded_host;
    proxy_set_header X-Forwarded-Proto $http_x_forwarded_proto;
    proxy_set_header X-Original-URL $http_x_original_url;
    proxy_pass_request_body off;
    proxy_set_header Content-Length "";
}
```

---

# Testing and validation

## Test 1: Outpost reachability

Run:

```bash
curl -Ik https://app.example.com/outpost.goauthentik.io/ping
```

Expected:

```text
HTTP/2 204
```

### Interpretation
- `204` means the outpost path is reachable
- if this fails, the proxy path itself is broken

---

## Test 2: Auth endpoint

Run:

```bash
curl -Ik https://app.example.com/outpost.goauthentik.io/auth/nginx
```

Expected:
- `401` if not authenticated
- `2xx` if already authenticated

Bad result:
- `404` means Authentik could not match the request correctly

---

## Test 3: Direct internal outpost test

If the outpost is reachable internally:

```bash
curl -ik http://AUTHENTIK_IP:9000/outpost.goauthentik.io/auth/nginx \
  -H 'Host: app.example.com' \
  -H 'X-Original-URL: https://app.example.com/'
```

Expected:
- `401` unauthenticated
- `2xx` authenticated

### Interpretation
- if direct internal works but external does not, the problem is in your proxy chain
- if direct internal also returns `404`, check Authentik provider/outpost config

---

## Test 4: Relay endpoint validation

If you are using the relay pattern, test the relay directly:

```bash
curl -ik https://ak-relay.example.com/outpost.goauthentik.io/auth/nginx \
  -H 'Host: ak-relay.example.com' \
  -H 'X-Forwarded-Host: app.example.com' \
  -H 'X-Forwarded-Proto: https' \
  -H 'X-Original-URL: https://app.example.com/'
```

Expected:
- `401` unauthenticated
- `2xx` authenticated

If this returns `404`, the relay is not restoring the protected app context correctly.

---

# Troubleshooting workflow

## Symptom: NPM shows 500
Check logs for:

```text
auth request unexpected status: 404 while sending to client
```

If present, continue.

## Step 1: Check `/ping`

```bash
curl -Ik https://app.example.com/outpost.goauthentik.io/ping
```

- `204` → path is reachable
- anything else → fix routing first

## Step 2: Check `/auth/nginx`

```bash
curl -Ik https://app.example.com/outpost.goauthentik.io/auth/nginx
```

- `401` → good for unauthenticated flow
- `2xx` → good for authenticated flow
- `404` → host/provider matching problem

## Step 3: Verify headers
Confirm your outpost block includes:

```nginx
proxy_set_header Host $host;
proxy_set_header X-Original-URL $scheme://$http_host$request_uri;
```

If using a relay, confirm the relay restores the protected host before forwarding to Authentik.

## Step 4: Verify Authentik provider
Check:
- provider type is forward auth single application
- external host matches the protected app URL
- app/provider is assigned to the correct outpost

## Step 5: Review network path
If the auth subrequest crosses another reverse proxy, confirm the original app hostname is preserved all the way to Authentik.

---

# Deployment guidance

## Use direct internal access whenever possible

Best practice:

- app-facing NPM talks directly to embedded outpost or standalone outpost
- Authentik UI can still be behind a different reverse proxy
- auth subrequests should avoid unnecessary extra proxy hops

## Use a standalone outpost for segmented environments

This is usually the most robust answer when:

- the Authentik server is not directly reachable from the app-facing proxy
- you have multiple proxy layers
- you want predictable header handling

## Use the relay pattern only when needed

The relay pattern is valid, but it is a fallback design.

Prefer it only when:
- direct internal outpost access is not possible
- a standalone outpost cannot be deployed near the app-facing proxy
- you are comfortable validating custom header behavior across multiple reverse proxies

---

# Quick reference

## Good
```nginx
proxy_pass http://INTERNAL_OUTPOST:9000/outpost.goauthentik.io;
proxy_set_header Host $host;
proxy_set_header X-Original-URL $scheme://$http_host$request_uri;
```

## Good relay pattern
```nginx
# App-facing NPM
proxy_pass https://ak-relay.example.com/outpost.goauthentik.io;
proxy_set_header Host ak-relay.example.com;
proxy_set_header X-Forwarded-Host $host;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Original-URL $scheme://$http_host$request_uri;
```

```nginx
# Relay proxy
proxy_pass http://AUTHENTIK_IP:9000/outpost.goauthentik.io;
proxy_set_header Host $http_x_forwarded_host;
proxy_set_header X-Forwarded-Host $http_x_forwarded_host;
proxy_set_header X-Forwarded-Proto $http_x_forwarded_proto;
proxy_set_header X-Original-URL $http_x_original_url;
```

## Bad
```nginx
proxy_pass https://auth.example.com/outpost.goauthentik.io;
proxy_set_header Host auth.example.com;
```

## Healthy test results
```text
/ping       -> 204
/auth/nginx -> 401 or 2xx
```

## Broken test results
```text
/ping       -> 204
/auth/nginx -> 404
client app  -> 500
```

---

# Final recommendations

1. Use **Forward auth (single application)** in Authentik for each protected app.
2. Make the provider external host match the protected app URL.
3. Send `/outpost.goauthentik.io` to an **internally reachable outpost** whenever possible.
4. Preserve:
   - `Host $host`
   - `X-Original-URL $scheme://$http_host$request_uri`
5. Prefer a **standalone outpost near the app-facing NPM** in segmented or multi-proxy homelabs.
6. Use the **relay pattern** only when direct reachability is not possible, and validate the full header chain carefully.

---

# Minimal starter block

For a quick starting point, this is the minimum block most people need:

```nginx
location / {
    proxy_pass $forward_scheme://$server:$port;
    auth_request /outpost.goauthentik.io/auth/nginx;
    error_page 401 = @goauthentik_proxy_signin;
}

location /outpost.goauthentik.io {
    proxy_pass http://AUTHENTIK_IP:9000/outpost.goauthentik.io;
    proxy_set_header Host $host;
    proxy_set_header X-Original-URL $scheme://$http_host$request_uri;
    proxy_pass_request_body off;
    proxy_set_header Content-Length "";
}

location @goauthentik_proxy_signin {
    internal;
    return 302 /outpost.goauthentik.io/start?rd=$scheme://$http_host$request_uri;
}
```
