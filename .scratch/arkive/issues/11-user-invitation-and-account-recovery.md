# User Onboarding, Invitations & Account Recovery

Type: grilling
Status: resolved
Blocked by: 05

## Question

How should Arkive onboard non-technical family and community users and handle password recovery without requiring external SMTP email servers or complicated verification infrastructure?

Specific decisions to resolve with the user:
1. **Invitation Mechanism:** How should Admins invite new users? (e.g. single-use or multi-use invitation links with pre-assigned roles like Uploader or Viewer and pre-selected Groups)?
2. **Password Recovery without SMTP:** In self-hosted setups without mail servers, how should users recover lost passwords? (e.g. offline emergency backup recovery codes generated at account creation vs Admin-generated one-time reset links)?
3. **First-Run Instance Setup:** How should a newly deployed Arkive instance bootstrap its first administrator account securely?
4. **Session Persistence:** What should the default session duration be for remembered devices on the SvelteKit PWA, and how are revoked sessions invalidated across Valkey?

## Answer

**Verdict: Implement an invitation-based onboarding model with customizable single-use and multi-use invite links, provide offline emergency recovery keys combined with Admin-assisted reset links and CLI overrides to eliminate SMTP mail server dependencies, enforce an automatic first-run setup wizard that permanently locks upon initial Admin creation, and manage 30-day sliding sessions via secure HTTP-only cookies backed by Valkey for instant multi-device revocation.**

### Agreed Decisions
1. **Configurable Tokenized Invitation Links**:
   - Admins generate invite links (`/join/:token`) with configurable parameters: single-use default (`max_uses = 1`) or multi-use count cap, expiration timestamps (24h, 7d, 30d, or never), and pre-assigned roles (`Viewer` or `Uploader`).
   - Group memberships (defined in ADR-0007) are pre-assigned to the invite and attached to the user during registration.
   - Registration runs inside a database transaction with a row lock (`FOR UPDATE`), hashing passwords via Argon2id (64MB memory, 1 iteration, 1 thread) tuned for the Pentium E5400.
2. **Three-Tier SMTP-Free Password Recovery**:
   - *Tier 1 (Admin One-Time Reset Links):* Admins generate single-use 15-minute reset links (`/reset-password/:token`) from the dashboard for family members.
   - *Tier 2 (Offline Emergency Recovery Key):* During registration, users receive a 24-character alphanumeric key (e.g. `ARK-7K9F-W2M4-9P8X-3T6Q-B5N1`) stored as an Argon2id hash. Entering this key on the login page resets the password, generates a replacement recovery key, and invalidates all existing sessions.
   - *Tier 3 (Host CLI Override):* Console command `arkive admin reset-password --username <user>` resolves complete Admin lockouts directly on the host.
3. **First-Run Bootstrap Wizard with Permanent Lock**:
   - Application startup checks if the database contains zero users and caches `isBootstrapped = false` in memory.
   - Unbootstrapped instances redirect web traffic to `/setup` and return HTTP 503 on API calls.
   - Wizard collects primary Admin credentials and displays the master emergency recovery key.
   - Upon submission, `setup_completed` is set in system settings and `isBootstrapped` becomes true, permanently deactivating `/setup` (returns HTTP 404/403).
4. **30-Day Sliding Sessions with Instant Valkey Revocation**:
   - Issue 32-byte cryptographically secure random session tokens in `HttpOnly`, `SameSite=Lax` cookies (`arkive_session`) with auto-detected TLS flags (`SECURE_COOKIES=auto`).
   - Sessions persist in Valkey (`session:<token>`) with a 30-day sliding TTL and a 24-hour write throttle on sliding expiration updates.
   - Maintain active session index sets in Valkey (`user_sessions:<user_id>`), allowing single device logout, remote device logout, or global kill-switches (on password changes) to execute an atomic `UNLINK` in under 1ms.
