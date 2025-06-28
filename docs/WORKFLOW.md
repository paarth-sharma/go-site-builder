# Distributable Website Builder - Complete Workplan
*Go + HTMX + TEMPL + Tailwind + Nix*

## Things to note
* A strating point, lots of boilerplate code
* Subject to change, will depend on how each stage plays out

## Phase 1: Foundation & Core Architecture (Weeks 1-2)

### 1.1 Project Structure Setup
```
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

### 1.2 Core Dependencies (Minimal Set)
```go
// go.mod - Keep dependencies minimal
module github.com/yourorg/website-builder

go 1.21

require (
    github.com/a-h/templ v0.2.543        // Template engine
    github.com/gorilla/mux v1.8.1        // HTTP router (or chi)
    github.com/golang-migrate/migrate/v4 // Database migrations
    github.com/mattn/go-sqlite3 v1.14.18 // SQLite driver
    github.com/lib/pq v1.10.9            // PostgreSQL driver
    github.com/joho/godotenv v1.5.1      // Environment configuration
)
```

### 1.3 Domain Model Design
```go
// internal/domain/models.go
type Tenant struct {
    ID        string    `json:"id"`
    Subdomain string    `json:"subdomain"`
    Name      string    `json:"name"`
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
}

type Website struct {
    ID          string            `json:"id"`
    TenantID    string            `json:"tenant_id"`
    Name        string            `json:"name"`
    Domain      string            `json:"domain,omitempty"`
    Theme       string            `json:"theme"`
    Settings    map[string]string `json:"settings"`
    IsPublished bool              `json:"is_published"`
    CreatedAt   time.Time         `json:"created_at"`
    UpdatedAt   time.Time         `json:"updated_at"`
}

type Page struct {
    ID        string            `json:"id"`
    WebsiteID string            `json:"website_id"`
    Path      string            `json:"path"`
    Title     string            `json:"title"`
    Content   []Component       `json:"content"`
    MetaTags  map[string]string `json:"meta_tags"`
    CreatedAt time.Time         `json:"created_at"`
    UpdatedAt time.Time         `json:"updated_at"`
}

type Component struct {
    ID       string                 `json:"id"`
    Type     string                 `json:"type"` // text, image, form, etc.
    Props    map[string]interface{} `json:"props"`
    Children []Component            `json:"children,omitempty"`
    Order    int                    `json:"order"`
}
```

## Phase 2: Core Application Development (Weeks 3-4)

### 2.1 Application Layer Architecture
```go
// internal/app/services.go
type WebsiteService struct {
    repo WebsiteRepository
    events EventPublisher
}

type PageService struct {
    repo PageRepository
    websiteRepo WebsiteRepository
    events EventPublisher
}

type TenantService struct {
    repo TenantRepository
    dbManager DatabaseManager
}
```

### 2.2 HTMX + TEMPL Integration
```go
// web/templates/components/editor.templ
package templates

import "github.com/yourorg/website-builder/internal/domain"

templ PageEditor(page domain.Page) {
    <div id="page-editor" class="min-h-screen bg-gray-50">
        <div class="container mx-auto px-4 py-8">
            <div class="grid grid-cols-12 gap-6">
                <!-- Component Library -->
                <div class="col-span-3 bg-white rounded-lg shadow p-4">
                    <h3 class="font-semibold mb-4">Components</h3>
                    @ComponentLibrary()
                </div>
                
                <!-- Canvas -->
                <div class="col-span-6 bg-white rounded-lg shadow p-4">
                    <div id="canvas" 
                         class="min-h-96 border-2 border-dashed border-gray-300 rounded"
                         hx-post="/api/pages/{page.ID}/components"
                         hx-trigger="drop"
                         hx-swap="innerHTML">
                        @PageCanvas(page.Content)
                    </div>
                </div>
                
                <!-- Properties Panel -->
                <div class="col-span-3 bg-white rounded-lg shadow p-4">
                    <h3 class="font-semibold mb-4">Properties</h3>
                    <div id="properties-panel">
                        Select a component to edit properties
                    </div>
                </div>
            </div>
        </div>
    </div>
}
```

### 2.3 HTTP Handlers with HTMX
```go
// internal/interfaces/http/handlers/page_handler.go
func (h *PageHandler) UpdateComponent(w http.ResponseWriter, r *http.Request) {
    // Extract tenant context
    tenantID := GetTenantID(r.Context())
    
    // Parse component data
    var component domain.Component
    if err := json.NewDecoder(r.Body).Decode(&component); err != nil {
        http.Error(w, "Invalid request", http.StatusBadRequest)
        return
    }
    
    // Update component
    page, err := h.pageService.UpdateComponent(r.Context(), tenantID, pageID, component)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    // Return updated component HTML
    templates.ComponentPreview(component).Render(r.Context(), w)
}
```

## Phase 3: Multi-Tenancy Implementation (Weeks 5-6)

### 3.1 Tenant Resolution Middleware
```go
// internal/infrastructure/middleware/tenant.go
func TenantMiddleware(tenantService *app.TenantService) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // Extract tenant from subdomain or header
            host := r.Host
            subdomain := extractSubdomain(host)
            
            tenant, err := tenantService.GetBySubdomain(r.Context(), subdomain)
            if err != nil {
                http.Error(w, "Tenant not found", http.StatusNotFound)
                return
            }
            
            // Add tenant to context
            ctx := context.WithValue(r.Context(), "tenant_id", tenant.ID)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}
