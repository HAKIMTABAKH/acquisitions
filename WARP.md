# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Project Overview

Node.js/Express API for acquisitions management using:
- **Runtime**: Node.js with ES modules (`type: "module"`)
- **Database**: PostgreSQL (Neon serverless) with Drizzle ORM
- **Authentication**: JWT tokens stored in httpOnly cookies
- **Validation**: Zod schemas
- **Logging**: Winston

## Development Commands

### Run Development Server
```bash
npm run dev
```
Uses Node's `--watch` flag for auto-reload on file changes.

### Code Quality
```bash
npm run lint              # Check for linting errors
npm run lint:fix          # Auto-fix linting errors
npm run format            # Format all files with Prettier
npm run format:check      # Check formatting without modifying
```

### Database Operations
```bash
npm run db:generate       # Generate migration files from schema changes
npm run db:migrate        # Apply pending migrations to database
npm run db:studio         # Open Drizzle Studio (database GUI)
```

**Important**: After modifying models in `src/models/*.js`, always run `db:generate` then `db:migrate`.

## Architecture

### Path Aliases
The project uses Node.js subpath imports (defined in `package.json`):
- `#config/*` → `./src/config/*`
- `#models/*` → `./src/models/*`
- `#routes/*` → `./src/routes/*`
- `#controllers/*` → `./src/controllers/*`
- `#middleware/*` → `./src/middleware/*`
- `#utils/*` → `./src/utils/*`
- `#validations/*` → `./src/validations/*`
- `#services/*` → `./src/services/*`

Always use these aliases for imports across the codebase.

### Request Flow
```
index.js → server.js → app.js → routes → controllers → services → models
                                              ↓
                                         validations
```

1. **Entry**: `src/index.js` loads environment variables and imports `server.js`
2. **Server**: `src/server.js` starts the Express app on specified PORT
3. **App**: `src/app.js` configures middleware (helmet, cors, morgan, cookie-parser) and routes
4. **Routes**: Define endpoints and map to controllers (e.g., `auth.routes.js`)
5. **Controllers**: Handle request/response logic, validate input with Zod schemas
6. **Services**: Contain business logic (e.g., user creation, password hashing)
7. **Models**: Define Drizzle ORM schemas (PostgreSQL tables)

### Key Patterns

**Controllers** (`src/controllers/`):
- Use `try/catch` blocks
- Validate request body with Zod schemas (`.safeParse()`)
- Format validation errors with `formatValidationsError()` utility
- Call service layer functions
- Generate JWT tokens and set httpOnly cookies
- Log operations with Winston logger
- Return structured JSON responses

**Services** (`src/services/`):
- Encapsulate database operations using Drizzle ORM
- Handle business logic (e.g., password hashing with bcrypt)
- Check for duplicate entries before insertion
- Use `.returning()` to get inserted record data
- Throw errors with descriptive messages for controller handling

**Validations** (`src/validations/`):
- Define Zod schemas for request body validation
- Apply transformations (`.trim()`, `.toLowerCase()`)
- Set constraints (`.min()`, `.max()`)
- Define enums for restricted values

**Models** (`src/models/`):
- Use Drizzle ORM's `pgTable` to define database schemas
- Export table definitions for use in services
- Include timestamps (`created_at`, `updated_at`)

**Configuration**:
- **Database**: `src/config/database.js` exports `db` (Drizzle instance) and `sql` (Neon client)
- **Logger**: `src/config/logger.js` exports Winston instance with file and console transports

**Utilities** (`src/utils/`):
- `jwt.js`: Token signing/verification wrapper
- `cookie.js`: Cookie management with secure defaults
- `format.js`: Error formatting helpers

### Environment Variables
Required in `.env` (see `.env.exemple`):
- `PORT`: Server port (default: 3000)
- `NODE_ENV`: Environment mode (development/production)
- `LOG_LEVEL`: Winston log level (default: info)
- `DATABASE_URL`: PostgreSQL connection string (Neon serverless)
- `JWT_SECRET`: Secret key for JWT signing (change in production!)

### Code Style
- **Indentation**: 2 spaces (enforced by ESLint)
- **Quotes**: Single quotes
- **Semicolons**: Required
- **Line endings**: Unix (LF)
- Use `const` over `let`, no `var`
- Prefer arrow functions
- Unused variables starting with `_` are allowed
- Console statements are permitted

### API Endpoints
Current routes (mounted at `/api/auth/`):
- `POST /api/auth/sign-up` - User registration (implemented)
- `POST /api/auth/sign-in` - User login (stub)
- `POST /api/auth/sign-out` - User logout (stub)

Health checks:
- `GET /health` - Returns server status, timestamp, and uptime
- `GET /api` - API availability check
