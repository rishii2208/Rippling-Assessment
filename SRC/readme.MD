# Boostly API

Boostly is a campus recognition platform for celebrating student contributions. Students can send kudos with credits, endorse recognitions, redeem earned credits for vouchers, and climb a leaderboard that rewards community impact. Business rules guarantee fairness: credits reset monthly with optional carry-forward, self-recognition is blocked, sending limits prevent abuse, and immutable ledger entries maintain financial transparency.

## How It Works

- **Recognition Flow**: A sender transfers credits to a peer while the service validates balance and monthly limits. Ledger entries record both the debit and credit, and monthly quota counters enforce the 100-credit cap.
- **Endorsements**: Other students can cheer a recognition once. Duplicate endorsements are prevented by a composite uniqueness constraint, while auto-incremented counters keep aggregate views fast.
- **Redemption**: Students redeem earned credits (only those received, never sent) at a fixed rate of ₹5 per credit. Successful redemptions persist vouchers, debit the ledger, and return the updated balance.
- **Monthly Reset**: At the start of each month, an APScheduler job grants every student 100 new credits, carries forward up to 50 unused credits from the previous month, resets send limits, and logs ledger adjustments.
- **Leaderboard**: Aggregates recognitions and endorsements to highlight top recipients, ordering by total credits received and breaking ties by student ID.

State transitions rely on SQLAlchemy models backed by PostgreSQL. A `credit_ledger` table keeps append-only events so balances and histories are auditable. Service-layer functions wrap checks and writes in a single session transaction, guaranteeing consistency across quotas, recognitions, endorsements, and redemptions.

## Setup

1. **Clone & enter repo**
	 ```bash
	 git clone <repo-url>
	 cd rishi-raj-prajapati-2k22ec187
	 ```
2. **Create virtual environment (recommended)**
	 ```bash
	 python3 -m venv .venv
	 source .venv/bin/activate
	 ```
3. **Install dependencies**
	 ```bash
	 pip install fastapi "uvicorn[standard]" sqlalchemy psycopg2-binary pydantic apscheduler python-dotenv
	 ```
4. **Provision PostgreSQL**
	 ```bash
	 docker run --name boostly-postgres -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=boostly -p 5432:5432 -d postgres:14
	 ```
5. **Apply schema**
	 ```bash
	 psql postgresql://postgres:postgres@localhost:5432/boostly -f src/app/models/schema.sql
	 ```

## Environment Configuration

| Variable | Default | Purpose |
|----------|---------|---------|
| `BOOSTLY_DATABASE_URL` | `postgresql+psycopg2://postgres:postgres@localhost:5432/boostly` | SQLAlchemy connection string for the primary database. |

Create a `.env` file alongside `src/app/core/config.py` overrides to suit each environment. The settings class automatically loads variables prefixed with `BOOSTLY_`.

## Running the API

From the repository root:

```bash
uvicorn app.main:app --app-dir src --reload
```

This starts the service at `http://127.0.0.1:8000`, mounts OpenAPI docs at `/docs`, and registers the APScheduler job responsible for monthly resets.

## API Reference

All endpoints are served under `/api/v1`.

### Health Probe
- **GET** `/api/v1/health`
- **Response** `200 OK`
	```json
	{"status": "ok"}
	```

### Recognitions
- **POST** `/api/v1/recognitions`
	- Sample request
		```json
		{
			"sender_id": "aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa",
			"receiver_id": "bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb",
			"credits_transferred": 30,
			"message": "Thanks for running the robotics workshop!"
		}
		```
	- Sample response `201 Created`
		```json
		{
			"recognition_id": "44444444-4444-4444-4444-444444444444",
			"sender": {
				"student_id": "aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa",
				"campus_uid": "S1001",
				"display_name": "Alex Rao",
				"email": "alex.rao@example.edu"
			},
			"receiver": {
				"student_id": "bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb",
				"campus_uid": "S1002",
				"display_name": "Bianca Liu",
				"email": "bianca.liu@example.edu"
			},
			"credits_transferred": 30,
			"message": "Thanks for running the robotics workshop!",
			"month_bucket": "2025-11-01",
			"created_at": "2025-11-12T10:15:30+00:00"
		}
		```
	- Errors
		- `400` when balance or monthly limit is insufficient, or sender equals receiver
		- `404` when either student is missing

