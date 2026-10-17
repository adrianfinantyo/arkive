# User Onboarding, Invitations & Account Recovery

Type: grilling
Status: open
Blocked by: 05

## Question

How should Arkive onboard non-technical family and community users and handle password recovery without requiring external SMTP email servers or complicated verification infrastructure?

Specific decisions to resolve with the user:
1. **Invitation Mechanism:** How should Admins invite new users? (e.g. single-use or multi-use invitation links with pre-assigned roles like Uploader or Viewer and pre-selected Groups)?
2. **Password Recovery without SMTP:** In self-hosted setups without mail servers, how should users recover lost passwords? (e.g. offline emergency backup recovery codes generated at account creation vs Admin-generated one-time reset links)?
3. **First-Run Instance Setup:** How should a newly deployed Arkive instance bootstrap its first administrator account securely?
4. **Session Persistence:** What should the default session duration be for remembered devices on the SvelteKit PWA, and how are revoked sessions invalidated across Valkey?
