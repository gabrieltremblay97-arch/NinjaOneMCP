# NinjaOne MCP Server — Phase 2, 3 & 4 Implementation

You are extending an existing TypeScript MCP server that exposes NinjaOne RMM
data to Claude via the Model Context Protocol. The server runs on Cloudflare
Workers. The codebase lives in the current directory.

Before writing any code, read the following files in full to understand the
existing patterns:
- `src/index.ts` — MCP server entry point, tool registration, request handlers
- `src/ninja-api.ts` — NinjaOne API v2 client (OAuth2, KV token cache, pagination)
- `src/auth.ts` — Entra ID JWT validation and group membership check
- `wrangler.toml` — Cloudflare Workers config
- `package.json` — dependencies and scripts
- `TOOLS.md` (if it exists) — existing tool inventory

Match all existing code style exactly: same TypeScript patterns, same error
handling, same Zod schema style, same response shape
`{ content: [{ type: "text", text: JSON.stringify(...) }] }`.

---

## Known bugs to fix first (before adding anything new)

1. There is a duplicate tool definition and handler for `get_device_software`.
   Remove the second/duplicate registration entirely.

2. Check for any other duplicate tool names and remove duplicates.

After fixing bugs, run `npx tsc --noEmit` to confirm the project compiles
cleanly before proceeding.

---

## Phase 2 — Write operations with confirmation guardrails

### Guiding principle
Every tool that mutates state in NinjaOne must include a `confirm` boolean
parameter (default `false`). When `confirm` is false, the tool performs a
dry-run: it returns a human-readable description of exactly what would happen
and instructs the caller to re-invoke with `confirm: true` to proceed. This
gives Claude a chance to show the user what will happen before committing.

Example dry-run response:
```
DRY RUN — no changes made.
Would reset alert uid=abc123 (Disk Free Space ≤5% on device SCB-PC18).
Re-call with confirm=true to execute.
```

### 2.1 Alert management

Add to `ninja-api.ts`:
```typescript
// Reset (acknowledge) an alert
async resetAlert(uid: string): Promise<unknown>
// POST /v2/alert/{uid}/reset  (body: {})
```

Add tool to `index.ts`:
- **`reset_alert`** — Reset/acknowledge an active alert by UID. Parameters:
  `uid: string`, `confirm: boolean (default false)`. Dry-run shows alert
  summary before committing.

### 2.2 Device management

Add to `ninja-api.ts`:
```typescript
// Update device display name and/or user-defined fields
async updateDevice(id: number, body: { displayName?: string; userData?: Record<string, string> }): Promise<unknown>
// PATCH /v2/device/{id}

// Set or clear maintenance mode window
async setDeviceMaintenance(id: number, body: {
  enabled: boolean;
  start?: string;   // ISO 8601
  end?: string;     // ISO 8601
}): Promise<unknown>
// PUT /v2/device/{id}/maintenance

// Reboot a device
async rebootDevice(id: number, mode: 'NORMAL' | 'FORCED'): Promise<unknown>
// POST /v2/device/{id}/reboot/{mode}
```

Add tools to `index.ts`:
- **`update_device`** — Update a device's display name or user-defined fields.
  Parameters: `id: number`, `displayName?: string`, `userData?: object`,
  `confirm: boolean (default false)`.
- **`set_device_maintenance`** — Enable or disable maintenance mode on a device.
  Parameters: `id: number`, `enabled: boolean`, `start?: string (ISO 8601)`,
  `end?: string (ISO 8601)`, `confirm: boolean (default false)`.
- **`reboot_device`** — Reboot a device. Parameters: `id: number`,
  `mode: 'NORMAL' | 'FORCED' (default NORMAL)`, `confirm: boolean (default false)`.
  Dry-run must show device name and reboot mode.

### 2.3 OS & software patch application

The API methods already exist on the client (`applyDeviceOsPatches`,
`applyDeviceSoftwarePatches`). Add the confirmation wrapper:

- **`apply_os_patches`** — Apply specific OS patches to a device. Parameters:
  `id: number`, `patches: array of { id: string }`, `confirm: boolean (default false)`.
