# Finding: allowDangerousEmailAccountLinking on Google, LinkedIn, and SAML

## Severity
HIGH

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N = 9.1 (full account takeover for any registered user via OAuth account linking).

## Affected Code
- File: `lib/auth/auth-options.ts`
- Line: 32-36 (Google provider), 37-56 (LinkedIn provider), 96-131 (SAML provider, line 130: `allowDangerousEmailAccountLinking: true`)
- Function: `authOptions.providers[]`
- Endpoint: `/api/auth/signin/google`, `/api/auth/signin/linkedin`, `/api/auth/signin/saml`

## Description
NextAuth's `allowDangerousEmailAccountLinking: true` is a documented footgun: if the provider returns a profile with the same email as an existing user, NextAuth will log the OAuth caller in as the existing user. Google and LinkedIn are public providers where any attacker can register an account at any email (including the victim's). SAML has the same risk but is sometimes intentional for IdPs without email verification.

There is **no `signIn` callback override** in `authOptions` (the callbacks object at line 195-239 only contains `jwt` and `session`) — NextAuth's default behavior of "auto-link on matching email" is in effect.

## Proof of Concept

### Pre-conditions
- The victim's email is registered in `prisma.user` (any papermark user). The email may be public knowledge (the victim is an active team member; their email is in the dashboard's `viewers` API response for any team they belong to).

### Step-by-step exploitation (Google provider, simplest)

