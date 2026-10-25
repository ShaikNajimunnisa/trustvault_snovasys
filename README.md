# trustvault_snovasys
# TrustVault – Secure Document Vault

TrustVault is a web app for storing documents and sharing them securely. Each document has an owner, who decides who else can read or edit it. Every sensitive action is written to a tamper-evident audit log.

## Features
- **Authentication:** register and login with JWT tokens (2-hour expiry) and bcrypt-hashed passwords.
- **Roles:** `admin`, `owner` and `viewer`. Self-registration always creates a viewer, so nobody can make themselves an admin.
- **Documents:** upload PDF, JPG, PNG or TXT files (up to 10 MB) and download them securely.
- **Sharing:** owners grant `read` or `edit` access to other users and can revoke it at any time.
- **Audit log:** records login, upload, download, grant and revoke events. Each entry stores a SHA-256 hash that includes the previous entry's hash. Admins can click "Verify hash chain" to check that no entry was altered.
- **Security:** login rate limiting, helmet security headers, CORS off by default, upload type whitelist.

## Tech Stack
Node.js · Express · SQLite (better-sqlite3) · JWT · bcryptjs · multer · plain HTML/JS front end

## Project Structure
```
trustvault_snovasys/
├── README.md
├── render.yaml                  # Render deployment blueprint
└── backend/
    ├── package.json
    ├── Dockerfile
    ├── .env.example             # copy to .env
    ├── public/
    │   ├── index.html           # web UI
    │   └── app.js               # UI logic
    ├── src/
    │   ├── app.js               # server entry point
    │   ├── config.js            # env vars, data paths, JWT secret check
    │   ├── db.js                # SQLite tables
    │   ├── seed.js              # creates demo users or the first admin
    │   ├── middleware/
    │   │   ├── auth.js          # JWT verify, role check, token signing
    │   │   └── checkPermission.js   # per-document read/edit/owner check
    │   ├── routes/
    │   │   ├── auth.js          # register, login, user list
    │   │   ├── documents.js     # upload, list, download, share, revoke
    │   │   └── admin.js         # audit log, hash verification, users
    │   └── utils/
    │       └── audit.js         # hash-chained audit logging
    └── tests/
        └── authz.test.js        # automated tests
```

## Run Locally
Requires Node.js 20 or newer.
```bash
cd backend
npm install
cp .env.example .env          # Windows PowerShell: copy .env.example .env
npm run seed                  # creates demo users
npm start                     # http://localhost:3000
npm test                      # runs the 6 automated tests
```

Demo accounts (local development only):

| Role   | Email             | Password  |
|--------|-------------------|-----------|
| Admin  | admin@vault.com   | admin123  |
| Owner  | owner@vault.com   | owner123  |
| Viewer | viewer@vault.com  | viewer123 |

## API Overview
| Method | Endpoint | Access |
|--------|----------|--------|
| POST | `/api/auth/register`, `/api/auth/login` | public |
| GET | `/api/auth/users` | logged in |
| POST | `/api/documents/upload` | logged in |
| GET | `/api/documents` | logged in (admin sees all) |
| GET | `/api/documents/:id/download` | read access |
| GET | `/api/documents/:id/permissions` | document owner |
| POST | `/api/documents/:id/grant`, `/revoke` | document owner |
| GET | `/api/admin/audit`, `/audit/verify`, `/users` | admin only |
| GET | `/health` | public |

## Deployment

### Docker (VPS or cloud VM)
```bash
cd backend
docker build -t trustvault .
docker run -d --name trustvault -p 80:3000 -v trustvault-data:/data \
  -e JWT_SECRET="$(openssl rand -hex 32)" trustvault
docker exec -e ADMIN_EMAIL=you@example.com -e ADMIN_PASSWORD='YourLongPassword' \
  trustvault node src/seed.js
```
Put it behind HTTPS (Caddy, nginx with Let's Encrypt, or a cloud load balancer).

### Render
1. Push the project to GitHub.
2. In Render, choose New → Blueprint and select the repo (it reads `render.yaml`).
3. Set `ADMIN_EMAIL` and `ADMIN_PASSWORD`. `JWT_SECRET` is generated automatically.
4. Once deployed, open the Shell tab and run `node src/seed.js` to create the admin.

A persistent disk is required because the database and uploads are stored on it, so use a paid instance.

## Environment Variables
| Variable | Purpose |
|----------|---------|
| `JWT_SECRET` | Required in production (16+ characters) |
| `NODE_ENV` | Set to `production` on the server (disables demo users) |
| `DATA_DIR` | Folder for `trustvault.db` and `uploads/` (use a persistent disk) |
| `CORS_ORIGIN` | Optional list of allowed origins, comma-separated |
| `PORT` | Server port (default 3000) |
| `ADMIN_EMAIL`, `ADMIN_PASSWORD` | Used by `node src/seed.js` to create the admin |
