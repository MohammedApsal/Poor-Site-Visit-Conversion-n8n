AI Real Estate Site-Visit Conversion Engine

Turn a buyer's “I’ll visit the property” into a completed site visit with automated booking, navigation, reminders, gate access, attendance tracking, and sales follow-up.

Built with n8n + PostgreSQL + WhatsApp + Google Calendar + Slack.

🚀 Problem

In real estate, getting a buyer interested is only the beginning.

A common operational gap happens after the buyer agrees to visit the property:

“I’ll come tomorrow.”

Then the sales team has to coordinate the appointment, share directions, communicate gate-pass details, remind the buyer, inform the assigned sales representative, track arrival, and follow up afterward.

When these steps depend on manual communication, small missed actions can turn an interested buyer into a missed site visit.

This project automates that operational journey.

💡 Solution

The AI Real Estate Site-Visit Conversion Engine manages the complete site-visit lifecycle.

Flow
Buyer Agrees to Visit
        ↓
Booking Request
        ↓
Validate & Authenticate
        ↓
Idempotency Check
        ↓
Check Lead / Project / Sales Rep
        ↓
Reserve Appointment Slot
        ↓
Generate Visit Assets
        ↓
Google Maps Navigation Link
        ↓
Gate Pass
        ↓
Google Calendar Event
        ↓
Queue Notifications
        ↓
WhatsApp + Email + Sales Rep Alert
        ↓
Automated Reminders
        ↓
Buyer Arrives
        ↓
Gate Check-In
        ↓
Visit Completed
        ↓
Feedback Request
        ↓
Sales Follow-Up

The workflow is divided into dedicated automation processes rather than one large workflow.

🏗️ Architecture

The system contains five major workflows:

Workflow	Responsibility
WF-01 — Booking Orchestrator	Booking, reservation, navigation, calendar and notification queue
WF-02 — Message Dispatcher	Sends WhatsApp, Email and Slack messages
WF-03 — Precision Reminders	24h, 2h, 30m and arrival reminders
WF-04 — Gate Check-In & No-Show Sweeper	Attendance tracking and automatic no-show handling
WF-05 — Post-Visit Follow-Up	Feedback and sales re-engagement

🔄 Workflow 1 — Booking Orchestrator

A booking request enters through an n8n webhook.

POST /sitevisit/v2/book
        ↓
Validate + Authenticate
        ↓
Check Idempotency
        ↓
Get Lead / Project
        ↓
Reserve & Create Booking
        ↓
Build Visit Assets
        ↓
Google Calendar
        ↓
Update Booking State
        ↓
Create Outbox Actions
        ↓
Return Booking Confirmation

The webhook accepts a request_id, lead phone, sales representative, project, and proposed visit time. Authentication and required-field validation are performed before processing.

The system also checks the idempotency table so repeated requests can return the previously generated response instead of creating another booking.

📅 Appointment Reservation

The workflow creates a site-visit record in PostgreSQL and checks whether the assigned representative already has another visit within the defined booking window.

If the slot is unavailable, the API returns:

409 Conflict

with a slot_unavailable response.

📍 Navigation & Gate Pass

After reservation, the workflow generates visit assets including:

Google Maps driving link
Gate-pass code
Project address
Assigned sales representative
Driver/share text

Example navigation URL:

https://www.google.com/maps/dir/?api=1&destination=LAT,LNG&travelmode=driving
📆 Google Calendar

The system creates a calendar event for the scheduled visit and adds the buyer's email as an attendee when available.

The booking record is then updated with the calendar event ID and gate-pass information.

📩 Multi-Channel Messaging

Instead of sending messages directly from the booking flow, notification actions are written to a PostgreSQL outbox.

The system queues:

WhatsApp
Email
CRM / Slack

This separates booking logic from message delivery.

⚙️ Workflow 2 — Message Dispatcher

A scheduled n8n process runs every minute and claims pending outbox actions.

It uses PostgreSQL:

FOR UPDATE SKIP LOCKED

to safely claim pending records without multiple workers processing the same rows concurrently.

Messages are then routed based on channel:

Outbox
   ↓
Route By Channel
   ├── WhatsApp
   ├── Email
   └── Slack

💬 Example WhatsApp Confirmation

The current workflow sends a confirmation containing:

🚗 Site Visit Confirmed

Project
Date & Time
Gate Pass PIN
Sales Manager
Phone Number
Google Maps Navigation

It also supports a DRIVER reply for forwardable driver/cab directions.

🔔 Workflow 3 — Precision Reminders

A scheduled process runs every 10 minutes and detects upcoming visits.

Reminder stages include:

24 Hours Before
       ↓
2 Hours Before
       ↓
30 Minutes Before
       ↓
Arrival Window

Each reminder is queued through the same outbox mechanism.

🚪 Workflow 4 — Gate Check-In & No-Show Detection

