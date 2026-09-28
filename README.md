IT Access & Onboarding Agent

An automated workflow built in n8n that reads employee transition requests from a Google Form, uses Google Gemini to classify the request and decide the IT access level, and sends the right notification email through Gmail. No manual triage is needed.

Show Image

Table of Contents
1. Problem Statement
2. Solution Overview
3. Architecture
4. Tech Stack
5. How It Works
6. AI Output Schema
7. Routing Rules
8. Setup Guide
9. Sample Run
10. Troubleshooting
11. Limitations and Future Improvements
12. Project Documents
13. Author

Problem Statement

In many companies, IT staff read every access request by hand and decide what access a person needs. This is slow and inconsistent:

--Access levels (standard vs. elevated) are judged case by case.

--New hires and role changers wait for IT to read and route their request.

--The form has only two event types (new hire, role change), but a role change may really be a technical promotion or a department move, and someone has to sort it by hand.

--IT and managers each get hand-written emails for every request.

Solution Overview

When the form is submitted, the workflow:

--Receives the response through a webhook.

--Sends the details to an AI Agent (Google Gemini).

--Classifies the request into one of four transition types.

--Decides whether the person needs standard or elevated access.

--Writes a short summary and picks up the manager's email.

--Routes the request with a Switch node and sends the matching Gmail message.

Architecture

Google Form
    |
Google Apps Script (on form submit)
    |
n8n Webhook (POST)
    |
AI Agent  <--  Google Gemini Chat Model
    |     <--  Structured Output Parser
    |
Switch (4 rules on transition_type)
    |--- new_hire             -> Gmail: Send a message
    |--- role_change          -> Gmail: Send a message1
    |--- technical_promotion  -> Gmail: Send a message2
    |--- department_change    -> Gmail: Send a message3

Tech Stack

Tool	                              Role

Google Forms	 ----     Captures the employee transition request

Google Apps     ----    Script	Posts the form response to n8n as JSON

n8n Cloud	       ----   Workflow orchestration and hosting

Google Gemini	   ----   LLM used for reasoning and classification

n8n AI Agent node	 ---- Analyses the request and drafts the summary

Structured Output Parser  ---- 	Forces the AI answer into a fixed JSON format

Switch node	   ----     Rule-based routing on transition_type

Gmail node	   ----    Sends the notification email


How It Works

Form fields

Field	Description
Employee name ----	Person joining or changing role

Employee Email ----	The employee's email address

Event type ----	New hire or Role change

Role / Job Title ----	The new role

Department ----	The department joined or moved to

Current Role ----	The present role (left blank for a new hire)

Manager Email ----	Reporting manager's email

Start Date ----	When the change takes effect

Requested Access Level ----	Standard or Elevated, as requested by the submitter


AI Agent

--Prompt: built from the webhook body, for example Employee Name: {{ $json.body["Employee name"] }}.

--System message: tells the agent it is an IT onboarding assistant, to read the request, and to return the required JSON fields without outputting schema definitions or code templates.

--Model: Google Gemini Chat Model.

--Output format: enforced by the Structured Output Parser.

Notes

The form only offers two event types. The AI decides whether a role change is really a technical_promotion or a department_change by reading the current role, new role and department.
The requested access level is passed to the AI, but the access_level in the output is decided by the rule in the schema (role and department). Treat the form choice as the requested level and the AI result as the recommended level.
AI Output Schema

The Structured Output Parser uses this JSON schema:

json
{
  "type": "object",
  "properties": {
    "transition_type": {
      "type": "string",
      "enum": ["new_hire", "role_change", "technical_promotion", "department_change"],
      "description": "Classify the request. new_hire for brand new employees. role_change for same-level moves. technical_promotion or department_change when moving to engineering, data, or a different team requiring new tool bundles."
    },
    "access_level": {
      "type": "string",
      "enum": ["standard", "elevated"],
      "description": "Choose 'elevated' if moving to IT, engineering, or a technical role requiring admin tools. Choose 'standard' for regular office or business roles."
    },
    "summary": {
      "type": "string",
      "description": "Summary of the transition details."
    },
    "manager_email": {
      "type": "string",
      "description": "Manager email address."
    }
  },
  "required": ["transition_type", "access_level", "summary", "manager_email"]
}

