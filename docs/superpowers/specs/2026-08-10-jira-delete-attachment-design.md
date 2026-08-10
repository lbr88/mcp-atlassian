# Jira Attachment Deletion Design

## Goal

Expose permanent deletion of one explicitly identified Jira attachment through
the Jira client and a first-class `jira_delete_attachment` FastMCP tool.

## Scope

The operation accepts exactly one non-empty `attachment_id`. It does not infer
an attachment from an issue or filename and does not support bulk deletion,
globs, or issue-wide deletion.

## Client behavior

`AttachmentsMixin.delete_attachment(attachment_id: str) -> None` validates that
the ID is non-empty, builds Jira's attachment resource URL, and invokes the
Atlassian client's existing `delete` operation. It selects REST API version 3
for Jira Cloud and version 2 for Jira Server/Data Center, matching the
repository's established `config.is_cloud` convention.

The Atlassian client treats Jira's HTTP 204 response as success and raises its
normal HTTP error for not-found, permission, and other Jira API failures. The
mixin does not translate or suppress those exceptions, allowing the MCP layer
to surface them consistently with existing Jira tools.

## MCP behavior

The server registers `jira_delete_attachment` in `jira_attachments` with Jira,
write, attachment, and destructive metadata. `@check_write_access` blocks the
operation in read-only mode before the Jira fetcher is called.

On success the tool returns a JSON object containing a concise message and the
exact deleted attachment ID:

```json
{
  "message": "Attachment deleted successfully",
  "attachment_id": "10001"
}
```

The parameter and tool descriptions state that deletion is permanent and that
callers must resolve and verify the exact attachment ID before invoking it.

## Testing

Focused unit tests mock only Jira's external HTTP/client boundary. Client tests
cover Cloud v3 and Server/Data Center v2 URL selection, a successful 204-style
delete, empty-ID validation, and representative HTTP errors. MCP tests cover
the structured confirmation, error propagation, and read-only blocking without
calling a real Jira instance.

## Documentation

Regenerate the Jira attachment tool page after registering the tool, and update
the manual toolset index so `jira_delete_attachment` is discoverable.
