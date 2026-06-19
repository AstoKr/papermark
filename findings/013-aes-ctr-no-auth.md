# Finding: AES-256-CTR Without Integrity on Document Passwords (Password Forgery)

## Severity
HIGH

## CVSS Estimate
CVSS:3.1/AV:N/AC:L/PR:H/UI:N/S:U/C:H/I:H/A:N = 6.8 (requires prior DB write access; the lack of an auth tag turns a passive DB read into an active forgery for any attacker who can write to the `Link.password` column).

## Affected Code
- File: `lib/utils.ts`
- Line: 595-622 (`generateEncrpytedPassword` — uses `crypto.createCipheriv("aes-256-ctr", ...)`)
- Line: 624-652 (`decryptEncrpytedPassword` — uses `crypto.createDecipheriv("aes-256-ctr", ...)`)
- Function: `generateEncrpytedPassword`, `decryptEncrpytedPassword`
- Endpoint: n/a — used by all link creation, update, and verification paths

## Description
Document passwords are encrypted with `crypto.createCipheriv("aes-256-ctr", …)` (line 618). CTR mode is a stream cipher and produces no authentication tag; `decryptEncrpytedPassword` does not call `setAuthTag()` because none exists. An attacker who can write to the `Link.password` column can **XOR-flip arbitrary bits of the plaintext** and the server will accept the forged password on next view. This breaks password-gated link-document access.

There is also a **secondary defect**: the AES key is derived as `sha256(secret).digest("base64").substring(0, 32)` (line 612-616), which produces 32 ASCII characters (not 32 bytes). This reduces the effective key space and embeds base64 charset bias (only `[A-Za-z0-9+/]` characters).

## Proof of Concept

### Pre-conditions
- The attacker has read+write access to the `Link.password` column (e.g. via backup leak, SQL injection elsewhere, admin-token compromise, or the chain via F-008 — the AES-CTR password-forgery link requires a separate DB write primitive that is not provided by Chain 7 in isolation, per the validation issues in `methodology-raw/06-validated.md`).

### Step-by-step exploitation

1. **Obtain the encrypted password row** for a target link. The row format is `IV_HEX:CIPHERTEXT_HEX` (32 hex chars for the IV, colon, then hex-encoded ciphertext). The IV is randomly generated per encryption (line 617).

2. **Forge a password** by XOR-flipping the ciphertext. The CTR mode's key stream is determined by `(key, IV)`. If the attacker knows the original plaintext `P_orig` and wants to forge plaintext `P_forged`, they compute:
   ```js
   const P_orig = "originalPassword123";
   const P_forged = "forgedPass!";
   const C = "<original_ciphertext_hex>";  // from the row
   // CTR mode: ciphertext bytes = plaintext bytes XOR keystream
   // So: ciphertext_for_forged = ciphertext_orig XOR (P_orig XOR P_forged)
   const buf_orig = Buffer.from(P_orig, "utf8");
   const buf_forged = Buffer.from(P_forged, "utf8");
   const buf_cipher = Buffer.from(C, "hex");
   const buf_result = Buffer.alloc(buf_cipher.length);
   for (let i = 0; i < buf_cipher.length; i++) {
     buf_result[i] = buf_cipher[i] ^ buf_orig[i] ^ buf_forged[i];
   }
   const new_ciphertext = buf_result.toString("hex");
   ```

3. **Write the forged row** back to the database:
   ```sql
   UPDATE "Link" SET password = '<IV_HEX>:<new_ciphertext>' WHERE id = '<target_link_id>';
   ```

4. **Submit the forged password** on the link's view page:
   ```
   https://app.papermark.com/view/<linkId>?password=forgedPass!
   ```

5. **Server-side verification** at `decryptEncrpytedPassword` (line 624-652) returns the forged plaintext. The handler's `verifyPassword(link.password, userProvidedPassword)` matches. The victim sees the document.

6. **Bonus — recover the AES key** by exploiting the secondary defect: `sha256(secret).digest("base64").substring(0, 32)` produces a 32-ASCII-character key with only base64 charset. The effective key space is `64^32` instead of `256^32` (32 ASCII characters chosen from 64 base64 characters), which is `2^192` instead of `2^256`. This is still 2^192 (a lot), but the **bias** in the character distribution (some characters appear more often) is the real issue — for a known-plaintext attack (e.g. the attacker knows the original password was empty), the keystream is revealed byte-by-byte. With multiple known-plaintext pairs, the attacker can reconstruct the key and forge arbitrary ciphertexts without ever writing to the DB.

