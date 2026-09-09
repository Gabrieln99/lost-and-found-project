# Lost & Found dApp

Decentralized platform for publishing, searching, and rewarding the recovery
of lost items/pets. A smart contract acts as the trusted intermediary for the
reward: the owner's funds are locked (escrow) until the finder's recovery is
confirmed, removing the need for a central authority or commission.

University project for FIPU (Informatics), combining two courses:
**Raspodijeljeni sustavi** (Distributed Systems) and **Blockchain aplikacije**
(Blockchain Applications).

## Core flow

1. Owner connects wallet (MetaMask), fills out a lost-item form (title,
   description, location, photo).
2. Frontend sends the photo plus the title/description/location to the
   storage service, which pins the photo to Pinata (IPFS), bundles it with
   the text fields into a JSON metadata document (`{title, description,
   location, image: "ipfs://<photo CID>"}`), pins that JSON to Pinata too,
   and returns the JSON's CID. That CID (not the raw photo's) is what goes
   on-chain.
3. Owner submits the listing and locks the reward (ETH) in the smart
   contract, passing the metadata CID as `itemCID`.
4. A listener service watches the chain, picks up the new listing event, and
   pushes listing data to the backend.
5. Finder connects their wallet, browses listings, and reports a find on a
   matching listing.
6. The listener detects the "reported" event; backend updates status so all
   users see it.
7. Owner and finder coordinate a handover (via the backend's temporary
   chat/contact storage).
8. Owner releases the funds through the contract; listener detects it and
   backend marks the listing "Resolved".

## Repositories

Two separate repos, each with its own CI/CD:

- **`lost-and-found-front`** — Vue.js app only. Frontend CI/CD deploys to
  Netlify.
- **`lost-and-found-back`** — Solidity smart contract (Hardhat) + aiohttp
  storage service + SQLite. Backend CI/CD builds a Docker image and deploys
  to Render.

Both pipelines run **test → lint → build** before allowing a deploy.

## Tech stack

**Frontend** (`lost-and-found-front`)
- Vue.js, Vite (local dev), scaffolded via the official `create-vue` CLI
- Vue Router — multi-view navigation (publish listing, browse listings,
  listing detail/report-found, etc.)
- Pinia — shared state across views (connected wallet address, contract
  instance, listings cache)
- ethers.js — wallet/contract communication via MetaMask
- Form validation: **VeeValidate v4** (Vue 3-compatible). Do not use the
  legacy `vuelidate` package — it targets Vue 2; if Vuelidate is preferred
  over VeeValidate, use `@vuelidate/core` ("Vuelidate Next") instead.
- Tests: Vitest · Lint: ESLint, plus `oxlint` (bundled by `create-vue` as a
  fast pre-lint pass; `eslint-plugin-oxlint` disables the ESLint rules it
  duplicates, so the two don't conflict) · Build: Vite

**Smart contract** (`lost-and-found-back`)
- Solidity, Hardhat (local dev/testing)
- Listing lifecycle: `Open → Reported → Resolved`, plus `Reported → Open`
  (owner rejects a false find) and `Open → Cancelled` (owner self-refunds
  before anyone reports)
- `@openzeppelin/contracts` (`ReentrancyGuard`) — defense-in-depth on the
  two ETH-transfer functions, on top of checks-effects-interactions ordering
- **Deployed to Sepolia** at `0x1945e05F857505C4282168d7Fa2974b36353B8d1`
  (see `deployments/sepolia.json` for the transaction hash, deployer
  address, and block-explorer link — that file is the source of truth,
  not this doc). Deployed via `.github/workflows/deploy-sepolia.yml`,
  triggered manually via `workflow_dispatch` (not automatically on merges
  to `main`), using the `SEPOLIA_RPC_URL` and `SEPOLIA_PRIVATE_KEY` GitHub
  secrets. Re-running that workflow redeploys a fresh instance at a new
  address — update `deployments/sepolia.json` (and any frontend env var
  pointing at it) if that ever happens.
- Tests: Hardhat/Cyfrin-style unit tests · Lint: Solhint
- Note: Solidity's algebraic-data-types / pattern-matching (`match`, `data`)
  is an experimental **Core Solidity** prototype, not part of the stable
  0.8.x compiler used by Hardhat. Do not rely on it for contract validation
  in this project — use standard `require`/`enum`/`struct` checks instead.
- **Known accepted limitations** (deliberate, not bugs — in scope for a
  university project, not a production system):
  - `confirmRecovery` and `cancelListing` push ETH via
    `recipient.call{value: ...}("")`. If the recipient (finder or owner) is
    a contract that reverts on receiving ETH, the whole transaction reverts
    — status stays `Reported`/`Open` and the reward stays escrowed rather
    than being lost. No pull-payment/withdraw-pattern fallback is
    implemented; the caller can simply retry (e.g. from a different,
    ETH-accepting address is not possible since the finder/owner is fixed,
    but the owner can `rejectReport` a stuck `Reported` listing to unblock
    it, or otherwise must wait for the recipient contract's logic to allow
    receiving ETH).
  - Once a listing is `Reported`, there is no timeout or forced-resolution
    path if the owner goes silent — funds stay locked until the owner calls
    `confirmRecovery` or `rejectReport`. No on-chain dispute arbitration is
    implemented.

**Storage service** (`lost-and-found-back`)
- aiohttp REST API, communicates with the smart contract layer
- Pinata API (JWT/API key) for IPFS pinning
  - `POST /upload` — pins a raw image, returns its CID (generic utility)
  - `POST /listing-metadata` — pins an image plus title/description/location
    as a bundled JSON document (`{title, description, location, image:
    "ipfs://<CID>"}`), returns the JSON's CID. This is the CID the frontend
    passes on-chain as `itemCID`, so the contract points at a metadata blob,
    not a bare image.
- SQLite for temporary owner↔finder contact/chat data (not built yet)
- Validation: Pydantic
- **Rate limiting, no auth layer** (deliberate scoping decision, not an
  oversight): `/upload` and `/listing-metadata` are rate-limited per client
  (in-memory sliding window, `RATE_LIMIT_REQUESTS_PER_MINUTE`, default 10
  requests/minute, shared across both endpoints since they both burn the
  same Pinata quota; `X-Forwarded-For`'s first hop is preferred over the
  raw peer address so this still works correctly behind Render's reverse
  proxy in production). There is deliberately **no API key/auth layer** in
  front of them, even though anyone who knows the URL can call them.
  Rationale: this service only brokers IPFS uploads — it holds no funds
  and makes no on-chain state changes itself. The actual value transfer
  (the escrowed reward) happens entirely on-chain via `LostAndFound.sol`,
  gated by wallet signatures the smart contract verifies independently of
  this service. Worst case if the rate limit is bypassed or set too
  loose is wasted Pinata quota, not stolen funds — proportionate for a
  university project's storage layer, not something that would be
  acceptable if this service touched money directly. The rate limiter
  itself is intentionally simple (in-memory, per-process, not
  distributed/persistent) for the same reason: a single small container,
  not a fleet needing shared rate-limit state.

**Listener**
- Not a separate deployed service — it's a background task inside the
  aiohttp storage service (same process/container) that watches on-chain
  events and updates listing status accordingly. Production only ships one
  container (aiohttp server + SQLite), so any listener code lives in
  `lost-and-found-back`, not its own repo or container.

## Conventions

- Git for version control; two independent repos as above.
- Frontend and storage service communicate over REST; the storage service
  talks to Pinata and returns the CID used on-chain.
- Full CI/CD command list (test/lint/build invocations) lives in
  `AGENTS.md`, not here — this file stays limited to architecture and
  decisions.