- **`apply_software_patches`** — Apply specific software patches. Parameters:
  `id: number`, `patches: array of { id: string }`, `confirm: boolean (default false)`.

Both dry-runs must list patch names/IDs that would be applied.

### 2.4 Ticketing (full CRUD)

Add to `ninja-api.ts`:
```typescript
// List all ticket boards
async getTicketBoards(): Promise<unknown>
// GET /v2/ticketing/ticket/boards

// Get tickets from a board (paginated)
async getTickets(boardId: number, pageSize?: number, after?: number): Promise<unknown>
// GET /v2/ticketing/ticket?boardId={boardId}&pageSize=...&after=...

// Get single ticket detail
async getTicket(ticketId: number): Promise<unknown>
// GET /v2/ticketing/ticket/{ticketId}

// Get ticket activity log
async getTicketLog(ticketId: number): Promise<unknown>
// GET /v2/ticketing/ticket/{ticketId}/log-entry

// Create a new ticket
async createTicket(body: {
  boardId: number;
  subject: string;
  description?: string;
  status: string;        // e.g. "OPEN"
  priority?: string;     // e.g. "HIGH", "MEDIUM", "LOW"
  severity?: string;
  type?: string;
  assignedAppUserId?: number;
  deviceId?: number;
}): Promise<unknown>
// POST /v2/ticketing/ticket

// Update an existing ticket
async updateTicket(ticketId: number, body: {
  subject?: string;
  description?: string;
  status?: string;
  priority?: string;
  severity?: string;
  assignedAppUserId?: number;
}): Promise<unknown>
// PATCH /v2/ticketing/ticket/{ticketId}

// Add a comment to a ticket
async addTicketComment(ticketId: number, comment: string, appUserId?: number): Promise<unknown>
// POST /v2/ticketing/ticket/{ticketId}/log-entry  body: { comment, appUserId }
```

Add tools to `index.ts`:
- **`get_ticket_boards`** — List all ticket boards. No parameters.
- **`get_tickets`** — List tickets from a board. Parameters: `boardId: number`,
  `pageSize?: number (default 25)`, `after?: number`.
- **`get_ticket`** — Get full detail for a single ticket. Parameter: `ticketId: number`.
- **`get_ticket_log`** — Get activity log for a ticket. Parameter: `ticketId: number`.
- **`create_ticket`** — Create a new ticket. Parameters: `boardId: number`,
  `subject: string`, `description?: string`, `status?: string (default OPEN)`,
  `priority?: string`, `severity?: string`, `deviceId?: number`,
  `assignedAppUserId?: number`, `confirm: boolean (default false)`.
- **`update_ticket`** — Update an existing ticket. Parameters: `ticketId: number`,
  plus optional `subject`, `description`, `status`, `priority`, `severity`,
  `assignedAppUserId`. `confirm: boolean (default false)`.
- **`add_ticket_comment`** — Add a comment/note to a ticket. Parameters:
  `ticketId: number`, `comment: string`, `appUserId?: number`,
  `confirm: boolean (default false)`.

### 2.5 Custom field writes

Add to `ninja-api.ts`:
```typescript
// Update custom fields on a device
async updateDeviceCustomFields(deviceId: number, fields: Record<string, unknown>): Promise<unknown>
// PATCH /v2/device/{deviceId}/custom-fields

// Update custom fields on an organization
async updateOrganizationCustomFields(orgId: number, fields: Record<string, unknown>): Promise<unknown>
// PATCH /v2/organization/{orgId}/custom-fields
```

Add tools to `index.ts`:
- **`update_device_custom_fields`** — Write custom field values on a device.
  Parameters: `deviceId: number`, `fields: object (key-value pairs)`,
  `confirm: boolean (default false)`. Dry-run shows field names and new values.
- **`update_org_custom_fields`** — Write custom field values on an organization.
  Parameters: `orgId: number`, `fields: object`, `confirm: boolean (default false)`.

---

## Phase 3 — Webhooks & event-driven integration

### 3.1 Webhook configuration tools