- **GET** `/api/v1/recognitions`
	- Query params: `sender_id`, `receiver_id`, `limit` (default 50), `offset`
	- Returns ordered list of recognition objects like above.

### Endorsements
- **POST** `/api/v1/endorsements`
	- Request
		```json
		{
			"recognition_id": "44444444-4444-4444-4444-444444444444",
			"endorser_id": "cccccccc-cccc-cccc-cccc-cccccccccccc"
		}
		```
	- Response `201 Created`
		```json
		{
			"endorsement_id": "66666666-6666-6666-6666-666666666666",
			"recognition_id": "44444444-4444-4444-4444-444444444444",
			"endorser": {
				"student_id": "cccccccc-cccc-cccc-cccc-cccccccccccc",
				"campus_uid": "S1003",
				"display_name": "Carlos Menon",
				"email": "carlos.menon@example.edu"
			},
			"created_at": "2025-11-12T12:05:30+00:00"
		}
		```
	- Errors: `400` for duplicate endorsements, `404` for missing recognition or student.

- **GET** `/api/v1/endorsements?recognition_id=...`
	- Returns endorsements for the specified recognition, newest first.

### Redemptions
- **POST** `/api/v1/redemptions`
	- Request
		```json
		{
			"student_id": "bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb",
			"credits_redeemed": 40
		}
		```
	- Response `201 Created`
		```json
		{
			"redemption": {
				"redemption_id": "88888888-8888-8888-8888-888888888888",
				"student": {
					"student_id": "bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb",
					"campus_uid": "S1002",
					"display_name": "Bianca Liu",
					"email": "bianca.liu@example.edu"
				},
				"credits_redeemed": 40,
				"voucher_value": 200,
				"status": "ISSUED",
				"created_at": "2025-11-12T14:30:00+00:00"
			},
			"available_balance": 35
		}
		```
	- Errors: `400` if requested credits exceed redeemable balance or none are available, `404` if student missing.

### Leaderboard
- **GET** `/api/v1/leaderboard?limit=10`
	- Response `200 OK`
		```json
		[
			{
				"student_id": "bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb",
				"campus_uid": "S1002",
				"display_name": "Bianca Liu",
				"email": "bianca.liu@example.edu",
				"total_credits_received": 80,
				"recognitions_received": 3,
				"endorsements_received": 5
			}
		]
		```

## Scheduler & Manual Reset

The APScheduler job runs on the first day of each month at 00:05 UTC. It uses `MonthlyQuota` and `CreditLedger` to:

- Reset each student's `send_limit` to 100 and `credits_sent` to 0.
- Grant baseline `MONTHLY_RESET` +100 credits.
- Carry forward up to 50 unused credits via `CARRY_FORWARD` entries while logging expirations through `CARRY_FORWARD_EXPIRED`.

Manual test of the reset logic:

```bash
python -c "from app.jobs import run_reset_once; print(run_reset_once())" --app-dir src
```

This prints a summary with processed students and total carry-forward applied. You can seed specific timestamps via `run_reset_once(current_time=...)` to simulate boundaries.

## Testing Suggestions

- Use pytest with an in-memory Postgres instance or transactional fixtures to validate rule enforcement (self-recognition, duplicate endorsements, over-redemption, leaderboard ordering).
- Exercise the monthly reset by creating ledger fixtures for two consecutive months and asserting carry-forward behavior.

## Troubleshooting

- Ensure PostgreSQL is reachable at the URL in `BOOSTLY_DATABASE_URL`.
- Apply the SQL schema before first run so enums and tables exist.
- If APScheduler logs that the job failed, inspect the stack trace—unapplied migrations or constraint violations are the most common causes.

With this setup, Boostly offers an auditable credit economy for student communities while keeping the operational footprint light.
