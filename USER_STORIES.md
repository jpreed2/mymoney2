# User Stories with Acceptance Criteria - MyMoney

## Story Format
Each story follows the format:
- **As a** [user type]
- **I want** [functionality]
- **So that** [business value]

Acceptance criteria use Given/When/Then format for clarity.

---

## 🔐 Authentication & Security Stories

### STORY-001: User Registration with Strong Password Requirements
**As a** new user
**I want** to create an account with my email and password
**So that** I can securely access the MyMoney platform

**Acceptance Criteria:**
- Given I am on the signup page
  - When the page loads
  - Then I should see password requirements list:
    - At least 12 characters
    - At least 1 uppercase letter
    - At least 1 lowercase letter
    - At least 1 number
    - At least 1 symbol (!@#$%^&* or similar)
  - And the "Create Account" button should be disabled initially
- Given I enter a valid password meeting all requirements
  - When I type the password
  - Then each requirement should show a green checkmark when satisfied
  - And the password regex validation should match: `/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[^\w\s]).{12,}$/`
- Given I enter mismatched passwords
  - When password and confirm password don't match
  - Then I should see "Passwords do not match" error
  - And the submit button should remain disabled
- Given I enter a weak password
  - When the password doesn't meet requirements
  - Then I should see "Password does not meet requirements" message
  - And unmet requirements should show in red
- Given I successfully register with valid data
  - When registration completes
  - Then I should be redirected to `/signin?registered=1`
  - And the signin page should display "Account created successfully"
- Given I try to register with an existing email
  - When I submit the form
  - Then I should see "This email is already registered" error
  - And the form should remain on signup page
- Given the API is unreachable
  - When the page detects API is down
  - Then I should see "API unreachable. Form disabled." warning
  - And a "Retry" button should be available
  - And the submit button should show "Offline" and be disabled
- Given I'm already authenticated
  - When I visit the signup page
  - Then I should be immediately redirected to `/accounts`

**Technical Notes:**
- Password must be hashed using Argon2 before storage
- Email must be unique in database (enforced by unique constraint)
- Password policy is configurable via environment variables:
  - `PASSWORD_MIN_LENGTH` (default: 12)
  - `PASSWORD_REQUIRE_UPPER` (default: true)
  - `PASSWORD_REQUIRE_LOWER` (default: true)
  - `PASSWORD_REQUIRE_NUMBER` (default: true)
  - `PASSWORD_REQUIRE_SYMBOL` (default: true)
- Form should show "Creating..." state while mutation is in flight
- GraphQL error codes should be properly mapped to user-friendly messages

---

### STORY-002: User Login with Session Management
**As a** registered user
**I want** to sign in with my credentials
**So that** I can access my accounts securely

**Acceptance Criteria:**
- Given I have valid credentials
  - When I sign in with email and password
  - Then I should receive a JWT access token (or session cookie)
  - And I should be redirected to `/accounts` page
  - And my user info should be stored in localStorage/session
- Given I enter incorrect credentials
  - When I attempt to sign in
  - Then I should see "Invalid credentials" error message
  - And no token/session should be issued
  - And I should remain on the signin page
- Given I am already signed in (session active)
  - When I visit the signin page
  - Then I should be immediately redirected to `/accounts`
  - And SSR should detect session cookie and redirect server-side
- Given I just registered successfully
  - When I'm redirected to signin with `?registered=1`
  - Then I should see "Account created successfully" message
- Given my session/token expires
  - When I make an authenticated API request
  - Then I should be redirected to `/signin` page
  - And my session should be cleared
- Given I click sign out
  - When the logout completes
  - Then my session should be destroyed
  - And I should be redirected to signin page

**Technical Notes:**
- JWT should have configurable expiration (`JWT_EXPIRES_IN`, default: 1h)
- Token can be stored in httpOnly cookie or Authorization header
- Session checking happens via `checkSession` API call
- Failed signin attempts should be rate-limited
- Both REST (`/users/signin`) and GraphQL (`signIn` mutation) endpoints available
- Response includes user object with id and email
- Form shows "Signing in..." state during authentication

---

### STORY-003: JWT Refresh Token Flow
**As a** signed-in user
**I want** my session to automatically refresh
**So that** I don't have to re-login frequently

**Acceptance Criteria:**
- Given my access token is about to expire
  - When I make an API request
  - Then a new access token should be generated using refresh token
- Given my refresh token is valid
  - When I request a new access token
  - Then I should receive new access and refresh tokens
- Given my refresh token is expired
  - When I request a new access token
  - Then I should be redirected to signin page
- Given I sign out
  - When the signout completes
  - Then both tokens should be invalidated

**Technical Notes:**
- Refresh token: 7-day expiration
- Implement token rotation for security
- Store refresh tokens in database for revocation

---

### STORY-004: Rate Limiting Protection
**As a** system administrator
**I want** API endpoints to be rate-limited
**So that** the system is protected from abuse

**Acceptance Criteria:**
- Given a user makes 100 requests in 1 minute
  - When they make the 101st request
  - Then they should receive a 429 error
- Given different endpoints
  - When rate limits are configured
  - Then each should have appropriate limits:
    - Auth endpoints: 5 requests/minute
    - Read endpoints: 100 requests/minute
    - Write endpoints: 30 requests/minute
- Given a rate limit is exceeded
  - When the response is sent
  - Then it should include retry-after header

**Technical Notes:**
- Use @nestjs/throttler for implementation
- Store rate limit data in memory/Redis
- Different limits for authenticated vs anonymous

---

## 💼 Account Management Stories

### STORY-005: Create Financial Account
**As a** authenticated user
**I want** to add a new financial account
**So that** I can track my finances

**Acceptance Criteria:**
- Given I am on the accounts page
  - When I see the form at the top of the page
  - Then I should see input fields for:
    - Name (required, placeholder: "Name")
    - Number (required, placeholder: "Number")
    - Description (optional, placeholder: "Description")
  - And an "Add" button to submit
- Given I fill in valid account details
  - When I click the "Add" button
  - Then the account should be created via GraphQL `addAccount` mutation
  - And it should appear in the account list immediately
  - And the form should be cleared for the next entry
- Given I submit without required fields (name or number)
  - When I try to create an account
  - Then HTML5 validation should prevent submission
  - And no account should be created
- Given I create multiple accounts
  - When I add each account
  - Then they should all appear in the list
  - And each should show name, number, and description (if provided)
- Given I use special characters in account details
  - When I create an account with characters like & # $ € £ ¥
  - Then the account should be created successfully
  - And special characters should display correctly
- Given there's no authenticated user
  - When I try to submit the form
  - Then the submission should fail gracefully (early return)
  - And no errors should be shown to the user

**Technical Notes:**
- GraphQL mutation: `addAccount(userId, name, number, description)`
- Account list uses GraphQL query: `accountsByUser(userId)`
- Form uses React Hook Form with onChange validation
- Optimistic UI updates for better UX
- Form clears via `reset()` after successful submission
- Account display structure:
  - `p.font-medium` for name
  - `p.text-sm` for number
  - `p.text-xs` for description

---

### STORY-006: View Account List
**As a** authenticated user
**I want** to see all my financial accounts
**So that** I can have an overview of my finances

**Acceptance Criteria:**
- Given I have multiple accounts
  - When I visit the accounts page
  - Then I should see all accounts in a list/table
- Given I have no accounts
  - When I visit the accounts page
  - Then I should see an empty state message
- Given accounts have different properties
  - When displayed in the list
  - Then I should see: name, number, description for each
- Given I am not authenticated
  - When I try to access accounts page
  - Then I should be redirected to signin

**Technical Notes:**
- Implement server-side rendering for SEO
- Use GraphQL query with proper caching
- Include loading and error states

---

### STORY-007: Edit Account Details
**As a** account owner
**I want** to update my account information
**So that** I can keep my records accurate

**Acceptance Criteria:**
- Given I have an account in my list
  - When I click the "Edit" button on that account
  - Then the account should switch to edit mode
  - And I should see input fields with current values pre-populated
  - And I should see "Save" and "Cancel" buttons
- Given I'm in edit mode
  - When I modify account details (name, number, or description)
  - And I click "Save"
  - Then the `updateAccount` mutation should be called
  - And the account should be updated in the database
  - And the UI should reflect changes immediately
  - And edit mode should close, returning to view mode
- Given I'm in edit mode
  - When I click the "Cancel" button
  - Then no changes should be saved
  - And edit mode should close
  - And the original values should be displayed
  - And the account should return to view mode
- Given I enter invalid data
  - When I try to save with empty required fields
  - Then validation should prevent submission
  - And appropriate error messages should appear
- Given multiple accounts exist
  - When I edit one account
  - Then only that account should enter edit mode
  - And other accounts should remain in view mode

**Technical Notes:**
- GraphQL mutation: `updateAccount(id, name, number, description)`
- Edit state managed locally with `editingId` state variable
- Inline editing implementation (not modal)
- Form maintains separate state for edited values
- Optimistic updates ensure immediate UI feedback
- Cancel functionality restores original display without API call

---

### STORY-008: Delete Account
**As a** account owner
**I want** to remove accounts I no longer need
**So that** I can keep my account list organized

**Acceptance Criteria:**
- Given I have an account in my list
  - When I click the "Delete" button on that account
  - Then a confirmation dialog should appear
  - With message: "Are you sure you want to delete this account?"
- Given the confirmation dialog is shown
  - When I click "Confirm"
  - Then the `deleteAccount` mutation should be called with the account ID
  - And the account should be deleted from the database
  - And removed from the list immediately (optimistic update)
  - And the UI should update without page refresh
- Given the confirmation dialog is shown
  - When I click "Cancel" or close the dialog
  - Then no deletion should occur
  - And the dialog should close
  - And the account should remain in the list
- Given I delete the last account
  - When the deletion completes
  - Then I should see the empty state message
- Given an account has associated transactions (future feature)
  - When I try to delete it
  - Then I should see a warning about data loss
  - And require additional confirmation

**Technical Notes:**
- GraphQL mutation: `deleteAccount(id)` returns Boolean
- Uses browser's native `confirm()` dialog for simplicity
- Optimistic updates remove item from UI immediately
- Consider implementing soft delete for data recovery (future)
- Add audit logging for compliance (future)
- No undo functionality currently implemented

---

### STORY-009: Search and Filter Accounts
**As a** user with many accounts
**I want** to search and filter my accounts
**So that** I can quickly find specific accounts

**Acceptance Criteria:**
- Given I type in the search box
  - When I enter text
  - Then accounts should filter by name/number match
- Given I use filter dropdowns
  - When I select filters
  - Then only matching accounts should display
- Given no accounts match
  - When filters are applied
  - Then I should see "no results" message
- Given I clear filters
  - When I click clear
  - Then all accounts should display

**Technical Notes:**
- Implement client-side filtering for performance
- Add debouncing to search input
- Persist filter state in URL params

---

## 🔄 Transaction Management Stories (Future)

### STORY-010: Add Transaction to Account
**As a** account owner
**I want** to record transactions
**So that** I can track my financial activity

**Acceptance Criteria:**
- Given I select an account
  - When I click "Add Transaction"
  - Then I should see a transaction form
- Given I enter transaction details
  - When I submit the form
  - Then transaction should be saved
  - And account balance should update
- Given I enter an invalid amount
  - When I try to save
  - Then I should see validation error
- Given I categorize a transaction
  - When saved
  - Then it should be tagged appropriately

**Technical Notes:**
- Transaction types: debit, credit, transfer
- Include date, amount, description, category
- Update account balance automatically

---

### STORY-011: View Transaction History
**As a** account owner
**I want** to see all transactions for an account
**So that** I can review my financial activity

**Acceptance Criteria:**
- Given an account has transactions
  - When I view the account
  - Then I should see a transaction list
- Given transactions exist
  - When displayed
  - Then I should see: date, amount, description, balance
- Given I want to filter by date
  - When I select a date range
  - Then only matching transactions display
- Given I want to export transactions
  - When I click export
  - Then I should download a CSV file

**Technical Notes:**
- Implement pagination for large datasets
- Sort by date descending by default
- Include running balance calculation

---

## 📊 Reporting & Analytics Stories

### STORY-012: View Account Dashboard
**As a** user
**I want** to see an overview dashboard
**So that** I can quickly understand my financial position

**Acceptance Criteria:**
- Given I have multiple accounts
  - When I view the dashboard
  - Then I should see total balance across accounts
- Given I have transactions (future)
  - When viewing dashboard
  - Then I should see spending trends chart
- Given different time periods
  - When I select a period
  - Then charts should update accordingly
- Given I click on a metric
  - When the click registers
  - Then I should see detailed breakdown

**Technical Notes:**
- Use chart library (Chart.js/D3)
- Implement caching for calculations
- Make dashboard configurable

---

### STORY-013: Export Financial Data
**As a** user
**I want** to export my financial data
**So that** I can use it in other applications

**Acceptance Criteria:**
- Given I click export
  - When I select format
  - Then I should see options: CSV, JSON, PDF
- Given I select CSV
  - When export completes
  - Then I should download a properly formatted file
- Given I select date range
  - When I export
  - Then only data in range should be included
- Given large dataset
  - When exporting
  - Then I should see progress indicator

**Technical Notes:**
- Implement streaming for large exports
- Include all related data in export
- Add export audit logging

---

## 🛠️ Technical & DevOps Stories

### STORY-014: API Documentation Access
**As a** developer
**I want** to access comprehensive API documentation
**So that** I can integrate with the MyMoney platform

**Acceptance Criteria:**
- Given I visit /rest-docs
  - When page loads
  - Then I should see ReDoc documentation
- Given I visit /swagger
  - When page loads
  - Then I should see Swagger UI
- Given I visit /graphql
  - When using GraphQL playground
  - Then I should see schema documentation
- Given I need examples
  - When viewing docs
  - Then I should see request/response examples

**Technical Notes:**
- Auto-generate from code annotations
- Include authentication examples
- Keep documentation version-synced

---

### STORY-015: Health Check Monitoring
**As a** DevOps engineer
**I want** health check endpoints
**So that** I can monitor system status

**Acceptance Criteria:**
- Given I call /health
  - When system is healthy
  - Then I should receive 200 OK
  - And see component statuses
- Given database is down
  - When I call /health
  - Then I should receive 503
  - And see which component failed
- Given I need detailed info
  - When I call /health/detailed
  - Then I should see metrics and versions
- Given monitoring system polls
  - When checking health
  - Then response time should be <100ms

**Technical Notes:**
- Check database connectivity
- Include version information
- Add memory/CPU metrics

---

### STORY-016: Automated Testing Pipeline
**As a** development team
**I want** automated testing on every commit
**So that** we catch bugs early

**Acceptance Criteria:**
- Given code is pushed
  - When pipeline triggers
  - Then unit tests should run
  - And integration tests should run
  - And E2E tests should run
- Given tests fail
  - When pipeline completes
  - Then build should fail
  - And team should be notified
- Given tests pass
  - When pipeline completes
  - Then code coverage should be reported
  - And build artifacts should be created
- Given coverage drops below 80%
  - When pipeline runs
  - Then build should warn/fail

**Technical Notes:**
- Use GitHub Actions or Jenkins
- Parallel test execution
- Cache dependencies for speed

---

## 🎨 User Experience Stories

### STORY-017: Mobile Responsive Design
**As a** mobile user
**I want** to access MyMoney on my phone
**So that** I can manage accounts on the go

**Acceptance Criteria:**
- Given I access on mobile
  - When pages load
  - Then layout should be mobile-optimized
- Given I use touch gestures
  - When interacting
  - Then swipe/tap should work properly
- Given different screen sizes
  - When testing
  - Then app should work on phones and tablets
- Given slow mobile connection
  - When loading
  - Then critical content should load first

**Technical Notes:**
- Use responsive CSS Grid/Flexbox
- Implement touch-friendly controls
- Test on real devices

---

### STORY-018: Dark Mode Support
**As a** user who prefers dark themes
**I want** to toggle dark mode
**So that** I can use the app comfortably at night

**Acceptance Criteria:**
- Given I click theme toggle
  - When action completes
  - Then theme should switch
- Given I set dark mode
  - When I return to app
  - Then preference should persist
- Given system is in dark mode
  - When I first visit
  - Then app should default to dark
- Given I switch themes
  - When transition occurs
  - Then it should be smooth (no flash)

**Technical Notes:**
- Use CSS variables for theming
- Store preference in localStorage
- Respect system preferences

---

## 🔧 System Health & Error Handling Stories

### STORY-019: API Availability Detection
**As a** user
**I want** to know when the API is unavailable
**So that** I understand why features aren't working

**Acceptance Criteria:**
- Given the API is unreachable
  - When I load any page
  - Then I should see an API status indicator
  - And forms should be disabled with "Offline" message
- Given the API becomes unavailable while I'm using the app
  - When the app detects the outage
  - Then I should see a notification banner
  - And a "Retry" button should be available
- Given I click the "Retry" button
  - When the API is still down
  - Then it should show "Checking..." state
  - And update the status accordingly
- Given the API becomes available again
  - When the status updates
  - Then forms should be re-enabled
  - And the warning banner should disappear

**Technical Notes:**
- Service status managed via React Context (`useServiceStatus`)
- Health check endpoint: `/health`
- Status properties: `apiUp`, `checking`, `refresh()`
- Forms show disabled state when API is down
- Automatic retry with exponential backoff (future)

---

### STORY-020: Server-Side Rendering (SSR) Authentication
**As a** authenticated user
**I want** pages to load with my data already rendered
**So that** I have a faster, smoother experience

**Acceptance Criteria:**
- Given I have a valid session cookie
  - When I request the accounts page
  - Then SSR should detect my session via `getServerSideProps`
  - And fetch my account data server-side
  - And render the page with data already loaded
- Given I don't have a session cookie
  - When I request the accounts page
  - Then SSR should redirect me to `/signin` server-side
  - And I should never see a flash of the accounts page
- Given my session is invalid or expired
  - When SSR checks my session
  - Then it should clear the invalid session
  - And redirect to `/signin` with appropriate message

**Technical Notes:**
- Session detection via `hasSessionCookie()` utility
- Server-side props: `getServerSideProps` with cookie inspection
- API URL resolution via `getServerApiUrl()` for SSR
- Props passed: `hasSession: boolean`
- Prevents client-side redirect flash

---

### STORY-021: GraphQL Error Handling
**As a** developer
**I want** consistent error handling across GraphQL operations
**So that** users see appropriate error messages

**Acceptance Criteria:**
- Given a GraphQL mutation fails with a known error
  - When the error is caught
  - Then it should map to a user-friendly message:
    - "Email already registered" → "This email is already registered"
    - "Invalid credentials" → "Invalid email or password"
    - Network errors → "Connection failed. Please try again."
- Given a GraphQL query fails
  - When loading data
  - Then a loading state should show first
  - And error state should replace it with retry option
- Given an unexpected GraphQL error occurs
  - When the error is caught
  - Then show generic message: "Something went wrong. Please try again."
  - And log detailed error for debugging

**Technical Notes:**
- Apollo Client error handling with `MockedProvider` for tests
- Error structure: `graphQLErrors[0].message`
- Extension codes: `CONFLICT`, `UNAUTHORIZED`, etc.
- Network errors vs GraphQL errors distinction
- Console error suppression in tests via patterns

---

## Definition of Done (DoD)

For a story to be considered complete:

1. **Code Complete**
   - Feature implemented according to acceptance criteria
   - Code follows project conventions
   - TypeScript types properly defined

2. **Testing**
   - Unit tests written and passing (>80% coverage target)
   - Integration tests for API endpoints (GraphQL and REST)
   - E2E tests for critical user paths
   - Test scenarios must cover:
     - Happy path functionality
     - Error states and edge cases
     - Loading and transition states
     - API unavailability handling
     - Form validation (client and server)
     - Authentication state changes

3. **Documentation**
   - API documentation updated
   - README updated if needed
   - Inline code comments for complex logic

4. **Code Quality**
   - Code review completed
   - Linting passes (npm run lint)
   - Type checking passes (TypeScript strict)

5. **Security**
   - Input validation implemented
   - Security best practices followed
   - No sensitive data exposed

6. **Performance**
   - Page load time <2 seconds
   - API response time <200ms
   - No memory leaks

7. **Deployment**
   - Changes work in Docker environment
   - Environment variables documented
   - Database migrations tested