Routing Rules

The Switch node runs in Rules mode and checks {{ $json.output.transition_type }}.

Rule	Equals	Output name	Gmail node

1	new_hire	New Hire Onboarding	Send a message

2	role_change	Role Change Update	Send a message1

3	technical_promotion	Technical Promotion & IT Provisioning	Send a message2

4	department_change	Department Change & Tool Upgrade	Send a message3

Setup Guide

1. Create the Google Form

Add the five fields listed above. Keep the field names the same, because the AI Agent prompt reads them by name.

2. Create the n8n workflow

--Add a Webhook node (method POST).

--Add an AI Agent node and set Source for Prompt to Define below. Turn on Require Specific Output Format.

--Connect a Google Gemini Chat Model and add your Gemini credentials.

--Connect a Structured Output Parser and paste the schema above.

--Add a Switch node with the four rules above.

--Add four Gmail nodes, one per Switch output, and connect your Google account.

--Set the workflow to Active.

3. Connect the form to n8n

In the form, open Extensions > Apps Script and add a script like this example:

javascript

function onFormSubmit(e) {
  
  var payload = {};
  
  for (var key in e.namedValues) {
    payload[key] = e.namedValues[key][0];
  }
  
  UrlFetchApp.fetch("YOUR_N8N_PRODUCTION_WEBHOOK_URL", {
    method: "post",
    contentType: "application/json",
    payload: JSON.stringify(payload)
  });
}

Then add an installable trigger (Triggers > Add Trigger) that runs onFormSubmit on form submit.

Use the production webhook URL, not the test URL. Do not commit your real webhook URL or credentials to GitHub.

Sample Run

Input

Field	Value
Employee name	Kavya Sharma
Event type	New hire
Role / Job Title	IT Support
Department	IT
Manager Email	manager.it@company.com
Start Date	9/29/2026

AI output

json
{
  "transition_type": "new_hire",
  "access_level": "elevated",
  "summary": "Kavya Sharma is joining the IT department as a new hire in the IT Support role, reporting to manager.it@company.com with a start date of 9/29/2026.",
  "manager_email": "manager.it@company.com"
}

Result: the run succeeded in about 30 seconds using 656 tokens. The Switch routed it to the New Hire Onboarding branch and the Gmail node sent the email.

Troubleshooting

Problem	                ----                         What to check

Form submitted but nothing happens	---- Apps Script trigger, production webhook URL, workflow set to Active

AI step fails	---- Gemini credentials and API usage limit

Output format error	---- Schema matches the prompt. If Auto-Fix Format is on, the parser needs a model connected

No Gmail branch runs	---- Compare transition_type in the execution log with the four Switch rules

Email not received	---- Gmail credential, recipient address, spam folder

Wrong classification	---- Enter a specific role and department, and review elevated access before granting it

Limitations and Future Improvements

1.The AI makes the final classification, so unclear input can give a wrong result. Review elevated access before granting it.

2.The AI sees the requested access level, so a submitter's wording or choice can influence its decision. Add a line to the system message such as "Decide access_level from role and department only, and mention in the summary if it differs from the requested level", and review elevated access before granting it.

3.The form collects the current role but not the current department. Adding a "Current Department" field would help separate a department_change from a technical_promotion.

4.The line between technical_promotion and department_change is left to the model. The schema description could be made more specific.

5.Possible extensions: create tickets automatically, log every request in a Google Sheet, add a Slack notification.

Project Documents

1.docs/User Guide.pdf: user guide

2.docs/Presentation.pptx: project presentation

3.docs/Explanation.pdf: slide-by-slide explanation

4.workflow/it-access-onboarding-agent.json: exported n8n workflow (remove credentials before uploading)

5.images/: workflow canvas and node screenshots

Author

Komal

1.GitHub: github.com/Komal240501

2.LinkedIn: linkedin.com/in/komal-lunawat-a1886b352
