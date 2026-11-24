# Infra and monitoring

Only add these when the user asks or the app has real traffic. Most hosts (Vercel, Netlify, Cloudflare, Render, Fly) already cover the first two.

## Browser caching for static files
- Hashed filenames plus `Cache-Control: public, max-age=31536000, immutable`.
- HTML gets `no-cache` so new deploys show up immediately.

## Load balancing
- Only when one instance is not enough.
- The app must be stateless first: sessions in the database or Redis, uploads in object storage, no local disk state.

## Tracing slow requests
- Use OpenTelemetry to trace requests across the server, database and external APIs, and send traces to whatever backend the team uses.
- Find the slowest endpoints first; fix those before anything else.

## Real-user monitoring
- Track real users' Core Web Vitals (the `web-vitals` library, the host's analytics, or an APM such as Sentry).
- Lab scores on a fast laptop are not what users get. Check the 75th percentile on mobile.

## Rule
Measure, fix the biggest number, measure again.
