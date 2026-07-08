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

## 1. HRMS Orchestrator

**Trigger:** a SharePoint employee/position item is created or modified.
**Role:** the lifecycle state machine for the whole system. It branches first on
employment type (on-roll vs off-roll), then on lifecycle status, and performs the right
action — approvals, document generation, vacancy creation, or notifications.

**Connectors:** SharePoint · Approvals · Outlook (shared mailbox) · Word Online ·
OneDrive · Office 365 Users.

```mermaid
flowchart TD
    T["Trigger: employee / position record<br/>created or modified"] --> R{On-roll or Off-roll?}

    R -->|On-roll| ON{Lifecycle status?}
    ON -->|Resigned| ON1["Collect exit docs NOC / NDC / exit process<br/>&rarr; email exit mailer to HR + payroll<br/>&rarr; create vacant position<br/>&rarr; compute notice-period details"]
    ON -->|Exited| ON2["Create vacant position"]
    ON -->|Offer initiated| ON3["Compose candidate details + salary breakup<br/>&rarr; start &amp; wait for approval"]
    ON3 --> ON3a{Approved?}
    ON3a -->|Yes| ON3y["Update record as approved<br/>&rarr; send offer email"]
    ON3a -->|No| ON3n["Update record as rejected"]
    ON3y --> ON3z["Send offer-alert notification"]
    ON3n --> ON3z
    ON -->|Offer rejected| ON4["Create vacant position"]
    ON -->|Accepted + joined| ON5["Email joining confirmation (CSV summary)<br/>&rarr; populate Word template &rarr; convert to PDF<br/>&rarr; email IT for mail-ID creation<br/>&rarr; look up reporting manager"]

    R -->|Off-roll| OFF{Lifecycle status?}
    OFF -->|Resigned| OFF1["Create vacant position"]
    OFF -->|Exited| OFF2["Create vacant position"]
    OFF -->|Offered| OFF3["Create SharePoint document set<br/>&rarr; copy form attachments via HTTP to SharePoint<br/>&rarr; notify"]
    OFF -->|Joined + active| OFF4["Send welcome / announcement adaptive card"]
```

---

## 2. Candidate Document Approval

**Trigger:** a file is created or modified in the candidate-documents library.
**Role:** routes each uploaded document through an approval, then stamps SharePoint's
content-approval status and notifies on rejection.

**Connectors:** SharePoint · Approvals · Outlook (shared mailbox).

```mermaid
flowchart TD
    T["Trigger: file created / modified<br/>in document library"] --> C{Pending review?}
    C -->|Yes| A["Start &amp; wait for approval"]
    A --> D{Approved?}
    D -->|Yes| Y["Set content-approval status = Approved"]
    D -->|No| N["Set status = Rejected<br/>&rarr; email uploader via shared mailbox"]
    C -->|No| Z["Compose folder path, shareable link,<br/>leaf name &amp; mail ID"]
    Y --> Z
    N --> Z
```

---

## 3. Mail-ID & Onboarding

**Trigger:** a new-joiner alert email arrives in a shared mailbox.
**Role:** turns an inbound joiner email into the downstream HRIS import files and the
mail-ID onboarding request.

**Connectors:** SharePoint · Outlook · HTML-to-text conversion.

```mermaid
flowchart TD
    T["Trigger: new joiner alert email"] --> P["HTML &rarr; text, normalize body<br/>compose employee ID + mail ID"]
    P --> F["Filter to the matching joiner"]
    F --> G["Get record from headcount/budget list<br/>+ candidate details"]
    G --> GEN["Generate HRIS import templates as CSV:<br/>biographical · person-info · employment history<br/>job history · e-code UDF"]
    GEN --> CF["Create the CSV files in SharePoint"]
    CF --> M["Email the onboarding package (shared mailbox)"]
    M --> U["Update the record status"]
```

---

## 4. Off-roll Resignation Tracker

**Trigger:** a resignation alert email from the staffing vendor.
**Role:** keeps the headcount/budget list current for contract (off-roll) staff without
manual entry.

**Connectors:** SharePoint · Outlook · HTML-to-text conversion.

```mermaid
flowchart TD
    T["Trigger: vendor resignation email"] --> P["HTML &rarr; text<br/>extract employee ID + date of resignation<br/>reformat date"]
    P --> G["Get matching rows from headcount/budget list"]
    G --> L["For each match"]
    L --> C{Record found?}
    C -->|Yes| U["Update item: mark resignation + date"]
```

---

## 5. Attendance Tracking

**Trigger:** an attendance-data email arrives.
**Role:** parses the attendance payload into a SharePoint list and confirms back to the
sender with an adaptive card.

**Connectors:** SharePoint · Outlook.

```mermaid
flowchart TD
    T["Trigger: attendance data email"] --> P["Compose table details + chunk the payload<br/>compute dynamic column counts"]
    P --> L["For each chunk"]
    L --> G["Get list items"]
    G --> L2["For each row"]
    L2 --> CI["Create attendance item"]
    CI --> C{Write ok?}
    C -->|Yes| R["Reply to email + update item<br/>&rarr; compose confirmation card (ID / Name / HQ)"]
```

---

## 6. Capture HTTP Request

**Trigger:** an HTTP request (invoked from the Power Apps front end).
**Role:** a server-side endpoint that lets the app update a record and send a
confirmation email in one call.

**Connectors:** SharePoint · Outlook.

```mermaid
flowchart TD
    T["Trigger: HTTP request received"] --> RS["Respond to caller"]
    RS --> C["Compose Name + ID from payload"]
    C --> G["Get matching list items"]
    G --> L["For each match"]
    L --> U["Update item"]
    U --> E["Send confirmation email"]
```

---

## 7. Capture Joining-Kit Details

**Trigger:** an HTTP request (invoked from the Power Apps front end).
**Role:** a lightweight endpoint that captures joining-kit form details and emails a
confirmation.

**Connectors:** Outlook.

```mermaid
flowchart TD
    T["Trigger: HTTP request received"] --> C["Compose joining-kit details from payload"]
    C --> RS["Respond to caller"]
    RS --> E["Send confirmation email"]
```
