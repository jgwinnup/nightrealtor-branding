# Night Realtor - Development Plan

## Project Overview

**Night Realtor** is a web-based real estate discovery and analysis tool designed for brokers and agents. It leverages LLMs to perform intelligent analysis of real estate listings, combining data from multiple sources to uncover insights that might otherwise be missed.

### Core Technology Stack

- **Runtime**: Bun.js (fast JavaScript runtime with built-in bundler, test runner, and package manager)
- **Backend Framework**: Hono (lightweight web framework optimized for Bun)
- **Frontend**: React with TypeScript
- **Database**: SQLite (prototype), with vector database support (e.g., SQLite with sqlite-vec extension)
- **Mapping**: Leaflet.js or Mapbox GL JS for clean line-art maps
- **AI/LLM**: Anthropic Claude API (or OpenAI API as fallback)
- **Authentication**: JWT-based auth with bcrypt for password hashing

### Key Data Sources

1. **IDX Broker API** - Primary listing data source
   - Documentation: https://developers.idxbroker.com/course/idx-broker-api/
   - Provides MLS listing data

2. **Montgomery County Ohio Real Estate Tax Information**
   - Website: https://www.mcrealestate.org/search/advancedsearch.aspx?mode=advanced
   - Requires web scraping or form automation (Puppeteer/Playwright)

3. **User-uploaded property lists** - Text files with addresses or parcel IDs

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                     Frontend (React)                     │
│  ├─ Map Interface (Leaflet/Mapbox)                      │
│  ├─ Client Selector                                     │
│  ├─ Property Analysis Dashboard                         │
│  ├─ User Management (Admin)                             │
│  └─ Settings Configuration                              │
└─────────────────────────────────────────────────────────┘
                            │
                            ↓
┌─────────────────────────────────────────────────────────┐
│                   API Layer (Hono/Bun)                   │
│  ├─ Authentication & Authorization                       │
│  ├─ Property Data Aggregation                           │
│  ├─ LLM Analysis Endpoints                              │
│  ├─ Client Management                                   │
│  └─ System Configuration                                │
└─────────────────────────────────────────────────────────┘
                            │
                ┌───────────┴───────────┐
                ↓                       ↓
┌──────────────────────┐    ┌──────────────────────┐
│   SQLite Database    │    │   External APIs      │
│  ├─ Users           │    │  ├─ IDX Broker       │
│  ├─ Clients         │    │  ├─ Claude API       │
│  ├─ Properties      │    │  └─ MC Tax Data      │
│  ├─ Analyses        │    │     (scraped)        │
│  ├─ Settings        │    └──────────────────────┘
│  └─ Vector Store    │
│     (embeddings)    │
└──────────────────────┘
```

## Database Schema

### Core Tables

```sql
-- Users table (agents/brokers)
CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    first_name TEXT NOT NULL,
    last_name TEXT NOT NULL,
    role TEXT NOT NULL CHECK(role IN ('agent', 'broker', 'admin')),
    is_active BOOLEAN DEFAULT 1,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Clients table (buyers/sellers)
CREATE TABLE clients (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    agent_id INTEGER NOT NULL,
    first_name TEXT NOT NULL,
    last_name TEXT NOT NULL,
    email TEXT,
    phone TEXT,
    notes TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (agent_id) REFERENCES users(id)
);

