# CRM Chat & Voicebot Automation

AI-powered customer CRM automation built with **n8n, VAPI, Google Gemini, and Weboxic CRM**.

This project provides two customer-facing channels:

* 💬 AI Chatbot
* 📞 AI Voicebot

Both channels collect the same required customer information and use a shared CRM automation workflow to create or update customer records in Weboxic CRM.

## Architecture

```text
                    CUSTOMER
                       |
              +--------+--------+
              |                 |
           CHAT             VOICE
              |                 |
         n8n Chat           VAPI Assistant
              |                 |
              +--------+--------+
                       |
                       v
              crm_chat_messages
                       |
                       v
                CRM Automation
                       |
                       v
             CRM Data Extractor
                       |
                       v
              Completion Check
                       |
                       v
             CRM Decision Agent
                       |
                       v
          Existing CRM Contacts
                       |
                Mobile / Email
                  Matching
                  /       \
                 /         \
            Existing       New
               |             |
               v             v
        Update Contact   Create Contact
               \             /
                \           /
                 v         v
                Weboxic CRM
```

## Features

* AI-powered customer data collection
* n8n web chat integration
* VAPI voicebot integration
* Shared CRM automation workflow
* Google Gemini-powered data extraction and decision making
* Weboxic CRM integration
* Existing customer detection
* Automatic contact creation
* Automatic contact updates
* Mobile number matching
* Email matching
* Phone number normalization
* Conversation/session storage
* Duplicate contact prevention
* Separate customer-facing and internal CRM automation layers

## Required Customer Details

The system collects exactly these 7 required contact details:

1. First Name
2. Last Name
3. Email
4. Mobile
5. Landline
6. Country
7. State

An optional service requirement can also be captured.

The CRM process only continues when all 7 required fields have been collected.

---

# Chat Workflow

## Customer chat workflow

The chat workflow uses the n8n Chat Trigger and an AI Agent to collect customer information.

### Main components

* `When chat message received`
* `AI Agent`
* `Simple Memory`
* `Insert row`
* `Insert row1`
* `Call n8n Workflow Tool`

The AI Agent:

* Collects the 7 required details.
* Keeps responses short and professional.
* Avoids asking for information already provided.
* Does not request unnecessary personal information.
* Does not invent customer information.
* Triggers CRM Automation after all required details are collected.

The conversation is stored in the `crm_chat_messages` Data Table.

---

# Voice Workflow

## VAPI Voice Assistant

The voice assistant handles the customer conversation independently.

VAPI collects the required customer details and sends the completed structured information to the n8n webhook.

### VAPI → n8n

Production webhook:

```text
POST https://soleprocoder.app.n8n.cloud/webhook/vapi-crm
```

The submitted data contains:

```json
{
  "first_name": "",
  "last_name": "",
  "email": "",
  "mobile": "",
  "landline": "",
  "country": "",
  "state": "",
  "service_requirement": ""
}
```

The VAPI flow does not expose n8n, CRM, webhook, API, or automation details to the customer.

---

# CRM Chat Messages

Both chat and voice workflows use the same n8n Data Table:

```text
crm_chat_messages
```

The table contains:

```text
session_id
role
message
```

The voice workflow stores the structured customer information as a JSON message.

Example:

```json
{
  "first_name": "John",
  "last_name": "Test",
  "email": "test@gmail.com",
  "mobile": "0345516115",
  "landline": "0519285115",
  "country": "Pakistan",
  "state": "Islamabad",
  "service_requirement": "Website development"
}
```

---

# CRM Automation

The shared workflow is:

```text
CRM Automation
```

It is called by both the chat and voice flows.

Input:

```text
sessionId
```

The workflow:

1. Retrieves the customer's conversation.
2. Extracts the required customer details.
3. Checks whether all 7 fields are complete.
4. Retrieves existing Weboxic CRM contacts.
5. Compares the new customer against existing contacts.
6. Creates a new contact if no match exists.
7. Updates the existing contact if a match is found.

---

# CRM Data Extractor

The `CRM Data Extractor` uses Google Gemini to extract:

```text
First Name
Last Name
Email
Mobile
Landline
Country
State
```

Rules include:

* Use only customer-provided information.
* Never guess or invent information.
* Combine information from the complete conversation.
* Use the latest customer-provided value.
* Ignore unrelated information.
* Leave missing values empty.

The extractor also returns:

```text
COMPLETE: YES
```

or:

```text
COMPLETE: NO
```

---

# CRM Matching

The final CRM AI Agent compares the new customer against existing Weboxic CRM contacts.

## Primary match

Mobile number.

Phone comparison ignores:

* spaces
* hyphens
* parentheses
* leading `+`

## Secondary match

Email address.

Email comparison:

* is case-insensitive
* ignores surrounding spaces

A customer is considered an existing customer when either:

```text
Mobile matches
```

or:

```text
Email matches
```

First name and last name are not used alone for matching.

---

# Create Contact

The `Create Contact` tool is called only when no existing contact matches by Mobile or Email.

Endpoint:

```text
POST https://api.weboxic.com/api/v1/crm-contacts
```

Payload:

```json
{
  "first_name": "",
  "last_name": "",
  "email": "",
  "mobile": "",
  "landline": "",
  "country": "",
  "state": ""
}
```

Mobile and landline values are normalized before being sent to Weboxic CRM.

Normalization removes non-numeric characters:

```javascript
.replace(/[^0-9]/g, '')
```

For example:

```text
0345 551 6115
```

becomes:

```text
034555116115
```

---

# Update Contact

The `Update Contact` tool is called only when an existing Weboxic CRM contact has been confirmed.

Endpoint:

```text
PUT https://api.weboxic.com/api/v1/crm-contacts/{existing_contact_id}
```

The `existing_contact_id` must be the actual UUID returned by Weboxic CRM.

The workflow never:

* invents a UUID
* guesses a UUID
* constructs a UUID
* uses a placeholder UUID
* uses `unknown`
* uses `null`
* uses `undefined`

Payload:

```json
{
  "first_name": "",
  "last_name": "",
  "email": "",
  "mobile": "",
  "landline": "",
  "country": "",
  "state": ""
}
```

---

# CRM Decision Logic

```text
New Customer
     |
     v
Compare Mobile
     |
     +---- Match ----> Update Contact
     |
     +---- No Match
              |
              v
         Compare Email
              |
         +---- Match ----> Update Contact
         |
         +---- No Match ----> Create Contact
```

Exactly one CRM action is performed:

```text
Create
```

or:

```text
Update
```

Never both for the same customer.

---

# Security

Weboxic CRM authentication is handled through n8n credentials.

Sensitive credentials must never be committed to Git.

Do not commit:

* API tokens
* Bearer tokens
* passwords
* private keys
* secret credentials

The workflow JSON may contain credential IDs/names, but actual credential values must remain inside n8n's credential system.

---

# Technology Stack

| Technology      | Purpose                               |
| --------------- | ------------------------------------- |
| n8n             | Workflow automation                   |
| VAPI            | Voice AI                              |
| Google Gemini   | AI agents and data extraction         |
| Weboxic CRM     | Customer CRM                          |
| n8n Data Tables | Conversation/session storage          |
| Webhooks        | VAPI → n8n communication              |
| JavaScript      | Data transformation and normalization |

---

# Workflow Structure

## Customer chat workflow

```text
When chat message received
        ↓
AI Agent
        ↓
Insert conversation data
        ↓
CRM Automation Tool
```

## Voice workflow

```text
VAPI
        ↓
Webhook
        ↓
Code in JavaScript
        ↓
Insert row2
        ↓
AI Agent1
        ↓
CRM Automation Tool
        ↓
Respond to Webhook
```

## CRM Automation workflow

```text
When Executed by Another Workflow
        ↓
Get row(s)
        ↓
CRM Data Extractor
        ↓
If
        ↓
Limit
        ↓
CRM Decision Agent
        ↓
HTTP Request
        ↓
Split Out
        ↓
Code in JavaScript
        ↓
Code in JavaScript1
        ↓
AI Agent
       / \
      /   \
Create   Update
Contact  Contact
```

---

# Project Goal

The goal of this automation is to provide a unified AI-powered customer intake system where customers can interact through either chat or voice while the backend automatically manages their CRM records.

The customer-facing AI handles the conversation.

The internal AI agents handle data extraction and CRM decisions.

Weboxic CRM remains the final customer record system.
