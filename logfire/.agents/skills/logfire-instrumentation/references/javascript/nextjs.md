# Next.js Instrumentation

Use this for Next.js apps. Instrument server-side Next telemetry separately from optional browser tracing.

## Server-Side Tracing

Install in the Next.js app package:

```bash
npm install @vercel/otel logfire
```

Create `instrumentation.ts` in the project root, or `src/instrumentation.ts` if the app uses `src`:

```ts
import { registerOTel } from '@vercel/otel'

export function register() {
  registerOTel({
    serviceName: process.env.LOGFIRE_SERVICE_NAME ?? 'nextjs-app',
  })
}
```

Set server-only env vars in `.env.local`, deployment secrets, or the hosting dashboard:

```bash
OTEL_EXPORTER_OTLP_ENDPOINT=https://logfire-api.pydantic.dev
OTEL_EXPORTER_OTLP_HEADERS='Authorization=your-write-token'
LOGFIRE_SERVICE_NAME=nextjs-app
```

Do not prefix write-token variables with `NEXT_PUBLIC_`.

## Manual Server Spans

Use the runtime-agnostic `logfire` package in server components, route handlers, server actions, and other server-only code:

```tsx
import * as logfire from 'logfire'

export default async function Page() {
  return logfire.span('render home page', {
    callback: async () => {
      logfire.info('loading homepage data')
      return <main>Hello</main>
    },
  })
}
```

Route handler error reporting:

```ts
import * as logfire from 'logfire'

export async function POST(request: Request) {
  try {
    return Response.json(await createOrder(await request.json()))
  } catch (error) {
    logfire.reportError('create order route failed', error)
    throw error
  }
}
```

## Client-Side Browser Tracing

Add browser tracing only when the app has or can safely add a same-origin proxy. Install:

```bash
npm install @pydantic/logfire-browser @opentelemetry/auto-instrumentations-web
```

Create a proxy file in the project root or `src` directory. For Next.js 16 and later use `proxy.ts`. For older apps that already use `middleware.ts`, follow the existing file convention unless the app has migrated to `proxy.ts`.

The proxy below fails closed until you connect its two adapter functions to the app's existing server-side authentication and rate limiter. Do not replace either `return false` with `return true`. If the app has no authentication or rate limiter to reuse, stop at server-side tracing and explain why browser tracing was not added.

```ts
// proxy.ts
import { NextRequest, NextResponse } from 'next/server'

async function isAuthenticated(request: NextRequest): Promise<boolean> {
  void request
  return false // Replace with the app's existing server-side authentication.
}

async function isWithinTelemetryLimit(request: NextRequest): Promise<boolean> {
  void request
  return false // Replace with the app's existing server-side rate limiter.
}

export default async function proxy(request: NextRequest) {
  const url = request.nextUrl.clone()

  if (url.pathname === '/logfire-proxy/v1/traces') {
    const allowedOrigin = process.env.LOGFIRE_PROXY_ALLOWED_ORIGIN
    if (!allowedOrigin) {
      return new NextResponse('Logfire proxy origin is not configured', { status: 500 })
    }
    if (request.method !== 'POST' || request.headers.get('origin') !== allowedOrigin) {
      return new NextResponse('Forbidden', { status: 403 })
    }
    if (!(await isAuthenticated(request))) {
      return new NextResponse('Unauthorized', { status: 401 })
    }
    if (!(await isWithinTelemetryLimit(request))) {
      return new NextResponse('Too many requests', { status: 429 })
    }

    const token = process.env.LOGFIRE_TOKEN
    if (!token) {
      return new NextResponse('Logfire token is not configured', { status: 500 })
    }

    const requestHeaders = new Headers({ Authorization: token })
    for (const name of ['content-type', 'content-encoding']) {
      const value = request.headers.get(name)
      if (value) requestHeaders.set(name, value)
    }

    return NextResponse.rewrite(new URL('https://logfire-api.pydantic.dev/v1/traces'), {
      request: {
        headers: requestHeaders,
      },
    })
  }

  return NextResponse.next()
}

export const config = {
  matcher: '/logfire-proxy/v1/traces',
}
```

Replace the two fail-closed adapter bodies with calls to the app's real authentication and rate-limit APIs before enabling the client component. Set `LOGFIRE_PROXY_ALLOWED_ORIGIN` to the app's public origin, such as `https://app.example.com`, with no trailing slash. An explicit origin keeps this check correct behind a reverse proxy or content delivery network (CDN). Keep the exact path, method, origin, and forwarded-header allowlist. Forwarding all request headers could send application cookies or session credentials to Logfire. Set `LOGFIRE_TOKEN` server-side to a Logfire write token. It can be the same write token value used in `OTEL_EXPORTER_OTLP_HEADERS`, but it must not use a `NEXT_PUBLIC_` prefix.

Create a client-only component:

```tsx
'use client'

import { getWebAutoInstrumentations } from '@opentelemetry/auto-instrumentations-web'
import * as logfire from '@pydantic/logfire-browser'
import { useEffect, useRef } from 'react'

export function ClientInstrumentation() {
  const configured = useRef(false)

  useEffect(() => {
    if (!configured.current) {
      logfire.configure({
        traceUrl: '/logfire-proxy/v1/traces',
        serviceName: 'nextjs-browser',
        instrumentations: [getWebAutoInstrumentations()],
      })
      configured.current = true
    }
  }, [])

  return null
}
```

Mount this component once at the app root and do not return the asynchronous SDK cleanup from its effect. The ref prevents React Strict Mode's development-only second effect setup from configuring Logfire twice. Tests, previews, or app shells that replace the whole telemetry setup should await the cleanup returned by `configure()` before configuring a replacement.

Import this Client Component normally from an App Router Server Component. If the app needs `next/dynamic` with `ssr: false`, put that dynamic import in another Client Component; Next.js rejects `ssr: false` directly in a Server Component.

## Vercel Deployment Notes

- Add OTLP and Logfire token values to the Vercel project environment.
- If spans do not appear after changing tracing env vars, clear the Vercel data cache for the project and redeploy.
- Keep server and browser service names distinct when both are enabled.
