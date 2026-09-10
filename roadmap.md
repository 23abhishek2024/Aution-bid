Here is a comprehensive, step-by-step roadmap to build **Prime Bid** from scratch to deployment. Breaking this complex system down into manageable phases is the best way to tackle it.

### Phase 1: Environment & Initial Setup (The Foundation)
1. **Initialize Git Repository:** Create a new folder and run `git init`.
2. **Backend Setup:**
   * Create a `backend/` folder.
   * Run `npm init -y` to create `package.json`.
   * Install core dependencies: `npm install express pg socket.io jsonwebtoken bcrypt dotenv cors express-rate-limit`.
   * Setup your MVC folder structure: `src/routes`, `src/controllers`, `src/models`, `src/middlewares`, `src/services`, `src/workers`.
3. **Frontend Setup:**
   * Create a `frontend/` folder.
   * Initialize React using Vite (recommended for speed): `npm create vite@latest . -- --template react`.
   * Install dependencies: `npm install axios react-router-dom socket.io-client tailwindcss` (if using Tailwind).

### Phase 2: Database Setup & Connectivity
1. **Local Database Setup:** Install PostgreSQL locally or run it via Docker (`docker run --name pg -p 5432:5432 -e POSTGRES_PASSWORD=secret -d postgres`).
2. **Define Schema:** Create a `schema.sql` file containing your `CREATE TABLE` statements for `users`, `auctions`, `bids`, and `jobs` (as defined in your HLD).
3. **Connect to Node.js:** Create a `db.js` file in your backend to establish a connection pool using the `pg` package.

### Phase 3: Core API & Middlewares (Backend Scaffold)
1. **Express App Setup:** Create `app.js` to initialize Express, setup CORS, and parse JSON.
2. **Build Middlewares:**
   * **Logger:** Write a simple middleware to `console.log` incoming requests and attach a unique `req.id`.
   * **Rate Limiter:** Implement `express-rate-limit` to prevent spam.
   * **Auth Middleware:** Write a middleware to extract the JWT from the `Authorization` header, verify it, and attach `req.user`.
3. **Authentication API:** Build the `/api/auth/register` (hash password with bcrypt) and `/api/auth/login` (generate JWT) routes.

### Phase 4: The Core Engine (Business Logic & Concurrency)
1. **Auction CRUD:** Create controllers and routes to Create, Read, Update, and Delete auctions.
2. **Bidding Logic (Critical Step):**
   * Create the POST `/api/bids` endpoint.
   * Implement **PostgreSQL Transactions** (`BEGIN`, `COMMIT`, `ROLLBACK`).
   * Implement **Pessimistic Locking**: Use `SELECT * FROM auctions WHERE id = $1 FOR UPDATE` to lock the auction row so concurrent bids don't overwrite each other.
   * Validate the bid (is it higher than `current_price`? Is the auction active?).
   * Insert the new bid and update the auction price.

### Phase 5: Real-Time WebSockets
1. **Socket.IO Backend:** Attach Socket.IO to your Express HTTP server. Setup basic connection handling.
2. **Emit Events:** Inside your Bid Controller, right after committing the database transaction, emit a `PRICE_UPDATE` event with the new price and auction ID to all connected clients.

### Phase 6: Distributed Background Workers (Job Queue)
1. **The Scheduler:** Write a standalone script (or `setInterval` block) that periodically checks the database for auctions where `end_time` has passed and `status = 'ACTIVE'`. For each one, insert a `CLOSE_AUCTION` job into the `jobs` table.
2. **The Worker:** Write the script that polls the `jobs` table.
   * Crucial DB query: `SELECT id FROM jobs WHERE status = 'PENDING' FOR UPDATE SKIP LOCKED LIMIT 1`.
   * Process the job (e.g., mark auction as `CLOSED`, determine the highest bidder).
   * Update the job status to `COMPLETED`.

### Phase 7: Frontend UI Integration (React)
1. **Routing:** Set up `react-router-dom` with routes for `/login`, `/dashboard`, and `/auction/:id`.
2. **Auth State:** Create a React Context or use standard state to manage the user's JWT token and login status.
3. **API Integration:** Use `axios` to connect your frontend forms (login, create auction, place bid) to your Express backend.
4. **WebSocket Integration:** On the `/auction/:id` page, connect the `socket.io-client`. Listen for `PRICE_UPDATE` events and use `useState` to instantly update the UI without refreshing the page.

### Phase 8: AI Assistant (Optional/Final Feature)
1. **AI Endpoint:** Create a simple endpoint `/api/ai/generate`.
2. **Prompt Engineering:** Securely call the OpenAI API directly from Node.js with a hardcoded prompt template (e.g., `"Write a professional auction description for the following item: {item_name}"`).
3. **Frontend Button:** Add a "Generate with AI" button on the frontend "Create Auction" form.

### Phase 9: Containerization
1. **Dockerize Backend:** Write a `Dockerfile` for the Node.js app.
2. **Dockerize Frontend:** Write a `Dockerfile` for the React app (typically a multi-stage build using Nginx to serve the static files).
3. **Docker Compose:** Create a `docker-compose.yml` file to spin up the Database, Backend, and Frontend together locally with a single command (`docker-compose up`).

### Phase 10: Production Deployment
1. **Database:** Provision a managed PostgreSQL database (e.g., AWS RDS, Supabase, or Render PostgreSQL).
2. **Backend:** Deploy your Node.js Docker container to a cloud provider (e.g., AWS ECS, Render, or Railway). Ensure environment variables (DB URL, JWT Secret) are securely set. Set up a Load Balancer if running multiple instances.
3. **Frontend:** Deploy your React app to a CDN/Hosting provider (e.g., Vercel, Netlify, or AWS S3+Cloudfront). Ensure it points to your deployed backend API URL.

