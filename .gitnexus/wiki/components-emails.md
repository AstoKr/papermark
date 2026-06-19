# components — emails

# Email Components Module

The `components/emails` module provides a library of React email templates for Papermark. All emails use the [react-email](https://react.email/) library for rendering and [Tailwind CSS](https://tailwindcss.com/) for styling.

## Module Structure

```
components/emails/
├── shared/
│   └── footer.tsx          # Reusable footer component
├── custom-domain-setup.tsx
├── data-rooms-information.tsx
├── dataroom-digest-notification.tsx
├── dataroom-notification.tsx
├── dataroom-trial-24h.tsx
├── dataroom-trial-end.tsx
├── dataroom-trial-welcome.tsx
├── dataroom-upload-notification.tsx
├── deleted-domain.tsx
├── download-ready.tsx
├── email-updated.tsx
├── export-ready.tsx
├── hundred-views-congrats.tsx
├── installed-integration-notification.tsx
├── invalid-domain.tsx
├── onboarding-1.tsx through onboarding-4.tsx
├── otp-verification.tsx
├── slack-integration.tsx
├── team-invitation.tsx
├── thousand-views-congrats.tsx
├── upgrade-one-month-checkin.tsx
├── upgrade-personal-welcome.tsx
├── upgrade-plan.tsx
├── upgrade-six-month-checkin.tsx
├── verification-email-change.tsx
├── verification-link.tsx
├── viewed-dataroom-paused.tsx
├── viewed-dataroom.tsx
├── viewed-document-paused.tsx
├── viewed-document.tsx
├── welcome.tsx
└── year-in-review-papermark.tsx
```

## Email Categories

### Notification Emails

Emails that alert users to activity on their content:

| Component | Purpose | Key Props |
|-----------|---------|-----------|
| `ViewedDocument` | Notifies when someone views a document | `documentId`, `documentName`, `linkName`, `viewerEmail`, `locationString` |
| `ViewedDocumentPausedEmail` | Notifies about views when subscription is paused | `documentName`, `linkName` |
| `ViewedDataroom` | Notifies when someone views a dataroom | `dataroomId`, `dataroomName`, `linkName`, `viewerEmail`, `locationString` |
| `ViewedDataroomPausedEmail` | Notifies about dataroom views when subscription is paused | `dataroomName`, `linkName` |
| `DataroomNotification` | Single document added to dataroom | `dataroomName`, `documentName`, `senderEmail`, `url`, `unsubscribeUrl` |
| `DataroomDigestNotification` | Batch notification of document changes | `dataroomName`, `documents[]`, `frequency`, `preferencesUrl` |
| `DataroomUploadNotification` | Notifies when visitor uploads files via link | `dataroomId`, `dataroomName`, `uploaderEmail`, `documentNames[]`, `linkName` |

### Download & Export Emails

| Component | Purpose | Key Props |
|-----------|---------|-----------|
| `DownloadReady` | Notifies when dataroom download is prepared | `dataroomName`, `downloadUrl`, `expiresAt`, `isViewer` |
| `ExportReady` | Notifies when export is available | `resourceName`, `downloadUrl`, `email` |

### Onboarding Sequence

A four-email sequence for new users:

1. **`onboarding-1.tsx`** — Upload your first document
2. **`onboarding-2.tsx`** — Set link permissions
3. **`onboarding-3.tsx`** — Watch views come in real-time
4. **`onboarding-4.tsx`** — Custom domains and branding

### Welcome Emails

| Component | Purpose | Key Props |
|-----------|---------|-----------|
| `WelcomeEmail` | Welcome new users with getting started guide | `name` |
| `DataroomTrialWelcomeEmail` | Personal welcome from founder for dataroom trial | `name` |

### Trial Management

| Component | Purpose | Trigger |
|-----------|---------|---------|
| `DataroomTrialWelcomeEmail` | Welcome to trial | Trial created |
| `DataroomTrial24hReminderEmail` | 24 hours until trial ends | 24h before expiry |
| `DataroomTrialEnd` | Trial has expired | Trial expired |

### Plan Upgrade Emails

| Component | Purpose | Key Props |
|-----------|---------|-----------|
| `UpgradePlanEmail` | Confirmation after upgrading | `name`, `planType` |
| `UpgradePersonalEmail` | Personal welcome from co-founder | `name`, `planName` |
| `UpgradeOneMonthCheckin` | 1-month follow-up | `name` |
| `UpgradeSixMonthCheckin` | 6-month milestone | `name`, `planName` |

### Domain Management

| Component | Purpose | Trigger |
|-----------|---------|---------|
| `CustomDomainSetup` | Guide for custom domain setup | User action or upgrade |
| `InvalidDomain` | Domain not configured after X days | 14+ days unconfigured |
| `DeletedDomain` | Domain deleted after 30 days | 30 days unconfigured |

### Verification Emails

| Component | Purpose | Key Props |
|-----------|---------|-----------|
| `VerificationCodeEmail` | One-time login code | Magic link login |
| `OtpEmailVerification` | OTP for document/dataroom access | Protected content access |
| `EmailUpdated` | Email address change confirmation | Email changed |
| `ConfirmEmailChange` | Confirm new email address | Email change request |

### Team & Collaboration

| Component | Purpose | Key Props |
|-----------|---------|-----------|
| `TeamInvitation` | Invite user to team | `senderName`, `teamName`, `url` |

### Integration Emails

| Component | Purpose | Key Props |
|-----------|---------|-----------|
| `SlackIntegrationEmail` | Promote Slack integration | Onboarding sequence |
| `InstalledIntegrationNotification` | Integration installed | `team`, `integration` |

### Data Rooms Information

| Component | Purpose | Key Props |
|-----------|---------|-----------|
| `DataRoomsInformationEmail` | Promote data rooms features | Onboarding sequence |

### Milestone & Special Emails

| Component | Purpose |
|-----------|---------|
| `HundredViewsCongratsEmail` | Congratulations at 100 views |
| `ThousandViewsCongratsEmail` | Congratulations at 1000 views |
| `YearInReviewPapermarkEmail` | Annual usage statistics |

## Shared Components

### Footer Component

The `Footer` component (`components/emails/shared/footer.tsx`) provides consistent footer styling across emails.

```tsx
import { Footer } from "./shared/footer";

// Standard footer (most emails)
<Footer />

// With company address
<Footer withAddress={true} />

// Marketing footer variant (for WelcomeEmail)
<Footer marketing />

// Custom footer text
<Footer footerText="Custom message here" />

// React node for footer text
<Footer footerText={<><strong>Bold text</strong> in footer</>} />
```

**Props:**

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `withAddress` | `boolean` | `false` | Include company address (1111B S Governors Ave #28117, Dover, DE 19904) |
| `marketing` | `boolean` | `false` | Use marketing footer variant with unsubscribe link |
| `footerText` | `string \| ReactNode` | varies | Custom footer message text |

## Email Design Patterns

### Container Structure

All full-featured emails follow this structure:

```tsx
<Html>
  <Head />
  <Preview>{previewText}</Preview>
  <Tailwind>
    <Body>
      <Container className="mx-auto my-10 w-[465px] p-5">
        {/* Header */}
        <Text className="text-center text-2xl">
          <span className="font-bold tracking-tighter">Papermark</span>
        </Text>
        
        {/* Title */}
        <Text className="text-center text-xl">Email Title</Text>
        
        {/* Content */}
        <Text className="text-sm leading-6">Body content</Text>
        
        {/* CTA Button */}
        <Section className="my-8 text-center">
          <Button href="...">Call to Action</Button>
        </Section>
        
        {/* Footer */}
        <Footer />
      </Container>
    </Body>
  </Tailwind>
</Html>
```

### Minimal Emails

Some emails (milestone emails, check-ins) use a minimal design without containers:

```tsx
<Html>
  <Head />
  <Preview>{previewText}</Preview>
  <Tailwind>
    <Body className="font-sans text-sm">
      <Text>Hi{name && ` ${name}`},</Text>
      <Text>Email body...</Text>
    </Body>
  </Tailwind>
</Html>
```

### Dynamic Content Helpers

**`formatExpirationTime`** in `download-ready.tsx`:

```tsx
function formatExpirationTime(expiresAt?: string): string {
  // Calculates human-readable time remaining
  // Returns: "3 days", "5 hours", "less than an hour"
}
```

**`getPlanInfo`** in `custom-domain-setup.tsx`:

```tsx
const getPlanInfo = () => {
  if (hasAccess) {
    return {
      title: "Your custom domain is ready to set up! 🎉",
      subtitle: `Great news! Your ${currentPlan} plan includes custom domain access.`,
    };
  }
  return {
    title: "Interested in custom domains? 🚀",
    subtitle: "Learn how custom domains can enhance your document sharing experience.",
  };
};
```

## Usage Example

Import and render an email template:

```tsx
import { render } from "@react-email/components";
import ViewedDocument from "@/components/emails/viewed-document";
import DownloadReady from "@/components/emails/download-ready";

// Render email to HTML string
const html = await render(
  <ViewedDocument
    documentId="doc_123"
    documentName="Q4 Pitch Deck"
    linkName="Investor Link"
    viewerEmail="investor@acme.com"
    locationString="San Francisco, CA"
  />
);

// With Next.js API route for sending
export async function POST(req: Request) {
  const html = await render(
    <DownloadReady
      dataroomName="M&A Due Diligence"
      downloadUrl="https://app.papermark.com/downloads/abc123"
      email="buyer@acme.com"
      expiresAt="2024-12-31T23:59:59Z"
      isViewer={true}
    />
  );
  
  // Send via email provider (Resend, SendGrid, etc.)
  await sendEmail({
    to: "buyer@acme.com",
    subject: "Your download is ready",
    html,
  });
}
```

## Styling Conventions

- **Container width**: 465px for standard emails, 600px max for emails with tables
- **Font**: System font stack (`font-sans`)
- **Colors**: Black text on white background, gray for secondary text
- **Buttons**: Black background, white text, rounded corners, `12px 20px` padding
- **Links**: Blue-600 color with no underline (except in specific contexts)

## External Links Reference

Emails link to these external destinations:

| Destination | URL Pattern |
|-------------|-------------|
| App Dashboard | `https://app.papermark.com/dashboard` |
| Documents | `https://app.papermark.com/documents` |
| Datarooms | `https://app.papermark.com/datarooms` |
| Settings | `https://app.papermark.com/settings/*` |
| Billing | `https://app.papermark.com/settings/billing` |
| Domains | `https://app.papermark.com/settings/domains` |
| Integrations | `https://app.papermark.com/settings/integrations` |
| Documentation | `https://docs.papermark.com/*` |
| Marketing | `https://www.papermark.com/*` |
| Calendar Booking | `https://cal.com/marcseitz/papermark` |
| G2 Reviews | `https://www.g2.com/products/papermark/reviews` |