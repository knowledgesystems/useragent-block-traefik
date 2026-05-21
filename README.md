# Block User-Agent

[![Build Status](https://github.com/knowledgesystems/useragent-block-traefik/workflows/Main/badge.svg?branch=master)](https://github.com/knowledgesystems/useragent-block-traefik/actions)

Block User-Agent is a middleware plugin for [Traefik](https://github.com/traefik/traefik) which sends an HTTP error
response when the requested HTTP User-Agent header matches one of the configured [regular expressions](https://github.com/google/re2/wiki/Syntax).

## Configuration Options

| Field             | Type     | Default     | Description                                                                 |
|-------------------|----------|-------------|-----------------------------------------------------------------------------|
| `regex`           | []string | `[]`        | Deny list — requests with a matching User-Agent are blocked                 |
| `regexAllow`      | []string | `[]`        | Allow list — matching User-Agents bypass the deny list (checked first)      |
| `statusCode`      | int      | `403`       | HTTP status code returned when a request is blocked                         |
| `responseMessage` | string   | *(empty)*   | Response body returned when a request is blocked. Empty body if not set.    |

## Static Configuration

```toml
[experimental.plugins.blockuseragent]
    modulename = "github.com/knowledgesystems/useragent-block-traefik"
    version = "vX.Y.Z"
```

## Dynamic Configuration

To configure the `Block User-Agent` plugin you should create a [middleware](https://docs.traefik.io/middlewares/overview/) in
your dynamic configuration. The following example blocks all requests with a User-Agent matching `\bTheAgent\b`, with an
exception for User-Agents that also contain `Allowed`.

```toml
[http.routers]
  [http.routers.my-router]
    rule = "Host(`localhost`)"
    middlewares = ["block-foo"]
    service = "my-service"

# Block all user agents containing TheAgent, except those also containing Allowed
[http.middlewares]
  [http.middlewares.block-foo.plugin.blockuseragent]
    regexAllow = ["\bAllowed\b"]
    regex = ["\bTheAgent\b"]
    statusCode = 403
    responseMessage = "Access denied"

[http.services]
  [http.services.my-service]
    [http.services.my-service.loadBalancer]
      [[http.services.my-service.loadBalancer.servers]]
        url = "http://127.0.0.1"
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