```

### 3.2 Database Per Tenant Strategy
```go
// internal/infrastructure/database/manager.go
type DatabaseManager struct {
    masterDB *sql.DB
    tenantDBs map[string]*sql.DB
    mu sync.RWMutex
}

func (dm *DatabaseManager) GetTenantDB(tenantID string) (*sql.DB, error) {
    dm.mu.RLock()
    db, exists := dm.tenantDBs[tenantID]
    dm.mu.RUnlock()
    
    if exists {
        return db, nil
    }
    
    // Create new tenant database
    return dm.createTenantDatabase(tenantID)
}
```

### 3.3 Repository Pattern with Tenant Context
```go
// internal/infrastructure/repositories/website_repository.go
type WebsiteRepository struct {
    dbManager *database.DatabaseManager
}

func (r *WebsiteRepository) GetByID(ctx context.Context, tenantID, websiteID string) (*domain.Website, error) {
    db, err := r.dbManager.GetTenantDB(tenantID)
    if err != nil {
        return nil, err
    }
    
    var website domain.Website
    err = db.QueryRowContext(ctx, 
        "SELECT id, name, domain, theme, settings, is_published, created_at, updated_at FROM websites WHERE id = $1",
        websiteID).Scan(&website.ID, &website.Name, &website.Domain, &website.Theme, 
                        &website.Settings, &website.IsPublished, &website.CreatedAt, &website.UpdatedAt)
    
    return &website, err
}
```

## Phase 4: Website Builder UI (Weeks 7-8)

### 4.1 Drag & Drop Interface with HTMX
```html
<!-- Component library with draggable items -->
<div class="component-library">
    <div class="component-item" 
         draggable="true"
         data-component-type="text"
         data-component-props='{"content": "Sample text"}'>
        <i class="icon-text"></i>
        Text Block
    </div>
</div>

<script>
// Minimal JavaScript for drag & drop
document.addEventListener('DOMContentLoaded', function() {
    const canvas = document.getElementById('canvas');
    const componentItems = document.querySelectorAll('.component-item');
    
    componentItems.forEach(item => {
        item.addEventListener('dragstart', function(e) {
            e.dataTransfer.setData('text/plain', JSON.stringify({
                type: this.dataset.componentType,
                props: JSON.parse(this.dataset.componentProps)
            }));
        });
    });
    
    canvas.addEventListener('drop', function(e) {
        e.preventDefault();
        const componentData = JSON.parse(e.dataTransfer.getData('text/plain'));
        
        // Use HTMX to add component
        htmx.ajax('POST', `/api/pages/${pageId}/components`, {
            values: componentData,
            target: '#canvas',
            swap: 'beforeend'
        });
    });
});
</script>
```

### 4.2 Real-time Preview System
```go
// internal/interfaces/http/handlers/preview_handler.go
func (h *PreviewHandler) GeneratePreview(w http.ResponseWriter, r *http.Request) {
    page := getPageFromContext(r.Context())
    
    // Generate static HTML for preview
    html := h.renderer.RenderPage(page)
    
    // Return as HTMX response
    w.Header().Set("Content-Type", "text/html")
    w.Write([]byte(html))
}
```

## Phase 5: Build System & Deployment (Weeks 9-10)

### 5.1 Makefile for Build Automation
```makefile
# Makefile
.PHONY: build test dev deploy clean templ tailwind

# Variables
BINARY_NAME=website-builder
BUILD_DIR=./bin
TEMPL_VERSION=v0.2.543

# Install tools
install-tools:
	go install github.com/a-h/templ/cmd/templ@$(TEMPL_VERSION)
	curl -sLO https://github.com/tailwindlabs/tailwindcss/releases/latest/download/tailwindcss-linux-x64
	chmod +x tailwindcss-linux-x64
	sudo mv tailwindcss-linux-x64 /usr/local/bin/tailwindcss

