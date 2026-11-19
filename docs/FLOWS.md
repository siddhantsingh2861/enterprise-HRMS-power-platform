# Flows — detailed breakdown

Seven Power Automate flows make up the HRMS automation layer. Each section below gives
the trigger, what the flow does, and a diagram. All names, mailboxes, and lists are
described by role/function; identifiers and data have been removed.

- [1. HRMS Orchestrator](#1-hrms-orchestrator)
- [2. Candidate Document Approval](#2-candidate-document-approval)
- [3. Mail-ID & Onboarding](#3-mail-id--onboarding)
- [4. Off-roll Resignation Tracker](#4-off-roll-resignation-tracker)
- [5. Attendance Tracking](#5-attendance-tracking)
- [6. Capture HTTP Request](#6-capture-http-request)
- [7. Capture Joining-Kit Details](#7-capture-joining-kit-details)

---

## How to read the diagrams

| Shape / colour | Meaning |
| --- | --- |
| Blue pill | Trigger — what starts the flow |
| Yellow diamond | Decision / condition |
| Grey box | Processing step (compose, parse, look up) |
| Purple cylinder | Write to SharePoint (create / update record) |
| Green box | Outbound message — email, adaptive card, notification |
| Dashed pill | Branch ends with no action |
| Dashed frame | Loop (*Apply to each*) — steps inside repeat per item |

Mermaid source for every diagram lives in [`docs/diagrams/`](diagrams/).

---

## 1. HRMS Orchestrator

**Trigger:** a SharePoint employee/position item is created or modified.
**Role:** the lifecycle state machine for the whole system. It branches first on
employment type (on-roll vs off-roll), then on lifecycle status, and performs the right
action — approvals, document generation, vacancy creation, or notifications.

**Connectors:** SharePoint · Approvals · Outlook (shared mailbox) · Word Online ·
OneDrive · Office 365 Users.

```mermaid
flowchart TD
    T(["Employee / position record<br/>created or modified"]):::trigger --> R{"On-roll or<br/>off-roll?"}:::decision

    R -->|On-roll| ON
    R -->|Off-roll| OFF

    subgraph ONROLL[" "]
        ON{"Lifecycle<br/>status?"}:::decision

        ON -->|Resigned| ON1["Collect exit docs<br/>NOC · NDC · exit process"]:::action
        ON1 --> ON1b("Email exit mailer<br/>to HR + payroll"):::notify
        ON1b --> ON1c[("Create vacant position")]:::data
        ON1c --> ON1d["Compute notice-period details"]:::action

        ON -->|"Exited / offer rejected"| VAC1[("Create vacant position")]:::data

        ON -->|Offer initiated| ON3["Compose candidate details<br/>+ salary breakup"]:::action
        ON3 --> ON3w["Start &amp; wait for approval"]:::action
        ON3w --> ON3a{"Approved?"}:::decision
        ON3a -->|Yes| ON3y[("Mark record approved")]:::data
        ON3y --> ON3e("Send offer email"):::notify
        ON3a -->|No| ON3n[("Mark record rejected")]:::data
        ON3e --> ON3z("Send offer-alert notification"):::notify
        ON3n --> ON3z

        ON -->|Accepted + joined| ON5("Email joining confirmation<br/>CSV summary"):::notify
        ON5 --> ON5b["Populate Word template<br/>→ convert to PDF"]:::action
        ON5b --> ON5c("Email IT for<br/>mail-ID creation"):::notify
        ON5c --> ON5d["Look up reporting manager"]:::action
    end

    subgraph OFFROLL[" "]
        OFF{"Lifecycle<br/>status?"}:::decision

        OFF -->|"Resigned / exited"| VAC2[("Create vacant position")]:::data

        OFF -->|Offered| OFF3[("Create SharePoint<br/>document set")]:::data
        OFF3 --> OFF3b["Copy form attachments<br/>to SharePoint via HTTP"]:::action
        OFF3b --> OFF3c("Notify"):::notify

        OFF -->|Joined + active| OFF4("Send welcome /<br/>announcement adaptive card"):::notify
    end

    style ONROLL fill:none,stroke:#94a3b8,stroke-dasharray:6 4
    style OFFROLL fill:none,stroke:#94a3b8,stroke-dasharray:6 4

    classDef trigger fill:#dbeafe,stroke:#1d4ed8,color:#0f172a
    classDef decision fill:#fef3c7,stroke:#b45309,color:#0f172a
    classDef action fill:#f8fafc,stroke:#64748b,color:#0f172a
    classDef data fill:#ede9fe,stroke:#6d28d9,color:#0f172a
    classDef notify fill:#dcfce7,stroke:#15803d,color:#0f172a
    classDef noop fill:#ffffff,stroke:#94a3b8,stroke-dasharray:4 3,color:#64748b
```

---

## 2. Candidate Document Approval

**Trigger:** a file is created or modified in the candidate-documents library.
**Role:** routes each uploaded document through an approval, then stamps SharePoint's
content-approval status and notifies on rejection.

**Connectors:** SharePoint · Approvals · Outlook (shared mailbox).

```mermaid
flowchart TD
    T(["File created / modified<br/>in candidate-documents library"]):::trigger --> C{"Pending<br/>review?"}:::decision
    C -->|Yes| A["Start &amp; wait for approval"]:::action
    A --> D{"Approved?"}:::decision
    D -->|Yes| Y[("Set content-approval<br/>status = Approved")]:::data
    D -->|No| N[("Set content-approval<br/>status = Rejected")]:::data
    N --> NM("Email uploader<br/>via shared mailbox"):::notify
    C -->|No| Z["Compose folder path, shareable link,<br/>leaf name &amp; mail ID"]:::action
    Y --> Z
    NM --> Z

    classDef trigger fill:#dbeafe,stroke:#1d4ed8,color:#0f172a
    classDef decision fill:#fef3c7,stroke:#b45309,color:#0f172a
    classDef action fill:#f8fafc,stroke:#64748b,color:#0f172a
    classDef data fill:#ede9fe,stroke:#6d28d9,color:#0f172a
    classDef notify fill:#dcfce7,stroke:#15803d,color:#0f172a
    classDef noop fill:#ffffff,stroke:#94a3b8,stroke-dasharray:4 3,color:#64748b
```

---

## 3. Mail-ID & Onboarding

**Trigger:** a new-joiner alert email arrives in a shared mailbox.
**Role:** turns an inbound joiner email into the downstream HRIS import files and the
mail-ID onboarding request.

**Connectors:** SharePoint · Outlook · HTML-to-text conversion.

```mermaid
flowchart TD
    T(["New-joiner alert email"]):::trigger --> P["HTML → text, normalize body<br/>compose employee ID + mail ID"]:::action
    P --> F["Filter to the matching joiner"]:::action
    F --> G["Get record from headcount/budget list<br/>+ candidate details"]:::action
    G --> CSV["Generate HRIS import templates as CSV<br/>biographical · person info · employment history<br/>job history · e-code UDF"]:::action
    CSV --> CF[("Create the CSV files<br/>in SharePoint")]:::data
    CF --> M("Email onboarding package<br/>from shared mailbox"):::notify
    M --> U[("Update record status")]:::data


    classDef trigger fill:#dbeafe,stroke:#1d4ed8,color:#0f172a
    classDef decision fill:#fef3c7,stroke:#b45309,color:#0f172a
    classDef action fill:#f8fafc,stroke:#64748b,color:#0f172a
    classDef data fill:#ede9fe,stroke:#6d28d9,color:#0f172a
    classDef notify fill:#dcfce7,stroke:#15803d,color:#0f172a
    classDef noop fill:#ffffff,stroke:#94a3b8,stroke-dasharray:4 3,color:#64748b
```

---

## 4. Off-roll Resignation Tracker

**Trigger:** a resignation alert email from the staffing vendor.
**Role:** keeps the headcount/budget list current for contract (off-roll) staff without
manual entry.

**Connectors:** SharePoint · Outlook · HTML-to-text conversion.

```mermaid
flowchart TD
    T(["Resignation alert email<br/>from staffing vendor"]):::trigger --> P["HTML → text<br/>extract employee ID + resignation date<br/>reformat date"]:::action
    P --> G["Get matching rows<br/>from headcount/budget list"]:::action
    G --> C

    subgraph LOOP["Apply to each match"]
        C{"Record<br/>found?"}:::decision
        C -->|Yes| U[("Mark resignation + date")]:::data
        C -->|No| S(["No change"]):::noop
    end

    style LOOP fill:none,stroke:#94a3b8,stroke-dasharray:6 4

    classDef trigger fill:#dbeafe,stroke:#1d4ed8,color:#0f172a
    classDef decision fill:#fef3c7,stroke:#b45309,color:#0f172a
    classDef action fill:#f8fafc,stroke:#64748b,color:#0f172a
    classDef data fill:#ede9fe,stroke:#6d28d9,color:#0f172a
    classDef notify fill:#dcfce7,stroke:#15803d,color:#0f172a
    classDef noop fill:#ffffff,stroke:#94a3b8,stroke-dasharray:4 3,color:#64748b
```

---

## 5. Attendance Tracking

**Trigger:** an attendance-data email arrives.
**Role:** parses the attendance payload into a SharePoint list and confirms back to the
sender with an adaptive card.

**Connectors:** SharePoint · Outlook.

```mermaid
flowchart TD
    T(["Attendance data email"]):::trigger --> P["Compose table details<br/>chunk the payload · compute column counts"]:::action
    P --> G

    subgraph CHUNK["Apply to each chunk"]
        G["Get list items"]:::action
        G --> CI

        subgraph ROW["Apply to each row"]
            CI[("Create attendance item")]:::data
            CI --> C{"Write ok?"}:::decision
            C -->|Yes| R("Reply to email<br/>+ update item"):::notify
            R --> CC["Compose confirmation card<br/>ID · Name · HQ"]:::action
            C -->|No| S(["No action"]):::noop
        end
    end

    style CHUNK fill:none,stroke:#94a3b8,stroke-dasharray:6 4
    style ROW fill:none,stroke:#94a3b8,stroke-dasharray:6 4

    classDef trigger fill:#dbeafe,stroke:#1d4ed8,color:#0f172a
    classDef decision fill:#fef3c7,stroke:#b45309,color:#0f172a
    classDef action fill:#f8fafc,stroke:#64748b,color:#0f172a
    classDef data fill:#ede9fe,stroke:#6d28d9,color:#0f172a
    classDef notify fill:#dcfce7,stroke:#15803d,color:#0f172a
    classDef noop fill:#ffffff,stroke:#94a3b8,stroke-dasharray:4 3,color:#64748b
```

---

## 6. Capture HTTP Request

**Trigger:** an HTTP request (invoked from the Power Apps front end).
**Role:** a server-side endpoint that lets the app update a record and send a
confirmation email in one call.

**Connectors:** SharePoint · Outlook.

```mermaid
flowchart TD
    T(["HTTP request<br/>from the Power App"]):::trigger --> RS["Respond to caller"]:::action
    RS --> C["Compose name + ID<br/>from payload"]:::action
    C --> G["Get matching list items"]:::action
    G --> U

    subgraph LOOP["Apply to each match"]
        U[("Update item")]:::data
        U --> E("Send confirmation email"):::notify
    end

    style LOOP fill:none,stroke:#94a3b8,stroke-dasharray:6 4

    classDef trigger fill:#dbeafe,stroke:#1d4ed8,color:#0f172a
    classDef decision fill:#fef3c7,stroke:#b45309,color:#0f172a
    classDef action fill:#f8fafc,stroke:#64748b,color:#0f172a
    classDef data fill:#ede9fe,stroke:#6d28d9,color:#0f172a
    classDef notify fill:#dcfce7,stroke:#15803d,color:#0f172a
    classDef noop fill:#ffffff,stroke:#94a3b8,stroke-dasharray:4 3,color:#64748b
```

---

## 7. Capture Joining-Kit Details

**Trigger:** an HTTP request (invoked from the Power Apps front end).
**Role:** a lightweight endpoint that captures joining-kit form details and emails a
confirmation.

**Connectors:** Outlook.

```mermaid
flowchart TD
    T(["HTTP request<br/>from the Power App"]):::trigger --> C["Compose joining-kit details<br/>from payload"]:::action
    C --> RS["Respond to caller"]:::action
    RS --> E("Send confirmation email"):::notify

    classDef trigger fill:#dbeafe,stroke:#1d4ed8,color:#0f172a
    classDef decision fill:#fef3c7,stroke:#b45309,color:#0f172a
    classDef action fill:#f8fafc,stroke:#64748b,color:#0f172a
    classDef data fill:#ede9fe,stroke:#6d28d9,color:#0f172a
    classDef notify fill:#dcfce7,stroke:#15803d,color:#0f172a
    classDef noop fill:#ffffff,stroke:#94a3b8,stroke-dasharray:4 3,color:#64748b
```
