---

I Built an Over-Engineered Analytics Dashboard for My Indie iOS App - Here's How
PostHog is great. But staring at your own Grafana dashboard at 11pm feels different.I launched rCoon a few days ago - a photo-cleaning app for iOS that helps you delete blurry shots, duplicates, and short throwaway videos. It uses on-device ML so nothing ever leaves your phone. Classic indie side project.
PostHog is my analytics backend of choice. It's powerful, the HogQL query language is surprisingly fun, and the free tier is generous. But at some point I wanted a dashboard I actually enjoyed opening. Something that felt like a mission control, not a SaaS admin panel.
So I set up Grafana locally via Docker, wired it to PostHog and Sentry, and now I have a dashboard that shows me exactly what I care about - paying subscribers front and center, and everything else below.
Here's exactly how I did it.

---

The Stack
Grafana - running locally via Docker
Infinity datasource plugin - queries any HTTP/JSON API, perfect for PostHog
PostHog Query API - HogQL queries over REST
Sentry - for crash/health metrics
docker-compose - one command to spin it all up

No cloud hosting needed. This runs on my MacBook, I open it when I want it, done.

---

Project Structure
After some iteration, this is the file structure I landed on:
grafana/
├── dashboards/
│   └── rcoon-overview.json
├── provisioning/
│   ├── dashboards/
│   │   └── dashboard.yml
│   └── datasources/
│       ├── posthog.yml
│       └── sentry.yml
├── .env.example
└── docker-compose.yml
The key insight: Grafana supports provisioning - you can define datasources and dashboards as YAML/JSON files, so everything is version-controlled and reproducible. No clicking through the UI to configure things. Just docker compose up and it's all there.
[SCREENSHOT - file structure in VS Code]

---

Step 1 - docker-compose.yml
yaml
services:
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    env_file:
      - .env
    environment:
      - GF_INSTALL_PLUGINS=yesoreyeram-infinity-datasource
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
    volumes:
      - grafana-data:/var/lib/grafana
      - ./provisioning:/etc/grafana/provisioning
      - ./dashboards:/var/lib/grafana/dashboards
volumes:
  grafana-data:
The GF_INSTALL_PLUGINS line auto-installs the Infinity plugin on first boot. No manual plugin installation needed.
Create a .env file (never commit this):
env
GRAFANA_PASSWORD=yourpassword
POSTHOG_API_KEY=phx_your_personal_api_key

---

Step 2 - Configure the PostHog Datasource
provisioning/datasources/posthog.yml:
yaml
apiVersion: 1
datasources:
  - name: PostHog
    type: yesoreyeram-infinity-datasource
    access: proxy
    jsonData:
      base_url: https://us.posthog.com
    secureJsonData:
      httpHeaderValue1: "Bearer ${POSTHOG_API_KEY}"
    jsonData:
      httpHeaders:
        - name: Authorization
          value: "Bearer ${POSTHOG_API_KEY}"
Grafana reads this on startup and the datasource is ready without touching the UI.

---

Step 3 - Provision the Dashboard
provisioning/dashboards/dashboard.yml:
yaml
apiVersion: 1
providers:
  - name: rcoon
    folder: rCoon
    type: file
    options:
      path: /var/lib/grafana/dashboards
This tells Grafana to watch the /dashboards folder and auto-load any .json files as dashboards. You edit the JSON, save, and Grafana hot-reloads it.

---

Step 4 - Querying PostHog with HogQL
Every panel hits PostHog's query endpoint:
POST https://us.posthog.com/api/projects/{project_id}/query
Content-Type: application/json
Authorization: Bearer <your_key>
With a body like:
json
{
  "query": {
    "kind": "HogQLQuery",
    "query": "SELECT count(DISTINCT person_id) FROM events WHERE event = 'Application Opened' AND timestamp > now() - INTERVAL 7 DAY"
  }
}
In Grafana's Infinity panel, set:
Type: JSON
Method: POST
URL: https://us.posthog.com/api/projects/YOUR_PROJECT_ID/query
Body: your HogQL query JSON

The response comes back as { "results": [[value]] } - you point Infinity at results[0][0] and you've got your stat panel.

---

Step 5 - The Dashboard Layout
Here's how I structured the panels, top to bottom:
Row 1 - North Star
One giant stat panel: Paying Subscribers (total active). This is the number I actually care about. Everything else is context.
[SCREENSHOT - north star panel]
Row 2 - Revenue
MRR (current month)
Trial starts this week
Trial → paid conversion %
Monthly vs yearly split (pie chart)

Row 3 - Engagement
DAU / WAU / MAU time series
DAU/MAU ratio (stickiness %)
Feature usage breakdown - Blur vs Duplicate vs Short Videos

[SCREENSHOT - engagement row]
Row 4 - Funnel
Waterfall panel: Install → Onboarding complete → First scan → Paywall seen → Trial started → Paid
This is the most useful view. You can immediately see where people drop off.
[SCREENSHOT - funnel panel]
Row 5 - Health
Crash-free session rate (from Sentry)
App version distribution
Events with anomalies

---

Key HogQL Queries
DAU (last 30 days):
sql
SELECT 
  toDate(timestamp) as day,
  count(DISTINCT person_id) as dau
FROM events
WHERE event = 'Application Opened'
  AND timestamp > now() - INTERVAL 30 DAY
GROUP BY day
ORDER BY day ASC
Trial starts this week:
sql
SELECT count(DISTINCT person_id)
FROM events
WHERE event = 'trial_started'
  AND timestamp > now() - INTERVAL 7 DAY
Feature usage breakdown:
sql
SELECT 
  properties.$screen_name as feature,
  count() as sessions
FROM events
WHERE event = 'screen_viewed'
  AND timestamp > now() - INTERVAL 30 DAY
GROUP BY feature
ORDER BY sessions DESC

---

What I Learned
PostHog's native dashboards are genuinely good. I'm not replacing them - I still use them for ad-hoc exploration. This Grafana setup is for the "mission control" view I want at a glance.
Provisioning as code is worth the setup. Being able to git clone and docker compose up and have the full dashboard running is satisfying. It also makes writing this article much easier.
The North Star metric changes how you feel about the data. When paying subscribers is the first thing you see, everything else becomes "what's driving or blocking that number." It focuses the analysis.

---

What rCoon Actually Does
Since you read this far - rCoon is an iOS app that cleans up your photo library. It finds blurry shots using on-device ML (Apple's Vision framework), finds near-duplicates, and surfaces short throwaway videos. Everything runs on-device, nothing is uploaded. It's $2.99/month or $9.99/year with a 3-day free trial.
If your camera roll is a disaster like mine was, give it a try.

---

Get the Code
I'll be open-sourcing the Grafana config (minus the API keys, obviously) on my Github. Follow me here or check rcoon.app for updates.
If you found this useful or have questions about the setup, drop a comment - happy to go deeper on any part of it.

---

Built with Grafana, PostHog, Docker, and too much coffee.
