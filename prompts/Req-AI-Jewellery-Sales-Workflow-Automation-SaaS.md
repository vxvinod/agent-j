# Build a Production-Ready AI Jewellery Sales & Workflow Automation SaaS

## 1. ROLE

Act as a senior product architect, UX designer, AI engineer, backend engineer, frontend engineer, database architect, QA engineer, security engineer, and DevOps engineer.

I want to build a multi-tenant SaaS product for jewellery businesses in India.

The product should automate the workflow from:

**Customer enquiry → AI understanding → product recommendation → quotation → follow-up → appointment → salesperson handoff → order**

The product must be designed as a real commercial SaaS, not a demo.

Do not build unnecessary features just because they are technically interesting.

Prioritize:
1. Business value
2. Simplicity
3. Reliability
4. Security
5. Auditability
6. Low operating cost
7. Ability to scale
8. Easy onboarding for non-technical jewellery shop owners

---

# 2. PRODUCT VISION

Create an AI-powered operating layer for jewellery businesses.

The first MVP should solve one painful problem extremely well:

> **Convert WhatsApp/online jewellery enquiries into qualified leads, quotations, appointments and sales follow-ups.**

The system should eventually expand into:

- CRM
- Inventory
- Jewellery catalogue
- Quotations
- Orders
- Manufacturing workflow
- Customer management
- After-sales
- Analytics
- AI business assistant

But DO NOT build all of these initially.

---

# 3. TARGET USERS

### Primary user

Jewellery shop owner / showroom manager.

They should be able to:

- View enquiries
- View hot leads
- See customer conversations
- See quotations
- See follow-up tasks
- Assign leads to salespeople
- View appointments
- View conversion metrics

### Salesperson

They should be able to:

- View assigned leads
- See AI-generated customer summary
- Continue conversation
- Create/update quotation
- Schedule follow-up
- Mark lead status

### Admin

They should be able to:

- Create jewellery business
- Manage users
- Configure gold rates
- Configure making charges
- Configure wastage
- Configure GST
- Manage catalogue
- Configure WhatsApp integration
- View audit logs

---

# 4. MVP WORKFLOW

Build this exact workflow first:

```text
Customer
   ↓
WhatsApp / Web enquiry
   ↓
Message received
   ↓
AI extracts requirements
   ↓
Create/update customer
   ↓
Create lead
   ↓
AI classifies lead
   ↓
Recommend catalogue products
   ↓
Generate indicative quotation
   ↓
AI sends response
   ↓
Customer replies
   ↓
AI understands intent
   ↓
Follow-up automation
   ↓
Appointment
   ↓
Salesperson handoff
   ↓
Lead converted / lost
```

---

# 5. EXAMPLE

Customer sends:

"Anna, my daughter's marriage is next month. I need a 22 carat necklace, around 6 sovereigns, modern design, budget 4 lakhs. Do you have anything?"

AI should extract:

```json
{
  "occasion": "wedding",
  "product_type": "necklace",
  "purity": "22K",
  "estimated_weight_sovereigns": 6,
  "maximum_weight_sovereigns": 7,
  "budget": 400000,
  "style": "modern",
  "urgency": "high"
}
```

Then search the jewellery catalogue.

Return matching products.

The system should calculate the quotation using deterministic business rules.

IMPORTANT:

**Never allow an LLM to invent gold prices, taxes, weights, or final prices.**

AI can interpret requirements.

The backend must perform financial calculations.

---

# 6. AI RESPONSIBILITIES

Use AI only where it creates genuine value.

AI should:

### A. Understand customer messages

Extract:

- Product
- Occasion
- Budget
- Weight
- Purity
- Style
- Colour
- Stones
- Delivery requirement
- Location
- Urgency

Support:

- English
- Tamil
- Tanglish
- Mixed language
- Informal WhatsApp language

Example:

"Bro 5 pavun necklace venum 3 lakh kulla"

Should become:

```json
{
  "product_type": "necklace",
  "weight_sovereigns": 5,
  "budget": 300000
}
```

---

### B. Classify customer intent

Possible intents:

- Product enquiry
- Price enquiry
- Design enquiry
- Availability enquiry
- Custom order
- Appointment request
- Exchange enquiry
- Repair enquiry
- Complaint
- Purchase intent
- General question