# Generate templates
templ:
	templ generate

# Build CSS
tailwind:
	tailwindcss -i ./web/static/css/input.css -o ./web/static/css/output.css --minify

# Build application
build: templ tailwind
	CGO_ENABLED=1 GOOS=linux go build -ldflags="-s -w" -o $(BUILD_DIR)/$(BINARY_NAME) ./cmd/server

# Development server
dev: templ
	tailwindcss -i ./web/static/css/input.css -o ./web/static/css/output.css --watch &
	templ generate --watch &
	go run ./cmd/server

# Run tests
test:
	go test -v ./...

# Clean build artifacts
clean:
	rm -rf $(BUILD_DIR)
	rm -f web/templates/*_templ.go
```

### 5.2 Nix Flake Configuration
```nix
# flake.nix
{
  description = "Distributable Website Builder";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
    flake-utils.url = "github:numtide/flake-utils";
  };

  outputs = { self, nixpkgs, flake-utils }:
    flake-utils.lib.eachDefaultSystem (system:
      let
        pkgs = nixpkgs.legacyPackages.${system};
        
        website-builder = pkgs.buildGoModule rec {
          pname = "website-builder";
          version = "0.1.0";
          
          src = ./.;
          
          vendorHash = "sha256-XXXXXX"; # Update with actual hash
          
          nativeBuildInputs = with pkgs; [
            sqlite
            tailwindcss
          ];
          
          buildInputs = with pkgs; [
            sqlite
          ];
          
          preBuild = ''
            # Generate templates
            ${pkgs.templ}/bin/templ generate
            
            # Build CSS
            ${pkgs.tailwindcss}/bin/tailwindcss -i ./web/static/css/input.css -o ./web/static/css/output.css --minify
          '';
          
          meta = with pkgs.lib; {
            description = "Self-hostable website builder";
            homepage = "https://github.com/yourorg/website-builder";
            license = licenses.mit;
            maintainers = [ maintainers.yourname ];
          };
        };
      in
      {
        packages.default = website-builder;
        packages.website-builder = website-builder;
        
        devShells.default = pkgs.mkShell {
          buildInputs = with pkgs; [
            go
            templ
            tailwindcss
            sqlite
            air # Live reload for development
          ];
        };
        
        nixosModules.website-builder = { config, lib, pkgs, ... }:
          with lib;
          let
            cfg = config.services.website-builder;
          in {
            options.services.website-builder = {
              enable = mkEnableOption "Website Builder service";
              
              port = mkOption {
                type = types.int;
                default = 8080;
                description = "Port to listen on";
              };
              
              dataDir = mkOption {
                type = types.str;
                default = "/var/lib/website-builder";
                description = "Data directory";
              };
            };
            
            config = mkIf cfg.enable {
              systemd.services.website-builder = {
                description = "Website Builder Service";
                wantedBy = [ "multi-user.target" ];
                after = [ "network.target" ];
                
                serviceConfig = {
                  ExecStart = "${website-builder}/bin/website-builder";
                  User = "website-builder";
                  Group = "website-builder";
                  WorkingDirectory = cfg.dataDir;
                  Restart = "always";
                  RestartSec = 10;
                };
                
                environment = {
                  PORT = toString cfg.port;
                  DATA_DIR = cfg.dataDir;
                };
              };
              
              users.users.website-builder = {
                isSystemUser = true;
                group = "website-builder";
                home = cfg.dataDir;
                createHome = true;
              };
              
              users.groups.website-builder = {};
            };
          };
      });
}
```

### 5.3 Docker Configuration
```dockerfile
# Dockerfile
FROM golang:1.21-alpine AS builder

WORKDIR /app

# Install build dependencies
RUN apk add --no-cache gcc musl-dev sqlite-dev make curl

# Install templ and tailwindcss
RUN go install github.com/a-h/templ/cmd/templ@latest
RUN curl -sLO https://github.com/tailwindlabs/tailwindcss/releases/latest/download/tailwindcss-linux-x64 && \
    chmod +x tailwindcss-linux-x64 && \
    mv tailwindcss-linux-x64 /usr/local/bin/tailwindcss

# Copy source
COPY . .

# Build application
RUN make build

# Runtime image
FROM alpine:latest

RUN apk add --no-cache sqlite ca-certificates tzdata

WORKDIR /app

# Copy binary and static files
COPY --from=builder /app/bin/website-builder /app/website-builder
COPY --from=builder /app/web/static /app/web/static
COPY --from=builder /app/migrations /app/migrations

# Create data directory
RUN mkdir -p /app/data

EXPOSE 8080

VOLUME ["/app/data"]

CMD ["./website-builder"]
```

## Phase 6: Testing & Quality Assurance (Weeks 11-12)

### 6.1 Testing Strategy
```go
// internal/app/services_test.go
func TestWebsiteService_CreateWebsite(t *testing.T) {
    // Setup
    repo := &MockWebsiteRepository{}
    events := &MockEventPublisher{}
    service := app.NewWebsiteService(repo, events)
    
    // Test data
    website := &domain.Website{
        Name:     "Test Site",
        TenantID: "tenant-123",
        Theme:    "default",
    }
    
    // Execute
    result, err := service.CreateWebsite(context.Background(), website)
    
    // Assert
    assert.NoError(t, err)
    assert.NotEmpty(t, result.ID)
    assert.Equal(t, website.Name, result.Name)
}
```

### 6.2 Integration Tests
```go
// tests/integration/api_test.go
func TestPageAPI_CRUD(t *testing.T) {
    // Setup test server
    server := setupTestServer(t)
    defer server.Close()
    
    client := &http.Client{}
    
    // Test create page
    pageData := `{"title": "Test Page", "path": "/test"}`
    resp, err := client.Post(server.URL+"/api/pages", "application/json", strings.NewReader(pageData))
    assert.NoError(t, err)
    assert.Equal(t, http.StatusCreated, resp.StatusCode)
}
```

## Phase 7: Documentation & Distribution (Weeks 13-14)

### 7.1 Documentation Structure
```
docs/
├── README.md
├── INSTALLATION.md
├── CONFIGURATION.md
├── API.md
├── DEVELOPMENT.md
├── DEPLOYMENT.md
└── TROUBLESHOOTING.md
```

### 7.2 Release Automation
```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        goos: [linux, darwin, windows]
        goarch: [amd64, arm64]
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Go
      uses: actions/setup-go@v4
      with:
        go-version: '1.21'
    
    - name: Install dependencies
      run: make install-tools
    
    - name: Build
      run: |
        GOOS=${{ matrix.goos }} GOARCH=${{ matrix.goarch }} make build
        tar -czf website-builder-${{ matrix.goos }}-${{ matrix.goarch }}.tar.gz -C bin .
    
    - name: Upload artifacts
      uses: actions/upload-artifact@v3
      with:
        name: website-builder-${{ matrix.goos }}-${{ matrix.goarch }}
        path: website-builder-${{ matrix.goos }}-${{ matrix.goarch }}.tar.gz
```

## Implementation Timeline

| Phase | Duration | Key Deliverables |
|-------|----------|------------------|
| **Phase 1** | Weeks 1-2 | Project structure, core dependencies, domain models |
| **Phase 2** | Weeks 3-4 | Application layer, HTMX integration, basic CRUD |
| **Phase 3** | Weeks 5-6 | Multi-tenancy, database per tenant, tenant middleware |
| **Phase 4** | Weeks 7-8 | Website builder UI, drag & drop, real-time preview |
| **Phase 5** | Weeks 9-10 | Build system, Nix packages, Docker containers |
| **Phase 6** | Weeks 11-12 | Comprehensive testing, performance optimization |
| **Phase 7** | Weeks 13-14 | Documentation, release automation, distribution |

## Risk Mitigation

### Technical Risks
1. **HTMX Complexity**: Start with simple interactions, gradually add complexity
2. **Multi-tenant Data Isolation**: Implement thorough testing for tenant separation
3. **Performance**: Profile early and often, implement caching strategies
4. **Nix Learning Curve**: Begin with simple Nix expressions, iterate

### Operational Risks
1. **Scope Creep**: Maintain strict MVP feature set for initial release
2. **Dependency Management**: Regular security audits, minimal dependency philosophy
3. **Documentation Debt**: Write docs alongside code, not after

## Success Metrics

### Technical Metrics
- **Build Time**: < 2 minutes for full build
- **Binary Size**: < 50MB for complete application
- **Memory Usage**: < 100MB per tenant under normal load
- **Startup Time**: < 5 seconds for application startup

### Quality Metrics
- **Test Coverage**: > 80% for business logic
- **Documentation Coverage**: 100% of public APIs documented
- **Security**: Zero high/critical vulnerabilities in dependencies

## Next Steps

1. **Immediate**: Set up project structure and core dependencies
2. **Week 1**: Implement basic Go HTTP server with TEMPL integration
3. **Week 2**: Add HTMX interactions and basic page editing
4. **Weekly Reviews**: Assess progress against timeline and adjust scope

This workplan provides a comprehensive roadmap for building a production-ready, distributable website builder that meets your requirements for minimal dependencies, easy deployment, and multi-tenant scalability.