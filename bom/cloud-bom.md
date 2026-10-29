# Cloud System Bill of Materials (BOM)

**Project:** LoRa-Based Parking Monitoring System  
**Architecture:** Cloudflare Pages + Cloudflare Workers + Supabase  
**Objective:** Public parking availability website, private admin dashboard, cloud database, and gateway synchronization.

| No. | Component / Service | Specification | Qty. | Estimated Monthly Cost |
|---:|---|---|---:|---:|
| 1 | Frontend hosting | Cloudflare Pages — public website and admin dashboard | 1 | ₱0 |
| 2 | API hosting | Cloudflare Workers — gateway API and application endpoints | 1 | ₱0 within free-tier limits |
| 3 | Cloud database | Supabase PostgreSQL — parking lots, bays, status, and events | 1 project | ₱0 within free-tier limits |
| 4 | Authentication | Supabase Auth — private administrator login | 1 | ₱0 within free-tier limits |
| 5 | Domain name | Custom domain for the public website (optional) | 1 | ₱0 if using a provider subdomain; otherwise varies |
| 6 | HTTPS | HTTPS for website and gateway API connections | 1 | ₱0 using the proposed hosting services |
| 7 | Database migrations | SQL schema and version-controlled migration files | 1 set | ₱0 |
| 8 | Source control | GitHub repository for application code and configuration templates | 1 | ₱0 on the free plan |
| 9 | Monitoring and logs | Built-in service logs and basic health checks | 1 set | ₱0 within applicable limits |
| 10 | Automated backups | Database export or backup process | 1 | ₱0 if self-managed within available resources; storage or automation may cost extra |

## Estimated Budget

| Expense | Estimated cost |
|---|---:|
| Initial cloud setup | ₱0 service fees, excluding development labor |
| Monthly cloud services | ₱0 within free-tier limits |
| Optional custom domain | Varies by registrar and domain extension |
| Optional paid-tier upgrades | Depends on usage and availability requirements |

## Required Cloud Components

- **Public website:** Displays parking availability without requiring visitor login.
- **Admin dashboard:** Requires authenticated administrator access.
- **Cloud API:** Receives gateway updates over HTTPS and serves public parking data.
- **Database:** Stores parking lots, gateway records, bay status, and historical events.
- **Gateway authentication:** Each gateway uses its own secret or credential.
- **Access controls:** Public visitors can read only intended public data; administrative and gateway operations require appropriate authorization.

## Scope Exclusions

- Video streaming or camera storage
- Dedicated VPS or cloud server
- Paid analytics platform
- Paid email or SMS notifications
- Paid monitoring services

**Important:** ₱0/month is a target for initial development and testing, not a guaranteed production cost. Free-tier quotas and policies can change, and Supabase Free projects may pause after inactivity. Check current service limits before deployment and budget for paid service if reliable production uptime, backups, or higher usage becomes necessary.