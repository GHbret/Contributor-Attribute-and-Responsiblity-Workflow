# Deployment & Configuration

Companion to `contributor-elevated-actions.bpmn`
(process ID `contributorElevatedActions`).

---

## 1. Why the elevated actions work

Collibra's documentation is explicit that workflow execution is not governed by
the initiator's licence or permissions:

> "the actions performed by workflows are not restricted by the permissions of
> the users who are starting or participating in them"
> — [Workflow permissions](https://productresources.collibra.com/docs/collibra/latest/Content/Workflows/co_workflow-permissions.htm)

Script tasks call `attributeApi`, `responsibilityApi` and friends in the engine's
own execution context. The user triggers the workflow; the workflow performs the
write.

What *is* governed is who may start it and who may hold its task — see §3.

---

## 2. Import

Settings → **Workflows** → **Definitions** → upload
`contributor-elevated-actions.bpmn`. Sandbox first.

To replace an existing deployment, disable the definition, upload the new file,
then re-enable. Because the process ID is unchanged, the upload **replaces** the
definition in place and your scope and role configuration survive.

---

## 3. Configuration

### 3.1 Scope

In the definition's settings:

- **Applies to** = `Asset`
- In the assignment rule, **Type** = **`Asset`**

`Asset` is the root of the type hierarchy, so one rule covers every asset type.
Collibra's own guidance: *"Select Asset to associate the workflow with all asset
types."* Selecting a specific type (`Column`, say) restricts the workflow to that
type only — a common reason the action fails to appear where expected.

To change an existing rule: edit icon in the **Applies to** table → change
**Type** → Save.

### 3.2 Workflow role

Under **Roles → Start Workflow**, add the resource role whose holders should get
this capability. Leave **"Any user can start the workflow"** unchecked — it
overrides role restrictions entirely.

**Stop Workflow** and **Reassign Tasks** can stay empty. That means only
Workflow Administration / System Administration holders can cancel a running
instance or reassign its task, which is a sensible default: the role being
granted is "may edit metadata", not "may administer workflow instances". Add the
role to Stop Workflow only if users should be able to cancel their own runs.

### 3.3 Global permissions — both of them

Users need a **global role** carrying **two** permissions:

| Permission | Without it |
|---|---|
| `Start workflow` | The action never appears in the Actions menu |
| `Participate in workflow` | The workflow starts, then fails with *"no suitable candidate(s) … or they don't have permission to participate in workflows"* |

Settings → Roles and Permissions → **Global Permissions** → Edit → tick both on
the relevant role → Save. Users log out and back in.

Three things are commonly confused here, and each produces a different symptom:

- A **group** carries no permissions of its own. Membership of "Everyone" or
  "Users" grants nothing unless that group is a member of a global role.
- **Global permissions attach only to global roles**, never to resource roles.
  The resource role from §3.2 cannot carry them.
- **Administrators bypass both.** If an admin sees the action and a normal user
  doesn't, the difference is almost always the global permission, not the scope.

### 3.4 Nothing else to configure

The form lives in the BPMN as `flowable:formProperty` elements. There is no form
to rebuild in the Designer, and no role ID to edit in any script.

---

## 4. The form

One dropdown plus five fields. Only the action is mandatory; the rest depend on
which action is chosen.

| Field | Type | Used by |
|---|---|---|
| `action` | dropdown (5 options) | always |
| `attributeType` | text — name or UUID | the three attribute actions |
| `attributeValue` | long text | Set, Add another |
| `roleName` | text — name or UUID | the two role actions |
| `assignee` | text — username, full name, or UUID | the two role actions |
| `justification` | long text, optional | appended to the audit comment |

Name matching is case-insensitive for roles. Attribute type names are resolved
via `getAttributeTypeByName`; match the displayed name.

---

## 5. Process structure

```
Start  (flowable:initiator="initiator")
  → Describe the change            user task, candidateUsers = user(${initiator})
  → Which action?                  gateway on ${action}
      → Set attribute value        find by type → change, else add
      → Add another value          add
      → Clear attribute value      find by type → remove each
      → Assign role                resolve role + user → addResponsibility
      → Remove role                resolve role + user → find → removeResponsibility
  → Succeeded?                     gateway on ${actionSucceeded}
      → Record result  → End: Completed     (comments the change on the asset)
      → Report failure → End: Failed        (comments the error on the asset)
```

Every action script wraps its work in try/catch and sets `actionSucceeded` /
`errorMessage`, rather than relying on BPMN boundary error events. Both outcomes
comment on the asset, so a failed run is never mistaken for a clean one.

The failure handler resolves the asset defensively — `item`, then
`execution.getVariable("item")` — so that if `item` is itself what broke, the
handler still reports rather than failing identically and swallowing the
diagnosis.

---

## 6. Verified API surface

Every call below was confirmed against a running instance, not inferred. This is
the most useful part of this document for anyone writing Collibra script tasks.

**Bindings available without import:** `item` (the asset; `item.id`),
`initiator` (the starting user's username), `execution`, `string2Uuid`,
`loggerApi`, `attributeApi`, `attributeTypeApi`, `responsibilityApi`, `roleApi`,
`userApi`, `commentApi`.

### Attributes

```groovy
attributeTypeApi.getAttributeTypeByName("Note").getId()

attributeApi.findAttributes(
    FindAttributesRequest.builder()
        .assetId(item.id)          // singular, UUID
        .typeIds([attributeTypeId]) // PLURAL, List
        .build()).getResults()

attributeApi.addAttribute(
    AddAttributeRequest.builder()
        .assetId(uuid).typeId(uuid).value(string).build())

attributeApi.changeAttribute(
    ChangeAttributeRequest.builder().id(attributeInstanceId).value(string).build())

attributeApi.removeAttribute(attributeInstanceId)   // bare UUID
```

Note the inconsistency: `FindAttributesRequest` takes `assetId` singular but
`typeIds` plural. `AddAttributeRequest` takes `typeId` singular.

### Roles and users

```groovy
// RoleApi has no getRoleByName. List and match — role counts are small.
roleApi.findRoles(FindRolesRequest.builder().build()).getResults()
// → Role.getName(), Role.getId()

userApi.findUsers(FindUsersRequest.builder().name(input).build()).getResults()
// → User.getUserName(), User.getId()
```

### Responsibilities

```groovy
responsibilityApi.addResponsibility(
    AddResponsibilityRequest.builder()
        .resourceId(item.id)
        .resourceType(ResourceType.Asset)   // REQUIRED — omitting it fails with
        .roleId(uuid)                       // addResourceMemberIncompleteParameters
        .ownerId(uuid)
        .build())

responsibilityApi.findResponsibilities(
    FindResponsibilitiesRequest.builder()./* see note */.build()).getResults()

responsibilityApi.removeResponsibility(responsibilityId)   // bare UUID
```

`Responsibility` exposes **`getRole()`** and **`getOwner()`**, each returning a
`NamedResourceReference` carrying `.getId()`. There is no flat `getRoleId()` /
`getOwnerId()`. Guard for nulls — a group-held responsibility has no user owner.

*Note:* the BPMN tries `resourceIds([item.id])` and falls back to
`resourceId(item.id)`; which one this version accepts was never isolated, since
the fallback made it unnecessary to find out.

### Comments

```groovy
commentApi.addComment(
    AddCommentRequest.builder()
        .baseResourceType(ResourceType.Asset)
        .baseResourceId(item.id)
        .content("<b>html</b> is accepted")
        .build())
```

### Imports

Only classes on Collibra's
[approved imports list](https://developer.collibra.com/tutorials/future-proof-your-script-tasks)
may be imported; anything else is rejected at deploy time as a security
violation. Notably **absent**: `java.net.*` and every HTTP client, so script
tasks cannot call REST APIs. Also absent: `RemoveAttributeRequest`,
`ChangeResponsibilityRequest`, `RemoveResponsibilityRequest` — removals take a
bare UUID, and changing a responsibility is remove-then-add.

Imports must be the **first statements in the script**. An `import` inside a
`try {}` block is a syntax error, and Collibra reports it as
`collibra-script-task-validation-security-error: Unexpected input: 'import'`
rather than as a syntax problem — misleading if you don't know to look at
placement.

---

## 7. Writing a Collibra-native BPMN by hand

Match these or expect trouble. The reference is any workflow exported from
Collibra's own Designer.

- `targetNamespace="http://www.collibra.com/apiv2"` — **this is how Collibra
  determines the API version.** Any other value flags the workflow as using the
  deprecated API v1 regardless of what the scripts call. See §8.
- Default BPMN namespace — unprefixed `<process>`, `<scriptTask>` — not a
  `bpmn:` prefix
- `xmlns:flowable="http://flowable.org/bpmn"`,
  `xmlns:design="http://flowable.org/design"`,
  `design:palette="flowable-process-palette"`
- `typeLanguage` and `expressionLanguage` attributes on `<definitions>`
- `design:stencilid` extension element on every element
- `<conditionExpression xsi:type="tFormalExpression">` with the expression in
  CDATA — note `tFormalExpression`, not `bpmn:tFormalExpression`
- `flowable:autoStoreVariables="false"` on script tasks, and `//#importFile NONE`
  as the first line of each script
- No vendor extension namespace other than flowable/design — a `camunda:` block
  can get the whole file rejected

### Candidate user expressions

These are a small function language, not free text:

| Form | Meaning |
|---|---|
| `role(<roleName>)` | holders of that role on the workflow's item |
| `role(<roleName>;<communityName>)` and variants | role scoped to community, domain, entity level, or related assets |
| `user(<userName>)` | one specific user |
| `group(<groupName>)` | everyone in a group |

A bare value is rejected with *"The user expression 'x' is invalid."*

**`role(...)` does not resolve group-held or inherited responsibilities.** If a
role reaches users via a group, or is inherited from a parent resource — the
common case — `role()` will find nobody, the task will have no candidate, and
the run dies with no form shown. Collibra's *own* permission layer resolves the
same role correctly, which is why the Start Workflow role setting works where
the BPMN expression doesn't. This workflow therefore assigns the task to the
initiator and relies on the definition's role setting for access control.

If the definition has the *validate candidate users* setting enabled, a task
carrying only `flowable:assignee` and no `candidateUsers` is rejected with
*"no candidate users are defined for task"*.

---

## 8. Troubleshooting

| Symptom | Cause |
|---|---|
| Banner: *"This workflow uses version 1 of the Collibra Java API"* | `targetNamespace` is not `http://www.collibra.com/apiv2`. Not fixable by changing scripts; survives restarts. |
| Deploy rejected: *security validation errors … script 'X'* | An `import` inside a `try {}` block, or a class not on the approved-imports list. Check the server log for the exact class. |
| Action doesn't appear on the asset page | Assignment rule Type is a specific asset type; or missing `Start workflow` global permission; or the user lacks the Start Workflow resource role on *that* asset. Admins bypass the permission checks, so an admin seeing it proves only that the scope is right. |
| *"The user expression 'x' is invalid"* | Candidate expression not wrapped in `user(...)` / `role(...)` / `group(...)`. |
| *"no suitable candidate(s) … or they don't have permission to participate"* | Missing `Participate in workflow` global permission. |
| *"no candidate users are defined for task"* | Task has an assignee but no `candidateUsers`, with candidate validation enabled. |
| Run reports completion but nothing changed | Older builds only. Both outcomes now comment on the asset; check the asset's comments for the error. |
| *"attributeNotFoundId"* | An attribute *type* ID was supplied where an attribute *instance* ID was expected — the failure mode the current form design removes. |
| Script fails with `No signature of method` | Groovy usually lists valid alternatives in the same message. That enumeration is the fastest route to the right signature. |

Server-side detail lives in `dgc.log`. Filter on
`wf_process_definition_key=contributorElevatedActions` or the literal text of a
failure message — beware of matching entries from an older definition with a
different process ID.

---

## 9. Testing checklist

- [ ] Contributor-licensed user holding the role sees **Actions → Manage
      Attributes and Responsibilities** on an asset
- [ ] The form opens for that user
- [ ] A user without the role does not see the action
- [ ] The action appears on more than one asset type (confirms `Asset`-level scoping)
- [ ] Set an attribute value — on an empty attribute (creates) and a populated
      one (updates)
- [ ] Clear an attribute value
- [ ] Give someone a role, then take it away
- [ ] Confirm inherited responsibilities are untouched by the removal
- [ ] A deliberate error (a misspelled attribute name) comments the failure on
      the asset rather than completing silently
- [ ] The audit comment records the actor, and the justification when supplied

---

## 10. Known limitations

- Only string-valued attribute types are exercised. Numeric, date, boolean and
  value-list types may need conversion before `.value()` accepts them.
- All form fields display for every action; conditional visibility is not wired up.
- Responsibilities can be assigned to users, not groups.
- **Clear** removes every value of a multi-valued attribute type, not a chosen one.
- Removing a role only affects directly-assigned responsibilities; inherited ones
  are reported and skipped by design.
- Script tasks cannot call REST APIs — no HTTP client is on the approved-imports
  list. Any HTTP-based integration must live outside the workflow engine.

---

## Sources

- [Workflow permissions](https://productresources.collibra.com/docs/collibra/latest/Content/Workflows/co_workflow-permissions.htm)
- [How to manage the new workflow permissions](https://productresources.collibra.com/docs/collibra/latest/Content/Tutorials/howto_new-workflow-permissions.htm)
- [View and edit workflow definition settings](https://productresources.collibra.com/docs/collibra/latest/Content/Workflows/ManageWorkflows/co_general-wf-settings.htm)
- [Deploy a workflow](https://productresources.collibra.com/docs/collibra/latest/Content/Workflows/ManageWorkflows/ta_deploy-wf.htm)
- [Defining users, roles, and permissions](https://productresources.collibra.com/docs/collibra/latest/Content/Settings/UsersAndGroups/co_user-roles-permissions.htm)
- [Future-proof your script tasks](https://developer.collibra.com/tutorials/future-proof-your-script-tasks) — approved imports list
- [Java API v1 to v2 mapping](https://developer.collibra.com/workflows/designing-workflows/processes/process-execution/java-api-v1-to-v2-mapping)
- [Candidate user expressions](https://developer.collibra.com/workflows/designing-workflows/processes/shape-repository/user-task/candidate-user-expressions)