-- Properties table
CREATE TABLE properties (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    address TEXT NOT NULL,
    city TEXT,
    state TEXT,
    zip TEXT,
    parcel_id TEXT,
    latitude REAL,
    longitude REAL,
    property_type TEXT,
    bedrooms INTEGER,
    bathrooms REAL,
    square_feet INTEGER,
    lot_size REAL,
    year_built INTEGER,
    listing_price REAL,
    tax_assessed_value REAL,
    idx_listing_id TEXT,
    idx_data JSON, -- Full IDX broker data
    tax_data JSON, -- Full Montgomery County tax data
    raw_data JSON, -- Any other raw data
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Client properties (watchlist/interested properties)
CREATE TABLE client_properties (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    client_id INTEGER NOT NULL,
    property_id INTEGER NOT NULL,
    status TEXT DEFAULT 'watching' CHECK(status IN ('watching', 'interested', 'offer_made', 'under_contract', 'closed', 'passed')),
    notes TEXT,
    added_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (client_id) REFERENCES clients(id),
    FOREIGN KEY (property_id) REFERENCES properties(id),
    UNIQUE(client_id, property_id)
);

-- Brokerage listings
CREATE TABLE brokerage_listings (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    property_id INTEGER NOT NULL,
    listing_agent_id INTEGER,
    status TEXT DEFAULT 'active' CHECK(status IN ('active', 'pending', 'sold', 'expired', 'withdrawn')),
    list_date DATE,
    expiration_date DATE,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (property_id) REFERENCES properties(id),
    FOREIGN KEY (listing_agent_id) REFERENCES users(id)
);

-- AI Analyses
CREATE TABLE analyses (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    property_id INTEGER NOT NULL,
    analysis_type TEXT NOT NULL,
    prompt TEXT NOT NULL,
    response TEXT NOT NULL,
    model TEXT NOT NULL,
    insights JSON, -- Structured insights extracted
    confidence_score REAL,
    created_by INTEGER,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (property_id) REFERENCES properties(id),
    FOREIGN KEY (created_by) REFERENCES users(id)
);

-- Vector embeddings for semantic search
CREATE TABLE property_embeddings (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    property_id INTEGER NOT NULL,
    embedding BLOB NOT NULL, -- Store as binary vector
    embedding_model TEXT NOT NULL,
    content_hash TEXT, -- Hash of content that was embedded
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (property_id) REFERENCES properties(id)
);

-- System settings
CREATE TABLE settings (
    key TEXT PRIMARY KEY,
    value TEXT NOT NULL,
    description TEXT,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Default settings to insert
INSERT INTO settings (key, value, description) VALUES
    ('brokerage_name', 'Your Brokerage Name', 'Name of the brokerage'),
    ('default_map_center_lat', '39.7589', 'Default map center latitude (1501 E 5th St, Dayton OH)'),
    ('default_map_center_lng', '-84.1847', 'Default map center longitude'),
    ('default_map_zoom', '13', 'Default map zoom level'),
    ('idx_api_key', '', 'IDX Broker API key'),
    ('claude_api_key', '', 'Anthropic Claude API key');
```

## Phase 1: Foundation & Core Infrastructure (Week 1-2)

### 1.1 Project Setup
- [ ] Initialize Bun project with TypeScript
- [ ] Set up project structure
  ```
  night-realtor/
  ├── src/
  │   ├── server/
  │   │   ├── index.ts          # Main server entry
  │   │   ├── db/
  │   │   │   ├── schema.sql    # Database schema
  │   │   │   ├── migrations/   # Database migrations
  │   │   │   └── connection.ts # Database connection
  │   │   ├── routes/
  │   │   │   ├── auth.ts
  │   │   │   ├── properties.ts
  │   │   │   ├── clients.ts
  │   │   │   ├── analyses.ts
  │   │   │   └── settings.ts
  │   │   ├── middleware/
  │   │   │   ├── auth.ts
  │   │   │   └── error.ts
  │   │   ├── services/
  │   │   │   ├── idx-broker.ts
  │   │   │   ├── tax-scraper.ts
  │   │   │   ├── llm-service.ts
  │   │   │   └── geocoding.ts
  │   │   └── types/
  │   └── client/
  │       ├── src/
  │       │   ├── App.tsx
  │       │   ├── components/
  │       │   ├── pages/
  │       │   ├── hooks/
  │       │   ├── services/
  │       │   ├── types/
  │       │   └── utils/
  │       ├── public/
  │       └── index.html
  ├── package.json
  ├── tsconfig.json
  ├── .env.example
  └── README.md
  ```
- [ ] Configure environment variables
- [ ] Set up SQLite database with initial schema
- [ ] Implement database migration system

### 1.2 Authentication System
- [ ] Create user registration endpoint
- [ ] Create login endpoint (JWT token generation)
- [ ] Implement JWT validation middleware
- [ ] Create password reset functionality
- [ ] Add role-based access control (RBAC) middleware
- [ ] Build basic auth UI (login/register pages)

### 1.3 Basic API Structure
- [ ] Set up Hono server with routing
- [ ] Implement error handling middleware
- [ ] Add request logging
- [ ] Create API response utilities (success/error formatting)
- [ ] Set up CORS configuration

## Phase 2: Data Integration (Week 3-4)

### 2.1 IDX Broker Integration
- [ ] Create IDX Broker API client
- [ ] Implement authentication with IDX API
- [ ] Create endpoints to fetch:
  - [ ] Featured listings
  - [ ] Sold listings
  - [ ] Property details by MLS ID
  - [ ] Agent listings
- [ ] Map IDX data to internal property schema
- [ ] Implement data caching strategy
- [ ] Schedule periodic data refresh

### 2.2 Montgomery County Tax Data Scraper
- [ ] Set up Playwright/Puppeteer for browser automation
- [ ] Build scraper for advanced search form
- [ ] Extract property tax information:
  - [ ] Parcel ID
  - [ ] Tax assessed value
  - [ ] Property characteristics
  - [ ] Sales history
  - [ ] Tax payment history
- [ ] Implement rate limiting and error handling
- [ ] Create data normalization pipeline
- [ ] Add scraper to background job queue

### 2.3 Property Data Management
- [ ] Create property CRUD endpoints
- [ ] Implement property search and filtering
- [ ] Build property import from text file (addresses/parcel IDs)
- [ ] Create geocoding service (use Nominatim or Google Geocoding API)
- [ ] Implement property data enrichment pipeline:
  1. Parse input (address/parcel ID)
  2. Geocode address
  3. Fetch IDX data if available
  4. Fetch tax data
  5. Store in database
- [ ] Add property deduplication logic

## Phase 3: Map Interface (Week 5-6)

### 3.1 Map Component Setup
- [ ] Choose mapping library (Leaflet.js recommended for clean lines)
- [ ] Create React map component
- [ ] Implement custom map tiles or style for line-art aesthetic
  - Option 1: Use Mapbox with custom style
  - Option 2: Use OpenStreetMap with custom renderer
  - Option 3: Use Stadia Maps with minimalist style
- [ ] Add map controls (zoom, pan, reset view)
- [ ] Implement configurable default map center (from settings)

### 3.2 Map Layers
- [ ] Create layer system architecture
- [ ] Implement brokerage listings layer
  - [ ] Custom markers for active listings
  - [ ] Marker clustering for dense areas
  - [ ] Click to view listing details
- [ ] Implement client properties layer
  - [ ] Different marker style from brokerage listings
  - [ ] Color coding by status (watching, interested, etc.)
  - [ ] Click to view property analysis
- [ ] Add layer toggle controls
- [ ] Implement layer filtering (by price, type, etc.)

### 3.3 Map Interactions
- [ ] Property detail popup on marker click
- [ ] Draw tools for area selection (optional feature)
- [ ] Property comparison (select multiple properties)
- [ ] Export map view as image

## Phase 4: Client Management (Week 6-7)

### 4.1 Client Management Interface
- [ ] Create client list page
- [ ] Build client creation/edit form
- [ ] Implement client search and filtering
- [ ] Add client detail view
- [ ] Display client's property watchlist

### 4.2 Client Property Management
- [ ] Add properties to client watchlist
- [ ] Remove properties from watchlist
- [ ] Update property status for client
- [ ] Add notes to client properties
- [ ] Property import from text file for specific client

### 4.3 Client Selector
- [ ] Create client dropdown/selector component
- [ ] Implement client switching
- [ ] Update map layers when client changes
- [ ] Show client info in header/sidebar

## Phase 5: LLM Analysis Engine (Week 8-9)

### 5.1 LLM Service Setup
- [ ] Create Claude API client (or OpenAI as fallback)
- [ ] Implement rate limiting for API calls
- [ ] Create prompt templates for different analysis types
- [ ] Build response parsing and structuring

### 5.2 Analysis Types
Implement the following analysis capabilities:

#### Market Analysis
- [ ] Comparative Market Analysis (CMA)
  - Compare property to similar recent sales
  - Identify pricing trends
  - Suggest optimal listing price
- [ ] Neighborhood analysis
  - Analyze area characteristics
  - Identify growth indicators
  - Compare to other neighborhoods

#### Property Analysis
- [ ] Investment potential assessment
  - Calculate potential ROI
  - Identify renovation opportunities
  - Estimate rental income potential
- [ ] Risk assessment
  - Identify red flags (foundation issues, flood zone, etc.)
  - Assess market volatility
  - Evaluate location risks

#### Opportunity Detection
- [ ] Undervalued properties
  - Compare listing price to assessed value
  - Identify below-market listings
  - Detect motivated sellers
- [ ] Off-market opportunities
  - Analyze properties not on MLS
  - Identify pre-foreclosures
  - Find expired listings worth revisiting

#### Document Analysis
- [ ] Parse and analyze listing descriptions
- [ ] Extract key features and amenities
- [ ] Identify missing information
- [ ] Suggest description improvements

### 5.3 Analysis Interface
- [ ] Create analysis dashboard
- [ ] Build analysis request form (select property + analysis type)
- [ ] Display analysis results with formatting
- [ ] Save/export analysis as PDF
- [ ] Analysis history for each property
- [ ] Share analysis with clients

### 5.4 Vector Search (Optional Enhancement)
- [ ] Install sqlite-vec extension for SQLite
- [ ] Generate embeddings for property descriptions
- [ ] Implement semantic property search
- [ ] Find similar properties using vector similarity

## Phase 6: Admin & Configuration (Week 10)

### 6.1 User Management (Admin Only)
- [ ] Create user management interface
- [ ] List all users with filtering
- [ ] Create/edit/deactivate users
- [ ] Assign roles (agent, broker, admin)
- [ ] View user activity logs

### 6.2 System Settings
- [ ] Create settings management interface
- [ ] Configure brokerage name
- [ ] Set default map center and zoom
- [ ] Configure API keys (IDX, Claude, etc.)
- [ ] Set data refresh intervals
- [ ] Configure analysis templates

### 6.3 Brokerage Configuration
- [ ] Upload brokerage logo
- [ ] Set brokerage contact information
- [ ] Configure listing display preferences
- [ ] Set up automated reporting schedules

## Phase 7: Polish & Optimization (Week 11-12)

### 7.1 UI/UX Refinement
- [ ] Implement consistent design system
- [ ] Add loading states and skeletons
- [ ] Improve error messages and validation
- [ ] Add tooltips and help text
- [ ] Implement responsive design for mobile/tablet
- [ ] Add keyboard shortcuts for power users

### 7.2 Performance Optimization
- [ ] Implement data pagination
- [ ] Add response caching
- [ ] Optimize database queries (indexes)
- [ ] Lazy load map markers
- [ ] Compress API responses
- [ ] Implement service worker for offline capability

### 7.3 Testing
- [ ] Unit tests for core services
- [ ] Integration tests for API endpoints
- [ ] End-to-end tests for critical flows
- [ ] Load testing for concurrent users
- [ ] Test IDX and tax data scraping reliability

### 7.4 Documentation
- [ ] API documentation
- [ ] User guide for agents
- [ ] Admin guide for system configuration
- [ ] Developer documentation for future maintenance
- [ ] Deployment guide

## Technical Implementation Details

### LLM Analysis Prompts

#### Example: Comparative Market Analysis
```typescript
const cmaPrompt = `You are a real estate analysis expert. Analyze the following property and comparable sales to provide a comprehensive Comparative Market Analysis.

TARGET PROPERTY:
${JSON.stringify(targetProperty, null, 2)}

COMPARABLE SALES (Last 6 months, within 1 mile):
${JSON.stringify(comparables, null, 2)}

Please provide:
1. Market value estimate with range
2. Key factors affecting value (positive and negative)
3. Comparison to similar properties
4. Market trends in the area
5. Recommended listing strategy
6. Potential concerns or red flags

Format your response as structured JSON with sections for each analysis point.`;
```

#### Example: Investment Analysis
```typescript
const investmentPrompt = `Analyze this property's investment potential for a real estate investor.

PROPERTY DATA:
${JSON.stringify(property, null, 2)}

AREA RENTAL RATES:
${JSON.stringify(rentalComps, null, 2)}

Provide:
1. Estimated monthly rental income (range)
2. Cash flow analysis (with typical financing)
3. Cap rate calculation
4. Appreciation potential (1, 5, 10 years)
5. Renovation opportunities and costs
6. Risk factors
7. Investment score (1-10) with justification

Return structured JSON with detailed analysis.`;
```

### IDX Broker API Integration Example

```typescript
// src/server/services/idx-broker.ts
export class IDXBrokerService {
  private apiKey: string;
  private baseUrl = 'https://api.idxbroker.com';

  constructor(apiKey: string) {
    this.apiKey = apiKey;
  }

  async getFeaturedListings() {
    const response = await fetch(`${this.baseUrl}/clients/featured`, {
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded',
        'accesskey': this.apiKey,
        'outputtype': 'json'
      }
    });
    return response.json();
  }

  async getPropertyDetails(listingId: string) {
    const response = await fetch(
      `${this.baseUrl}/clients/properties/${listingId}`,
      {
        headers: {
          'accesskey': this.apiKey,
          'outputtype': 'json'
        }
      }
    );
    return response.json();
  }

  async searchProperties(criteria: SearchCriteria) {
    const response = await fetch(`${this.baseUrl}/clients/search`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded',
        'accesskey': this.apiKey,
        'outputtype': 'json'
      },
      body: new URLSearchParams(criteria as any)
    });
    return response.json();
  }
}
```

### Montgomery County Tax Scraper Example

```typescript
// src/server/services/tax-scraper.ts
import { chromium } from 'playwright';

