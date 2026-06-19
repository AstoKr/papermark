# Finding: SAML CredentialsProvider.authorize Upserts User Row from IdP-Controlled userinfo

## Severity
MEDIUM

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N = 8.5 (cross-tenant account creation / takeover via permissive IdP; partial — requires IdP misconfig).

## Affected Code
- File: `lib/auth/auth-options.ts`
- Line: 138-178 (the `saml-idp` CredentialsProvider.authorize function), 157 (`const { email, firstName, lastName, requested } = userInfo as any;` — `requested` is destructured but **never used** to scope the upsert)
- Function: `authOptions.providers[]` (the `saml-idp` CredentialsProvider)
- Endpoint: `POST /api/auth/callback/saml-idp`

## Description
The `saml-idp` CredentialsProvider (lines 132-178 of `auth-options.ts`) exchanges a SAML `code` for an access token via BoxyHQ Jackson's `oauthController.token`, then calls `oauthController.userInfo(access_token)` to read the SAML userinfo. It then upserts a `prisma.user` keyed on `email` from the IdP. The credentials are not validated against an existing user — any email the IdP returns creates a new row (or matches an existing one).

The `requested` field on line 157 is destructured from `userInfo` but **never used to scope the upsert** — the only scoping signal is `email`. Combined with `allowDangerousEmailAccountLinking: true` on the `saml` provider (F-009), an enterprise tenant whose IdP allows the user to choose their own email attribute (or whose IdP is misconfigured to return an email the attacker controls) can register/login as a Papermark user matching that email.

## Proof of Concept