## Impact
- **Password forgery** for any link with `enablePassword = true` — the attacker can submit any password they choose for any link.
- **Defense-in-depth gap** — the lack of an auth tag means any DB write access becomes a forgery primitive. The Slack integration path (`lib/integrations/slack/utils.ts:41-74`) uses AES-256-GCM with a 12-byte IV and stores the auth tag — the correct pattern to mirror.
- **No immediate exploit** without a separate DB write primitive. The chain via F-008 (timing-attack + AES-CTR) requires DB write access that the chain does not establish independently (per validation issue 4 in `methodology-raw/06-validated.md`).

## Evidence
```ts
// lib/utils.ts:595-622 (encrypt)
export async function generateEncrpytedPassword(
  password: string,
): Promise<string> {
  if (!password) return "";
  // ... format check ...
  const encryptedKey: string = crypto
    .createHash("sha256")
    .update(String(process.env.NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY))
    .digest("base64")
    .substring(0, 32);  // ← 32 ASCII chars, not 32 bytes
  const IV: Buffer = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv("aes-256-ctr", encryptedKey, IV);  // ← CTR mode, no auth tag
  let encryptedText: string = cipher.update(password, "utf8", "hex");
  encryptedText += cipher.final("hex");
  return IV.toString("hex") + ":" + encryptedText;
}
```
```ts
// lib/utils.ts:624-652 (decrypt)
export function decryptEncrpytedPassword(password: string): string {
  if (!password) return "";
  const encryptedKey: string = crypto
    .createHash("sha256")
    .update(String(process.env.NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY))
    .digest("base64")
    .substring(0, 32);
  // ... format check ...
  try {
    const IV: Buffer = Buffer.from(textParts[0], "hex");
    const encryptedText: string = textParts[1];
    const decipher = crypto.createDecipheriv("aes-256-ctr", encryptedKey, IV);
    let decrypted: string = decipher.update(encryptedText, "hex", "utf8");
    decrypted += decipher.final("utf8");
    return decrypted;
  } catch (error) {
    return password;
  }
}
```
There is no `setAuthTag()` call. There is no `getAuthTag()` call. CTR mode is a stream cipher with no integrity.

## Methodology
- **M2 — SAST** (`methodology-raw/02-synthesized-sast.md` S-001)
- **M1 — Reachability** (`methodology-raw/01-reachability-analysis.md` F1: 7 entry points confirmed)
- **M7 — Chain synthesis** (`methodology-raw/05-chains.md` Chain 7 — weakened; see validation issue 4)

## Chain Potential
- **Chain 7** (`methodology-raw/05-chains.md`, validation issue 4) — the AES-CTR password-forgery link is **weakened** because it requires DB write access that the chain does not establish independently. The strong chain is timing-attack → 26-route blast radius. AES-CTR is a related defense-in-depth concern.
- **F-001/F-004** (4-purpose NEXTAUTH_SECRET) — the AES key is `NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY`, not `NEXTAUTH_SECRET`, so the two are independent. But the published placeholder `my-superstrong-document-secret` (line 68 of `.env.example`) is also a known string, so a deployment that didn't rotate it has a known AES key.

## Remediation
1. **Migrate to AES-256-GCM** with `getAuthTag()` persisted alongside the ciphertext:
   ```ts
   const IV: Buffer = crypto.randomBytes(12);  // GCM standard IV size
   const cipher = crypto.createCipheriv("aes-256-gcm", encryptedKey, IV);
   let encryptedText: string = cipher.update(password, "utf8", "hex");
   encryptedText += cipher.final("hex");
   const authTag: Buffer = cipher.getAuthTag();
   return IV.toString("hex") + ":" + authTag.toString("hex") + ":" + encryptedText;
   ```
   On decryption, call `setAuthTag()` and surface errors as "decryption failed":
   ```ts
   const decipher = crypto.createDecipheriv("aes-256-gcm", encryptedKey, IV);
   decipher.setAuthTag(authTag);
   let decrypted: string = decipher.update(encryptedText, "hex", "utf8");
   decrypted += decipher.final("utf8");
   ```

2. **Use the full 32-byte SHA-256 digest** as the key (not the 32-ASCII-character `substring(0, 32)` of the base64 encoding):
   ```ts
   const encryptedKey: Buffer = crypto
     .createHash("sha256")
     .update(String(process.env.NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY))
     .digest();  // 32 raw bytes
   ```

3. **Rotate the `NEXT_PRIVATE_DOCUMENT_PASSWORD_KEY`** for any deployment that has used the placeholder `my-superstrong-document-secret` (line 68 of `.env.example`). Existing encrypted passwords will need to be re-encrypted under the new key during the rollover.
