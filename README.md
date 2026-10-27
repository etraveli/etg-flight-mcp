# Flight Network MCP B2B Server

The **Flight Network MCP Server** lets AI assistants search for and book flights on behalf of users through the Etraveli Group B2B API. Connect it to your preferred AI assistant and let your users book flights in natural language — from search all the way through to payment.

---

## Getting Access

Access to the Flight Network MCP Server is managed by ETG. Your account is assigned an access level (tier 1 to 4) that decides which tools your AI assistant can use. To request access or an upgrade, contact us (see [Support](#support)).

### Access Levels

Higher tiers include all tools from the tiers below.

| Level | Tools available | Rate limit (per minute) | Rate limit (per day) |
|---|---|---|---|
| `TIER_1` | `search_flights` | 15 requests | 150 requests |
| `TIER_2` | `search_flights`, `select_flight` | 20 requests | 300 requests |
| `TIER_3` | `search_flights`, `select_flight`, `pick_seats`, `select_baggage`, `add_flight_details` | 250 requests | 10000 requests |
| `TIER_4` | All tools, including `get_payment_url` | 300 requests | 30000 requests |

Rate limits apply per account across all requests. Requests above the limit receive an HTTP `429` response; wait and retry.

---

## Quick Start

The server communicates over HTTP using the Model Context Protocol (MCP). Your AI assistant connects to it and automatically discovers the available tools.

Authentication is OAuth 2.0 with a browser-based login: the first time your AI assistant uses a tool, it opens a sign-in page. Your AI assistant handles the rest.

---

## Installation

### Claude.ai (web)

> **Requires:** Pro, Max, Team, or Enterprise plan.

1. Open **Settings → Customize → Connectors**
2. Click **+** → **Add custom connector**
3. Enter the MCP server URL as distributed to the user by email
4. Click **Advanced settings** and set:
    - **OAuth Client ID:** the client ID provided to you
5. Click **Add**

Claude will prompt you to authenticate via your browser the first time a tool is used. Connectors are shared across claude.ai, Claude Desktop, and mobile — you only need to add it once.

### Claude Desktop

Claude Desktop shares the same Connectors system as claude.ai. If you have already added the connector on claude.ai, it is available in Claude Desktop automatically.

To connect through the Claude Desktop config file instead, add the server using the [`mcp-remote`](https://www.npmjs.com/package/mcp-remote) bridge:

- **macOS:** `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Windows:** `%APPDATA%\Claude\claude_desktop_config.json`

```json
{
  "mcpServers": {
    "flight_network": {
      "command": "npx",
      "args": [
        "mcp-remote",
        "https://url-as-distributed-to-user-by-email",
        "--static-oauth-client-info",
        "{\"client_id\":\"<CLIENT-ID>\"}"
      ]
    }
  }
}
```

Fully quit and restart Claude Desktop after editing the config file. The first tool call opens your browser to sign in.

### Claude Code

Add the server with a single command:

```bash
claude mcp add-json flight-network '{"type":"http","url":"https://url-as-distributed-to-user-by-email","oauth":{"clientId":"<CLIENT-ID>"}}'
```

Then start Claude Code, run `/mcp`, select `flight-network` and select **Authenticate**. Your browser opens for sign-in.

Verify that the Flight Network server is connected:

```bash
claude mcp list
```

You should see `flight-network` listed as an available MCP server.

### OpenAI ChatGPT

> **Note:** MCP servers are currently only supported in ChatGPT Plus and Pro plans.

1. Go to **Settings → Connectors**
2. Scroll down and click **Advanced Settings**
3. Enable **Developer mode**
4. Go back to **Settings → Connectors** and click **Create** in the **Browser Connectors** section
5. Set the server URL to the MCP server URL provided to you
6. In the **Description** field, add:

    > Flight Network flight search and booking service. Search and book flights directly from ChatGPT.

7. In the **Authentication** section, select **OAuth** and enter the client ID provided to you
8. Click **Create**

To use the connector in a new chat, click **+ → More**, enable **Developer mode**, then select it from the **Add Sources** tab.

### OpenAI Codex (CLI)

Add the server with a single command, replacing `<CLIENT-ID>` with the value provided to you:

```bash
codex mcp add flight-network --url https://url-as-distributed-to-user-by-email --oauth-client-id <CLIENT-ID>
```

Then sign in. Your browser opens the first time:

```bash
codex mcp login flight-network
```

Alternatively, add the server directly to `~/.codex/config.toml`:

```toml
[mcp_servers.flight-network]
url = "https://url-as-distributed-to-user-by-email"

[mcp_servers.flight-network.oauth]
client_id = "<CLIENT-ID>"
```

Run `codex mcp login flight-network` afterwards. Restart Codex if it was already running.

---

## How It Works

The AI assistant guides the user through the full flight booking flow by calling the tools below in sequence. All session state is held server-side — the AI only needs to pass the values it collects from the user.

### Booking Flow Overview

1. `search_flights` — find available flights
2. `select_flight` — pick a flight option; optionally compare branded fare tiers
3. `pick_seats` — select seats (optional)
4. `select_baggage` — select baggage add-ons (optional)
5. `add_flight_details` — submit traveler names, contact info, and passport details
6. `get_payment_url` — get a secure payment link; user completes payment in browser

The AI prompts the user for information at each step and only calls the next tool once all required fields are collected. No partial or incomplete bookings are submitted downstream.

### Step 1 — Find available flights

Example of find flights user prompts:

- "find flights from London to Paris tomorrow" (searches any airports and defaults to 1 Adult traveler with no airline filtering)
- "find flights from London to Paris tomorrow except with Ryanair" (searches any airports and defaults to 1 Adult traveler with airline Ryanair excluded from the search results)
- "find flights from London to Paris tomorrow only with British Airways" (searches any airports and defaults to 1 Adult traveler and only include airline British Airways from the search results)
- "find flights from London to Paris tomorrow me, my wife and our newborn" (2 Adults and 1 Infant (age 2 year or less))
- "find flights from London to Paris tomorrow only direct flights" (only includes direct flights)
- "find flights from Heathrow to Charles de Gaulle tomorrow" (searches specific airports)

Supports up to 9 travelers

### Step 2 — Pick a flight option

User is presented with top 5 flight options with a brief description of the flight itineraries overview with price, flight times and layovers

User select one option between 1 to 5

### Step 3 — Pick seats

User is presented with option whether to pick seats or not

User selects yes or no. If users selects yes, user is presented with seat options and selects seats per flight path

### Step 4 — Select extra baggage

User is presented with option whether to pick extra baggage or not

User select yes or no. If user select yes, user is presented with extra baggage options and selects extra baggage per flight segment. A flex ticket (rebooking option) may also be offered at this step

### Step 5 — Provide travelers information

User is asked to provide details for each traveler: first name, last name, gender, and home address. The main traveler must also provide an email address and phone number. Depending on the flight and destination, additional fields such as date of birth and passport details (number, issue date, expiry date) may also be required.

### Step 6 — Pay for your booking

The AI provides a secure payment link. The user opens the link in their browser and completes payment on the secure payment page. Once payment is done, the user will receive a booking confirmation by email.

---

## Available Tools

Access to tools depends on your access level. Higher tiers include all tools from the tiers below.

| Tool | Description | Minimum access level |
|---|---|---|
| `search_flights` | Search for flights by origin, destination, date, and traveler count. Returns numbered flight cards. | `TIER_1` |
| `select_flight` | Select a flight by option number. If the airline offers multiple fare brands, the AI presents a comparison table so the user can select the right tier. | `TIER_2` |
| `pick_seats` | Displays an interactive seat map and lets each traveler select a seat. | `TIER_3` |
| `select_baggage` | Lists available checked and cabin baggage add-ons for selection or skip. | `TIER_3` |
| `add_flight_details` | Submits traveler names, contact details, and any required document fields (date of birth, passport number, etc.). | `TIER_3` |
| `get_payment_url` | Returns a secure payment link. The user opens the link in their browser and completes payment on the secure payment page. Booking confirmation is sent to the user by email. | `TIER_4` |

---

## Session State

Each booking session is tracked server-side using a `sessionId` generated at the search step and threaded through all subsequent tool calls. Session data (selected flight, traveler requirements, seat and baggage selections) expires after a short idle period — typically 10–30 minutes — so users should complete their booking in a single session.

---

## Support

For access requests, integration questions, or to report issues, contact us at: [b2bmcpsignup@etraveligroup.com](mailto:b2bmcpsignup@etraveligroup.com)