export class TaxDataScraper {
  async searchByAddress(address: string) {
    const browser = await chromium.launch({ headless: true });
    const page = await browser.newPage();
    
    try {
      await page.goto('https://www.mcrealestate.org/search/advancedsearch.aspx?mode=advanced');
      
      // Fill in address in the search form
      await page.fill('#ctl00_ContentPlaceHolder1_AdvancedSearch1_txtAddress', address);
      await page.click('#ctl00_ContentPlaceHolder1_AdvancedSearch1_btnSearch');
      
      // Wait for results
      await page.waitForSelector('.search-results-table');
      
      // Extract data
      const propertyData = await page.evaluate(() => {
        // Extract property details from the page
        return {
          parcelId: document.querySelector('.parcel-id')?.textContent,
          assessedValue: document.querySelector('.assessed-value')?.textContent,
          // ... more fields
        };
      });
      
      await browser.close();
      return propertyData;
    } catch (error) {
      await browser.close();
      throw error;
    }
  }
}
```

## Deployment Considerations

### Production Hosting Options
1. **VPS Hosting** (Recommended for prototype)
   - DigitalOcean Droplet or Hetzner VPS
   - Run Bun directly with systemd
   - Nginx reverse proxy
   - Simple backup strategy for SQLite

2. **Platform-as-a-Service**
   - Railway.app (great Bun support)
   - Fly.io
   - Render

3. **Serverless** (Future consideration)
   - Cloudflare Workers (Bun compatible)
   - Note: May need different database strategy

### Environment Variables
```bash
# .env
DATABASE_URL=./data/night-realtor.db
JWT_SECRET=your-secret-key-here
JWT_EXPIRES_IN=7d

