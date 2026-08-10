# Hyde Salesforce KT Summary

## Overview
This document consolidates KT sessions from:
- 30-Mar-2026
- 01-Apr-2026
- 08-Apr-2026
- 10-Apr-2026

Sources: Knowledge Sharing sessions. References: turn1search2, turn1search1, turn1search3, turn2search1.

---

# 1. Hyde Salesforce Architecture

## Core Platforms
- Salesforce Sales Cloud
- Service Cloud
- Marketing Cloud
- Experience Cloud (My Account Portal)
- Northgate (NEC) Legacy Housing CRM
- Total Mobile (Repairs)
- AllPay (Payments)
- DIPA Integration Platform

```mermaid
flowchart LR
    NEC[Northgate NEC] --> DIPA[DIPA Integration Layer]
    DIPA --> SF[Salesforce]
    SF --> Portal[My Account Portal]
    SF --> MC[Marketing Cloud]
    SF --> TM[Total Mobile]
    TM --> DIPA
    DIPA --> DataLake[Data Lake]
    DIPA --> Workday[Workday]
```

---

# 2. Core Salesforce Data Model

## Person Account Centric Model

```mermaid
erDiagram
    PERSON_ACCOUNT ||--o{ CASE : creates
    PERSON_ACCOUNT ||--o{ FINANCIAL_ACCOUNT : owns
    PERSON_ACCOUNT ||--o{ AGREEMENT : has
    AGREEMENT }o--|| PROPERTY : relates_to
    PROPERTY }o--|| ADMIN_GROUPING : grouped_under
    PERSON_ACCOUNT ||--o{ DISABILITY : records
    PERSON_ACCOUNT ||--o{ WORK_ORDER : raises
```

### Key Objects
- Person Account (Customer/Tenant)
- Occupancy Agreement
- Property
- Administrative Grouping
- Financial Account
- Cases
- Work Orders
- Disabilities & Vulnerabilities
- Person Notes

---

# 3. Customer Journey

```mermaid
flowchart TD
    Tenant --> Portal
    Portal --> Case
    Portal --> Repair
    Portal --> Payment
    Repair --> WorkOrder
    WorkOrder --> TotalMobile
    Payment --> AllPay
```

---

# 4. Northgate and Salesforce Relationship

## Current State

```mermaid
flowchart LR
    NEC[Northgate] --> Customer
    Customer --> Salesforce
```

### Important Notes
- Customers are initially created in Northgate.
- Salesforce receives customer data via DIPA.
- Salesforce aims to become the future customer master system.
- Tenancy remains tightly coupled with Northgate.

## Target Future State

```mermaid
flowchart LR
    Salesforce --> CustomerOnboarding
    CustomerOnboarding --> NEC
```

---

# 5. Testing Framework (Story 1542)

## Business Problem
QA automation requires dynamic customer segmentation.

### Old Approach
```text
IsTHCH = True/False
```

### New Approach
```text
Belongs To
- THCH
- Clove
- River
- Others
```

```mermaid
flowchart LR
    Customer --> Picklist[Belongs To]
    Picklist --> THCH
    Picklist --> Clove
    Picklist --> River
    THCH --> Automation
    Clove --> Automation
    River --> Automation
```

Benefits:
- Dynamic queries
- Real production-like data
- Better automated testing

---

# 6. Security Monitoring (Story 1541)

```mermaid
flowchart LR
    Salesforce --> Events[Event Monitoring]
    Events --> Taegis
    Taegis --> Dashboards
    Dashboards --> SecurityTeam
```

Purpose:
- Monitor session hijacking.
- Detect suspicious behaviour.
- Enable future automated security controls.

---

# 7. Connected Apps & Integrations

Preferred authentication pattern:

```mermaid
sequenceDiagram
    participant App
    participant Salesforce

    App->>Salesforce: JWT Authentication
    Salesforce-->>App: Access Token
    App->>Salesforce: API Request
    Salesforce-->>App: Response
```

---

# 8. AllPay Payment Integration