Add to `ninja-api.ts`:
```typescript
// Get current webhook configuration
async getWebhookConfig(): Promise<unknown>
// GET /v2/webhook

// Set (upsert) webhook configuration
async setWebhookConfig(body: {
  webhookUrl: string;
  secret?: string;
  activities?: string[];   // Activity types to subscribe to
  expandedActivities?: string[];
}): Promise<unknown>
// PUT /v2/webhook

// Delete webhook configuration
async deleteWebhookConfig(): Promise<unknown>
// DELETE /v2/webhook
```

Add tools to `index.ts`:
- **`get_webhook_config`** — Show the current NinjaOne webhook configuration
  (URL, subscribed events). No parameters.
- **`set_webhook_config`** — Configure a webhook endpoint for NinjaOne to push
  events to. Parameters: `webhookUrl: string`, `secret?: string`,
  `activities?: string[]`, `confirm: boolean (default false)`.
- **`delete_webhook_config`** — Remove the webhook configuration. Parameters:
  `confirm: boolean (default false)`.

### 3.2 Scheduled / polling queries

Add to `ninja-api.ts`:
```typescript
// Get devices that haven't checked in since a given timestamp
async getStaleDevices(sinceHours: number): Promise<unknown>
// GET /v2/devices with lastContact filter, computed client-side from current time

// Get devices with pending patches
async getDevicesWithPendingPatches(patchStatus?: 'PENDING' | 'FAILED'): Promise<unknown>
// GET /v2/queries/os-patches filtered by status
```

Add tools to `index.ts`:
- **`get_stale_devices`** — List devices that haven't checked in for more than N
  hours. Parameter: `sinceHours: number (default 48)`. Useful for "what hasn't
  reported recently?" queries.
- **`get_devices_pending_patches`** — List devices with pending or failed OS
  patches across the entire fleet. Parameter: `status?: 'PENDING' | 'FAILED'
  (default PENDING)`.

### 3.3 System activities feed

Add to `ninja-api.ts`:
```typescript
// Get system-wide activity log with filtering
async getActivities(params?: {
  pageSize?: number;
  after?: number;
  before?: number;
  type?: string;       // Activity type filter
  deviceId?: number;
  userId?: number;
  status?: string;
}): Promise<unknown>
// GET /v2/activities with query params
```

Add tool to `index.ts`:
- **`get_activities`** — Query the NinjaOne activity log. Parameters:
  `pageSize?: number (default 50)`, `after?: number (cursor)`,
  `type?: string`, `deviceId?: number`, `userId?: number`, `status?: string`.
  Returns audit trail of all NinjaOne actions: logins, policy changes,
  script runs, device events.

---

## Phase 4 — Script execution & policy management

### 4.1 Script / automation execution

Add to `ninja-api.ts`:
```typescript
// Run a saved automation script on a device
async runDeviceScript(deviceId: number, body: {
  id: number;          // Script/automation ID from NinjaOne
  runAs?: string;      // e.g. "SYSTEM", "LOGGED_ON_USER"
  parameters?: string; // Script parameters string
  timeout?: number;    // Seconds
}): Promise<unknown>
// POST /v2/device/{deviceId}/script/run

// Get the result/status of a script run
async getScriptResult(deviceId: number, activityId: number): Promise<unknown>
// GET /v2/device/{deviceId}/script/run/{activityId}/result

// List available automations/scripts
async getAutomations(): Promise<unknown>
// GET /v2/automation-scripts  (or /v2/scripting/automation — check OpenAPI spec)
```

Add tools to `index.ts`:
- **`list_automations`** — List all saved automation scripts/tasks available in
  NinjaOne. No parameters. Returns script IDs, names, and descriptions.
- **`run_device_script`** — Execute a saved automation script on a specific device.
  Parameters: `deviceId: number`, `scriptId: number`, `runAs?: string (default SYSTEM)`,
  `parameters?: string`, `timeout?: number (default 300)`,
  `confirm: boolean (default false)`.
  Dry-run must show device name, script name, runAs context, and parameters.
- **`get_script_result`** — Poll the result of a previously triggered script run.
  Parameters: `deviceId: number`, `activityId: number`.

