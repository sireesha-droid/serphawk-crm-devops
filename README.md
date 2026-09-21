# SerpHawk CRM — AWS Deployment

A CRM application deployed on AWS as part of the SERP Hawk DevOps Engineer technical assessment.

**Live application:** http://3.104.132.154:3000
**Login:** `admin@example.com` / `password123`

---


Tech Stack

- **Frontend:** Next.js 16 / React 19 / TypeScript, containerized with Docker
- **Backend:** Python / FastAPI, containerized with Docker
- **Database:** PostgreSQL hosted on AWS RDS (`db.t4g.micro`)
- **Hosting:** AWS EC2, Ubuntu 22.04 (`t3.small`)
- **EBS:20 GB
- **Containerization:** Docker & Docker Compose
- **Monitoring:** Amazon CloudWatch


## Architecture

Internet → EC2 (t3.small) running two Docker containers:
- Next.js frontend (port 3000)
- FastAPI backend (port 8000)

EC2 connects to Amazon RDS (PostgreSQL, db.t4g.micro, Free Tier). CloudWatch automatically monitors both EC2 and RDS. Security Group on EC2 allows ports 22 (SSH, restricted), 3000, and 8000. See `aws-architecture-diagram.svg` in this repo for a visual diagram.

The frontend calls the backend over REST (`/clients`, `/login`, `/projects`, etc.) and a WebSocket connection (`/ws/chat/{thread_id}`) for real-time messaging. The backend connects to RDS PostgreSQL via `DATABASE_URL`.

## AWS Services Used & Why

| Service | Purpose | Reasoning |
|---|---|---|
| **EC2** | Hosts both Docker containers | The app has a persistent WebSocket connection, which needs a server that stays running — this rules out serverless (Lambda). A single EC2 instance is the simplest correct fit for one app with two containers; ECS/EKS would be unnecessary complexity for this scale. `t3.small` was used instead of the Free Tier `t2/t3.micro` after the micro instance ran out of disk/memory headroom while building the frontend's Docker image. |
| **Security Group** | Firewall rules on the EC2 instance | SSH (22) restricted, ports 3000 and 8000 opened for the app. |
| **VPC** | Default AWS networking | Used automatically with EC2 and RDS; no custom VPC needed for this scope. |
| **RDS (PostgreSQL)** | Managed database | db.t4g.micro, Single-AZ, 20GB — Free Tier eligible. Chosen over self-hosting Postgres in a container so backups, patching, and availability are handled by AWS rather than manually. RDS's built-in "Set up EC2 connection" feature was used to automatically configure Security Group rules so only the application EC2 instance can reach the database. |
| **EBS** | EC2's attached disk | Resized from the default 8GB to 20GB — the default size was insufficient to build the frontend's Docker image (Next.js's build output and `node_modules` require significant temporary disk space during the image build step). |
| **Elastic IP** | Static public IP | Attached to the EC2 instance so the deployed URL stays constant across instance restarts (a plain EC2 public IP changes on restart otherwise). |
| **CloudWatch** | Monitoring | Automatic basic metrics (CPU, network) for both EC2 and RDS, included at no extra cost. |

**File storage:** Uploaded files are currently stored on the EC2 instance's local disk (`static/uploads/`). Moving this to S3 would be a natural next improvement for durability across instance restarts, but was out of scope given the assignment timeline.

## Running Locally

### Prerequisites
- Python 3.10+
- Node.js 20+
- Docker Desktop (for the containerized approach)
- A PostgreSQL database (e.g. a free RDS or Neon instance)

### Option A: Docker Compose (recommended)

1. Clone the repository
2. Create a `.env` file in the project root:

   `DATABASE_URL=postgresql://<user>:<password>@<host>/<db>`

3. Run: `docker compose up --build`
4. Frontend: http://localhost:3000
   Backend docs: http://localhost:8000/docs

### Option B: Run directly (no Docker)

**Backend:**
```
python -m venv venv
venv\Scripts\Activate      # Windows
pip install -r requirements.txt
python -m uvicorn main:app --reload
```

**Frontend** (separate terminal):
```
cd frontend
npm install
npm run dev
```

### Seeding an admin user

The database starts empty. Create a default admin login with:
```
python seed_db.py
```
This creates `admin@example.com` / `password123`.

## Deployment Steps (AWS)

1. Launched an EC2 instance (Ubuntu 22.04, t3.small)
2. Configured Security Group to allow SSH (22, restricted), and ports 3000 / 8000
3. Installed Docker, Docker Compose, and Git on the instance
4. Cloned the source code directly from GitHub onto the instance
5. Created an RDS PostgreSQL database (db.t4g.micro, Free Tier), using its "Set up EC2 connection" feature to automatically configure Security Group access
6. Resized the EC2 instance's EBS volume from 8GB to 20GB to accommodate the frontend's Docker build
7. Set `DATABASE_URL` in a `.env` file on the instance, pointing to RDS
8. Set `NEXT_PUBLIC_API_BASE_URL` (frontend build arg, in `docker-compose.yml`) to the EC2 instance's Elastic IP
9. Ran `docker compose up -d --build` to build and start both containers in detached mode
10. Seeded an admin user and verified the app by logging in from a browser outside the EC2 instance

## Deployment / Configuration Files

- `Dockerfile` — backend image (Python 3.10-slim, FastAPI/Uvicorn)
- `frontend/Dockerfile` — frontend image (Node 20-alpine, Next.js build + start)
- `docker-compose.yml` — orchestrates both containers together
- `aws-architecture-diagram.svg` — visual architecture diagram

## Environment Variables

| Variable | Required | Purpose |
|---|---|---|
| `DATABASE_URL` | Yes | PostgreSQL connection string |
| `OPENAI_API_KEY` | No | Enables AI features. App runs without it; those features degrade gracefully. |
| `SENDER_EMAIL` / `SENDER_PASSWORD` | No | Outlook SMTP credentials for email notifications. Not configured in this deployment. |
| `NEXT_PUBLIC_API_BASE_URL` | Yes (frontend build) | The backend's public URL, baked into the frontend at build time. |

**No credentials, API keys, or passwords are committed to this repository.** `.env` and other sensitive/generated files are excluded via `.gitignore`.

## Known Limitations / Future Improvements

- Uploaded files are stored on local disk rather than S3 — would require changing the upload logic in `main.py`.
- No HTTPS/custom domain configured — the app is served over plain HTTP on the instance's public IP.
- RDS is currently publicly accessible (restricted to the EC2 instance's Security Group). A stricter setup would use a private subnet.
- Email notification feature is not configured, since it requires a dedicated email account's credentials.
