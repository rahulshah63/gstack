          
  Prompt Template for /orchestrate --parallel                                                                                                                  
                                                                                                                                                               
  Copy-paste this and fill in the blanks:                                                                                                                    
                                                                                                                                                               
  /orchestrate --parallel "                                                                                                                                  
  Product: [TITLE]                                                                                                                                           

  Description: [1-3 sentences on what you're building]

  Key features:
  - [Feature 1]
  - [Feature 2]
  - [Feature 3]

  Reference: [URL, repo, or screenshot if available]

  Tech stack: [e.g. Next.js + Supabase + Stripe]

  Constraints: [deadlines, must-use libraries, existing schema, etc.]
  "

  Examples

  Simple:
  /orchestrate --parallel "
  Product: InvoiceFlow

  Description: A SaaS invoicing app where teams can create, send, and track invoices with Stripe payments.

  Key features:
  - Dashboard with invoice list, status filters, totals
  - Invoice builder with line items, tax, discounts
  - Stripe checkout integration for payment links
  - PDF export and email delivery
  - Team management with roles (admin, member, viewer)

  Tech stack: Next.js 14, Supabase, Stripe, Resend
  "

  With references:
  /orchestrate --parallel "
  Product: DevPulse

  Description: Engineering metrics dashboard that pulls from GitHub and Linear to show team velocity, PR cycle time, and deployment frequency.

  Key features:
  - GitHub PR analytics (cycle time, review time, merge rate)
  - Linear sprint burndown and velocity trends
  - Team leaderboard with configurable metrics
  - Slack weekly digest

  Reference: Similar to LinearB (https://linearb.io) but self-hosted
  Tech stack: Next.js, Postgres, GitHub API, Linear API

  Constraints: Must work with GitHub Enterprise. No paid analytics dependencies.
  "

  The orchestrator will decompose this into workstreams (e.g., "Stripe integration", "Invoice builder UI", "Team management + auth"), order them into waves
  based on dependencies, show you the plan, and after you approve, fan out parallel agents — each running its own full pipeline (plan → code → review → QA →
  PR).