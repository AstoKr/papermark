# ee — emails

# `ee/emails/pause-resume-reminder` Module

The `pause-resume-reminder.tsx` module provides an email template component that notifies team administrators when a paused subscription is about to automatically resume billing. This is a customer retention and communication component within the Papermark email system.

## Purpose

When users pause their Papermark subscription, they receive a reminder email three days before the subscription automatically resumes. This gives administrators time to:

- Review upcoming billing charges
- Cancel if they no longer need the service
- Update billing information
- Contact support with concerns

## Props Interface

The component accepts the following optional props:

```typescript
interface PauseResumeReminderEmailProps {
  teamName?: string;      // Name of the team/workspace
  userName?: string;      // First name of the recipient
  resumeDate?: string;    // Formatted date when billing resumes
  plan?: string;          // Subscription plan name (e.g., "Pro")
  userRole?: string;      // Recipient's role (e.g., "Admin")
}
```

All props have sensible defaults, so the component can be rendered with minimal configuration.

## Email Structure

The email is organized into distinct sections:

```
┌─────────────────────────────────────┐
│         Papermark Logo              │
│                                     │
│   Subscription Resume Reminder       │
│                                     │
│   Hello {userName},                 │
│                                     │
│   Reminder message about            │
│   subscription resuming in 3 days   │
│                                     │
│   ┌─────────────────────────────┐   │
│   │  📅 Resume Details:        │   │
│   │    Team: {teamName}         │   │
│   │    Plan: {plan}             │   │
│   │    Resume Date: {resumeDate}│   │
│   │    Your Role: {userRole}    │   │
│   └─────────────────────────────┘   │
│                                     │
│   What happens next?                │
│   • Subscription resumes on date    │
│   • Billing restarts                 │
│   • Features restored               │
│   • Data remains unchanged          │
│                                     │
│   [ Manage Subscription Button ]    │
│                                     │
│   Closing text and support info     │
│                                     │
│   ─────────────────────────────     │
│   Footer / Legal notice             │
└─────────────────────────────────────┘
```

## Key Components

### Logo Section

Renders the Papermark logo from the marketing domain using the `Img` component from React Email. The source URL is constructed from `NEXT_PUBLIC_MARKETING_URL` environment variable with a fallback to `https://www.papermark.com`.

### Details Card

A bordered, light gray background section (`bg-[#f9fafb]`) displays key subscription information in a scannable format. This uses emoji (📅) for visual hierarchy rather than icons.

### Call-to-Action Button

A centered black button links to `/settings/billing` on the marketing domain. Styled with:
- Black background (`bg-[#000000]`)
- White text
- Rounded corners
- Padding: `px-5 py-3`

### Footer

Contains legal boilerplate explaining why the recipient received the email, with a subtle top border separator.

## Styling Approach

The component uses:

- **Tailwind CSS** via the `<Tailwind>` wrapper from React Email
- **Responsive container** with fixed 465px width centered on desktop
- **System font stack** (`font-sans`)
- **Consistent spacing** using Tailwind margin/padding utilities
- **Neutral color palette** with black text and gray secondary text (`text-[#6b7280]`, `text-[#666]`)

## Usage Example

```tsx
import PauseResumeReminderEmail from "@/ee/emails/pause-resume-reminder";

const EmailComponent = () => {
  return (
    <PauseResumeReminderEmail
      teamName="Acme Corp"
      userName="Sarah"
      resumeDate="March 15, 2024"
      plan="Pro"
      userRole="Admin"
    />
  );
};
```

## Integration Notes

This component is designed to be rendered by an email service layer (likely Resend or similar) when a scheduled job triggers the resume reminder. The component itself contains no logic for:

- Scheduling or timing
- Recipient lookup
- Email sending

It is purely a presentation component that accepts data via props and renders a styled HTML email.

## Environment Dependencies

| Variable | Purpose | Default |
|----------|---------|---------|
| `NEXT_PUBLIC_MARKETING_URL` | Base URL for logo and CTA links | `https://www.papermark.com` |

The CTA button links to `${baseUrl}/settings/billing`, so users clicking the button are taken directly to their billing management page.