# Project Instructions — Lost & Found dApp

This project spans two repositories. Apply the sections below according to
which repo you're working in.

## Frontend (`lost-and-found-front`)

Vue 3 + Vite project. When working on routing, components, forms, state, or
wallet-connect UI, follow standard Vue 3 Composition API conventions already
used in the repo — don't introduce Vue 2 patterns or Options API unless the
existing codebase already uses them.

- ethers.js is the only library used to talk to MetaMask / the contract.
  Don't add a second web3 library (e.g. web3.js) alongside it.
- Form validation uses **VeeValidate v4** (or `@vuelidate/core` — "Vuelidate
  Next" — if that's what the repo already has). Never add the legacy
  `vuelidate` package; it targets Vue 2 and is incompatible.
- Prefer Vite-native conventions (env vars via `import.meta.env`, etc.) over
  generic Node/Webpack solutions.

## Smart contract (`lost-and-found-back/contracts`)

Solidity + Hardhat project.

- Use standard `require` / `enum` / `struct` checks for validation. Do not
  use Core Solidity's algebraic-data-types / `match` syntax — it's an
  experimental prototype, not part of the stable 0.8.x compiler Hardhat
  uses.
- Keep the contract auditable: one clear state machine (listing lifecycle:
  `Open → Reported → Resolved`, plus `Reported → Open` when the owner rejects
  a false find, and `Open → Cancelled` when the owner self-refunds before
  anyone reports), explicit access control on fund-release functions,
  checks-effects-interactions ordering around any ETH transfer, backed by
  OpenZeppelin's `ReentrancyGuard` on the two functions that move ETH
  (`confirmRecovery`, `cancelListing`).
- Every new function or modifier needs a matching Hardhat unit test in the
  same PR.

## Storage service (`lost-and-found-back/storage-service`)

aiohttp REST API.

- Pydantic models are the single source of truth for request/response
  validation — don't hand-roll validation elsewhere in the service.
- Keep the service's only responsibilities: talking to Pinata (IPFS
  pinning/CID retrieval) and storing temporary owner↔finder contact data in
  SQLite. On-chain logic stays in the contract, not here.

## Skill Selection

- Use a Solidity/Hardhat skill (if available) for contract structure,
  testing, and deployment scripts.
- Use a Vue/Vite skill (if available) for frontend architecture and
  component-level decisions.
- Use both when a task touches the full flow — e.g. a new listing field
  that needs a form input, a contract field, and validation on both ends.

## Coding Standards

- Before changing anything, check `package.json` (frontend), `hardhat.config`
  (contract), and the storage service's dependency file — follow the
  conventions and package manager already in use.
- Preserve existing naming conventions, folder structure, and formatting.
- Make the smallest safe change that solves the task; avoid large rewrites
  unless explicitly requested.
- Never commit secrets (Pinata API key/JWT, private keys, RPC URLs) — these
  come from environment variables / GitHub Actions secrets, never hardcoded.


## Verification

Don't invent commands — read the actual scripts in `package.json` /
`hardhat.config` first and use those. As of this writing:

| Repo / layer     | Test               | Lint                          | Build                          |
| ----------------- | ------------------ | ------------------------------ | ------------------------------- |
| Frontend          | `npm run test`     | `npm run lint`                | `npm run build`                |
| Smart contract    | `npx hardhat test` | `npx solhint 'contracts/**/*.sol'` | `npx hardhat compile`      |
| Storage service   | `pytest`           | `ruff check .`                | `docker build -t lost-and-found-back .` |

Smart contract additionally runs `npx hardhat coverage` (Solidity coverage via
`solidity-coverage`, bundled in `@nomicfoundation/hardhat-toolbox`) in CI —
target >90% branch coverage on ETH-transfer, state-transition, access-control,
and cancellation/refund logic.

Run the relevant test → lint → build sequence for whatever you touched
before considering a task done. CI (GitHub Actions) re-runs the same steps
on push and blocks deploy (Netlify for frontend, Render for backend) on
failure. Never commit directly to `main`.

## Reference docs

- Vue 3: https://vuejs.org/guide/introduction.html
- ethers.js: https://docs.ethers.org/v6/
- Hardhat: https://hardhat.org/docs
- Solhint: https://protofire.github.io/solhint/
- Pinata API: https://docs.pinata.cloud/
