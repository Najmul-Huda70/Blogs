
# How I Solved Cross-Domain Auth in Production
 
## Overview
  
This project deploys its frontend and backend on **separate domains** — the frontend on Vercel, the backend on Render. That split is great for scaling and separation of concerns, but it turns something as simple as "log the user in" into a much harder problem: browsers are deliberately strict about sharing cookies across domains, and getting that wrong doesn't throw a loud error — it just quietly logs people out.

 ## The Challenge
 
**Getting cross-domain auth right, not just working**
 
The frontend (Vercel) and backend (Render) sit on separate domains, so Better Auth's session cookies needed `sameSite: "none"`, `secure: true`, and exact trusted origins — the same rule even in local dev. Getting this wrong doesn't error loudly; it just silently logs users out.
 
This is the kind of bug that's brutal to debug because everything *looks* fine — no stack trace, no failed request in the network tab that screams "auth broken." The session just doesn't persist, and the user gets bounced back to the login screen for no obvious reason.
 
**The fix:**
JWTs signed with `jose` on the Express backend, verified in `proxy.ts` on every protected route — one source of truth for who's logged in, on either domain.
 
By moving verification into a single proxy layer instead of scattering `req.user` checks across route handlers, there's now exactly one place that decides whether a request is authenticated — regardless of which domain it originated from.

 
## Result
 
- Sessions persist reliably across the Vercel ↔ Render boundary, in both local dev and production.
- One centralized verification point (`proxy.ts`) instead of duplicated auth checks per route.
- No more silent logouts caused by mismatched cookie attributes.
  
## Key Takeaways
 
- Cross-domain cookies need `sameSite: "none"` **and** `secure: true` together — one without the other fails silently.
- Trusted origins must match *exactly*, including in local development, not just in production config.
- Centralizing token verification in one middleware layer beats scattering auth checks across handlers — it's the single source of truth for "who is logged in."
---
 
