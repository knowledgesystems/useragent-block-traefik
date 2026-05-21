# Block User-Agent & Path

[![Build Status](https://github.com/knowledgesystems/useragent-block-traefik/workflows/Main/badge.svg?branch=master)](https://github.com/knowledgesystems/useragent-block-traefik/actions)

A middleware plugin for [Traefik](https://github.com/traefik/traefik) that blocks HTTP requests based on User-Agent headers and/or URL paths matching configured [regular expressions](https://github.com/google/re2/wiki/Syntax).

## Configuration Options

| Field             | Type     | Default     | Description                                                                 |
|-------------------|----------|-------------|-----------------------------------------------------------------------------|
| `regex`           | []string | `[]`        | Deny list — requests with a matching User-Agent are blocked                 |
| `regexAllow`      | []string | `[]`        | Allow list — matching User-Agents bypass the deny list (checked first)      |
| `pathRegex`       | []string | `[]`        | Path deny list — requests with a matching URL path are blocked (takes priority over User-Agent rules) |
| `statusCode`      | int      | `403`       | HTTP status code returned when a request is blocked                         |
| `responseMessage` | string   | *(empty)*   | Response body returned when a request is blocked. Empty body if not set.    |

### Evaluation Order

1. **Path blocking** (`pathRegex`) is evaluated first. If the request path matches, the request is blocked immediately regardless of User-Agent.
2. **User-Agent allow list** (`regexAllow`) is checked next. If the User-Agent matches, the request is allowed through.
3. **User-Agent deny list** (`regex`) is checked last. If the User-Agent matches, the request is blocked.

## Static Configuration

```toml
[experimental.plugins.blockuseragent]
    modulename = "github.com/knowledgesystems/useragent-block-traefik"
    version = "vX.Y.Z"
```

## Dynamic Configuration

To configure the plugin you should create a [middleware](https://docs.traefik.io/middlewares/overview/) in
your dynamic configuration.

### Block by User-Agent

The following example blocks all requests with a User-Agent matching `\bTheAgent\b`, with an
exception for User-Agents that also contain `Allowed`.

```toml
[http.middlewares]
  [http.middlewares.block-foo.plugin.blockuseragent]
    regexAllow = ["\bAllowed\b"]
    regex = ["\bTheAgent\b"]
    statusCode = 403
    responseMessage = "Access denied"
```

### Block by Path

The following example blocks requests to specific API endpoints with a 404 response:

```toml
[http.middlewares]
  [http.middlewares.block-paths.plugin.blockuseragent]
    pathRegex = ["^/api/molecular-profiles/co-expressions/fetch$"]
    statusCode = 404
    responseMessage = "Not Found"
```

### Combined (User-Agent + Path)

You can combine both User-Agent and path blocking in a single middleware:

```toml
[http.middlewares]
  [http.middlewares.block-combined.plugin.blockuseragent]
    regex = ["\bscraper\\b"]
    pathRegex = ["^/api/expensive-endpoint$"]
    statusCode = 403
    responseMessage = "Forbidden"
```

## Kubernetes

### 1. Enable the plugin in Traefik's `values.yaml`

When installing Traefik via Helm, enable the plugin under `additionalArguments`:

```yaml
additionalArguments:
  - "--experimental.plugins.blockuseragent.moduleName=github.com/knowledgesystems/useragent-block-traefik"
  - "--experimental.plugins.blockuseragent.version=vX.Y.Z"
```

### 2. Create a Traefik `Middleware` resource

#### Block by User-Agent

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: block-useragent
  namespace: my-namespace
spec:
  plugin:
    blockuseragent:
      regexAllow:
        - "\\bAllowed\\b"
      regex:
        - "\\bTheAgent\\b"
        - "^$"
      statusCode: 403
      responseMessage: "Access denied"
```

#### Block by Path

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: block-paths
  namespace: my-namespace
spec:
  plugin:
    blockuseragent:
      pathRegex:
        - "^/api/molecular-profiles/co-expressions/fetch$"
        - "^/api/other-expensive-endpoint"
      statusCode: 404
      responseMessage: "Not Found"
```

### 3. Annotate the Ingress

Add the middleware annotation to any Ingress where you want to apply the block:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  namespace: my-namespace
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: my-namespace-block-useragent@kubernetescrd
spec:
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-service
                port:
                  number: 80
```

> **Note:** The middleware annotation value follows the format `<namespace>-<middleware-name>@kubernetescrd`.
