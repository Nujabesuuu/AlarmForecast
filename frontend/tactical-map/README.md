# Forecast map

Next.js 16 / React 19 frontend for the air raid alert forecast: a choropleth of Ukraine's
25 regions with a 24-hour timeline, risk statistics and a per-region hourly panel.

```bash
npm install
cp .env.example .env.local   # NEXT_PUBLIC_API_URL → Flask /forecast endpoint
npm run dev                  # http://localhost:3000
```

Data source order: the live API from `NEXT_PUBLIC_API_URL`, then the bundled model
snapshot in `public/sample-forecast.json`, then generated demo data. The header badge
shows which one is on screen.

Deploys to Vercel with this folder as the root directory — see
[docs/DEPLOYMENT.md](../../docs/DEPLOYMENT.md).
