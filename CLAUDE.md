# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Next.js 15+ SaaS application using App Router, TypeScript, Tailwind CSS, and PostgreSQL with DrizzleORM.

## Essential Commands

### Development
```bash
npm run dev          # Start development server with PGlite database (port 3000)
npm run dev:db       # Start PGlite database server only (port 4000)
npm run build        # Production build with memory database
npm run start        # Start production server
```

### Database Management
```bash
npm run db:generate  # Generate database migrations from schema changes
npm run db:migrate   # Apply pending migrations
npm run db:studio    # Open Drizzle Studio for database exploration
npm run db:check     # Check database consistency
```

### Testing
```bash
npm test             # Run all tests (unit, integration, e2e)
npm run test:unit    # Run unit tests with Vitest
npm run test:e2e     # Run E2E tests with Playwright
npm run test:e2e:ui  # Run E2E tests with UI mode
npm run test:ci      # Run tests in CI mode with coverage
```

### Code Quality
```bash
npm run lint         # Run ESLint
npm run lint:fix     # Fix linting issues
npm run format       # Format code with Prettier
npm run format:fix   # Apply formatting fixes
npm run typecheck    # Run TypeScript type checking
npm run check        # Run all checks (lint, format, typecheck)
```

### Storybook & Analysis
```bash
npm run storybook    # Start Storybook (port 6006)
npm run build-storybook
npm run build:analyze  # Analyze bundle size
```

## Architecture & Key Patterns

*** Frameworks
- Started using the Next-js-Boilerplate located on Github https://github.com/ixartz/Next-js-Boilerplate

### Directory Structure
- `src/app/[locale]/`: Internationalized routes with Next.js App Router
  - `(auth)/`: Authentication group routes (sign-in, sign-up, sign-out)
  - `(marketing)/`: Public pages
  - `dashboard/`: Protected dashboard pages
- `src/components/`: Reusable React components
- `src/models/`: DrizzleORM database schemas
- `src/validations/`: Zod validation schemas
- `src/libs/`: Third-party library configurations (Clerk, DB, monitoring)
- `tests/`: Test files (e2e with Playwright, integration tests)

### Database Architecture
- Uses DrizzleORM with PostgreSQL
- Local development uses PGlite (in-browser PostgreSQL)
- Schema defined in `src/models/Schema.ts`
- Migrations stored in `migrations/` directory
- Connection handled via `src/libs/DB.ts` and `src/utils/db-connection.ts`

### Authentication Flow
- Clerk handles all authentication
- Protected routes use middleware in `src/middleware.ts`
- Auth components in `src/app/[locale]/(auth)/`
- Dashboard access requires authentication

### Internationalization
- Supported locales: English (en) and French (fr)
- Translations in `src/locales/`
- Locale routing handled by `[locale]` dynamic segment
- Uses next-intl with Crowdin integration

### Component Development
- Components use TypeScript with strict typing
- Styling with Tailwind CSS classes
- Form handling with React Hook Form + Zod
- Component stories for Storybook in `*.stories.tsx` files

### API Routes
- Located in `src/app/api/`
- Webhook handlers for Clerk in `src/app/api/webhooks/`
- Monitoring endpoints for health checks

## Testing Strategy

### Unit Tests
- Use Vitest with React Testing Library
- Test files: `*.test.ts` or `*.test.tsx`
- Focus on component logic and utilities

### E2E Tests
- Use Playwright
- Test files in `tests/e2e/` with `*.e2e.ts` extension
- Cover critical user journeys

### Running Specific Tests
```bash
npm run test:unit -- path/to/test  # Run specific unit test
npm run test:e2e -- --grep "test name"  # Run specific E2E test
```

## Environment Configuration
- Development: `.env` file (not committed)
- Copy `.env.example` to `.env` for local setup
- Required: Clerk keys, database URL
- Optional: Sentry, PostHog, monitoring services

## Deployment Notes
- Uses semantic versioning with conventional commits
- GitHub Actions handle CI/CD
- Automatic releases on main branch
- Monitoring with Checkly and Better Stack
