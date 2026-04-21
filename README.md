# Vibe Coder Learnings

Hard-won lessons from building production apps with Supabase, Google Auth, and LLMs. Written for developers who move fast and want to avoid the potholes we already hit.

## What's in here

### [lessons-learned.md](./lessons-learned.md)
A distilled reference of ~35 lessons across authentication, Supabase, edge functions, database design, client security, and AI agents. Each point is classified by severity (critical security risk → reliability issue) so you can triage what to prioritize.

### [supabase-google-auth-setup-guide.md](./supabase-google-auth-setup-guide.md)
A step-by-step setup guide for:
- Supabase project setup with tables, RLS, and edge functions
- Google OAuth for web apps
- Google OAuth for desktop apps (Electron) — including the local callback server, CSRF hardening, and proactive token refresh
- Connecting Google tokens to Supabase sessions
- Edge function security patterns

Both files are written to be practical and copy-paste friendly. The guide references the lessons file for deeper context on each decision.

## Context

These learnings come from building [Overmind](https://github.com/linhnguyen/Overmind) — a production Electron + Supabase + LLM app — and a Slack AI agent built on top of it. The security points were validated through a formal security audit; the reliability points were each discovered by breaking something in production.

## Contributing

Found a pattern we missed? Open a PR. The bar is: would this have saved you at least an hour?
