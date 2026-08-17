### Product Definition

# Product name: Lead Flow 

### One-line description

A simple SaaS CRM that helps small businesses capture, organize, follow up with, and convert customer leads without losing them.

### Core problem: A small business may receive enquiries from:

WhatsApp,Instagram,Phone calls,Website,Google,Referrals

### But the information is scattered.

The owner may forget:

Who contacted me?

What did they want?

Who should follow up?

When should I contact them?

Did they become a customer?

How much revenue came from these leads?

LeadFlow centralizes this process.

2. Our Initial Target Customer

We should not target every business initially.

Our first target:

Small service businesses that regularly receive customer enquiries and need follow-ups.

Examples:

Real-estate agents
Coaching institutes
Salons
Gyms
Interior designers
Digital agencies
Local service providers

For the first MVP, we'll keep the product generic enough that the same system works for all of them.

Later we can create industry-specific versions.

3. Core User Types

We'll begin with three roles.

Owner

The business owner can:

Create business
Manage team
Create/manage leads
Assign leads
View analytics
Manage subscription

Manager Can:

View leads
Create leads
Assign leads
Manage follow-ups
View team activity

Sales Agent Can:

View assigned leads
Update lead status
Add notes
Schedule follow-ups
Convert leads

This gives us real RBAC (Role-Based Access Control) rather than simply having a login page.

4. MVP

This is extremely important.

We are not building WhatsApp + AI + payments + 100 features on day one.

Our MVP is:

Authentication
        ↓
Business
        ↓
Team
        ↓
Leads
        ↓
Pipeline
        ↓
Follow-ups
        ↓
Customers
        ↓
Dashboard

Authentication
Register
Login
Logout
Refresh/access token strategy
Forgot password
Reset password
Business
Create business
Business profile
Business settings
Team
Invite team member
Assign role
Activate/deactivate member
Lead

A lead will contain information such as:

Name
Phone
Email
Source
Requirement
Status
Assigned employee
Notes
Created date
Lead pipeline

NEW
 ↓
CONTACTED
 ↓
INTERESTED
 ↓
NEGOTIATION
 ↓
CONVERTED

And:

LOST

can happen from any active stage.

Follow-up

Lead
 ↓
Follow-up date
 ↓
Follow-up type
 ↓
Notes
 ↓
Completed / Pending

Customer

When a lead converts:

Lead
 ↓
CONVERTED
 ↓
Customer
Dashboard

Initially:

Total Leads
New Leads
Active Leads
Converted Leads
Lost Leads
Pending Follow-ups
Conversion Rate

5. Multi-Tenant SaaS

This is one of the most important architectural decisions.

Suppose we have:

Business A
 ├── Owner
 ├── Employees
 └── Leads


Business B
 ├── Owner
 ├── Employees
 └── Leads

Business A must never be able to access Business B's data.

Therefore, most business-owned records will contain:

businessId

For example:

Lead
├── _id
├── businessId
├── name
├── phone
├── status
└── assignedTo

Every relevant backend query will be scoped to the authenticated user's business.

This is the foundation of our SaaS architecture.

6. High-Level Architecture

We'll use:

                    LeadFlow
                       │
           ┌───────────┴───────────┐
           │                       │
       Frontend                 Backend
           │                       │
        React                 Node.js
           │                  Express.js
           │                       │
      API Client              REST APIs
           │                       │
           └───────────┬───────────┘
                       │
                    MongoDB
                       │
        ┌──────────────┼──────────────┐
        │              │              │
      Users         Businesses       Leads
                                       │
                                  Follow-ups
                                       │
                                   Customers

Later:

Backend
 ├── Authentication
 ├── Authorization
 ├── Validation
 ├── Logging
 ├── Notifications
 ├── Payments
 ├── AI
 └── Background Jobs

But we won't build those prematurely.

7. Initial Database Model

Our first conceptual model:

User
  │
  └──── BusinessMembership ──── Business
                                      │
                    ┌─────────────────┼─────────────────┐
                    │                 │                 │
                  Lead             Customer          Team
                    │
                 FollowUp

One important design decision:

User ≠ Business

A user is a person.

A business is the organization.

Membership connects them.

This will make future SaaS features much easier.

8. Initial Tech Stack

Frontend
React
Vite
React Router
Redux Toolkit
Axios
Tailwind CSS
Backend
Node.js
Express.js
MongoDB
Mongoose
JWT
bcrypt
Development
Git
GitHub
Postman
ESLint
Prettier

We'll add technologies later only when there is a genuine requirement.

9. Repository Strategy

We'll use one repository:

leadflow/
│
├── client/
│
├── server/
│
├── docs/
│
├── .gitignore
└── README.md

This also avoids the Git repository collision problem you've encountered before.

docs/ will eventually contain
docs/
├── requirements/
├── architecture/
├── database/
├── api/
└── decisions/

That documentation itself becomes useful when explaining the project in interviews.

10. Development Rule

From tomorrow, we're going to follow this cycle:

Understand
    ↓
Design
    ↓
Implement
    ↓
Test
    ↓
Review
    ↓
Refactor
    ↓
Commit

Not:

Copy code
 ↓
Something works
 ↓
Move on

That's the difference between a tutorial project and an engineering project.

Today's First Milestone

Before we touch the actual backend tomorrow, we should lock these five artifacts:

01 — Product Requirements
02 — System Architecture
03 — Database ER/relationship design
04 — API architecture
05 — Repository structure

I've already started 01 — Product Requirements above.