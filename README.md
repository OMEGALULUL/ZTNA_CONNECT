# ZTNA Connect

A GitHub Pages portfolio published behind **Cloudflare Zero Trust Access** — built as a live demo for BG ITversity Connect. It ships with a second zero-trust app: **save://**, a file platform on Cloudflare R2 that you can drive from inside the terminal.

Anyone who visits must first prove their identity (email + one-time code) before the page — or any file operation — is ever served. This is **clientless Zero Trust Network Access (ZTNA)** in action: identity-based access control enforced at the edge, with no VPN and no client software.

**Live demo URLs:**
- `https://ztna.blueberryservices.co.za` — the terminal portfolio
- `https://save.blueberryservices.co.za` — the R2 file platform (upload UI)

---

## Architecture

```
Attendee browser
      │
      │ 1. https://ztna.blueberryservices.co.za
      ▼
┌───────────────────────────────────────────────┐
│            Cloudflare Edge (proxy)             │
│                                               │
│   ┌──────────────────────────────────────┐    │
│   │     Cloudflare Access (ZTNA gate)    │    │
│   │  Policy: Email OTP → allow           │    │
│   │  Unauthenticated → 302 to login      │    │
│   └──────────────────────────────────────┘    │
│                      │                        │
│                      │ 2. authenticated only  │
└──────────────────────┼────────────────────────┘
                       ▼
            GitHub Pages (origin)
         omegalulul.github.io/ZTNA_CONNECT
```

The portfolio itself is a plain static site on GitHub Pages. Cloudflare fronts it with a proxied DNS record, and Cloudflare Access sits in front of the hostname as the security gate. The origin is never exposed — visitors interact only with Cloudflare.

### save:// — the file platform

```
save.blueberryservices.co.za
      │
      ▼
┌────────────────────────────────────────────────┐
│              Cloudflare Edge (proxy)           │
│                                                │
│   ┌────────────────────────────────────────┐   │
│   │      Cloudflare Access (ZTNA gate)     │   │
│   │   same policy: Email OTP → allow       │   │
│   └────────────────────────────────────────┘   │
│                     │                          │
└─────────────────────┼──────────────────────────┘
                      ▼
             Worker "save-drive"
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
   upload site     REST API        R2 bucket
   (drag & drop)   /api/files     save-blueberry
```

One Worker serves the whole platform: the drag-and-drop upload page, a small REST API, and access to the R2 bucket. The same Worker is also mounted at `ztna.blueberryservices.co.za/api/*`, so the LUNIX terminal talks to it same-origin — one identity session drives both apps.

Terminal integration:

```bash
save              # menu
save upload       # opens the save:// drop site
save down         # lists the bucket in the terminal
save down file.txt# downloads a file
save rm file.txt  # deletes a file
man save          # manual page
```

---

## How it works (layer by layer)

| Layer | Component | Role |
|---|---|---|
| 1. DNS | `CNAME ztna → omegalulul.github.io`, `CNAME save → blueberryservices.co.za` (both proxied) | Routes traffic through Cloudflare; origin IP stays hidden |
| 2. TLS | Cloudflare Universal SSL | HTTPS for `*.blueberryservices.co.za` at the edge |
| 3. Gate | Cloudflare Access apps (self-hosted) | Enforce policy on every request, for both hostnames, before anything reaches an origin |
| 4. Policy | `Email OTP — Conference Attendees` (decision: allow, include: everyone) | Anyone with a valid email can authenticate with a one-time code; session lasts 24h |
| 5. Origin (ztna) | GitHub Pages (`ZTNA_CONNECT` repo) | Serves the static portfolio (index.html) |
| 6. Origin (save) | Worker `save-drive` + R2 bucket `save-blueberry` | Upload UI + REST API + object storage; the terminal talks to the same Worker via `ztna.../api/*` |

### Request flow

1. Visitor requests `ztna.blueberryservices.co.za` (or `save.blueberryservices.co.za`).
2. Cloudflare Access checks for a valid session cookie (`CF_Authorization`).
3. No session → `302` redirect to the Access login (`*.cloudflareaccess.com`).
4. User enters email → receives one-time code → Access validates it.
5. Session issued → request forwarded to the origin (GitHub Pages or the save Worker) → page/file operation proceeds.

