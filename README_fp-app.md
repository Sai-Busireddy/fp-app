# Fingerprint & Face Verification — Proof‑of‑Concept

A full‑stack biometric identity system that lets you **capture, register, and instantly match fingerprints and faces** through a web UI.

| Layer | Tech |
|-------|------|
| **Frontend** | Next.js (App Router, TypeScript), Tailwind CSS, shadcn/ui, Framer Motion, NextAuth.js |
| **Backend**  | FastAPI, Python 3.11, supabase‑py, OpenCV, InsightFace |
| **Data**     | Supabase (PostgreSQL + Storage + Auth) &pgvector extension |
| **Container**| Docker & docker‑compose |

<p align="center"><img src="docs/screenshot.png" alt="UI screenshot" width="700"></p>

---

## ✨ Features

* Camera or file‑upload capture for **face & fingerprint** images  
* One‑click **registration** with user metadata  
* **Vector search** (pgvector) for < 200 ms similarity matches  
* JSON REST API (`/api/*`) + auto‑generated Swagger docs  
* Auth‑ready: drop in a NextAuth provider and JWT secret  
* **Docker‑first** but equally simple virtual‑env + pnpm workflow

---

## 1. Quick Start (Docker)

```bash
# clone & enter
git clone https://github.com/your-org/fp-app.git
cd fp-app

# copy env templates
cp frontend/.env.example frontend/.env
cp backend/.env.example  backend/.env
# then edit the values marked ❗ below

# one‑liner
docker compose up --build
```

| Service | URL (default) |
|---------|---------------|
| Frontend | <http://localhost:3000> |
| Backend  | <http://localhost:8000/docs> (Swagger + Redoc) |

---

## 2. Environment Variables ❗

Both `frontend/.env` and `backend/.env` need a **Supabase project**.

| Key | Example | Notes |
|-----|---------|-------|
| `SUPABASE_URL` | `https://xyzcompany.supabase.co` | _Base URL_ from project settings (not the Postgres conn string). |
| `SUPABASE_ANON_KEY` | `eyJhbGciOi...` | _Anon public key_. |
| **Frontend‑only** |
| `NEXT_PUBLIC_HOST_URL` | `http://localhost:3000` | Used to build absolute URLs in NextAuth callbacks. |
| `NEXT_PUBLIC_API_URL` | `http://127.0.0.1:8000` | Where the FastAPI service runs. |
| **Backend‑only** |
| `JWT_SECRET_KEY` | any‑random‑string | Required if you enable authenticated routes later. |

> **Pro‑tip**: Never commit filled `.env` files—use `.gitignore`.

---

## 3. Creating the Supabase schema

1. Enable **pgvector** under _Database → Extensions_.  
2. Run the SQL below in the SQL editor.

```sql
-- Users table
create table if not exists users (
  id            uuid primary key default gen_random_uuid(),
  display_name  text,
  platform      text default 'local',
  image_url     text,
  face_vec      vector(512),
  thumb_vec     text,                 -- ORB descriptors (base64)
  inserted_at   timestamptz default now()
);

-- Public social‑media faces
create table if not exists public_profiles (
  id            uuid primary key default gen_random_uuid(),
  platform      text,
  profile_id    text,
  display_name  text,
  image_url     text,
  face_vec      vector(512),
  inserted_at   timestamptz default now()
);

-- Face‑matching helper (cosine similarity)
create or replace function match_public_faces(
  query_vec  vector(512),
  k          int     default 5,
  threshold  real    default 0.36
)
returns table (
  id           uuid,
  display_name text,
  score        real,
  platform     text
) language sql stable as $$
  select
    p.id,
    p.display_name,
    1 - (p.face_vec <=> query_vec) as score,
    p.platform
  from public_profiles p
  where p.face_vec is not null
    and (p.face_vec <=> query_vec) < threshold
  order by p.face_vec <=> query_vec
  limit k;
$$;
```

3. Create a **Storage bucket** called `images` and set it to “public”.

---

## 4. Local dev without Docker

```bash
# backend
cd backend
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
uvicorn main:app --reload --port 8000

# frontend (new tab)
cd ../frontend
pnpm install
pnpm dev          # runs on :3000
```

### Next.js image domains

If you pull avatars from Twitter/Google, add them once:

```ts title="frontend/next.config.ts"
const nextConfig = {
  images: {
    domains: ['pbs.twimg.com', 'lh3.googleusercontent.com', 'avatars.githubusercontent.com'],
  },
};
export default nextConfig;
```

---

## 5. API Reference (v1)

| Method | Route | Body | Description |
|--------|-------|------|-------------|
| `POST` | `/api/register/user` | `{ displayName, faceFile, thumbFile }` (multipart) | Register a user with both biometrics. |
| `POST` | `/api/users/search` | `{"face": <base64‑jpg>}` | Top‑K face match. |
| `POST` | `/api/face/search` | `{"face": <base64‑jpg>}` | (InsightFace branch) High‑accuracy search. |
| `GET`  | `/api/users/{id}` | — | Fetch user + signed image URL. |

Swagger docs are live at `/docs`.

---

## 6. Testing & linting

```bash
# backend
pytest
ruff check .

# frontend
pnpm test          # vitest
pnpm lint          # eslint
```

---

## 7. Troubleshooting

| Symptom | Fix |
|---------|-----|
| **`next/image` “hostname not configured”** | Add the host to `images.domains` (see §4). |
| **`cv2.error: OpenCV(4.xx)…`** on Ubuntu | `pip install opencv-python-headless` or apt `libgl1`. |
| **Slow first search (>2 s)** | The InsightFace model (~100 MB) downloads on first run—wait once. |
| “No thumb features found” | Ensure the finger occupies >50 % of frame; ORB can fail on blurred images. |

---

## 8. Deployment notes

* **Vercel + Railway** is the quickest pair: set the same env vars.  
* Behind **ngrok** or other tunnels, update `NEXT_PUBLIC_HOST_URL`.  
* For production HTTPS, put Cloudflare or an ALB in front of both ports.

---

## 9. Contributing

Pull requests are welcome! Please open an issue first to discuss major changes.

```bash
git checkout -b feat/my-awesome-idea
# make changes
git commit -s -m "feat: my awesome idea"
git push origin feat/my-awesome-idea
```

---

## 10. License

MIT © 2025 Your Name
