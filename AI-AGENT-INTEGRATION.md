# AI-agent booking support

This repository contains the existing human-facing booking page at `booking.html`. It remains the primary accessible booking experience and continues to use the same Vercel backend, Google Calendar and Stripe Checkout flow.

The AI-agent API is implemented in the companion repository `alinaszaputko-psych/booking-backend`. Agents should discover the public profile and OpenAPI specification there rather than scraping this page or attempting to automate its form.

- Professional profile: https://booking-backend-kappa-drab.vercel.app/api/agent-profile
- Available slots: https://booking-backend-kappa-drab.vercel.app/api/agent-availability
- OpenAPI: https://booking-backend-kappa-drab.vercel.app/openapi/agent-booking.yaml
- Client-authorised checkout: https://booking-backend-kappa-drab.vercel.app/api/agent-checkout

The agent flow requires the client's explicit consent and sends the client to Stripe Checkout. The agent must not collect card details or clinical history. Payment and successful calendar fulfilment are required before an appointment can be described as confirmed.

See the backend repository's `AI-AGENT-INTEGRATION.md` for the complete flow and deployment checklist.