Unauthenticated requests never reach any origin — they're stopped at the Cloudflare edge. You can verify this yourself:

```bash
curl -I https://ztna.blueberryservices.co.za
# HTTP/2 302
# location: https://dry-rice-70ec.cloudflareaccess.com/cdn-cgi/access/login/...
# www-authenticate: Cloudflare-Access
```

---

## What is ZTNA?

**Zero Trust Network Access (ZTNA)** is a security model that replaces "trust the network, then trust anyone on it" with **"never trust, always verify"** — access decisions are made per request, per user, based on identity and policy, not on where the traffic comes from.

### ZTNA vs. VPN

| | Traditional VPN | ZTNA |
|---|---|---|
| Trust model | User is trusted once inside the network | Nothing is trusted by default; every request verified |
| Access scope | Full network access ("flat" trust) | Per-application, per-user policies |
| Attack surface | Entire internal network exposed | Only the specific app is reachable |
| Client | VPN software + credentials on every device | Optional — can be fully clientless (browser only) |
| Policy | Network-level (IP, port) | Identity-level (user, email, group, device posture) |
| User experience | Slow, clunky tunnels | Log in once, get a session |

### Key concepts used here

- **Identity-based access** — the policy allows *people* (verified by email OTP), not IPs. Anyone, anywhere can authenticate; nothing else about the requester matters.
- **Clientless access** — attendees authenticate in the browser. No app install, no device enrollment, works on any laptop or phone.
- **Edge enforcement** — the check happens at Cloudflare's edge, so the origin (GitHub Pages) never sees unauthenticated traffic.
- **Default deny** — Access apps have no implicit access; you must create an explicit Allow policy (ours is Email OTP + anyone).
- **Session control** — the policy sets a 24h session; after that, re-authentication is required.

### Why this matters

- The site stays on free GitHub Pages — no VPS, no exposed origin.
- Attendees get a seamless, credential-free signup (any valid email).
- You get **audit logs** of every login attempt in the Access dashboard — great for the demo and for real-world accountability.
- Tightening is trivial: switch the policy to require a specific email domain, email list, or add device posture checks later.

---

## Setup summary (how this was built)

1. **GitHub:** created `ZTNA_CONNECT` repo, pushed `index.html`, enabled Pages, set custom domain `ztna.blueberryservices.co.za` (Enforce HTTPS left off — Cloudflare handles TLS).
2. **Cloudflare DNS:** added proxied `CNAME ztna → omegalulul.github.io` and `CNAME save → blueberryservices.co.za`.
3. **Cloudflare Access:** created self-hosted apps for `ztna.blueberryservices.co.za` and `save.blueberryservices.co.za`, both 24h sessions.
4. **Policy:** `Email OTP — Conference Attendees` → decision *allow*, include *everyone* (one policy per app).
5. **R2 + Worker:** created bucket `save-blueberry`; deployed Worker `save-drive` (upload UI, REST API, R2 binding) and mounted it on `save.blueberryservices.co.za/*` and `ztna.blueberryservices.co.za/api/*`.

> Note: everything created via API — no cloudflared, no tunnels, pure edge ZTNA. Requests are gated by Access at the edge before the Worker is ever invoked.

---

## Demo tips (BG ITversity Connect)

- Always demo in **incognito** (or a fresh browser) — otherwise Cloudflare silently reuses your own session and skips the login.
- Show the Access dashboard: **Access → Apps → ZTNA Connect Portfolio → Policies** to display the rule live.
- Show **Access audit logs** after an attendee logs in — real evidence of identity-based access.
- The save demo packs well: `save down` lists the bucket in the terminal, `save upload` opens the drop site, then upload a file on the projector and watch it appear in `save down`.
- If the OTP email lands in spam, that's a talking point: email delivery is the only dependency of this auth model.
- Trivia for the audience: this page itself explains the demo in its "how it works" section below the terminal.