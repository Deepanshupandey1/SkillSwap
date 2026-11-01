# PeerLearn — MERN peer-mentorship platform

PeerLearn lets learners find a more experienced peer, request a focused session, collaborate live, and exchange credits after both participants confirm completion. This package keeps the supplied `frontend.zip` UI and connects it to an Express/MongoDB API; a small React/Vite shell loads the supplied static UI modules.

## Stack

- **MongoDB  + Mongoose** — users, skills, availability, bookings, sessions, ledger entries, ratings, messages, and seed data.
- **Express  + Node.js** — REST API, validation, authentication, credit settlement, health endpoint, and optional OpenAI recommendations.
- **React + Vite** — frontend app shell and development/build server, retaining the supplied SkillSwap UI.
- **Socket.IO** — authenticated room presence, editor updates, whiteboard deltas, chat, notes, and WebRTC signaling.
- **WebRTC** — peer-to-peer audio/video in the session room, with local/remote video, microphone, camera, and hang-up controls.

## Features

- Searchable/filterable peer directory (skills/topics, coding language, spoken language, experience/level, and available slots).
- AI matching assistant: enter a preferred skill and optional language/availability. Candidates are ranked from real MongoDB profiles and are limited to mentors above the learner's experience level. If `OPENAI_API_KEY` is configured, OpenAI adds a concise recommendation explanation and learning goal; without a key, ranked database matches still work.
- Session request → mentor acceptance → credit reservation → shared live room → both participants confirm completion → one-time settlement. The backend verifies that the mentor is more experienced and checks available credit on acceptance.
- Credit ledger and profiles, session notes/summary, final code and whiteboard snapshot, learner rating, helpfulness bonus, leaderboard, and milestone badges.
- Shared editor, whiteboard, cursor/presence, chat, and notes synchronized through Socket.IO and saved to MongoDB.
- WebRTC video/audio controls inside the live room. `getUserMedia` requires HTTPS in deployed environments (localhost is allowed). STUN helps many networks; restrictive NATs/firewalls may require a TURN service. If you configure `TURN_URLS`, `TURN_USERNAME`, and `TURN_CREDENTIAL` in `server/.env`, authenticated clients receive those ICE server settings from `/api/rtc/config`; production deployments should use short-lived TURN credentials.

## Requirements

- Node.js 20+ recommended
- npm
- Docker Desktop for the included MongoDB replica-set setup, or MongoDB Atlas
- Optional: an OpenAI API key for AI-generated recommendations

Credit transfers use multi-document MongoDB transactions, so do not run against a standalone MongoDB server. Use the included single-node replica set or Atlas.

**Fix for `cannot index parallel arrays [topics] [skills]`:** the server now removes a legacy compound index containing both array fields during database startup and creates only the safe indexes declared by the current `User` schema. Restart the updated backend after replacing the project files. Do not manually create a compound index on both `topics` and `skills`; use separate indexes instead.

## Start locally

### 1. Start MongoDB

From this project root:

```bash
docker compose up -d
```

Initialize the replica set once on a fresh Docker volume:

```bash
docker exec -it peerlearn-mongo mongosh --eval 'try { rs.status().ok } catch (e) { rs.initiate({_id:"rs0",members:[{_id:0,host:"localhost:27017"}]}) }'
```

PowerShell users can run the same command from PowerShell if Docker Desktop is active. If you use MongoDB Atlas, use your replica-set connection string and skip the Docker step.

### 2. Configure the environment and install dependencies

From the project root, copy `server/.env.example` to `server/.env` (PowerShell: `Copy-Item server/.env.example server/.env`). Set `JWT_SECRET` to a private random value of at least 32 characters. Keep `ENABLE_DEMO_AUTH=true` for the sample accounts; set it to `false` on deployments where seeded identities should not be used. Add your OpenAI key to `OPENAI_API_KEY` for AI explanations. Keep the key in the backend environment only, never in client code.

Install both workspace packages once, then seed the sample data:

```bash
npm install
npm run seed
```

### 3. Start the backend and frontend

Start the backend in terminal one:

```bash
npm run dev:server
```