IDX_API_KEY=your-idx-broker-api-key
CLAUDE_API_KEY=your-anthropic-api-key

# Optional
GEOCODING_API_KEY=your-geocoding-api-key
MAPBOX_TOKEN=your-mapbox-token

# App Configuration
PORT=3000
NODE_ENV=production
BASE_URL=https://your-domain.com
```

## Security Considerations

- [ ] Implement rate limiting on all endpoints
- [ ] Add SQL injection prevention (use parameterized queries)
- [ ] Validate and sanitize all user inputs
- [ ] Store API keys securely (never in code)
- [ ] Implement HTTPS in production
- [ ] Add CSRF protection
- [ ] Secure password requirements (min length, complexity)
- [ ] Implement account lockout after failed login attempts
- [ ] Add audit logging for sensitive operations
- [ ] Regular security updates for dependencies

## Future Enhancements (Post-MVP)

### Advanced Features
- [ ] Mobile app (React Native)
- [ ] Email notifications for property updates
- [ ] Automated CMA report generation
- [ ] Integration with DocuSign for contracts
- [ ] SMS notifications via Twilio
- [ ] Property showing scheduler
- [ ] Virtual tour integration
- [ ] Market report generator (PDF)
- [ ] Client portal for buyers/sellers
- [ ] Integration with other MLS systems
- [ ] Advanced analytics dashboard
- [ ] AI-powered lead scoring
- [ ] Predictive market analytics

### Scaling Considerations
- [ ] Migrate to PostgreSQL for better concurrency
- [ ] Implement Redis for caching
- [ ] Add job queue (BullMQ) for background tasks
- [ ] Horizontal scaling with load balancer
- [ ] CDN for static assets
- [ ] Dedicated vector database (Pinecone, Qdrant)

## Getting Started with Claude Code

When implementing this with Claude Code, work through the phases in order. Here's how to start:

### Phase 1 Kickoff Commands

```bash
# Create project directory
mkdir night-realtor && cd night-realtor

