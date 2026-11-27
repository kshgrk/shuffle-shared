# $reference syntax

Shuffle replaces every $reference in a node's parameters with text just before the node runs. It is Shuffle's own syntax: not JSONPath, not JavaScript, not Python indexing.

## Sources
| Reference | Resolves to |
|---|---|
| $exec | The trigger's input (webhook body, schedule argument, subflow argument). Also $webhook, $schedule, $userinput |
| $exec.email | A field of the trigger's input |
| $<label> | The whole result of the node with that label, as text |
| $<label>.body.id | A field inside that result |
| $<variable_name> | A workflow variable or execution variable |

Labels match case-insensitively, and spaces in a label become _: a node labelled "Get Emails" is $get_emails. Only nodes that run before this one on the same branch path can be referenced.

## Objects
Step in with dots: $get_ticket.body.fields.priority.name

## Lists
A list is stepped into with #, never with a bare number.

| Syntax | Means |
|---|---|
| .#0 | First item |
| .#1 | Second item |
| .#max (or .#last) | Last item |
| .#0-2 | Items 0 to 2, as a list |
| .# | Every item: the node runs once per item |
| .#.id | The id of every item: the node runs once per id |
| .#0.id | The id of the first item |

Example. If get_users returned {"status": 200, "body": {"users": [{"name": "Alice", "id": 101}, {"name": "Bob", "id": 102}]}}:
- $get_users.status gives 200
- $get_users.body.users.#0.name gives Alice
- $get_users.body.users.#max.id gives 102
- $get_users.body.users.#.id runs the node twice, with 101 and 102
- $get_users.body.users.0.name gives nothing: .0 on a list does not pick an item
- $get_users.users.#0.name gives nothing: the users list is under body

To loop over a list inside a list, pass each outer item to a subflow and loop inside it.

## When a reference doesn't resolve
There is no error. The node gets an empty or partial value and still shows SUCCESS, and later nodes run on that value. The run records it: get_workflow_run with node=<label> lists each $reference and what it resolved to. Fix the path, don't add workaround nodes.

## Inside text, JSON bodies and code
The value is pasted in as text where the reference stands.
- In a JSON body, a string value needs quotes around the reference: {"id": "$get_emails.body.messages.#0.id"}. An object or list does not: {"rows": $build_rows.message}
- In Python code, reference only the field you need. To parse a whole object, use a raw triple-quoted string, which keeps the JSON's backslash escapes intact: data = json.loads(r'''$get_emails.body'''). A plain '...' or '''...''' string changes the escapes and json.loads fails.

## Liquid
Every parameter also supports Liquid templates, applied after $references:
- {{ $get_users.body.users.#.score | plus: 1 }} changes each value
- {{ "  text  " | strip }} trims spaces
Use Liquid for small formatting; use a Shuffle Tools action or Python for real logic.
