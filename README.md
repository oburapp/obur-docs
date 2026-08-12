# Obur Docs

Shared documentation for the Obur project — product design, architecture decisions,
data model diagrams, and operational runbooks.

## Structure

| Path | Contents |
|------|----------|
| [`pdd/`](pdd/obur-pdd.md) | Product Design Document — vision, feature catalog, data model, tech stack |
| [`diagram/`](diagram/entity-relationship.md) | Architecture and data diagrams (entity-relationship, etc.) |
| [`adr/`](adr/README.md) | Architecture Decision Records |
| [`runbooks/`](runbooks/incident-response.md) | Operational runbooks (incident response, etc.) |
| [`CLAUDE.md`](CLAUDE.md) | Shared Git, commit, and code quality standards for all Obur repositories |

## Related Repositories

| Repository | Technology | Purpose |
|------------|------------|---------|
| [obur-backend](https://github.com/oburapp/obur-backend) | FastAPI + PostgreSQL | REST API |
| [obur-web](https://github.com/oburapp/obur-web) | Next.js | Web client |
| [obur-mobile](https://github.com/oburapp/obur-mobile) | Flutter | iOS & Android |

## Where to Start

- Product decisions and data model: [pdd/obur-pdd.md](pdd/obur-pdd.md)
- Shared engineering standards: [CLAUDE.md](CLAUDE.md)
- Architecture decisions: [adr/README.md](adr/README.md)