---

### C. Lead scoring

Create a transparent lead score.

Example:

```text
Budget provided             +20
Product specified           +20
Purchase timeline           +20
Design selected             +15
Asked for quotation         +15
Appointment requested       +10
```

Score:

0–30 = Cold

31–60 = Warm

61–80 = Hot

81–100 = Very Hot

Do not make the score a mysterious black box.

Store the reasons for the score.

---

### D. Conversation summary

Generate:

- Customer requirement
- Budget
- Preferred products
- Previous conversation
- Objections
- Next action
- Recommended follow-up date

Example:

"Customer wants a 22K bridal necklace, 6–7 sovereigns, budget ₹4L. Interested in modern designs. Asked for showroom appointment Saturday morning. High purchase intent."

---

### E. Follow-up generation

AI generates natural messages based on context.

Examples:

"Hi Priya, yesterday you were checking our 22K bridal necklace designs. We have shortlisted three options close to your budget. Would you like to see them?"

The system must prevent spam.

---

# 7. NON-AI RESPONSIBILITIES

Use deterministic software for:

- Gold price
- Weight calculation
- Making charge
- Wastage
- Stone price
- GST
- Discounts
- Final quotation
- Inventory
- Customer records
- Order status
- Appointment records
- User permissions
- Billing calculations

Never let the LLM directly modify financial values.

---

# 8. QUOTATION ENGINE

Create a configurable jewellery pricing engine.

Example:

```text
Metal value =
gold_rate_per_gram × net_gold_weight

Wastage =
metal value × wastage_percentage

Making charge =
configured making charge

Stone charge =
configured stone value

Subtotal =
metal value
+ wastage
+ making charge
+ stone charge

GST =
configured GST percentage

Final amount =
subtotal + GST
```

Support configuration by:

- Business
- Product category
- Purity
- Making charge type
- Percentage
- Per gram
- Fixed amount

Every quotation must store the exact pricing inputs used at the time it was created.

If the gold price changes tomorrow, an old quotation must remain reproducible.

---

# 9. PRODUCT CATALOGUE

Create catalogue management.

Each product should support:

```text
Product ID
SKU
Name
Category
Subcategory
Purity
Gross weight
Net gold weight
Stone weight
Making charge
Wastage
Price
Images
Description
Tags
Availability
```

Categories:

- Necklace
- Chain
- Ring
- Earrings
- Bracelet
- Bangles
- Pendant
- Bridal set
- Kids jewellery
- Men's jewellery

Allow custom categories.

---

# 10. PRODUCT RECOMMENDATION

Create a hybrid recommendation system.

Stage 1:

Use structured filters:

- Category
- Purity
- Budget
- Weight
- Availability

Stage 2:

Use semantic search for:

- Modern
- Traditional
- Minimal
- Bridal
- Temple
- Contemporary
- Heavy
- Lightweight

Do not rely entirely on an LLM.

Use embeddings/vector search where appropriate.

Return:

```text
Product
Why it matches
Estimated price
Weight
Image
Availability
```

---

# 11. CRM

Build a lightweight CRM.

Lead statuses:

```text
NEW
CONTACTED
QUALIFIED
QUOTATION_SENT
FOLLOW_UP
APPOINTMENT
NEGOTIATION
WON
LOST
```

Each lead should have:

- Customer
- Source
- Assigned salesperson
- Status
- Score
- Requirements
- Conversation
- Quotations
- Follow-ups
- Appointment
- Created date
- Updated date

---

# 12. FOLLOW-UP ENGINE

Create a workflow engine.

Example:

```text
Quotation sent
      ↓
Wait 1 day
      ↓
No response?
      ↓
Send follow-up
      ↓
Wait 2 days
      ↓
No response?
      ↓
Second follow-up
      ↓
Wait 4 days
      ↓
Final follow-up
```

Rules must be configurable.

Do not send messages automatically if:

- Customer opted out
- Conversation is closed
- Lead is marked lost
- Human salesperson has taken over
- Business disabled automation

Create a complete audit trail.

---

# 13. WHATSAPP ARCHITECTURE

Design the system to integrate with the official WhatsApp Business Platform/API.

Incoming message:

