## 1. Architecture Overview
### 1.1 Core Technology Stack

* **Backend:** Go 1.21+ with standard library + minimal dependencies
* **Frontend:** HTMX + TEMPL for server-side rendering
* **Styling:** Tailwind CSS (standalone CLI)
* **Database:** SQLite (single-tenant) → PostgreSQL (multi-tenant)
* **Deployment:** Nix packages + Docker containers
* **Build System:** Make + Go toolchain

### 1.2 System Design Principles

* **Onion Architecture:** Clear separation of concerns with dependency inversion
* **Database per Tenant:** Isolated data with shared application code
* **Stateless Application:** All state in database, enabling horizontal scaling
* **Configuration-driven:** Environment-based configuration with sensible defaults
* **Event-driven:** Internal event system for extensibility

### 1.3 Project Structure setup
```txt 
website-builder/
├── cmd/
│   ├── server/           # Main application binary
│   └── migrate/          # Database migration tool
├── internal/
│   ├── app/             # Application layer
│   ├── domain/          # Business logic/entities
│   ├── infrastructure/  # External dependencies
│   └── interfaces/      # HTTP handlers, templates
├── pkg/                 # Reusable packages
├── web/
│   ├── static/         # CSS, JS, images
│   └── templates/      # TEMPL files
├── migrations/         # SQL migration files
├── deployments/
│   ├── nix/           # Nix expressions
│   └── docker/        # Docker configurations
├── scripts/           # Build and deployment scripts
├── go.mod
├── Makefile
└── flake.nix         # Nix flake definition
```


