# AI-Powered Multi-Agent Business Development System

An end-to-end multi-agent workflow, built with **n8n**, **Google Gemini**, and **ElevenLabs**, that takes a lead from raw prospecting all the way to a booked demo call — with no manual handoffs.

## Overview

This project orchestrates **five specialized AI subagents** across two connected workflows: an outbound business development pipeline and a voice-driven account executive layer. A manager agent coordinates the outbound side; a conversational voice agent handles the inbound call and booking.

The system starts from a single input — an Ideal Customer Profile (ICP) document — and ends with a scheduled demo on Google Calendar, triggered entirely by a phone call.

## Architecture

### 1. Business Development Manager (Orchestrator)

- **Trigger:** New file created in Google Drive (the ICP document, as PDF)
- **Flow:** Downloads the file, extracts its content, and passes it to an AI Agent (Google Gemini) that runs through a fixed sequence of subagent calls
- **Output:** Sends a Telegram notification reporting success or failure at the end of the run

The manager calls three subagents, in order, each exposed as a tool:

| Subagent | Responsibility | Tools used |
|---|---|---|
| **Prospecting** | Searches the web for leads matching the ICP | Firecrawl (web scraping), Hunter (email finding/verification) |
| **RevOps (Revenue Operations)** | Creates the corresponding CRM records | Pipedrive (create organization → create person → create lead) |
| **SDR (Sales Development Representative)** | Retrieves the newly created contacts and drafts outreach | Pipedrive (get persons), Gmail (create draft) |

Each subagent is a **separate n8n workflow**, called as a tool by the orchestrator's AI Agent — not a hardcoded linear chain of nodes. This keeps responsibilities isolated and each subagent independently testable.

### 2. Account Executive Layer (Voice Agent)

Once a lead receives the outreach email and calls back, an **ElevenLabs conversational agent** answers the call and handles the booking flow. It also follows a fixed sequence, with a fallback for scheduling conflicts:

| Subagent | Trigger | Responsibility |
|---|---|---|
| **Deal Recording** | Webhook (called by the voice agent) | Finds the caller in Pipedrive; if found, creates a deal |
| **Demo Booking** | Webhook (called by the voice agent) | Checks Google Calendar availability and creates the event; retries with another time slot if the first isn't available |

Both subagents report their outcome back through Telegram and respond to the calling webhook so the voice agent knows how to continue the conversation.

## Tech Stack

- **Automation:** [n8n](https://n8n.io/) (self-hosted, Docker)
- **LLM:** Google Gemini (chat model for both the orchestrator and all subagents)
- **Voice:** [ElevenLabs](https://elevenlabs.io/) (conversational agent, inbound call handling)
- **Web scraping / lead sourcing:** Firecrawl
- **Email finding/verification:** Hunter.io
- **CRM:** Pipedrive
- **Scheduling:** Google Calendar
- **Email outreach:** Gmail
- **Notifications:** Telegram
- **Tunneling (dev):** ngrok

## Design Notes

- **One capable model for orchestration.** The manager and voice agents run on a full-capability Gemini model, not a lightweight one — smaller models struggled to reliably consolidate multi-tool-call results in testing.
- **Fixed sequence, not autonomous branching.** Both the manager and the voice agent follow a structured, ordered sequence of subagent calls rather than deciding dynamically — predictable and easier to debug, at the cost of flexibility.
- **Status visibility over silence.** Every workflow reports back to Telegram on success or failure, so runs can be monitored without opening n8n.
- **Two invocation patterns.** The three outbound subagents are called as tools by the manager's AI Agent inside n8n; the two account executive subagents are triggered via webhook, called directly by the ElevenLabs voice agent.

## Disclaimer

Exported workflow JSON files in this repository have been sanitized: real credentials, webhook IDs, calendar/CRM IDs, chat IDs, and email addresses have been replaced with placeholders. Credentials are never included in n8n exports by default, but all other identifying values were manually reviewed and scrubbed before publishing.

## License

This project is licensed under the [MIT License](LICENSE).
