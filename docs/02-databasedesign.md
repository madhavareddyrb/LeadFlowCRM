Step 2 — LeadFlow Database Design

Now we design the database before writing Mongoose schemas. This is important because changing the data model later can become expensive.

1. Core relationship

Our first version will use these six collections:

User
  │
  │
  ▼
BusinessMembership
  │
  ▼
Business
  │
  ├──────────► Lead
  │              │
  │              ├────────► FollowUp
  │              │
  │              └────────► Customer
  │
  └──────────► Team Members

The key principle is:

Business is the tenant.

Every business-owned resource will carry businessId.

2. User

A User represents a human using LeadFlow.

User
├── _id
├── firstName
├── lastName
├── email
├── passwordHash
├── phone
├── avatar
├── isEmailVerified
├── isActive
├── lastLoginAt
├── createdAt
└── updatedAt
Important decisions

We store:

passwordHash

not the actual password.

Email should have a unique index.

We will not put business-specific role information directly on User.

Why?

Because one person could eventually belong to multiple businesses.

For example:

Madhava
 ├── Company A → Manager
 └── Company B → Agent

That is why we need BusinessMembership.

3. Business

This represents the customer's company.

Business
├── _id
├── name
├── slug
├── industry
├── email
├── phone
├── logo
├── timezone
├── currency
├── ownerId
├── settings
├── createdAt
└── updatedAt

Example:

{
  name: "ABC Fitness",
  slug: "abc-fitness",
  industry: "gym",
  ownerId: "..."
}
Why slug?

Later we may have URLs such as:

leadflow.app/business/abc-fitness

or:

leadflow.app/form/abc-fitness

So a stable business identifier is useful.

4. BusinessMembership

This is one of our most important collections.

It connects:

User ↔ Business

Structure:

BusinessMembership
├── _id
├── businessId
├── userId
├── role
├── status
├── invitedBy
├── joinedAt
├── createdAt
└── updatedAt
Role
OWNER
MANAGER
AGENT

Status
INVITED
ACTIVE
SUSPENDED
REMOVED

Relationship:

User A
   │
   ├── Business A → OWNER
   ├── Business B → AGENT
   └── Business C → MANAGER

This gives us a genuine multi-tenant architecture.

Critical index

We should not allow the same user to be an active membership twice in the same business.

So eventually:

unique index:
businessId + userId
5. Lead

This is our primary business entity.

Lead
├── _id
├── businessId
├── createdBy
├── assignedTo
├── firstName
├── lastName
├── email
├── phone
├── source
├── status
├── requirement
├── value
├── tags
├── notes
├── lastContactedAt
├── nextFollowUpAt
├── convertedAt
├── lostReason
├── createdAt
└── updatedAt
Lead source

We can initially support:

WEBSITE
WHATSAPP
PHONE
INSTAGRAM
FACEBOOK
GOOGLE
REFERRAL
OTHER
Lead status
NEW
CONTACTED
INTERESTED
NEGOTIATION
CONVERTED
LOST
Example
Lead
├── businessId: ABC Fitness
├── assignedTo: Rahul
├── firstName: John
├── phone: ...
├── source: INSTAGRAM
├── status: INTERESTED
├── value: 15000
└── nextFollowUpAt: ...
6. FollowUp

A lead can have many follow-ups.

Therefore:

Lead
 │
 ├── FollowUp 1
 ├── FollowUp 2
 ├── FollowUp 3
 └── FollowUp 4

Collection:

FollowUp
├── _id
├── businessId
├── leadId
├── assignedTo
├── type
├── scheduledAt
├── status
├── notes
├── completedAt
├── createdBy
├── createdAt
└── updatedAt
Type
CALL
WHATSAPP
EMAIL
MEETING
OTHER
Status
PENDING
COMPLETED
MISSED
CANCELLED

This gives us a useful timeline:

Lead
 │
 ├── Aug 17 → CALL → Completed
 ├── Aug 19 → WHATSAPP → Completed
 └── Aug 22 → MEETING → Pending