## Payment Flow

```mermaid
sequenceDiagram
    participant Customer
    participant Portal
    participant AllPay
    participant NEC
    participant Salesforce

    Customer->>Portal: Enter Amount
    Portal->>AllPay: Open Iframe
    AllPay->>NEC: Payment Processing
    NEC->>Salesforce: Transaction Update
    Salesforce-->>Customer: Updated Payment Status
```

## Key Issues Solved

### iOS Issue
- Fixed by adding AllPay domains to Mobile Publisher allowed sites.

### Payment Status Issue
- AllPay was not consistently sending success/failure/cancel states.
- Alignment undertaken with AllPay team.

---

# 9. Marketing Cloud Integration

## Sources of Case Creation

```mermaid
flowchart TD
    Agent[Agent Console] --> SalesforceCase
    Portal[My Account Portal] --> SalesforceCase
    MarketingCloud --> API
    API --> SalesforceCase
```

Supported use cases:
- Name Change
- Citizenship Change
- Disability Update
- Vulnerability Update

---

# 10. Sandbox Strategy & CI/CD

## Environment Flow

```mermaid
flowchart LR
    DevSandbox --> Dev
    Dev --> INT
    INT --> UAT
    UAT --> Training
    Training --> Production
```

### Release Process

```mermaid
flowchart TD
    Story --> Development
    Development --> PR
    PR --> QA
    QA --> UAT
    UAT --> CAB
    CAB --> Production
```

Key Notes:
- Deployments every two weeks.
- CAB approval required.
- Hotfixes supported.

---

# 11. Sandbox Refresh Programme

Goals:
- Branch synchronization.
- Metadata cleanup.
- Data anonymisation.
- Reduction of manual deployment steps.

## Data Anonymisation

```mermaid
flowchart LR
    ProductionData --> HashFunction
    HashFunction --> AnonymizedData
    AnonymizedData --> UAT
```

Masked fields:
- Email
- Phone
- Mobile

---

# 12. Customer Organisation (B2B) POC

## Current Model

```mermaid
flowchart LR
    Tenant --> PersonAccount
```

## Proposed B2B Model

```mermaid
flowchart TD
    Organization[Customer Organization]
    Organization --> Contact1
    Organization --> Contact2
    Organization --> Contact3
    Organization --> Cases
```

### Why It Was Built
- Some customers are organisations rather than tenants.
- Person Accounts are not suitable.
- Standard Account + Contact model required.

### Why It Was Put On Hold
- Source data in NEC is not truly B2B.
- Requires large-scale data cleanup.

### Changes Implemented
- New Customer Organisation record type.
- New layouts.
- New FlexiPages.
- Flow modifications.
- Portal logic exclusions.

---

# 13. Data Team & DIPA

## Ownership
- Led by Satya.
- Responsible for integration platform.

```mermaid
flowchart LR
    Salesforce --> DIPA
    DIPA --> NEC
    DIPA --> TMC
    DIPA --> Workday
    DIPA --> DataLake
```

The Data Team is one of the most important teams the Salesforce developers interact with.

---

# 14. Key Things To Learn First

## Priority 1
- Person Accounts
- Properties
- Occupancy Agreements
- Financial Accounts
- Administrative Groupings

## Priority 2
- Case Management
- Repairs
- Work Orders
- Customer 360

## Priority 3
- DIPA
- Northgate Integration
- Total Mobile
- AllPay

## Priority 4
- Git Strategy
- Deployments
- CAB Process
- Sandbox Refresh

## Priority 5
- Marketing Cloud
- Connected Apps
- JWT
- Event Monitoring

---

# Executive Summary

Hyde's Salesforce platform is a housing-focused CRM built around Person Accounts and deeply integrated with Northgate (housing management), DIPA (integration middleware), Total Mobile (repairs), AllPay (payments), Marketing Cloud (customer engagement), and Experience Cloud portals. The strategic direction is to move customer mastery into Salesforce while retaining tenancy operations within Northgate until future transformation programmes are completed.