```text
WhatsApp
 ↓
Webhook
 ↓
Message validation
 ↓
Message storage
 ↓
AI processing
 ↓
Intent detection
 ↓
CRM update
 ↓
Workflow engine
 ↓
Response generation
 ↓
WhatsApp
```

Never expose WhatsApp credentials to the frontend.

Use encrypted secrets.

Design webhook processing to be idempotent.

A duplicate webhook must not create duplicate leads or messages.

---

# 14. HUMAN HANDOFF

This is mandatory.

AI should not pretend to be human.

Allow salesperson to take over a conversation.

When human takeover happens:

```text
AI automation = PAUSED
```

Salesperson handles the customer.

They can return the conversation to AI later.

---

# 15. DASHBOARD

Create a clean dashboard.

Top metrics:

```text
New Leads
Hot Leads
Pending Follow-ups
Appointments
Quotations
Won Sales
Conversion Rate
Estimated Pipeline Value
```

Show:

### Today's priorities

- 5 hot leads
- 8 follow-ups
- 3 appointments

### AI insights

Example:

"12 customers asked about bridal necklaces this week."

"Most enquiries are between ₹2L–₹5L."

"7 quotations have not received a response."

---

# 16. DATABASE

Use PostgreSQL.

Design a proper multi-tenant schema.

Minimum entities:

```text
organizations
users
roles
customers
leads
conversations
messages
products
product_images
gold_rates
pricing_rules
quotations
quotation_items
followups
appointments
orders
ai_interactions
workflow_runs
workflow_steps
audit_logs
integrations
```

Every business entity must belong to an organization/tenant.

Implement tenant isolation correctly.

---

# 17. TECH STACK

Prefer:

### Frontend

Next.js
TypeScript
React
Tailwind CSS

### Backend

Python
FastAPI
Pydantic

### Database

PostgreSQL

### Cache / queues

Redis

### Background jobs

Celery / RQ / equivalent reliable job system

### AI

Use an LLM API through an abstraction layer.

Do not hard-code the application to one model provider.

### Vector search

PostgreSQL + pgvector initially.

### Storage

S3-compatible object storage.

### Deployment

Docker.

Design for cloud deployment.

---

# 18. AI ARCHITECTURE

Do not create one giant prompt.

Create specialized components:

```text
Message Classifier
Requirement Extractor
Lead Scorer
Product Retriever
Response Generator
Conversation Summarizer
Follow-up Generator
```

Use structured outputs.

Every AI result should be validated with Pydantic/schema validation.

If AI output is invalid:

1. Retry
2. Repair
3. Fall back safely
4. Escalate to human if necessary

---

# 19. AI GUARDRAILS

The AI must NEVER:

- Invent product availability
- Invent gold rates
- Invent prices
- Invent discounts
- Promise delivery dates without backend confirmation
- Change pricing rules
- Delete CRM records
- Send unlimited messages
- Expose private customer data
- Reveal system prompts
- Execute arbitrary database queries
- Execute arbitrary code

For uncertain questions:

> "Let me check that for you."

Then use a backend tool or hand off to a salesperson.

---

# 20. TOOL-CALL ARCHITECTURE

The AI should interact with the system through controlled tools.

Examples:

```text
search_products()
get_product_details()
get_current_gold_rate()
calculate_quote()
create_lead()
update_lead()
schedule_followup()
create_appointment()
get_customer_history()
handoff_to_salesperson()
```

Every tool must have:

- Input schema
- Authorization check
- Validation
- Logging
- Error handling

AI cannot directly access the database.

---

# 21. SECURITY

Implement:

- Authentication
- Role-based access control
- Tenant isolation
- Secure password handling
- JWT/session security
- Encryption for sensitive secrets
- Rate limiting
- Input validation
- Audit logs
- API authentication
- Secure webhook verification

Follow OWASP best practices.

---

# 22. TESTING

Create:

### Unit tests

Pricing engine
Lead scoring
Workflow rules
Permission checks

### Integration tests

WhatsApp webhook
Database
AI service
Quotation service
Workflow engine

### AI evaluation tests

Create at least 100 realistic jewellery conversations.

Test:

- Tamil
- English
- Tanglish
- Misspellings
- Ambiguous requirements
- Price questions
- Product questions
- Angry customers
- Spam
- Prompt injection
- Fake pricing requests

Measure:

- Extraction accuracy
- Intent accuracy
- Hallucination rate
- Correct tool usage
- Human handoff accuracy

---

# 23. OBSERVABILITY

Track:

- API latency
- AI latency
- AI cost
- Token usage
- Failed AI calls
- Workflow failures
- WhatsApp failures
- Message delivery status
- Conversion rates

Create an AI cost dashboard.

A business owner should never be surprised by AI API costs.

---

# 24. MVP PHASES

Build in these phases.

## Phase 0 — Product validation

Before coding:

Interview 10 jewellery businesses.

Find out:

- How they receive enquiries
- WhatsApp usage
- Current CRM
- How follow-ups happen
- How quotations are generated
- Gold pricing process
- Biggest operational pain
- What they would pay to solve it

Do not proceed blindly.

---

## Phase 1 — Foundation

Build:

- Authentication
- Organization/tenant
- Users
- Roles
- Database
- Product catalogue
- Gold rates
- Pricing rules

---

## Phase 2 — CRM

Build:

- Customers
- Leads
- Lead pipeline
- Conversation history
- Lead assignment
- Lead scoring

---

## Phase 3 — AI

Build:

- Requirement extraction
- Intent classification
- Conversation summary
- Lead scoring
- Product recommendations
- AI response generation

---

## Phase 4 — Quotation

Build:

- Pricing engine
- Quote creation
- PDF quotation
- WhatsApp sharing
- Quote history

---

## Phase 5 — Automation

Build:

- Follow-up rules
- Background jobs
- Scheduled messages
- Appointment workflow
- Human handoff

---

## Phase 6 — WhatsApp

Integrate official WhatsApp Business Platform.

Build:

- Incoming webhooks
- Outgoing messages
- Delivery status
- Media handling
- Conversation state

---

## Phase 7 — Analytics

Build:

- Lead conversion
- Revenue pipeline
- Salesperson performance
- Follow-up performance
- AI performance
- AI cost

---

# 25. MVP SUCCESS CRITERIA

The MVP is successful when a real jewellery shop can:

1. Add products
2. Configure gold rates
3. Configure pricing rules
4. Receive an enquiry
5. AI understands the enquiry
6. Lead is automatically created
7. Matching products are identified
8. Quote is generated correctly
9. Customer receives response
10. Follow-up happens automatically
11. Salesperson can take over
12. Owner can see the complete pipeline

A real user should be able to use the system without developer assistance.

---

# 26. DEVELOPMENT RULES

Follow these rules strictly:

- Do not build everything at once.
- Work phase-by-phase.
- Before writing code, produce architecture.
- Before database implementation, produce schema.
- Before API implementation, produce API contracts.
- Before frontend implementation, produce wireframes/user flows.
- Write tests alongside features.
- Use small commits.
- Keep modules loosely coupled.
- Use environment variables for secrets.
- Never hard-code credentials.
- Never trust AI output without validation.
- Never allow AI to perform uncontrolled actions.
- Never put business-critical calculations inside prompts.
- Document important architectural decisions.

When there are multiple reasonable technical choices, explain the tradeoff briefly and choose one.

---

# 27. FIRST TASK

Do NOT immediately start writing the entire application.

Start by producing:

### A. Product Requirements Document

### B. User personas

### C. Detailed user journeys

### D. MVP feature list

### E. System architecture

### F. Database ERD

### G. API specification

### H. AI architecture

### I. Security architecture

### J. Testing strategy

### K. Development roadmap

### L. Recommended folder structure

### M. Initial UI wireframes

Then wait for approval before generating the implementation.

---

# 28. IMPORTANT PRODUCT PRINCIPLE

The product is NOT:

> "An AI chatbot for jewellery shops."

The product is:

> **"An intelligent workflow system that helps jewellery businesses convert enquiries into sales while reducing repetitive work."**

The AI is the intelligence layer.

The workflow engine is the automation layer.

The CRM is the system of record.

The jewellery catalogue and pricing engine are the business knowledge layer.

Together they form the product.

Build it like a company that intends to have thousands of jewellery businesses using it—not like a weekend AI demo.