# Initialize Bun project
bun init

# Install core dependencies
bun add hono @hono/node-server
bun add better-sqlite3 @types/better-sqlite3
bun add bcrypt jsonwebtoken
bun add @types/bcrypt @types/jsonwebtoken -d

# Install dev dependencies
bun add -d typescript @types/node

# Create directory structure
mkdir -p src/server/{db,routes,middleware,services,types}
mkdir -p src/client/src/{components,pages,hooks,services,types,utils}
mkdir -p src/client/public
```

Ask Claude Code to:
1. "Set up the initial project structure for Night Realtor following the plan"
2. "Create the SQLite database schema with all tables"
3. "Implement the authentication system with JWT"
4. "Build the Hono server with basic routing"

Then proceed through each phase, referencing this document as the blueprint.

## Success Metrics

- [ ] User can login and manage multiple clients
- [ ] Properties can be imported and displayed on map
- [ ] IDX Broker data successfully integrated
- [ ] Tax data successfully scraped and stored
- [ ] LLM analysis provides actionable insights
- [ ] Map displays multiple layers cleanly
- [ ] System is performant with 100+ properties
- [ ] Admin can configure all system settings
- [ ] Application is secure and production-ready

## Risk Mitigation

**Risk**: Montgomery County website changes break scraper
- **Mitigation**: Build flexible scraper with selectors config file, implement monitoring and alerts

**Risk**: IDX Broker API rate limits
- **Mitigation**: Implement caching, batch requests, store data locally

**Risk**: LLM API costs exceed budget
- **Mitigation**: Implement usage tracking, set spending limits, cache common analyses

**Risk**: SQLite performance issues with large datasets
- **Mitigation**: Proper indexing, pagination, consider PostgreSQL migration path

**Risk**: Geocoding API costs
- **Mitigation**: Cache all geocoded addresses, consider self-hosted Nominatim instance

## Conclusion

This development plan provides a comprehensive roadmap for building Night Realtor from prototype to production-ready application. The phased approach allows for iterative development and testing while maintaining focus on core functionality first.

The key to success is:
1. Build the foundation properly (authentication, data models)
2. Integrate data sources reliably
3. Create an intuitive, clean map interface
4. Leverage LLMs for meaningful insights
5. Polish the user experience

Start with Phase 1 and work methodically through each phase. Use this document as a living guide, updating it as requirements evolve and new insights emerge during development.
