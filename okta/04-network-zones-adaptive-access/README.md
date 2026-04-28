# Project 4: Network Zones and Adaptive Access

Context-aware authentication using Okta Network Zones to apply
location-based controls — denying access from sanctioned countries,
relaxing friction inside trusted networks, and enforcing strict MFA
everywhere else.

## Problem Statement

Authentication policy that treats every sign-in identically is the
enterprise security equivalent of a one-size-fits-all medical
prescription. A user signing in from a known corporate office on a
managed device presents fundamentally different risk than the same
user signing in from a sanctioned country through an anonymizer. A
mature identity program adapts to that risk signal rather than
applying uniform friction.

This project implements that adaptation in Okta. The user's location
becomes a first-class condition in policy decisions, producing three
distinct authentication outcomes from a single policy: denial,
reduced-friction allow, and full-friction allow. The architectural
pattern is adaptive access, and it is the operational implementation
of Zero Trust thinking applied to the authentication layer.

## What I Built

Two custom Network Zones plus three rules layered onto the existing
"Require MFA for All Users" policy from Project 2:

- **Trusted Corporate Network** (IP Zone) — a static IP zone scoped
  to a trusted gateway address, representing the corporate office or
  trusted egress point
- **Blocked High-Risk Locations** (Dynamic Zone) — a country-scoped
  dynamic zone covering OFAC-sanctioned jurisdictions (Cuba, Iran,
  North Korea)
- **Three-rule policy logic** that evaluates location first, then
  applies progressively stricter MFA controls based on the location
  signal

## Configuration Details

### Network Zones

| Zone | Type | Configuration | Purpose |
|---|---|---|---|
| Trusted Corporate Network | IP Zone | Single gateway IP (/32 CIDR) | Identifies sign-ins from trusted location |
| Blocked High-Risk Locations | Dynamic Zone | Cuba, Iran (Islamic Republic of), Korea (DPRK) | OFAC-aligned geographic blocking |

### Policy Rule Configuration (top-down evaluation order)

**Rule 1 — Block High-Risk Locations** (Priority 1)
| Setting | Value |
|---|---|
| Location | In zone — Blocked High-Risk Locations |
| Access | Denied |
| Status | Active |

**Rule 2 — Trusted Network — Reduced Friction** (Priority 2)
| Setting | Value |
|---|---|
| Location | In zone — Trusted Corporate Network |
| Access | Allowed |
| MFA | Required |
| MFA prompt frequency | After MFA lifetime expires (per device cookie) |
| MFA lifetime | 2 hours |
| Maximum global session lifetime | 1 day |
| Session lifetime | 12 hours |
| Cookies persist across browser sessions | No |

**Rule 3 — Require MFA on Sign-In** (Priority 3)
| Setting | Value |
|---|---|
| Location | NOT in zone — Trusted Corporate Network |
| Access | Allowed |
| MFA | Required |
| MFA prompt frequency | At every sign-in |
| Session lifetime | 2 hours |
| Cookies persist across browser sessions | No |

The priority order matters absolutely. The Deny rule must evaluate
first; otherwise an Allow rule could match a user from a blocked
location before the deny logic has a chance to fire. The Trusted
Network rule must evaluate before the catch-all rule for the inverse
reason — trusted-network users are a subset of "everyone," so the
specific rule needs to win before the general one catches them.

## Screenshots

### Network Zones inventory
![Networks page showing custom and built-in zones](screenshots/01-network-zones-list.png)

### Blocked High-Risk Locations dynamic zone
![Dynamic zone configuration showing OFAC-aligned country selections](screenshots/02-blocked-zone-dynamic.png)

### Three-rule policy stack with priority ordering
![Policy detail showing Deny / Trusted Allow / Catch-all Allow rules](screenshots/03-policy-rule-stack.png)

## Business Value

**Security teams** care because adaptive access is the operational
implementation of Zero Trust. The principle — never trust, always
verify, but proportional to context — only becomes a real control
when location, device, and behavior signals actually drive policy
decisions. Static "MFA for everyone" policies produce friction
without proportional risk reduction; adaptive policies match control
strength to threat profile.

**IT operations teams** care because adaptive access reduces the
single largest source of identity-related help desk tickets:
authentication friction. Trusted-network users authenticating once
per two hours instead of every sign-in is a measurable productivity
gain at scale. Blocked-zone users never reach the help desk because
they never reach authentication.

**Compliance teams** care because geographic access controls are
increasingly regulated. OFAC sanctions enforcement requires
organizations to prevent transactions with persons or entities in
sanctioned jurisdictions, including authentication into systems
containing US-person data. Healthcare data residency requirements
under HIPAA and state privacy laws (Texas, California) similarly
require demonstrable controls over where PHI can be accessed from.
A dynamic zone with country-scoped denial provides defensible
evidence that those controls exist and are enforced.

## Exam Domain Mapping

**Okta Certified Professional**
- Security Enforcement: Network zones, sign-on policy rules, policy
  conditions including IP-based location
- Administration and Troubleshooting: System Log captures
  policy.evaluate_sign_on events including zone match information

## Lessons Learned

- IP Zones and Dynamic Zones are distinct zone types in Okta; IP
  Zones cover static IP ranges only, while Dynamic Zones support
  country, ASN, and proxy/anonymizer conditions. Selecting the wrong
  type at zone creation results in missing the configuration fields
  needed for the intended use case.
- An empty IP Zone cannot be saved — Okta requires at least one
  Gateway IP or Trusted Proxy IP. The platform surfaces this as a
  validation error rather than allowing creation of a non-functional
  zone.
- Adding the admin's current IP to a Blocked zone (with the "Block
  access" checkbox enabled) is the most common Okta self-inflicted
  lockout scenario. The admin console proactively displays the
  current IP near the input field, but the warning text is easy to
  miss when moving quickly.
- Modifying an existing rule's Location condition from "Anywhere" to
  "NOT in zone" reframes the rule from "applies to everyone" to
  "applies to everyone except trusted-network users." The rule logic
  is unchanged; only the scope shrinks. This is the foundational
  pattern for layering adaptive access onto an existing policy
  without rewriting it.
- Rule priority within a single policy follows the same top-down
  evaluation logic as policy priority across multiple policies. Deny
  rules belong at priority 1; specific allow rules above general
  allow rules. The default policy at the bottom of any policy
  hierarchy serves as the safety net.