# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Night Realtor** is a web-based real estate discovery and analysis tool for brokers and agents, developed by **Night Realtor LLC**. It uses LLMs to perform intelligent analysis of real estate listings by aggregating data from multiple sources (county CAMA data, IDX Broker API, and user uploads).

**Current Status**: Phase 1 Foundation + Client Management Complete (as of 2025-11-08)
- 208,405 properties loaded (Montgomery County, OH)
- 203,667 properties enriched with CAMA data (97.7% match rate)
- Search-based property discovery with fuzzy matching
- Enhanced property details with modal view
- Click-to-select properties on map (zoom-based, configurable)
- Map layers: ZIP codes (blue) and School Districts (pink)
- Complete property characteristics, valuations, and sale history
- **Client management system with contacts and archiving** ✨ NEW
- **Property watchlists for client tracking** ✨ NEW
- **Watchlist properties displayed on map (purple markers)** ✨ NEW
- Admin-configurable map settings
- Database backup utilities
- Multi-county ready architecture
- Standardized camelCase API responses

## Technology Stack

- **Runtime**: Bun.js 1.x
- **Backend**: Hono web framework
- **Frontend**: React 19 with TypeScript + Vite
- **Database**: SQLite with WAL mode (using Bun's built-in `bun:sqlite` module)
- **Mapping**: Leaflet.js 1.9 with react-leaflet, CartoDB Positron tiles, marker clustering
- **AI/LLM**: Configurable (Claude API or local Ollama via LLM_PROVIDER env var)
- **Auth**: JWT-based with bcrypt ✅ **IMPLEMENTED**
- **Web Scraping**: Playwright/Puppeteer (for Montgomery County tax data, NOT YET IMPLEMENTED)

## Project Structure

The intended directory structure is:
```
night-realtor/
├── src/
│   ├── server/              # Backend (Hono/Bun)
│   │   ├── index.ts
│   │   ├── db/              # Database schema and migrations
│   │   ├── routes/          # API route handlers
│   │   ├── middleware/      # Auth and error handling
│   │   ├── services/        # External API integrations
│   │   └── types/
│   └── client/              # Frontend (React)
│       └── src/
│           ├── components/
│           ├── pages/
│           ├── hooks/
│           ├── services/
│           ├── types/
│           └── utils/
├── docs/                    # Project documentation
└── assets/                  # Static assets
```

## Database Schema

The application uses SQLite with the following core tables:
- **users** ✅ - Agents/brokers (with roles: agent, broker, admin)
- **clients** ✅ - Buyers/sellers managed by agents with contact information
- **properties** ✅ - Real estate properties with MLS and tax data
- **client_properties** ✅ - Client watchlists with status tracking (watching, interested, toured, etc.)
- **brokerage_listings** ❌ - Listings managed by the brokerage (NOT YET USED)
- **analyses** ✅ - LLM-generated property analyses with structured insights
- **property_embeddings** ❌ - Vector embeddings for semantic search (TABLE NOT CREATED)
- **settings** ✅ - System configuration (brokerage info, map defaults)
- **audit_logs** ✅ - Comprehensive audit trail for compliance and security

Properties store data from multiple sources in JSON columns (idx_data, tax_data, raw_data, cama_data).

**CAMA Data** (Computer Assisted Mass Appraisal) includes:
- Dwelling characteristics (style, exterior, basement, heating, square footage)
- Land information (lot size, acreage, neighborhood codes)
- Valuations (assessed and appraised land/building/total)
- Sale history (last sale date, price, previous owner)
- Owner information and property classification

**See `docs/IMPLEMENTATION-STATUS.md` for complete feature status.**
**See `docs/montgomery-county-cama-integration.md` for CAMA data details.**

## Architecture

The application follows a 3-tier architecture:

1. **Frontend (React)** - Map interface, client selector, property dashboard, admin panel
2. **API Layer (Hono)** - Authentication, property aggregation, LLM endpoints, client management
3. **Data Layer** - SQLite database + External APIs (IDX Broker, Claude, Montgomery County tax scraper)

### Key Services

- **llm-service.ts** ✅ - Configurable LLM client (Claude API or Ollama) for property analysis
- **audit-logger.ts** ✅ - Comprehensive audit logging for compliance and security
- **idx-broker.ts** ❌ - IDX Broker API client for MLS listing data (NOT YET IMPLEMENTED)
- **tax-scraper.ts** ❌ - Playwright-based scraper for Montgomery County tax records (NOT YET IMPLEMENTED)
- **geocoding.ts** ❌ - Address geocoding service (NOT YET IMPLEMENTED)

### LLM Analysis Types

The system supports multiple analysis types:
- **Market Analysis**: CMA, neighborhood analysis, pricing trends
- **Property Analysis**: Investment potential, risk assessment
- **Opportunity Detection**: Undervalued properties, off-market opportunities
- **Document Analysis**: Listing description parsing and improvement

## Data Sources

1. **Montgomery County GIS** - Address points shapefile with coordinates ✅ INTEGRATED
2. **Montgomery County CAMA** - Computer Assisted Mass Appraisal database ✅ INTEGRATED
   - 254,308 total property records (214,016 residential)
   - Complete property characteristics, valuations, and sale history
   - 97.7% match rate with imported properties
3. **IDX Broker API** - MLS listing data (https://developers.idxbroker.com/) ❌ NOT YET IMPLEMENTED
4. **County Tax Scraper** - Additional tax data via web scraping ❌ LOWER PRIORITY (CAMA covers 90%)
5. **User Uploads** - Text files with addresses or parcel IDs ❌ NOT YET IMPLEMENTED

## Development Commands

```bash
# Run both backend and frontend dev servers (recommended)
bun run dev          # Backend (Hono) on port 3000
bun run dev:client   # Frontend (Vite) on port 5173

# Backend only
cd src/server && bun --watch index.ts

# Frontend only
cd src/client && bun run dev

# Build for production (NOT YET CONFIGURED)
bun run build

# Run tests (NOT YET IMPLEMENTED)
bun test
```

### Important Notes

- **Database**: Using Bun's built-in `bun:sqlite` module (NOT `better-sqlite3` which has compatibility issues)
- **Database WAL Files**: `.db-shm`, `.db-wal`, `.db-journal` are in `.gitignore` (runtime files only)
- **API Endpoints**:
  - Auth: `/api/auth` (register, login, me, logout)
  - Users: `/api/users` (CRUD with RBAC)
  - Properties: `/api/properties` (GET with limit=0 default)
  - Property Search: `/api/properties/search?q=query&limit=20`
  - Property by Coordinates: `/api/properties/by-coordinates?lat=X&lng=Y&maxDistance=100`
  - Clients: `/api/clients` (CRUD with contacts, role-based filtering)
  - Client Watchlists: `/api/clients/:id/properties` (add, update, remove properties)
  - Analyses: `/api/analyses/:propertyId`
  - Settings: `/api/settings` (admin only, batch update support)
  - Public Settings: `/api/settings/public/map` (map configuration, no auth)
  - Health: `/health`
- **Database Location**: `./data/night-realtor.db` (177.89 MB, 208,405 properties)
- **Backups Location**: `./backups/` (compressed SQL dumps)
- **Utilities**: `./utils/` (import, backup, inspection scripts)
- **Logo**: Located in `assets/images/night-realtor-skyline.png` (optimized)
- **Default Credentials**: admin@nightrealtor.com / password123 (CHANGE IN PRODUCTION)

## Codebase Statistics

**As of 2025-11-04:**

- **Total Commits**: 32
- **TypeScript Files**: 36 files, 5,432 lines
- **CSS Files**: 18 files, 3,103 lines
- **Python Utilities**: 4 files, 741 lines
- **Documentation**: 11 markdown files
- **Database**: 177.89 MB (208,405 properties)
- **Test Coverage**: Not yet implemented

**Key Metrics**:
- Backend: 2,100+ lines (Hono + services)
- Frontend: 3,300+ lines (React components)
- Utilities: 740+ lines (data import, backup)
- Documentation: 2,500+ lines (comprehensive guides)

## Implementation Status

**As of 2025-11-04:**

### ✅ Completed (Phase 1 - Foundation)
**Core Infrastructure**:
- Backend setup (Bun + Hono)
- Complete database schema with all tables
- JWT authentication with RBAC (agent/broker/admin roles)
- User management API (full CRUD)
- Comprehensive audit logging system

**Property Management**:
- 208,405 properties imported (Montgomery County, OH)
- Property search API with fuzzy matching
- Search-based property discovery (max 20 results default)
- Click-to-select properties by coordinates (Haversine distance calculation)
- Enhanced property popups with key information
- Full property details modal with scrollable content
- Address normalization (proper capitalization)
- Coordinate-based property lookup API

**Frontend**:
- Authentication UI (login/register)
- Interactive map (Leaflet with clustering)
- Click-to-select properties (crosshair cursor, zoom-based activation)
- Map layers: ZIP Code Boundaries and School Districts
- Non-interactive layers (clicks pass through to map)
- Search bar with expandable UI
- Theme system (light/dark mode)
- User menu with role-based options
- Property details modal with scroll support
- Layers menu for toggling map overlays
- Admin settings UI (map configuration)
- About dialog

**Utilities**:
- Database backup utility (gzip compression)
- Property import from ArcGIS shapefiles
- Shapefile to GeoJSON conversion (ZIP codes, school districts)
- Coordinate reprojection (EPSG:3735 → EPSG:4326)
- Coordinate transformation (State Plane → WGS84)
- Shapefile inspection tools

### 🚧 Partially Implemented
- LLM analysis (endpoint exists, limited types)
- Admin UI (user management done, settings/audit log viewer pending)

### ✅ Completed (Phase 3 - Client Management) ✨ NEW
- **Client CRUD API**: Complete with contacts and archiving
- **Property Watchlists**: Add, update status, notes, remove
- **Client UI**: List page, watchlist page, header selector
- **Map Integration**: Purple markers for watchlist properties
- **API Standardization**: camelCase responses throughout
- **Audit Logging**: Full tracking of client operations

### ❌ Not Yet Started
- **Phase 2**: IDX Broker integration, tax scraper, geocoding, CAMA frontend integration
- **Phase 4**: Multi-county expansion (framework ready, needs data)
- **Phase 5**: Advanced LLM analysis types, vector search
- **Phase 6**: Testing, performance optimization
- **Phase 7**: Production deployment

**Completed This Session (2025-11-08)**:
- ✅ Client management system with contacts
- ✅ Property watchlists with status tracking
- ✅ Watchlist display on map (purple markers)
- ✅ API field name standardization (camelCase)
- ✅ Enhanced watchlist popups with status badges
- ✅ Client archiving functionality
- ✅ Documentation updates

**See `docs/IMPLEMENTATION-STATUS.md` for detailed feature tracking.**
**See `docs/ROADMAP.md` for multi-county expansion plans.**

## Key Implementation Notes

### Authentication (✅ IMPLEMENTED)
- JWT tokens with 7-day expiration (configurable via JWT_EXPIRES_IN)
- Role-based access control (agent, broker, admin)
- bcrypt password hashing with 10 salt rounds
- Middleware: `authMiddleware`, `requireRole()`, `optionalAuth`
- Token stored in localStorage on frontend
- Auto-validation on app load via `/api/auth/me`
- Logout is client-side (remove token) with audit logging

### Audit Logging (✅ IMPLEMENTED)
- **Service**: `src/server/services/audit-logger.ts`
- **Tracked Events**:
  - User: create, login, login.failed (with reason), logout, update, role.change, delete
  - Property: create, delete
  - Client: create, update, archive, restore, delete
  - Watchlist: property.add_to_watchlist, property.remove_from_watchlist
- **Data Captured**: timestamp, user_id, action, resource_type, resource_id, changes (JSON with old/new values), ip_address, user_agent, metadata
- **Query Methods**: `getUserActivity()`, `getResourceHistory()`, `getFailedLogins()`, `getRoleChanges()`, `getStats()`
- **Performance**: Prepared statements, indexed columns (timestamp, user_id, resource, action)
- **Documentation**: Full strategy in `docs/logging-strategy.md`

### Property Data Pipeline (⚠️ PARTIALLY IMPLEMENTED)
1. ✅ Manual property creation via API
2. ❌ Geocode address (NOT YET IMPLEMENTED)
3. ❌ Fetch IDX data if available (NOT YET IMPLEMENTED)
4. ❌ Scrape tax data (NOT YET IMPLEMENTED)
5. ❌ Store in database with deduplication (NO DEDUPLICATION LOGIC)

### Click-to-Select Properties (✅ IMPLEMENTED)
**Feature**: Click on map to select nearest property
- **Activation**: Automatic at zoom level 16+ (building level)
- **Distance**: Maximum 100m from click point (configurable)
- **Visual Feedback**: Cursor changes to crosshair when feature active
- **Popup**: Auto-opens property info popup (not full modal)
- **Button**: "View Full Details →" opens full modal from popup

**How it works**:
1. User zooms in to level 16+
2. Cursor changes to crosshair
3. Click anywhere on map
4. System finds nearest property using Haversine formula
5. If within max distance, shows popup with property info
6. Temporary marker appears at property location

**Admin Configuration** (Settings → Map Settings):
- Enable/disable feature
- Set minimum zoom level (1-20, default: 16)
- Set maximum selection distance (10-1000m, default: 100m)
- Settings stored in database, accessible via public API

**API Endpoint**: `/api/properties/by-coordinates?lat=X&lng=Y&maxDistance=100`
- Searches all properties in database
- Returns nearest property within max distance
- Uses Haversine distance calculation
- 404 if no properties found within range

### Map Layers (✅ IMPLEMENTED)
**Current Layers**:
- **ZIP Code Boundaries** - Blue boundaries with labels (zoom 11+)
  - Source: Montgomery County GIS shapefile
  - Non-interactive (clicks pass through to map)
  - 26 ZIP codes in dataset

- **School Districts** - Pink/magenta boundaries with labels (zoom 12+)
  - Source: Montgomery County school district shapefile
  - Non-interactive (clicks pass through to map)
  - 26 school districts (Kettering CSD, Centerville CSD, etc.)

**Layer Management**:
- Toggle layers via layers menu (☰ button in header)
- Layers stored in React state (not persisted)
- Non-interactive design allows click-to-select to work when layers enabled

**Future Layers**:
  - **Brokerage listings layer** - Active listings from the brokerage (clustered markers)

### Client Management (✅ IMPLEMENTED)
**Features**: Manage buyer/seller clients and property watchlists

**Client CRUD**:
- Create clients with contact information (phone, email, address)
- Role-based access: agents see their clients, brokers/admins see all
- Archive/restore functionality (soft delete with `archived_at` timestamp)
- Full audit trail for all client operations

**Property Watchlists**:
- Add properties to client watchlist via `/api/clients/:id/properties`
- Status tracking: watching → interested → toured → offer_made → under_contract → closed → passed
- Notes field for each watchlist entry
- Update status and notes independently
- Remove properties from watchlist

**Map Integration**:
- Purple/magenta markers for watchlist properties (vs blue for search results)
- Separate `MarkerClusterGroup` for performance
- Enhanced popups with:
  - Purple-themed status badge (📋 icon)
  - Watchlist status (WATCHING, INTERESTED, etc.)
  - Yellow-highlighted notes section (when notes exist)
  - Full dark mode support
- "View Full Details →" button opens PropertyDetails modal

**Client Selector**:
- Header dropdown showing selected client
- Displays client name and property count
- Clear selection returns to all properties view
- Selection persists across page navigation (via ClientContext)

**Client Pages**:
- **ClientList** (`/clients`): Search, filter, quick actions (view, edit, archive)
- **ClientWatchlist** (`/clients/:id/watchlist`): Property cards with status/notes editing

### API Standardization (✅ IMPLEMENTED)
**Convention**: camelCase for all API responses, snake_case in database

**Implementation**:
- Created `convertPropertyToApiFormat()` helper in `src/server/routes/properties.ts`
- Single conversion point at API boundary (backend)
- All property search endpoints return camelCase
- Frontend Property interface uses camelCase consistently
- Database remains snake_case (SQL convention)
- No ad-hoc field mapping throughout codebase

**Converted Fields**:
- `property_type` → `propertyType`
- `listing_price` → `listingPrice`
- `square_feet` → `squareFeet`
- `lot_size` → `lotSize`
- `year_built` → `yearBuilt`
- `tax_assessed_value` → `taxAssessedValue`
- `parcel_id` → `parcelId`
- `raw_data` → `rawData`
- `cama_data` → `camaData`
- `created_at` → `createdAt`
- `updated_at` → `updatedAt`

### Security Requirements
- Rate limiting on all endpoints
- Parameterized SQL queries to prevent injection
- Input validation and sanitization
- API keys stored in environment variables only
- HTTPS in production
- CSRF protection

## Environment Variables

```bash
DATABASE_URL=./data/night-realtor.db
JWT_SECRET=your-secret-key-here
JWT_EXPIRES_IN=7d
IDX_API_KEY=your-idx-broker-api-key
CLAUDE_API_KEY=your-anthropic-api-key
GEOCODING_API_KEY=your-geocoding-api-key  # Optional
MAPBOX_TOKEN=your-mapbox-token  # Optional
PORT=3000
NODE_ENV=development|production
BASE_URL=https://your-domain.com
```

## Important Considerations

### IDX Broker Integration
- Use proper headers: 'accesskey' for API key, 'outputtype': 'json'
- Implement caching to avoid rate limits
- Store full IDX response in properties.idx_data JSON column

### Tax Data Scraping
- Implement rate limiting and error handling
- Run scraper as background job
- Handle site changes gracefully with monitoring

### LLM Analysis
- Use structured JSON responses from Claude
- Implement usage tracking to control costs
- Cache common analyses
- Store prompts and responses in analyses table

### Database
- Create proper indexes for query performance
- Use JSON columns for flexible data storage from external APIs
- Implement migration system for schema changes
- Consider PostgreSQL migration path if scaling beyond SQLite

## Reference Documentation

Full development plan with detailed implementation examples is in [docs/night-realtor-development-plan.md](docs/night-realtor-development-plan.md).
