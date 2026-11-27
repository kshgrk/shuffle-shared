# What nodes return, and how to check it

A reference is only right if it matches the real output of the node it points at. Find that output; never assume it.

## Output by kind of node
| Node | Result |
|---|---|
| App action built from an API (Gmail, Notion, Jira, Slack, HTTP-style apps) | {"status": <HTTP code>, "body": <the API's JSON or text>, "url": ..., "headers": {...}, "success": true}. The API's data is always under .body. success is true even when status is 400 or 500: check status |
| Shuffle Tools execute_python | {"success": true, "message": <everything the code printed>}, or {"success": false, "message": "Exception: ..."} when it crashed. Printed JSON can be stepped into: $<label>.message.<field> |
| Other Shuffle Tools actions (filter list, merge lists, set value...) | Each has its own shape, e.g. filter list returns valid and invalid lists. Check the action's example from get_action or a real result |
| Triggers | $exec is whatever started the run: the webhook body, the schedule argument, the subflow argument |

## Finding a node's real output, best first
1. A real result from this workflow: get_workflow_run with node=<label> and max_chars up to 8000 shows that node's result from the run.
2. Your own test call: call the app action yourself with real values (BUILD LOOP step 4). Its response is exactly what the node will get, inside the {status, body} wrapper.
3. The action's example from get_action. It shows the shape, but fields can be missing in real data: optional fields, errors and empty results look different.
Treat a path you have only seen in an example as unverified until a real result confirms it.

## Hidden failures
A node can show SUCCESS and still have failed. In a run result look for:
- status 400 or higher in an app action's result
- "success": false in a Python or Shuffle Tools result
- an empty list or empty message where data was expected
- a $reference that resolved to nothing ("references" in get_workflow_run)
get_workflow_run marks the first two as FAILED in real_status; check the last two yourself in its results and "references". The first failed node is the one to fix: the nodes after it only ran on its bad output.

## Python nodes
- Only use one when references, Liquid or a Shuffle Tools action can't do the job.
- Print the result: print(json.dumps(rows)). The next node reads it from $<label>.message.
- References are pasted into the code as text before it runs. Use r'''...''' around a reference to a JSON object, and reference single fields where you can.
- Test the code with execute_python and real sample data before you add the node.
