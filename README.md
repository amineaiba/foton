# Foton

Backend for a cash-agent money-transfer system . Agents send and receive cash for customers; the API tracks agent wallet balances and the full transaction lifecycle. Built with Django REST Framework, consumed by a mobile app (frontend built separately, not part of this repo).

A live version is deployed at `foton.onrender.com`.

## Stack

- Python / Django REST Framework
- PostgreSQL
- JWT auth (SimpleJWT)
- Cloudinary (media storage in production)
- Twilio / console SMS backend (pluggable)

## Setup

```bash
# create + activate venv (Windows)
python -m venv venv
venv\Scripts\activate

# install deps
pip install -r requirements.txt

# configure environment
copy .env.example .env   # fill in DB, JWT lifetimes, SMS backend, etc.

# migrate + run
python manage.py migrate
python manage.py runserver
```

Create an agent/admin user with `python manage.py createsuperuser` (custom user model needs phone, email, full_name).

Run tests per app:

```bash
python manage.py test accounts
python manage.py test transactions
```

## Architecture

Two Django apps plus a shared services layer that holds all business logic (views stay thin):

- **accounts** — custom phone-based user model (`Users`), agent `Wallet` (DZD balance), `Profile`, OTP-based password reset.
- **transactions** — `Transaction` (core money-movement record), `MoneyRequester` (non-user sender/recipient), `IdempotencyLog` (dedupes transfer requests).
- **services/** — `TransactionService` (fee calc, transfer-code generation, send/claim logic), `account_service` (password-reset JWT flow), `NotificationService` (SMS on transfer/refund events).

### Money-transfer lifecycle

1. Agent sends money on behalf of a customer — own wallet is debited `amount + fee`, a PENDING transaction is created with a 10-digit transfer code, recipient gets an SMS with the code.
2. Receiving agent looks up the transaction by transfer code or by recipient phone + last name.
3. Receiving agent claims it — their wallet is credited, transaction marked COMPLETED. Row-level locking (`select_for_update`) prevents double-claims.
4. Unclaimed transactions expire and can later be refunded (claimed back by the sender).

All balance-mutating operations run inside `transaction.atomic()` with row locks to keep wallet balances consistent under concurrent access.

## API

Base URL: `https://foton.onrender.com/api`

| Method | Endpoint | Description |
|---|---|---|
| POST | `/auth/login/` | Login by phone + password, returns JWT pair |
| POST | `/token/refresh/` | Refresh access token |
| POST | `/token/verify/` | Verify a token is valid |
| POST | `/auth/logout` | Blacklist refresh token |
| POST | `/auth/password-reset/request/` | Send OTP for password reset |
| POST | `/auth/password-reset/verify/` | Verify OTP |
| POST | `/auth/password-reset/confirm/` | Set new password |
| GET | `/accounts/wallet/` | Get agent wallet balance |
| GET | `/accounts/profile/` | Get agent profile |
| PUT | `/accounts/profile/` | Update agent profile |
| POST | `/accounts/change-password/` | Change password (authenticated) |
| GET | `/transactions/calculate-fee/` | Preview fee before sending |
| POST | `/transactions/send/` | Send money (creates PENDING transaction) |
| POST | `/transactions/receive/lookup/` | Look up a transaction to claim |
| POST | `/transactions/receive/claim/` | Claim a transaction (or refund) |
| GET | `/transactions/history/` | Agent's transaction history |
| POST | `/transactions/lookup-user/` | Look up recipient before sending |
| POST | `/transactions/expire-trigger/` | Manually run expiry cleanup job |

All endpoints except login/refresh/verify/password-reset require `Authorization: Bearer <access_token>`.
