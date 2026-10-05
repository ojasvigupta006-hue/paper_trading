# Paper Trading Platform

A full-stack paper trading app: users register, get virtual cash, place market and limit orders, and watch prices, orders and P&L update live. No real money is involved.

Stack: FastAPI, PostgreSQL, SQLAlchemy, Alembic, JWT + bcrypt, WebSockets, React (Vite, canvas charts), pytest, Docker Compose, GitHub Actions. Backend deploys to Render, frontend to Vercel.

## Run it locally

With Docker (needs Docker Desktop):

    docker compose up --build

Then open http://localhost:3000 (API docs at http://localhost:8000/docs).

Without Docker you need Python 3.12, Node 20 and a local PostgreSQL:

    # backend
    cd backend
    python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
    pip install -r requirements.txt
    cp .env.example .env                                     # edit DATABASE_URL if needed
    alembic upgrade head
    uvicorn app.main:app --reload

    # frontend (second terminal)
    cd frontend
    npm install
    npm run dev                                              # http://localhost:5173

## Run the tests

Tests run against a real PostgreSQL database (the concurrency tests need real row locks). Create an empty database whose name contains "test", point DATABASE_URL at it, and run pytest. The test setup truncates tables, so it refuses to run against a database without "test" in its URL.

    createdb papertrade_test
    cd backend
    DATABASE_URL=postgresql+psycopg://postgres:postgres@localhost:5432/papertrade_test pytest -q

## How it works

Orders. POST /orders needs an Idempotency-Key header. The order is authenticated, validated by Pydantic, then processed in one database transaction:

1. Take a per-symbol advisory lock, then lock the account rows involved (own account plus any counterparties) in ascending id order with SELECT ... FOR UPDATE.
2. Check the idempotency key again now that the lock is held. A repeated key returns the original order instead of trading again.
3. Risk checks (app/services/risk.py): quantity and order value limits, valid price, buying power (cash minus cash reserved by resting buy orders), free shares (holdings minus shares reserved by resting sell orders). A failed check is stored as a REJECTED order with a reason.
4. Matching (app/services/matching.py): the incoming order first matches resting orders from other users with price-time priority, trading at the resting order's price. Whatever is left is filled by a simulated market maker at its bid or ask if the order is a market order or a limit order that crosses it. A limit order that cannot fill rests in the book.
5. Every fill updates cash, position (average price, realized P&L), both orders and the trade log together. Commit, and only then push events over WebSocket.

Background worker (app/services/ticker.py). An asyncio task moves prices every few seconds, fills resting limit orders the new quote satisfies, triggers price alerts, and broadcasts prices to all connected clients.

Market data. Prices are a small random walk around realistic reference levels (app/services/market_data.py). It is the only module that knows where prices come from, so a real data API can replace it later.

Auth. Passwords are hashed with bcrypt. Login returns a 15 minute access token and a 7 day refresh token (JWT). The frontend refreshes the access token automatically. WebSocket connections authenticate with the access token.

## Using the app

The dashboard is one screen: a watchlist with live sparklines on the left, the chart in the middle, and the order ticket with your account summary on the right. Holdings, open orders, order history, trades and alerts sit in tabs under the chart.

- The chart is drawn on canvas from the live price stream. Switch between candles and a line, and between 10 second, 30 second and 1 minute candles. Your resting limit orders, active alerts, average cost and past fills are drawn on it.
- Click anywhere on the chart to place a buy or sell limit at that price, or to set an alert there.
- The ticket sizes orders for you (25%, 50%, max of available cash, or all shares held), shows the cash left after the trade, and says whether a limit will fill straight away or wait.
- Holdings can be closed in one step (sell all at market, with a confirm click). Open orders can be cancelled one at a time or all at once.
- Keyboard: `/` search, up and down arrows switch stock, `B` buy, `S` sell.

The login page shows the live market without an account by polling the public `/market/quotes` endpoint.

## Deployment

Backend on Render:

1. Push this repo to GitHub.
2. In Render choose New, Blueprint, and select the repo. render.yaml creates the API (built from backend/Dockerfile, which runs migrations on start) and a PostgreSQL database.
3. Set CORS_ORIGINS to your Vercel URL once you have it. Copy the service's Deploy Hook URL.

Frontend on Vercel:

1. Import the repo, set the root directory to frontend.
2. Add the environment variable VITE_API_URL with your Render URL (for example https://paper-trading-api.onrender.com), then deploy.

CI/CD: .github/workflows/ci.yml runs the backend tests against a PostgreSQL service, checks the migrations, builds the frontend, builds the Docker images, and on pushes to main triggers a Render deploy. Add the Deploy Hook URL as a repository secret named RENDER_DEPLOY_HOOK. Vercel redeploys the frontend by itself when main changes.

Free hosting tiers may put the API to sleep when idle, which pauses the price ticker and drops WebSocket connections until the next request. Wake it before a demo by opening /health.

## Limitations

This is a simulation. Prices are synthetic, the market maker has unlimited liquidity, and there is no short selling, no stop orders, and no fees. Sessions are per browser (tokens in localStorage). WebSocket fan-out lives in a single API process, so running several API instances would need a shared message broker such as Redis.
