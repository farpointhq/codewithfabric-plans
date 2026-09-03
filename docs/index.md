# CodeWithFabric Plans

Implementation plans for the [CodeWithFabric](https://github.com/farpointhq/codewithfabric) billing portal.

## Active Plans

- **[Web Chat — Thin Client for codewithfabric.com](plans/web-chat-plan.md)** — logged-in users
  land on chat, not the dashboard: a Tailwind rebuild of the Fabric desktop chat surface behind
  two transports (hosted chat with a client-side tool loop, and remote desktop over the existing
  relay). Three phases: fix relay presence (#3803), hosted chat + landing, remote desktop.

- **[Reporting: Forge Coverage & Identity](plans/forge-namespaces-and-identity-plan.md)** — why
  Matt Kesby's team reads as 22-of-24 developers shipping nothing: a GitHub token that sees 7
  repos, 378 Bitbucket PRs fenced out of Impact for lack of churn data, and 13 contributors
  stored as raw Atlassian UUIDs. Plus the mapping UX that let 26 of 50 people go unmapped.

- **[Enterprise Account Consolidation](plans/enterprise-account-consolidation-plan.md)** — enforce
  the ownership invariant (only Owners and deputized Admins own teams; Members own nothing),
  retire the 38 stray solo Enterprise teams the provisioning page exposed, and land at exactly
  three Enterprise accounts: Farpoint, Matt Kesby, Amy.
- **[Homepage Product Video](plans/homepage-product-video-plan.md)** — put real product footage
  from the Fabric docs video set inline in the marketing sections, and remove the redundant
  "Meet the Future of Coding" section from the EN homepage.
  [▶ interactive mockup](plans/homepage-video-mockup/)
