# Noble Trading App Website Workflow

## Website Integration Technology
- [NextJS](https://nextjs.org)
- [Vercel](https://vercel.com)
- [Clerk](https://clerk.com)
- [Supabase](https://supabase.com)
- [Helio Payments](https://hel.io)
- [Tailwind](https://tailwindcss.com)
- [ShadCN Charts](https://ui.shadcn.com/docs/components/chart)
- [OneSignal](https://onesignal.com)

## User Journery & Onboarding Steps
- Waitlist : Clerk Waitlist
- Invitation : Clerk Invitation
- Login : Clerk Authentication
- Join Discord : Set Clerk metadata to control UX display of membership plans
- Select membership plan & complete payment
- Redirect to payment success page

## Initial metadata settings
{
public_metatdata {
  "role" : "",
  "discord": "false",
  "planid": "false",
  "planstatus": "false"
  }
}

## On Discord Join : Display membership plans
{
public_metatdata {
  "role" : "",
  "discord": "true",
  "planid": "false",
  "planstatus": "false"
  }
}

## On Successful Paymnent & Renewal
{
public_metatdata {
  "role" : "",
  "discord": "true",
  "planid": "planname",
  "planstatus": "true"
  }
}

## On Expired Membership
{
public_metatdata {
  "role" : "",
  "discord": "true",
  "planid": "false",
  "planstatus": "false"
  }
}