7. Customer

When a lead becomes a paying customer:

Lead
  ↓
CONVERTED
  ↓
Customer

For MVP:

Customer
├── _id
├── businessId
├── leadId
├── name
├── email
├── phone
├── value
├── convertedBy
├── convertedAt
├── notes
├── createdAt
└── updatedAt
Why retain leadId?

Because we want to trace:

Customer
   ↓
Original Lead
   ↓
Source
   ↓
Follow-ups
   ↓
Conversion

Later this will help analytics.

8. Complete relationship model

Our conceptual model is now:

                  ┌──────────────┐
                  │     User     │
                  └──────┬───────┘
                         │
                         │
                  ┌──────▼──────────────┐
                  │ BusinessMembership  │
                  └──────┬──────────────┘
                         │
                         ▼
                  ┌──────────────┐
                  │   Business   │
                  └──────┬───────┘
                         │
             ┌───────────┼─────────────┐
             │           │             │
             ▼           ▼             ▼
           Lead       Customer     Membership
             │
       ┌─────┴─────┐
       │           │
       ▼           ▼
   FollowUp      User
9. Important MongoDB design decision

We're not going to embed every child object inside Lead.

For example, we will not initially do:

Lead
 └── followUps: [
       {...},
       {...},
       {...}
     ]

We'll use a separate FollowUp collection.

Why?

Because follow-ups can grow over time, and we want to:

search
filter
paginate
sort
query by date
query by employee

efficiently.

This is a good example of designing for the actual product rather than simply making the schema look simple.

10. Multi-Tenant Security Rule

This becomes a non-negotiable backend rule.

Every business-owned query must be scoped by:

businessId

Bad:

Lead.findById(req.params.id)

Potential problem: someone could try another lead's ID.

Better:

Lead.findOne({
  _id: req.params.id,
  businessId: req.user.businessId
})

Eventually we'll centralize this logic rather than trusting every developer to remember it manually.

This is one of the security principles we'll practice throughout the project.

11. Index strategy

We won't blindly index every field.

Likely indexes for MVP:

User
email
Business
slug
ownerId
Membership
businessId + userId
businessId + role
Lead
businessId
businessId + status
businessId + assignedTo
businessId + createdAt
businessId + nextFollowUpAt
FollowUp
businessId + scheduledAt
businessId + assignedTo + status
leadId

This will matter once the dataset becomes large.

12. One correction to our earlier design

Earlier I mentioned:

Team

as a possible entity.

We don't need a separate Team collection for MVP.

BusinessMembership already represents the team.

So our actual initial collections are:

1. User
2. Business
3. BusinessMembership
4. Lead
5. FollowUp
6. Customer

That's cleaner.

13. Database flow example

Imagine a real customer:

Madhava
   ↓
Creates "Madhava Fitness"
   ↓
Business created
   ↓
Membership created
   ↓
Invites Rahul
   ↓
Rahul becomes AGENT
   ↓
Rahul creates Lead "John"
   ↓
John gets CONTACTED
   ↓
Follow-up scheduled
   ↓
Follow-up completed
   ↓
Negotiation
   ↓
Converted
   ↓
Customer created

That's the actual business lifecycle we're building.

Today's database milestone

We now have the conceptual database design.

The next step should be:

Step 3 — API Architecture

We'll define:

/api/v1/auth
/api/v1/businesses
/api/v1/memberships
/api/v1/leads
/api/v1/follow-ups
/api/v1/customers

and design the actual API contracts:

POST /register
POST /login
GET /me


POST /businesses
GET /businesses/:id


POST /leads
GET /leads
PATCH /leads/:id
DELETE /leads/:id


POST /follow-ups
GET /follow-ups
PATCH /follow-ups/:id

We'll also establish our standard API response format, error format, HTTP status conventions, authentication middleware, authorization middleware, and validation strategy before tomorrow's coding begins.