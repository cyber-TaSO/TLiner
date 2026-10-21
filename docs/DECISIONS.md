# Decisions

- **One server, many deployments.** Local, LAN and hosted modes run the same server; data moves between them via project export/import (no live sync in v1).
- **Storage:** local disk / Cloudflare R2 for active footage; Google Drive as archive/backup tier only.
