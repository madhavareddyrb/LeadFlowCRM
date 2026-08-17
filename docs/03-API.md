Step 3 — Industrial API Architecture

Now we define the API contract before implementing it. The goal is that tomorrow we can start coding without constantly changing our backend structure.

1. API versioning

We'll use:

/api/v1

So our backend URLs become:

/api/v1/auth
/api/v1/businesses
/api/v1/memberships
/api/v1/leads
/api/v1/follow-ups
/api/v1/customers

Why v1?

Because later we can introduce:

/api/v2

without immediately breaking existing clients.

2. Authentication APIs

Base:

/api/v1/auth
Register
POST /api/v1/auth/register

Request:

{
  "firstName": "Madhava",
  "lastName": "Reddy",
  "email": "madhava@example.com",
  "password": "********"
}

Response:

{
  "success": true,
  "message": "Account created successfully",
  "data": {
    "user": {}
  }
}
Login
POST /api/v1/auth/login

Request:

{
  "email": "madhava@example.com",
  "password": "********"
}

Response will eventually contain the authenticated session/token information.

Current user
GET /api/v1/auth/me

This requires authentication.

Purpose:

Token
 ↓
Authentication middleware
 ↓
User
 ↓
Return current session information
Logout
POST /api/v1/auth/logout

We'll decide during implementation whether our final token strategy uses secure HTTP-only cookies, token rotation, or another approach. We should not hard-code an insecure localStorage-only architecture just because it's common in tutorials.

Password reset
POST /api/v1/auth/forgot-password
POST /api/v1/auth/reset-password

We'll build these after the core authentication flow.

3. Business APIs

Base:

/api/v1/businesses
Create business
POST /api/v1/businesses

Authenticated user:

User
 ↓
Create Business
 ↓
Create OWNER membership

This is an important transaction-like workflow.

Get my businesses
GET /api/v1/businesses

Useful because eventually one user could belong to multiple businesses.

Get business
GET /api/v1/businesses/:businessId

Authorization must verify the user belongs to that business.

Update business
PATCH /api/v1/businesses/:businessId

Only authorized roles can do this.

4. Membership APIs

Base:

/api/v1/memberships
Invite member
POST /api/v1/memberships

Request:

{
  "businessId": "...",
  "email": "rahul@example.com",
  "role": "AGENT"
}
List members
GET /api/v1/memberships?businessId=...
Change member role
PATCH /api/v1/memberships/:membershipId

Example:

{
  "role": "MANAGER"
}
Suspend member
PATCH /api/v1/memberships/:membershipId/status

This is better than immediately deleting the membership.

We want to preserve historical information.

5. Lead APIs

This is the heart of LeadFlow.

Base:

/api/v1/leads
Create lead
POST /api/v1/leads

Request:

{
  "firstName": "John",
  "lastName": "Doe",
  "phone": "+919999999999",
  "email": "john@example.com",
  "source": "INSTAGRAM",
  "requirement": "Looking for annual gym membership",
  "value": 15000
}

The backend determines:

businessId
createdBy

from authenticated context.

The client should not be trusted to send these security-sensitive values.

That's an important engineering rule.

List leads
GET /api/v1/leads

Eventually:

GET /api/v1/leads
    ?status=INTERESTED
    &assignedTo=USER_ID
    &source=INSTAGRAM
    &page=1
    &limit=20
    &search=john

We'll implement filtering and pagination properly rather than returning thousands of records at once.

Get single lead
GET /api/v1/leads/:leadId
Update lead
PATCH /api/v1/leads/:leadId

Example:

{
  "status": "INTERESTED",
  "assignedTo": "...",
  "value": 20000
}
Delete/archive lead

For a SaaS product, I prefer soft deletion over immediately destroying important business data.

We can eventually use:

isDeleted
deletedAt
deletedBy

rather than physically deleting everything.

6. Follow-up APIs

Base:

/api/v1/follow-ups
Create follow-up
POST /api/v1/follow-ups

Request:

{
  "leadId": "...",
  "assignedTo": "...",
  "type": "CALL",
  "scheduledAt": "2026-08-18T10:30:00+05:30",
  "notes": "Call about membership pricing"
}
Get follow-ups
GET /api/v1/follow-ups

Possible filters:

?status=PENDING
&assignedTo=...
&from=...
&to=...

This will later power:

Today's Follow-ups
Overdue Follow-ups
Upcoming Follow-ups
Update follow-up
PATCH /api/v1/follow-ups/:followUpId

For example:

{
  "status": "COMPLETED"
}
7. Customer APIs

Base:

/api/v1/customers

We should not allow a random client request to directly convert anything it wants into a customer.

Instead:

PATCH /api/v1/leads/:leadId/convert

Business action:

Lead
 ↓
CONVERTED
 ↓
Customer created

This gives us one explicit business operation.

Then:

GET /api/v1/customers

and:

GET /api/v1/customers/:customerId
8. Dashboard API

