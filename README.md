# Contributor-Attribute-and-Responsiblity-Workflow
Extend Collibra metadata editing to the people closest to the data. A governed, audited BPMN workflow for attribute and responsibility changes that would otherwise require a Creator license.
# Collibra Contributor Elevated Actions

A Collibra workflow that lets **Contributor-licensed users** add, change and clear
asset attributes and assign or unassign responsibilities — actions that normally
require a Creator licence.

Users work entirely in **names** (attribute example; "Note", role example; "Business Steward", user example; "eliza arquette"). The
workflow resolves identifiers and looks up existing values itself, so nobody has
to find a UUID. Every run leaves an audit comment on the asset.

## The problem

In Collibra, editing attributes and responsibility assignments requires a Creator
licence. Organizations frequently have subject-matter experts — analysts,
stewards-in-practice, data owners — who are licensed as Contributors and who are
exactly the right people to maintain this metadata, but who cannot.

The usual answers are to buy more Creator licences or to funnel every small edit
through someone who has one. This workflow offers a third option: a governed,
audited, role-restricted path that performs the change on the user's behalf.

## How it works

Collibra's documentation states that "the actions performed by workflows are not
restricted by the permissions of the users who are starting or participating in
them." Script tasks execute in the workflow engine's own context, not the
initiator's. So a Contributor can trigger a workflow that writes an attribute,
even though the same edit is blocked for them in the UI.

That makes the access control model the important part. Three independent gates
apply, and all three must pass:

| Gate | Where it's configured | What it controls |
|---|---|---|
| `Start workflow` + `Participate in workflow` global permissions | Settings → Roles and Permissions → Global Permissions, on a **global role** | Whether the user may start workflows and hold workflow tasks at all |
| **Start Workflow** role | The workflow definition's Roles settings — a **resource role** | Which role may launch *this* workflow on an asset |
| `candidateUsers="user(${initiator})"` | In the BPMN | The form goes only to the person who started it |

Nothing is hardcoded in the BPMN about *which* role is privileged. That decision
lives in the definition's Start Workflow setting, where Collibra resolves group
membership and inherited responsibilities correctly.

## Contents

| File | Purpose |
|---|---|
| `contributor-elevated-actions.bpmn` | The workflow. Import into Settings → Workflows → Definitions |
| `DEPLOYMENT.md` | Deployment, configuration, verified API surface, troubleshooting |
| `README.md` | This file |

## Requirements

- Collibra with the Flowable-based workflow engine (targets `http://www.collibra.com/apiv2`)
- A resource role to grant the capability to (any role works — you choose it at deploy time)
- A global role carrying **Start workflow** and **Participate in workflow**
- Permission to import workflow definitions

## Quick start

1. **Import** `contributor-elevated-actions.bpmn` via Settings → Workflows → Definitions.
2. **Scope it**: set **Applies to** = `Asset`, and in the assignment rule set
   **Type** = `Asset` — the root type, which covers every asset type. Picking a
   specific type here limits the workflow to that type only.
3. **Restrict it**: under **Roles → Start Workflow**, add the resource role whose
   holders should get this capability. Leave *"Any user can start the workflow"*
   unchecked.
4. **Grant global permissions**: on a global role held by those users, tick
   **both** `Start workflow` and `Participate in workflow`. Both are required —
   the first lets them launch it, the second lets them receive the form task.
5. **Use it**: open an asset → **Actions** → *Manage Attributes and
   Responsibilities*.

`DEPLOYMENT.md` covers each step in detail, including how to tell the two
permissions apart when something doesn't appear.

## Using it

One dropdown, five actions:

| Action | What it does | Fields used |
|---|---|---|
| Set an attribute value | Changes the value if one exists, creates it if not | Attribute name, Value |
| Add another value to an attribute | Appends a value to a multi-valued attribute | Attribute name, Value |
| Clear an attribute value | Removes the existing value(s) of that attribute | Attribute name |
| Give someone a role on this asset | Assigns a responsibility | Role name, Person |
| Take a role away from someone | Removes a directly-assigned responsibility | Role name, Person |

**Set** deliberately upserts. In the UI an attribute with no value still renders
as an empty field, so users reasonably think "update" — but with no value there
is no attribute instance to update. Set removes that distinction: it works
whether or not a value is already there.

All name fields also accept a UUID, so power users can paste identifiers
directly. Role and attribute name matching is case-insensitive.

**Removing roles only affects responsibilities assigned directly on the asset.**
Inherited ones (granted at a domain or community, or held via a group) are left
alone, and the workflow says so rather than failing silently.

## Audit trail

Because this grants elevated write access, every run comments on the asset:

> **Elevated action** — Updated Note.
> Requested by: *username*
> Reason: *optional justification from the form*

Failures comment too, carrying the underlying error. A run can't look like it
succeeded when it didn't.

## Status

All five actions are verified working end to end against a live Collibra
instance, with a Contributor-licensed user performing attribute writes and
responsibility changes their licence would otherwise block.

Known gaps, none blocking:

- Only **string-valued** attribute types have been exercised. Numeric, date,
  boolean and value-list types may need value conversion before they work.
- Form fields are all shown at once. Which ones matter depends on the action
  chosen; conditional visibility would make this clearer.
- The `Person` field resolves users; assigning a responsibility to a **group**
  is not implemented.
- Multi-valued attributes: **Clear** removes every value of that type, not a
  selected one.

## Roadmap

- Conditional field visibility driven by the selected action
- Value conversion for non-string attribute types
- Group assignees for responsibilities
- Optional approval step for a configurable subset of actions
- Relations, in addition to attributes and responsibilities

## Notes for anyone hand-authoring Collibra workflows

Several things cost real time to discover and are poorly documented. They're
written up in `DEPLOYMENT.md`, but the headline one:

**Collibra decides a workflow's API version from the root element's
`targetNamespace`.** A file declaring anything other than
`http://www.collibra.com/apiv2` is flagged as using the deprecated API v1 no
matter what its scripts actually call. That banner is immune to every
script-level change, survives restarts, and never appears on workflows built in
Collibra's own Designer.

`DEPLOYMENT.md` also carries the full verified API surface — exact builder
signatures and accessor names for the attribute, responsibility, role, user and
comment calls, each confirmed against a running instance rather than inferred
from documentation.