### 4.2 Policy management

Add to `ninja-api.ts`:
```typescript
// List all policies
async getPolicies(): Promise<unknown>
// GET /v2/policies

// Get policy detail
async getPolicy(policyId: number): Promise<unknown>
// GET /v2/policies/{policyId}  (check exact path in OpenAPI spec)

// Assign a policy to a device
async assignDevicePolicy(deviceId: number, policyId: number): Promise<unknown>
// PUT /v2/device/{deviceId}/policy/{policyId}  (verify in OpenAPI spec)

// Get policy overrides for a device
async getDevicePolicyOverrides(deviceId: number): Promise<unknown>
// GET /v2/device/{deviceId}/policy/overrides

// Reset policy overrides on a device (restore to org default)
async resetDevicePolicyOverrides(deviceId: number): Promise<unknown>
// DELETE /v2/device/{deviceId}/policy/overrides
```

Add tools to `index.ts`:
- **`get_policies`** — List all policies in NinjaOne with IDs, names, and types.
  No parameters.
- **`get_policy`** — Get full detail for a specific policy. Parameter: `policyId: number`.
- **`assign_device_policy`** — Assign a policy to a device. Parameters:
  `deviceId: number`, `policyId: number`, `confirm: boolean (default false)`.
  Dry-run shows current policy → new policy before committing.
- **`get_device_policy_overrides`** — Show any policy overrides active on a device.
  Parameter: `deviceId: number`.
- **`reset_device_policy_overrides`** — Remove all policy overrides from a device,
  restoring it to its org-level policy. Parameters: `deviceId: number`,
  `confirm: boolean (default false)`. Dry-run lists all overrides that would be removed.

### 4.3 Device approval workflow

Add to `ninja-api.ts`:
```typescript
// List devices pending approval
async getPendingDevices(): Promise<unknown>
// GET /v2/devices?df=approval_status=PENDING

// Approve or reject a list of devices
async approveDevices(deviceIds: number[], mode: 'APPROVE' | 'REJECT'): Promise<unknown>
// POST /v2/devices/approval/{mode}  body: { deviceIds: [...] }
```

Add tools to `index.ts`:
- **`get_pending_devices`** — List all devices awaiting approval. No parameters.
- **`approve_devices`** — Approve or reject pending devices. Parameters:
  `deviceIds: number[]`, `mode: 'APPROVE' | 'REJECT'`,
  `confirm: boolean (default false)`.
  Dry-run lists each device name and the action that would be taken.

---

## Documentation updates

After all phases are implemented:

1. Update `TOOLS.md` with every new tool: name, description, parameters,
   which phase it belongs to, and whether it has a confirm guard.

2. Update `README.md` to include a "Phase 2/3/4" section explaining:
   - The confirm pattern and how to use it
   - Webhook setup instructions
   - Script execution prerequisites (scripts must exist in NinjaOne first)
   - Policy management caveats

---

## API notes for the CA region

- Base URL: `https://ca.ninjarmm.com/api`
- All write endpoints require the NinjaOne API app to have **Management** scope
  (not just Monitoring). Verify the app registration has this before testing.
- The ticketing API requires the **Ticketing** module to be enabled on the tenant.
- Script execution requires the **Automation** module.
- Some endpoints (especially ticketing and script run) are not fully documented
  in the public OpenAPI spec. If an endpoint returns 404 or 405, check the
  NinjaOne developer community or the Swagger UI at
  `https://ca.ninjarmm.com/apidocs` (requires login) for the correct path.

---

## Implementation order

Work through in this sequence to minimise conflicts:

1. Fix the `get_device_software` duplicate bug
2. Run `npx tsc --noEmit` — must be clean before proceeding
3. Phase 2: alert reset → device tools → patch tools → ticketing → custom fields
4. `npx tsc --noEmit` again
5. Phase 3: webhook tools → stale device query → pending patches query → activities
6. `npx tsc --noEmit` again
7. Phase 4: script execution → policy management → device approval
8. Final `npx tsc --noEmit`
9. Update TOOLS.md and README.md