We don't need a separate Dashboard collection.

Dashboard data is derived from the database.

GET /api/v1/dashboard/overview

Example response:

{
  "success": true,
  "data": {
    "totalLeads": 125,
    "newLeads": 24,
    "activeLeads": 62,
    "convertedLeads": 21,
    "lostLeads": 18,
    "pendingFollowUps": 13,
    "conversionRate": 16.8
  }
}

Later we'll use MongoDB aggregation pipelines for this.

That gives you real backend experience beyond simple CRUD.

9. Standard API response format

We need consistency across the entire application.

Success
{
  "success": true,
  "message": "Lead created successfully",
  "data": {}
}
Collection
{
  "success": true,
  "message": "Leads fetched successfully",
  "data": [],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 125,
    "totalPages": 7
  }
}
10. Standard error format

Every API error should follow one structure.

{
  "success": false,
  "message": "Lead not found",
  "error": {
    "code": "LEAD_NOT_FOUND"
  }
}

For validation:

{
  "success": false,
  "message": "Validation failed",
  "error": {
    "code": "VALIDATION_ERROR",
    "fields": {
      "email": "Invalid email address",
      "phone": "Phone number is required"
    }
  }
}

This makes frontend error handling much easier.

11. HTTP status conventions

We'll consistently use:

Situation	Status
Successful GET/POST/PATCH	200 / 201
Successful DELETE	204
Bad request	400
Unauthenticated	401
Authenticated but not allowed	403
Resource doesn't exist	404
Conflict	409
Validation failure	422
Unexpected server error	500

We won't randomly return 200 for everything.

12. Middleware architecture

Our request pipeline should eventually look like:

HTTP Request
     ↓
Helmet / Security
     ↓
CORS
     ↓
Request Logger
     ↓
JSON Parser
     ↓
Rate Limiting
     ↓
Route
     ↓
Authentication
     ↓
Tenant Context
     ↓
Authorization
     ↓
Validation
     ↓
Controller
     ↓
Service
     ↓
Repository / Model
     ↓
MongoDB

Not every endpoint needs every middleware, but this is the architectural direction.

13. Controller vs Service

This is an important decision.

We won't put all business logic inside controllers.

Bad:

exports.createLead = async (req, res) => {
   // 150 lines of everything
}

Instead:

Route
 ↓
Controller
 ↓
Service
 ↓
Model

For example:

lead.routes.js
      ↓
lead.controller.js
      ↓
lead.service.js
      ↓
Lead model

The controller handles HTTP.

The service handles business logic.

The model handles database interaction.

This makes the backend easier to test and maintain.

14. Backend folder architecture

Tomorrow we'll build toward:

server/
│
├── src/
│   ├── config/
│   ├── controllers/
│   ├── middlewares/
│   ├── models/
│   ├── routes/
│   ├── services/
│   ├── validators/
│   ├── utils/
│   ├── constants/
│   ├── app.js
│   └── server.js
│
├── tests/
│
├── .env
├── .env.example
├── package.json
└── README.md

As the application grows, we may move to a more feature-oriented structure. We won't blindly follow a folder pattern if it starts becoming difficult to maintain.

15. The most important security rule

Never trust:

businessId
userId
createdBy
ownerId

coming from the frontend when those values can be derived from authenticated context.

For example, we should not rely on:

{
  "businessId": "business-of-attacker"
}

from the request body.

Instead:

JWT/session
   ↓
Authenticated User
   ↓
Membership
   ↓
Authorized Business
   ↓
Database query scoped to businessId

This is the architecture we will repeatedly apply.

16. Final API map

Our initial API architecture is now:

/api/v1
│
├── /auth
│   ├── POST   /register
│   ├── POST   /login
│   ├── POST   /logout
│   ├── GET    /me
│   ├── POST   /forgot-password
│   └── POST   /reset-password
│
├── /businesses
│   ├── POST   /
│   ├── GET    /
│   ├── GET    /:businessId
│   └── PATCH  /:businessId
│
├── /memberships
│   ├── POST   /
│   ├── GET    /
│   ├── PATCH  /:membershipId
│   └── PATCH  /:membershipId/status
│
├── /leads
│   ├── POST   /
│   ├── GET    /
│   ├── GET    /:leadId
│   ├── PATCH  /:leadId
│   └── PATCH  /:leadId/convert
│
├── /follow-ups
│   ├── POST   /
│   ├── GET    /
│   └── PATCH  /:followUpId
│
├── /customers
│   ├── GET    /
│   └── GET    /:customerId
│
└── /dashboard
    └── GET    /overview
Foundation status

We've now completed:

✅ Product definition
✅ MVP scope
✅ User roles
✅ Multi-tenant strategy
✅ Database architecture
✅ API architecture

The remaining setup work today is repository/project initialization + environment strategy + Git conventions + engineering rules.

Then tomorrow we can begin actual backend implementation, starting from the repository and Express foundation rather than jumping straight into CRUD.