API: `http://localhost:5000`

Check the live database connection at `http://localhost:5000/api/health`. It returns `database: "connected"` only after MongoDB answers a ping; it returns HTTP 503 if the database is disconnected or unreachable.

Start the frontend in terminal two:

```bash
npm run dev:client
```

Open `http://localhost:5173`. The sign-in screen supports account creation and login, or the seeded demo account. The frontend environment defaults to the local API at `http://localhost:5000/api` and Socket.IO at `http://localhost:5000`; optionally override those in `client/.env` using `VITE_API_URL` and `VITE_SOCKET_URL`.

## Demo accounts

The seed script creates these accounts with password `PeerLearn123!`:

- `riya@peerlearn.demo` — beginner DSA learner
- `rohan@peerlearn.demo` — advanced DSA / C++ mentor
- `priya@peerlearn.demo` — advanced React / JavaScript mentor
- `sneha@peerlearn.demo` — advanced Python / ML mentor
- `arjun@peerlearn.demo` — advanced OS / networking mentor
- `karan@peerlearn.demo` and `aarav@peerlearn.demo` — intermediate peers

The sign-in screen supports your own account registration and login, or the demo account above. The demo user menu lets each browser tab switch to a seeded identity. Open two tabs and set one to a learner and the other to a mentor when testing a live room. Demo balances/history are illustrative; newly registered accounts receive 500 welcome credits and their own credit ledger.

## Recommended smoke test

1. Log in as Riya in one browser tab and Rohan in a second tab. Password: `PeerLearn123!`.
2. As Riya, search for DSA / C++ and request an Easy session with Rohan. Rohan should see the request.
3. As Rohan, accept it. Join the session from both accounts. Credits are reserved at acceptance but final transfer only settles once both participants confirm completion.
4. Open **Start call** in the live room and grant microphone/camera permissions. Test mute, camera toggle, and hang-up.
5. Type in the shared editor, draw on the whiteboard, chat, and write session notes. Verify both clients update and reload the session to verify saved state.
6. Confirm completion from both accounts, then check the learner's ledger, mentor earnings, recap, rating, and leaderboard.
7. In the directory AI panel, choose `DSA` and language `C++`; with `OPENAI_API_KEY` configured, the assistant also explains why the available matches fit.

## Credit policy

| Event | Credits |
|---|---:|
| Signup welcome balance | +500 once per account |
| Easy session (15 min) | 25 |
| Medium session (30 min) | 50 |
| Hard session (45 min) | 100 |
| Eligible 4- or 5-star helpfulness bonus | +5 to mentor |

## Main folders

```text
peerlearn/
├── package.json                    # Root npm workspace scripts
├── client/                         # React/Vite shell + supplied SkillSwap UI
│   ├── src/main.jsx                # Mounts UI inside React app shell
│   └── src/supplied-ui/            # Original frontend modules adapted to API
├── server/                         # Express + MongoDB + Socket.IO API
│   └── src/
│       ├── models/                 # Users, bookings, sessions, credits, ratings, messages
│       ├── routes/                  # REST API including AI recommendations
│       ├── sockets/                # Room sync and WebRTC signaling
│       └── seed/                   # Demo users/questions
└── docker-compose.yml              # MongoDB replica set
```

## Important verification notes

Static verification performed on this package: JavaScript syntax, React JSX parsing, 77 local import paths, named import/export references, package JSON, and checks that frontend record seeds/local room state are removed. A production Vite build and live integration run still need to be performed in an environment where npm dependencies can be installed and MongoDB is reachable. The app needs installed npm dependencies, a running MongoDB replica set, and browser permissions to exercise end-to-end. A valid `OPENAI_API_KEY` is required for generated AI explanations; without it, deterministic profile-based recommendations remain available. Video connectivity across restrictive networks can require a properly secured TURN service. If you configure `TURN_URLS`, `TURN_USERNAME`, and `TURN_CREDENTIAL` in `server/.env`, authenticated clients receive those ICE server settings from `/api/rtc/config`; production deployments should use short-lived TURN credentials. Keep `.env` private and do not commit API keys.
