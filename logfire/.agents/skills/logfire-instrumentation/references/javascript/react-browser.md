# React And Browser Instrumentation

Use this for browser-only telemetry in React, Vite, or other SPA projects. If the app is Next.js, read [nextjs.md](./nextjs.md) instead.

## Security Requirement

Browser telemetry must go through an authenticated backend proxy. Do not put a Logfire write token or `Authorization` header in browser code. If a static frontend has no backend or proxy, do not add direct browser export; explain that a server-side proxy is required.

## Install

```bash
npm install @pydantic/logfire-browser @opentelemetry/auto-instrumentations-web
```

## Configure In Browser-Only Code

For React, add a provider mounted once near the app root:

```tsx
import { getWebAutoInstrumentations } from '@opentelemetry/auto-instrumentations-web'
import * as logfire from '@pydantic/logfire-browser'
import { useEffect, useRef, type ReactNode } from 'react'

export function LogfireProvider({ children }: { children: ReactNode }) {
  const configured = useRef(false)

  useEffect(() => {
    if (!configured.current) {
      logfire.configure({
        traceUrl: '/logfire-proxy/v1/traces',
        serviceName: 'web-app',
        instrumentations: [getWebAutoInstrumentations()],
      })
      configured.current = true
    }
  }, [])

  return children
}
```

Mount this provider once at the app root and do not return the asynchronous SDK cleanup from its effect. The ref prevents React Strict Mode's development-only second effect setup from configuring Logfire twice. Tests, previews, or app shells that replace the whole telemetry setup should await the cleanup returned by `configure()` before configuring a replacement.

For non-React browser entrypoints, run `configure()` from the client entry file before adding manual spans.

## Backend Proxy Shape

The proxy must:

- accept requests from the frontend origin only
- forward `/v1/traces` OTLP requests to Logfire
- add `Authorization: <write-token>` server-side
- apply normal app authentication, CORS, origin checks, and rate limiting

Use a path like `/logfire-proxy/v1/traces` so the browser sends same-origin requests.

## Manual Client Events

```ts
button.addEventListener('click', () => {
  logfire.info('checkout button clicked')
})
```

Report client errors:

```ts
window.addEventListener('error', (event) => {
  logfire.reportError('uncaught browser error', event.error, { filename: event.filename }, { tags: ['browser'] })
})

window.addEventListener('unhandledrejection', (event) => {
  logfire.reportError('unhandled browser rejection', event.reason, {}, { tags: ['browser'] })
})
```

## Browser-Specific Cautions

- Configure only in browser runtime code. Avoid importing `@pydantic/logfire-browser` from SSR modules.
- Use `diagLogLevel: logfire.DiagLogLevel.ALL` only during local troubleshooting.
- Browser `configure()` returns an async cleanup function. Use it in tests and providers.
- Browser does not install automatic pending-span processing; call `startPendingSpan()` explicitly for long operations.
- Avoid high-volume spans for every mouse movement, render, or keystroke.
