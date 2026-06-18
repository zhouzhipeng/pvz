```markdown
# GDevelop MCP Improvement Suggestions

## Context

I can accept direct project JSON modification when it is paired with MCP/editor validation. The ideal workflow is not “never touch JSON,” but “allow controlled JSON edits, then ask MCP/GDevelop to validate, reload/sync, and save the editor project safely.”

MCP already knows the active project path from `gdevelop_get_editor_state`, so tools should not require passing the project JSON file path manually. They should operate on the currently open project, create backups/snapshots automatically, and validate through GDevelop before committing.

## Recommended Direction

Add a “validated direct JSON edit” workflow for advanced changes:

1. MCP reads the current in-memory project and/or saved JSON path.
2. The agent applies a JSON patch or generated replacement to the project model.
3. MCP validates by unserializing through GDevelop.
4. MCP runs event validation and generated JS preflight where relevant.
5. MCP reloads/syncs the editor state.
6. MCP saves only if validation passes.
7. MCP reports a compact semantic diff and backup/snapshot id.

This would keep most of the speed of direct JSON editing while avoiding stale/broken project state.

## Main Pain Points

1. Extension function event editing is harder than scene event editing.
   - Scene events have file-based validation/replacement tools.
   - Extension object functions currently require inline `events_json`.
   - Large function rewrites are slow and context-heavy.

2. Extension/object JSON patching is missing.
   - Need validated patch tools for extension functions, events-based objects, child objects, property descriptors, and child instances.

3. Prefab/custom object coordinate spaces are hard to debug.
   - `Object.X()` inside object functions can be misleading.
   - Widening a prefab area affects `IsCursorOnObject(Object)`.
   - Need geometry inspection for parent, children, points, hit areas, and cursor layer coordinates.

4. Resource property binding is unclear.
   - A `Resource` property on a prefab does not automatically prove the child Sprite frame is dynamically bound.
   - Need tools to inspect or create actual property-to-child-resource binding.

5. Runtime verification after extension edits is repetitive.
   - Extension edits make existing previews stale.
   - Save/close/relaunch/inspect should be a one-call flow.

## Suggested New MCP Tools

### 1. `apply_validated_project_json_patch`

Apply a JSON patch to the currently open project, without requiring the caller to pass the project path.

Inputs:
- `patch` or `patch_file`
- `dry_run`
- `summary_only`
- optional `scope`: scene, extension, extension_object, extension_function
- optional target fields: `scene_name`, `extension_name`, `parent_kind`, `parent_name`, `function_name`

Behavior:
- MCP discovers the active project path/state itself.
- Creates an automatic snapshot/backup.
- Applies patch to a temporary project model first.
- Validates GDevelop unserialization.
- Runs relevant event/function lint.
- Runs generated JS preflight for extensions.
- Applies to editor memory only if valid.
- Saves only when requested or after explicit successful validation.

### 2. `replace_extension_function_events_from_current_project_file`

Replace an extension function’s event body from a generated JSON file or patch, using the active project path internally.

Better name could be:
- `replace_extension_function_events_from_file`
- but no project JSON path should be required.

Inputs:
- `extension_name`
- `parent_kind`
- `parent_name`
- `function_name`
- `events_json_file`
- `dry_run`
- `summary_only`
- `include_generated_code`

Behavior:
- Validate event JSON.
- Validate function scope.
- Replace only if generated code preflight passes.
- Sync editor state.
- Return compact event diff.

### 3. `validate_current_project_json`

Validate the currently open project JSON/model after direct or generated changes.

Behavior:
- MCP knows the active project file.
- Unserializes with GDevelop.
- Checks scenes, extensions, resources, events, and generated code.
- Reports specific broken paths.
- Does not mutate unless paired with an apply/sync step.

### 4. `sync_editor_from_validated_project_json`

For cases where a controlled JSON edit was made externally.

Behavior:
- MCP reads the active project file path itself.
- Validates saved JSON first.
- If valid, reloads/syncs the editor project model.
- Warns if unsaved in-memory editor changes would be overwritten.
- Creates backup before sync.

### 5. `apply_validated_extension_patch`

Focused patch tool for extension internals.

Use cases:
- Add/remove/replace one condition.
- Add/remove/replace one action.
- Patch child Sprite frames.
- Patch events-based object area.
- Patch property descriptors.
- Patch child initial instance positions.

### 6. `inspect_custom_object_runtime_geometry`

Prefab coordinate debugging tool.

Should report:
- parent/custom object area
- rendered bounds
- hit-test bounds
- child local positions
- child scene positions
- point coordinates like `Card.PointX("centre")`
- cursor coordinates by layer
- whether hit testing uses parent area or visible child bounds

### 7. `inspect_prefab_property_bindings`

Should report:
- public/private properties
- child objects using static resources
- child objects actually bound to properties
- events that read/write/use each property
- warnings when a `Resource` property exists but no child/event uses it dynamically

### 8. `bind_child_sprite_resource_property`

High-level helper to bind an events-based object `Resource` property to a child preview sprite.

Behavior:
- If native binding exists, apply it.
- If not, generate supported runtime logic or explain limitation.
- Validate and read back the binding.

### 9. `save_and_relaunch_preview_paused`

One-call stale preview recovery:
- Save project.
- Close all running previews.
- Launch fresh paused debug preview.
- Wait until ready.
- Return debugger id, scene name, object counts, variables, and errors.

## Skill Doc Updates

Update `skills/gdevelop-mcp/SKILL.md` to support a controlled direct-JSON workflow.

### Add: Validated Direct JSON Workflow

The skill should distinguish unsafe direct JSON edits from controlled JSON edits.

Unsafe:
- modifying `PvZ.json` while editor has unsaved in-memory changes
- skipping MCP validation
- assuming editor memory matches disk
- saving from editor after an un-synced disk edit

Allowed with explicit user permission:
- generate a JSON patch/file
- apply through MCP validation
- sync editor model after validation
- save through MCP only after successful validation
- keep automatic backup/snapshot

### Add: Custom Object / Prefab Section

Include:
- custom object coordinate-space rules
- warning about widened object areas affecting `IsCursorOnObject(Object)`
- recommended hit testing against visible child points/bounds
- recipe for mouse-follow preview sprites
- stale preview warning after extension edits

### Add: Resource Property Binding Section

Clarify:
- A `Resource` property on an events-based object is not enough by itself.
- The tool/agent must verify that the child object frame or runtime event actually uses the property.
- Prefer a binding helper when available.

## Highest Priority Improvements

1. `apply_validated_project_json_patch`
2. `replace_extension_function_events_from_file`
3. `sync_editor_from_validated_project_json`
4. `inspect_custom_object_runtime_geometry`
5. `inspect_prefab_property_bindings`
6. `save_and_relaunch_preview_paused`

These would allow fast direct-JSON-style edits while preserving MCP validation, editor synchronization, and runtime safety.
```