# ⚡ Elpriser – ENTSO-E Prices Frontend

A Vue 3 + Vite single-page application that displays Swedish electricity prices (SE3/SE4 bidding zones) sourced from the ENTSO-E Transparency Platform via a companion backend API.

## Features

- **Current price** – live indicator for the ongoing hour, colour-coded against the daily average (🟢 low / 🟡 normal / 🔴 high)
- **Day navigation** – browse prices for the day before yesterday, yesterday, today, and tomorrow (when available)
- **Summary cards** – average, minimum, and maximum price for the selected day
- **Cheapest / most expensive hours** – quick overview of the best and worst hours to consume electricity
- **Bar chart** – hourly price visualisation with the current hour highlighted
- **Multi-day comparison table** – side-by-side averages and extremes across all available days
- **Exchange rate** – live EUR → SEK rate shown in the header

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Framework | [Vue 3](https://vuejs.org/) (`<script setup>` SFCs) |
| Build tool | [Vite](https://vitejs.dev/) |
| Web server | [nginx](https://nginx.org/) (production container) |
| Container | Docker |

## Prerequisites

- [Node.js](https://nodejs.org/) 18 or later
- npm (bundled with Node.js)
- Docker (optional, for containerised deployment)

## Getting Started

### Install dependencies

```bash
npm install
```

### Configure environment

Create a `.env.local` file in the project root (or set the variable in your shell) to point at the backend API:

```env
VITE_API_URL=http://localhost:8080
```

Leave `VITE_API_URL` empty to use relative URLs (useful when the frontend and API are served from the same origin).

### Run in development mode

```bash
npm run dev
```

The app will be available at `http://localhost:5173` by default.

### Build for production

```bash
npm run build
```

The compiled output is written to the `dist/` directory.

### Preview the production build locally

```bash
npm run preview
```

## Docker

Build and run the production image:

```bash
# Build
docker build -t entsoe-prices-frontend .

# Run (exposes port 80)
docker run -p 8080:80 entsoe-prices-frontend
```

The container serves the compiled frontend via nginx on port 80.

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `VITE_API_URL` | *(empty)* | Base URL of the backend API. Empty string means same-origin relative requests. |

## Project Structure

```
.
├── public/             # Static assets copied verbatim to dist/
├── src/
│   ├── assets/         # Images, fonts, and other imported assets
│   ├── components/     # Reusable Vue components
│   ├── App.vue         # Root application component
│   ├── main.js         # Application entry point
│   └── style.css       # Global styles
├── Dockerfile          # Multi-stage Docker build
├── nginx.conf          # nginx configuration for the production container
├── index.html          # HTML entry point
└── vite.config.js      # Vite configuration
```

## API Endpoints Used

The frontend consumes the following endpoints from the backend:

| Endpoint | Description |
|----------|-------------|
| `GET /api/analysis/summary/today` | Price summary for today |
| `GET /api/analysis/today` | Hourly prices for today |
| `GET /api/analysis/summary/{date}` | Price summary for a specific date (`YYYY-MM-DD`) |
| `GET /api/analysis/date/{date}` | Hourly prices for a specific date (`YYYY-MM-DD`) |
| `GET /api/analysis/exchange-rate` | Current EUR → SEK exchange rate |