### Pre-conditions
- Attacker has access to a permissive enterprise IdP (e.g. a self-registered IdP that doesn't validate `NameID` strictly, or an IdP that allows the user to set arbitrary email values in userinfo).
- The IdP returns a `userinfo.email` that the attacker controls (e.g. `attacker@evil.com` — they registered the IdP, so the IdP returns their chosen email).
- A papermark tenant with SSO connected to this IdP exists (the attacker is the IdP operator, so they configure a new papermark tenant with the IdP, or the papermark tenant connects to the attacker's IdP via standard SSO setup).

### Step-by-step exploitation

1. **Attacker authenticates against the permissive IdP** — login flow that returns `userinfo.email = "attacker@evil.com"` (the attacker's chosen email).

2. **The IdP redirects** to `https://app.papermark.com/api/auth/saml/callback?code=<jackson_issued_code>` (handled by Jackson's SAML callback).

3. **NextAuth code exchange** at `auth-options.ts:138-150`:
   ```ts
   const { access_token } = await oauthController.token({
     code: credentials.code,
     grant_type: "authorization_code",
     redirect_uri: getMainDomainUrl(),
     client_id: "dummy",
     client_secret: process.env.NEXTAUTH_SECRET!,
   });
   ```
   The Jackson `oauthController.token` validates the `code` using the same `NEXTAUTH_SECRET` (per F-001/F-004) and returns an `access_token`.

4. **Userinfo read** at line 154-156:
   ```ts
   const userInfo = await oauthController.userInfo(access_token);
   ```
   Returns `{ id, email: "attacker@evil.com", firstName, lastName, requested, ... }`. The `requested` field is the IdP's `requested` attribute (which Jackson uses for OIDC subject scoping) but papermark does not use it.

5. **User upsert** at line 162-166:
   ```ts
   const user = await prisma.user.upsert({
     where: { email },
     create: { email, name },
     update: { name: name || undefined },
   });
   ```
   - If `attacker@evil.com` is new, a new `prisma.user` row is created. The attacker is now logged in as this new user.
   - If `attacker@evil.com` already exists (e.g. someone else with the same email previously signed up), the upsert matches the existing row. The attacker is now logged in as that existing user — **full ATO**.

6. **Cross-tenant pivot** — if the papermark tenant the attacker is logging into has SSO configured with the attacker's permissive IdP, the `prisma.user` row is associated with that tenant via the `userTeam` table. The new user can be assigned ADMIN/MANAGER role on the tenant (SSO default in many setups), and from there chain into F-005/F-006/F-007 for cross-tenant folder destruction.

## Impact
- **Cross-tenant account creation**: an attacker who controls a permissive IdP can create `prisma.user` rows in papermark that are scoped to a tenant they have no legitimate relationship with.
- **Cross-tenant account takeover**: if the email the IdP returns matches an existing `prisma.user.email` (e.g. the victim signed up with personal email and the enterprise IdP is misconfigured to return the same email), the upsert silently matches the existing user.
- **No invitation check** — there's no `Invitation` table lookup. New users are created with the `userTeam` membership that the SAML flow assigns.
- **No domain scoping** — the IdP's email domain is not verified against the tenant's verified email domain.

## Evidence
```ts
// lib/auth/auth-options.ts:138-167
CredentialsProvider({
  id: "saml-idp",
  name: "IdP Login",
  credentials: { code: { type: "text" } },
  async authorize(credentials) {
    if (!credentials?.code) return null;
    try {
      const { oauthController } = await jackson();
      const { access_token } = await oauthController.token({
        code: credentials.code,
        grant_type: "authorization_code",
        redirect_uri: getMainDomainUrl(),
        client_id: "dummy",
        client_secret: process.env.NEXTAUTH_SECRET!,
      });
      if (!access_token) return null;
      const userInfo = await oauthController.userInfo(access_token);
      if (!userInfo) return null;
      const { email, firstName, lastName, requested } = userInfo as any;
      //                                          ^^^^^^^^^ requested is destructured but never used
      if (!email) return null;
      const name = [firstName, lastName].filter(Boolean).join(" ") || email;
      const user = await prisma.user.upsert({
        where: { email },           // ← keyed on email only, no domain scoping
        create: { email, name },
        update: { name: name || undefined },
      });
      return { id: user.id, email: user.email, name: user.name, profile: userInfo } as any;
    } catch (error) {
      console.error("[SAML] Error during SAML authorization:", error);
      return null;
    }
  },
}),
```

## Methodology
- **M1 — Structural analysis** (`methodology-raw/01-structural-analysis.md` Finding 8)
- **M2 — SAST** (`methodology-raw/02-synthesized-sast.md` S-006)
- **M7 — Chain synthesis** (`methodology-raw/05-chains.md` Chain 2, Chain 5)

## Chain Potential
- **Chain 5** (`methodology-raw/05-chains.md`) — the SAML ATO pivot is the unique value-add; the IDOR portion (F-005/F-006/F-007) is already public. The chain's full impact is "any attacker with an email account can destroy any tenant's folders."
- **F-001/F-004** (4-purpose NEXTAUTH_SECRET) — the Jackson OAuth `client_secret` is the same string as the SAML `clientSecret`, so a token recovery from the published placeholder immediately unlocks SAML ATO.
- **F-009** (allowDangerousEmailAccountLinking) — the SAML provider at line 130 has this flag set; the dangerous-linking amplifies the upsert's impact.

## Remediation
Add a `signIn` callback in `authOptions` that:
1. Looks up the Jackson `connections` table for the tenant that issued this SAML code.
2. Verifies the returned `email`'s domain matches the tenant's verified email domain.
3. Verifies the user has either been invited (`Invitation.email` matches) or is already a `userTeam` member.
4. Rejects with a clear error otherwise.

Also:
- **Remove `allowDangerousEmailAccountLinking: true`** from the SAML provider (line 130) and rely on the explicit `signIn` callback for tenant-bound user creation.
- **Use the destructured `requested` field** on line 157 to scope the upsert to the tenant that issued the `code`.
- **Add a `signIn` callback** in `authOptions` that maps the SAML `code` to the originating `tenant` via the Jackson `connections` table lookup.
