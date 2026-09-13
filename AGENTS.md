# AGENTS.md - Development Guidelines

## Critical Rules (Non-Negotiable)

- ❌ Never use fake production data
- ❌ Never bypass authorization checks
- ❌ Never commit secrets or API keys
- ❌ Never expose private CVs or sensitive files
- ❌ Never expose passwords/tokens in logs or responses
- ❌ Never trust frontend permissions - always validate on backend
- ❌ Never create mock/stub data permanently in production

## Backend Requirements

- Always write tests for business logic
- Always validate input on backend (never trust frontend)
- Always use RBAC with explicit permissions
- Always hash passwords properly
- Always sanitize user input
- Always implement rate limiting on sensitive endpoints
- Always log security events to audit_logs
- Always validate file uploads (MIME, size, extension)
- Always use prepared statements to prevent SQL injection
- Always implement CORS properly

## Frontend Requirements

- Always run `npm run lint`
- Always run `npm run type-check`
- Always run component tests before pushing
- Always consider RTL layout
- Always consider Persian UX (جلالی dates, Persian numbers, تومان currency)
- Always validate forms with Zod
- Always implement loading and error states
- Always handle edge cases

## Database Requirements

- Always use migrations for schema changes
- Always normalize data
- Always add appropriate indexes
- Always use foreign keys
- Always use soft deletes where appropriate
- Always add created_at/updated_at timestamps
- Always use transactions for critical operations
- Always validate database constraints

## Testing Checklist

Before marking feature as complete:
- ✅ Unit tests pass
- ✅ Feature tests pass
- ✅ Integration tests pass
- ✅ No console errors or warnings
- ✅ No TypeScript errors
- ✅ Lint passes
- ✅ Authorization working correctly
- ✅ Tested on mobile view
- ✅ Persian text displays correctly
- ✅ No hardcoded secrets

## Documentation Requirements

- Every new feature needs updating docs
- Every API endpoint needs documentation
- Every database migration needs explanation
- Every security decision needs reasoning

## Build & Deployment Checks

Before production:
```bash
npm install
composer install
php artisan migrate --force
npm run build
docker build .
php artisan test
npm run test:e2e
php artisan tinker < security-audit.php
```

Never claim success while:
- Commands are failing
- Tests are failing
- TypeScript has errors
- The application won't start
- Security vulnerabilities exist