1. **Attacker registers a Google account** at `attacker+corp.com@gmail.com` (or, more cleverly, at a personal Google account with the email `victim@corp.com` if the victim's domain is not Google Workspace-verified).

2. **Attacker navigates to** `https://app.papermark.com/api/auth/signin/google` and completes Google's consent screen using the attacker's Google identity.

3. **Google redirects** to NextAuth's callback with a profile `{ id, email: "victim@corp.com", verified_email: true, ... }`.

4. **NextAuth's internal signIn handler** finds the existing `prisma.user` row keyed on `email` (line 162-166 of the SAML-idp `CredentialsProvider.authorize` shows the upsert pattern, but the same internal logic runs in NextAuth core for the Google provider when `allowDangerousEmailAccountLinking: true` is set). NextAuth creates a session JWT for the existing user.

5. **Attacker is logged in as the victim.** All `pages/api/teams/[teamId]/**` handlers that gate on `getServerSession + userTeam` membership treat the attacker as the victim.

### Step-by-step exploitation (LinkedIn provider)

1. **Attacker creates a LinkedIn profile** with `email: "victim@corp.com"` (LinkedIn allows unverified emails; the `profile.email` field returned to the `LinkedInProvider.profile` callback at line 45-54 is whatever the LinkedIn account has set).

2. **Attacker navigates to** `https://app.papermark.com/api/auth/signin/linkedin` and completes LinkedIn's OAuth flow.

3. **NextAuth's profile callback** at line 45-54 returns `{ id: profile.sub, name, email: profile.email, image }` (the email is whatever LinkedIn returned — unverified).

4. **Same as Google: NextAuth's internal signIn handler** finds the existing `prisma.user` and creates a session for the victim.

### Step-by-step exploitation (SAML provider, Chain 5 alternative)

1. **Attacker has a permissive enterprise IdP** that returns `userinfo.email = "victim@corp.com"`.

2. **Attacker initiates SSO** at `https://app.papermark.com/api/auth/signin/saml`. The SAML flow goes through BoxyHQ Jackson, which issues a `code`.

3. **NextAuth's `saml` provider** at line 96-131 has `allowDangerousEmailAccountLinking: true` at line 130. The profile callback at line 114-125 returns the IdP-controlled `email` as the user's identifier.

4. **NextAuth's internal signIn handler** matches the email to the existing `prisma.user` row and creates a session for the victim.

## Impact
- **Full account takeover** for any registered papermark user.
- **No special prerequisites** beyond a victim email and a Google/LinkedIn account the attacker controls.
- **Cross-tenant IDOR pivot**: once logged in as the victim, the attacker can use the victim team memberships to delete folders in OTHER teams via F-005/F-006/F-007, or to read confidential data via the dashboard.
- **Audit trail noise**: the `accounts` table records a Google/LinkedIn/SAML account linked to the victim's `userId`; the audit log records the attacker's session as a "Sign In" event for the victim.

## Evidence
```ts
// lib/auth/auth-options.ts:32-36 (Google)
GoogleProvider({
  clientId: process.env.GOOGLE_CLIENT_ID as string,
  clientSecret: process.env.GOOGLE_CLIENT_SECRET as string,
  allowDangerousEmailAccountLinking: true,
}),
```
```ts
// lib/auth/auth-options.ts:37-56 (LinkedIn)
LinkedInProvider({
  clientId: process.env.LINKEDIN_CLIENT_ID as string,
  clientSecret: process.env.LINKEDIN_CLIENT_SECRET as string,
  authorization: { params: { scope: "openid profile email" } },
  issuer: "https://www.linkedin.com/oauth",
  jwks_endpoint: "https://www.linkedin.com/oauth/openid/jwks",
  profile(profile, tokens) {
    const defaultImage = "https://cdn-icons-png.flaticon.com/512/174/174857.png";
    return {
      id: profile.sub,
      name: profile.name,
      email: profile.email,    // ← unverified email from LinkedIn
      image: profile.picture ?? defaultImage,
    };
  },
  allowDangerousEmailAccountLinking: true,
}),
```
```ts
// lib/auth/auth-options.ts:96-131 (SAML)
{
  id: "saml",
  name: "BoxyHQ SAML",
  type: "oauth",
  ...
  profile: async (profile) => ({
    id: profile.id || profile.email,
    name: ...,
    email: profile.email,    // ← IdP-controlled email
    image: null,
  }),
  options: { clientId: "dummy", clientSecret: process.env.NEXTAUTH_SECRET as string },
  allowDangerousEmailAccountLinking: true,
},
```

## Methodology
- **M1 — Structural analysis** (`methodology-raw/01-structural-analysis.md` Finding 4)
- **M2 — Web synthesis** (`methodology-raw/00-early-web-intel.md` Directive D6 — NextAuth documented footgun)
- **M7 — Chain synthesis** (`methodology-raw/05-chains.md` Chain 2)

## Chain Potential
- **Chain 2** (`methodology-raw/05-chains.md`) — chains with F-005/F-006/F-007 (folder IDOR cluster) for cross-tenant destruction after ATO.
- **Chain 5** (`methodology-raw/05-chains.md`) — chains with F-013 (SAML upsert) for ATO via permissive IdP.
- **F-001/F-004** (4-purpose NEXTAUTH_SECRET) — the SAML provider's `clientSecret` is the same string, so a token recovery immediately unlocks SAML account linking.

## Remediation
1. **Remove `allowDangerousEmailAccountLinking: true`** from the Google and LinkedIn providers entirely. NextAuth's default behavior is to **reject** sign-in when the OAuth email matches an existing user, which is the safer default.

2. **For SAML**, keep the flag **only if every connected IdP enforces verified email** (most enterprise IdPs do — verify with each tenant). For IdPs that don't enforce verification, remove the flag and rely on an explicit `signIn` callback that:
   - Looks up the Jackson `connections` table for the tenant that issued this SAML code.
   - Verifies the returned `email`'s domain matches the tenant's verified email domain.
   - Verifies the user has either been invited (`Invitation.email` matches) or is already a `userTeam` member.
   - Rejects with a clear error otherwise.

3. **Implement a `signIn` callback** in `authOptions` that explicitly handles the linking policy. The current `callbacks` object (line 195-239) only contains `jwt` and `session` — no `signIn` override.

4. **Document the policy** in `SECURITY.md`: which providers are eligible for account linking, under what conditions, and how to disable linking for a specific provider per tenant.
