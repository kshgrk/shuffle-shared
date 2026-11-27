# put_agent_update operations

Every operation goes in the "operations" array of put_agent_update's body. How to reference data between nodes is in SKILL.md (WIRING DATA) and references/reference-syntax.md.

Supported Step-by-Step Operations

A workflow has two types of nodes: Actions and Triggers. If you need a trigger, use Webhook, Schedule, User Input, or Subflow. There is no rigid rule that triggers must always exist — only use a trigger if the workflow solution genuinely needs it.

Format: Webhook Trigger (Incoming HTTP requests)
{
"op": "add_node",
"node_type": "trigger",
"temp_id": "<your_temp_id>",
"data": {
  "app_name": "Webhook",
  "label": "<unique_node_name>",
  "x": 100,
  "y": 100
}
}

Format: Schedule Trigger (Time / cron interval triggers)
{
"op": "add_node",
"node_type": "trigger",
"temp_id": "<your_temp_id>",
"data": {
  "app_name": "Schedule",
  "label": "<unique_node_name>",
  "parameters": [
    { "name": "cron", "value": "*/15 * * * *" }
  ],
  "x": 100,
  "y": 100
}
}

Format: User Input Trigger (Human-in-the-loop approval / decision checkpoint that pauses execution awaiting manual user approval)
{
"op": "add_node",
"node_type": "trigger",
"temp_id": "<your_temp_id>",
"data": {
  "app_name": "User Input",
  "label": "<unique_node_name>",
  "parameters": [
    { "name": "alertinfo", "value": "## Approve Action?\n\nDetails: $exec" },
    { "name": "type", "value": "email" },
    { "name": "email", "value": "admin@example.com" },
    { "name": "options", "value": "boolean" },
    { "name": "subflow", "value": "" },
    { "name": "subflow_failure", "value": "" }
  ],
  "x": 100,
  "y": 100
}
}
Note on User Input parameters:
- "alertinfo": The prompt/markdown shown to the human.
- "type": Delivery method: "email", "sms", "subflow", or combinations like "subflow,email".
- "email": Target email for email notifications.
- "sms": Target phone number for SMS notifications.
- "subflow": Optional workflow UUID to trigger for approval handling.
- "subflow_failure": Optional workflow UUID to trigger if the human clicks Decline.
- "options": Response type (typically "boolean").

Format: Subflow Trigger (Triggered when this workflow is called as a subflow from another workflow)
{
"op": "add_node",
"node_type": "trigger",
"temp_id": "<your_temp_id>",
"data": {
  "app_name": "Shuffle Workflow",
  "label": "<unique_node_name>",
  "parameters": [
    { "name": "workflow", "value": "<the workflow_id>" },
    { "name": "argument", "value": "$exec" },
    { "name": "check_result", "value": "true" }
  ],
  "x": 100,
  "y": 100
}
}

1. ADDING A NODE (Action)
{
"op": "add_node",
"node_type": "action",
"temp_id": "a_temp_id_string",
"data": {
  "app_id": "<id of the app>",
  "action_name": "<name of action to use, or custom_action>",
  "label": "Authenticate User",
  "parameters": [
    { "name": "param_1", "value": "some_value" }
  ],
  "x": 0,
  "y": 0
}
}

POSITIONING (x/y): get_minimal_workflow returns the real x/y of every existing node — use it. The overall flow should progress left to right: a node's "x" should generally be greater than the node(s) before it in the sequence (e.g. roughly halfway between two neighbors' x values if inserting between them, or ~337 to the right of the last node if appending). "y" should match its predecessor UNLESS this node is one of several parallel branches running off the same source node — in that case, give each parallel branch its own distinct "y" (spread them apart, e.g. ±150-250) so they fan out and don't overlap, matching how the user's request implies the branches should look. If the user explicitly describes a specific layout or arrangement, follow that instead of these defaults. Only leave "x" and "y" at 0 when you have no neighbor position to go on (e.g. the very first action in a brand-new workflow) — the backend will auto-place it. NEVER reuse the exact same fixed x/y (e.g. 100,100) for multiple unrelated actions, that stacks them directly on top of each other.

Note: You can control WHERE in the sequence a new node is inserted (which also determines its horizontal position) by adding "insert_before": "<node_id>" or "insert_after": "<node_id>" alongside the data field.

2. CONNECTING NODES (Adding a Branch)
Connect your nodes using their real IDs (if they already exist) or the temp_id you assigned when creating them in the same payload. You can include an array of conditions if you are adding multiple to the same branch.
{
"op": "add_branch",
"temp_id": "branch_1",
"data": {
  "source_id": "<real_node_id or temp_id>",
  "destination_id": "<real_node_id or temp_id>",
  "conditions": [
    { "source": "$some_var", "condition": "equals", "destination": "true" }
  ]
}
}

3. EDITING A NODE
Only provide the fields you actually want to change.
{
"op": "edit_node",
"id": "<real_node_id>",
"data": {
  "label": "New Name",
  "parameters": [
    { "name": "param_1", "value": "new_value" }
  ]
}
}

To swap a node to another app or action (e.g. an HTTP node to Google Sheets), use edit_node with the new app_id and action_name. The node keeps its id, label, position and branches, and its parameters become those of the new action, so pass the values for them:
{
"op": "edit_node",
"id": "<real_node_id>",
"data": {
  "app_id": "<app_id from the app lookup>",
  "action_name": "<action of that app>",
  "parameters": [
    { "name": "<parameter of that action>", "value": "<value>" }
  ]
}
}

4. DELETING OR MOVING
{ "op": "delete_node", "node_type": "action", "id": "<real_node_id>" }
{ "op": "delete_branch", "id": "<real_branch_id>" }
{ "op": "move_node", "id": "<real_node_id>", "data": { "x": 250, "y": 300 } }

5. MANAGING CONDITIONS
You can add, edit, or delete conditions on an EXISTING branch.
NOTE: To edit or delete, you must use a Get/Read action first to find the exact condition ID.

Available operators: equals, larger than, less than, does not equal, startswith, endswith, is empty, contains, contains_any_of.

Add conditions to an existing branch:
{
"op": "add_condition",
"branch_id": "<real_branch_id or temp_id>",
"data": {
  "conditions": [
    { "source": "$var", "condition": "equals", "destination": "val" }
  ]
}
}

Edit existing conditions (only include fields you want to change):
{
"op": "edit_condition",
"branch_id": "<real_branch_id>",
"data": {
  "conditions": [
    { "id": "<real_condition_id>", "source": "$new_var", "condition": "contains", "destination": "new_val" }
  ]
}
}

Delete conditions:
{
"op": "delete_condition",
"branch_id": "<real_branch_id>",
"data": {
  "condition_ids": ["<real_condition_id>"]
}
}

6. SETTING THE START NODE
Defines the entry point of the workflow. You can use a real ID or a temp_id from the same payload.
IMPORTANT: The start node must always be an ACTION, never a trigger. If the workflow begins with a trigger, connect it to its first action via add_branch, then pass that action's id (the trigger's destination) here — NOT the trigger's own id.
{
"op": "set_start_node",
"id": "<real_node_id or temp_id>"
}

7. SAVING THE WORKFLOW
Your edits are stored as a real-time draft. When you are completely finished building or modifying the workflow, you MUST append this operation to your final payload to permanently save the workflow to the database. Call this only after all changes are complete — do not forget it.
{
"op": "save_workflow"
}