When a buyer reaches the property, the system accepts a check-in event through:

POST /sitevisit/checkin

The event can contain:

Visit ID
Gate-pass code
Check-in source

The system verifies the visit and changes its status to:

arrived

while recording the check-in timestamp and method.

No-show handling

Visits that remain in awaiting_checkin more than 30 minutes after the scheduled time are identified and marked as:

no_show

The workflow then creates:

Sales alert
Rep re-engagement task

🔄 Workflow 5 — Post-Visit Follow-Up

Once the buyer arrives, the workflow schedules a feedback request one hour later.

This creates an opportunity to continue the sales process after the physical visit instead of ending automation at check-in.

🗄️ Data Layer

The system uses PostgreSQL for operational state, including:

sitevisits.site_visits
sitevisits.reps
sitevisits.projects
sitevisits.outbox_actions
sitevisits.idempotency_keys

PostgreSQL acts as the system of record for booking state, messaging state and idempotency.

🔐 Reliability Features

The workflow includes several reliability-oriented patterns:

API Authentication

Booking requests can be protected with:

X-API-Key

and required fields are validated before processing.

Idempotency

Repeated requests with the same request ID can return the previously stored response.

Database-backed Outbox

Notifications are persisted before being dispatched, reducing the coupling between booking and external messaging services.

Concurrent Processing

The dispatcher uses row locking with FOR UPDATE SKIP LOCKED.

State-Based Visit Tracking

Example states include:

pending_confirmation
confirmed
awaiting_checkin
arrived
completed
no_show
cancelled
🧰 Tech Stack
Technology	Purpose
n8n	Workflow orchestration
PostgreSQL	Operational database & outbox
WhatsApp	Buyer communication
Google Calendar	Appointment scheduling
Google Maps	Navigation
Slack	Sales team notifications
Webhooks / HTTP	System integrations
JavaScript	n8n transformation & business logic
📂 Project Structure
real-estate-site-visit-engine/
│
├── workflows/
│   └── site-visit-engine.json
│
├── database/
│   ├── schema.sql
│   └── seed.sql
│
├── docs/
│   ├── architecture.md
│   ├── setup.md
│   └── api.md
│
├── screenshots/
│   ├── booking.png
│   ├── dispatcher.png
│   ├── reminders.png
│   └── checkin.png
│
├── examples/
│   └── booking-request.json
│
├── .env.example
├── .gitignore
├── LICENSE
└── README.md
🚀 Setup
1. Install n8n

Run n8n using your preferred deployment method.

For local development:

n8n start
2. Configure PostgreSQL

Create the required database and schema.

Configure credentials in n8n rather than hard-coding secrets in workflow JSON.

3. Configure integrations

Connect:

PostgreSQL
WhatsApp
Google Calendar
Email
Slack

The imported workflow currently contains placeholders such as the WhatsApp phone-number ID, Slack channel ID and sender email, so these must be configured for an actual deployment.

4. Import the workflow

Import the JSON file into n8n and configure the required credentials.

5. Configure environment variables

Example:

BOOKING_API_KEY=your_secure_api_key

Never commit real credentials or API keys to GitHub.

🧪 Example Booking Request
{
  "request_id": "REQ-10001",
  "lead_phone": "9876543210",
  "lead_email": "buyer@example.com",
  "lead_name": "Rahul",
  "rep_id": "REP-001",
  "project_id": "PROJECT-001",
  "proposed_time": "2026-10-01T11:00:00+05:30"
}
📡 Booking API
Book a Visit
POST /sitevisit/v2/book
Required Fields
request_id
lead_phone
rep_id
project_id
proposed_time
Successful Response
{
  "status": "booked",
  "visit_id": "123",
  "gate_pass_code": "VP-ABC123",
  "maps_url": "https://www.google.com/maps/dir/?api=1..."
}

📊 Business Impact

The goal of this automation is not simply to schedule appointments.

It creates an operational bridge between:

Interested Buyer
      ↓
Scheduled Visit
      ↓
Buyer Reaches Property
      ↓
Sales Team Knows Buyer Arrived
      ↓
Follow-Up

That helps reduce the manual coordination required around every site visit.

⚠️ Deployment Notes

This repository contains an automation architecture and workflow implementation. Before treating it as a production deployment, integration credentials, external service configuration, and failure handling should be tested in the target environment.

In particular, configure the WhatsApp phone-number ID, Slack channel, email sender and other environment-specific values before activation.

👨‍💻 Author

Mohammed Apsal M.

AI Automation Specialist
n8n • AI Agents • Workflow Automation • Generative AI

⭐ Project Summary

AI Real Estate Site-Visit Conversion Engine
Automating the journey from “I’ll visit the property” to “I’ve arrived.”

Built to demonstrate how workflow automation can solve a real operational problem in real estate rather than simply automate isolated tasks.
