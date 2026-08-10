# Jira Attachment Deletion Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add safe, explicit deletion of one Jira attachment through the client and FastMCP.

**Architecture:** Extend the existing Jira attachment mixin with a thin API-version-aware delete operation, then expose it through a write-protected destructive FastMCP tool. Reuse the Atlassian HTTP client and existing decorator so success and errors follow current repository behavior.

**Tech Stack:** Python 3.10+, atlassian-python-api, FastMCP, Pydantic, pytest, Ruff, mypy

## Global Constraints

- Accept one explicit `attachment_id` only.
- Use REST API v3 for Jira Cloud and v2 for Jira Server/Data Center.
- Deletion is permanent and must be blocked in read-only mode.
- Do not perform real Jira mutations in tests.
- Preserve the branch baseline at commit `9773a24e7ae2b61f8a6865a91ad0a68551a1d474`.

---

### Task 1: Jira client deletion behavior

**Files:**
- Modify: `src/mcp_atlassian/jira/attachments.py`
- Modify: `src/mcp_atlassian/jira/protocols.py`
- Test: `tests/unit/jira/test_attachments.py`

**Interfaces:**
- Consumes: `JiraConfig.is_cloud`, `self.jira.resource_url(resource, api_version=...)`, and `self.jira.delete(path)`
- Produces: `AttachmentsMixin.delete_attachment(attachment_id: str) -> None`

- [ ] **Step 1: Write failing client tests**

Add tests which configure `is_cloud` as `True` and `False`, call
`delete_attachment("10001")`, and verify the observable client boundary uses
`rest/api/3/attachment/10001` for Cloud and `rest/api/2/attachment/10001` for
Server/Data Center. Add tests for an empty ID and propagated `HTTPError` values
representing 404 and 403 responses.

- [ ] **Step 2: Run the client tests and verify RED**

Run:

```bash
uv run pytest tests/unit/jira/test_attachments.py -q
```

Expected: the new tests fail because `delete_attachment` is absent.

- [ ] **Step 3: Implement the minimal client method and protocol**

Implement:

```python
def delete_attachment(self, attachment_id: str) -> None:
    if not attachment_id:
        raise ValueError("attachment_id is required")
    api_version = "3" if self.config.is_cloud else "2"
    url = self.jira.resource_url(
        f"attachment/{attachment_id}", api_version=api_version
    )
    self.jira.delete(url)
```

Declare the same signature and contract on `AttachmentsOperationsProto`.

- [ ] **Step 4: Run the client tests and verify GREEN**

Run `uv run pytest tests/unit/jira/test_attachments.py -q` and expect all tests
to pass.

### Task 2: FastMCP deletion tool

**Files:**
- Modify: `src/mcp_atlassian/servers/jira.py`
- Test: `tests/unit/servers/test_jira_server.py`

**Interfaces:**
- Consumes: `AttachmentsMixin.delete_attachment(attachment_id: str) -> None`
- Produces: FastMCP tool `jira_delete_attachment` returning a JSON string with `message` and `attachment_id`

- [ ] **Step 1: Write failing MCP tests**

Register `delete_attachment` in the test MCP fixture. Add a success test that
invokes `jira_delete_attachment` with `{"attachment_id": "10001"}` and checks
the returned JSON. Add an API error propagation test and a read-only context
test proving the fetcher method is not called.

- [ ] **Step 2: Run the MCP tests and verify RED**

Run:

```bash
uv run pytest tests/unit/servers/test_jira_server.py -q
```

Expected: the new tests fail because the tool is absent.

- [ ] **Step 3: Implement the minimal MCP tool**

Register a Jira attachment tool with `write`, `attachments`, and
`toolset:jira_attachments` tags, `destructiveHint: True`, and
`@check_write_access`. Its description and parameter text must say deletion is
permanent and callers must resolve and verify the exact ID first. Call
`jira.delete_attachment(attachment_id)` and return:

```python
json.dumps(
    {
        "message": "Attachment deleted successfully",
        "attachment_id": attachment_id,
    },
    indent=2,
    ensure_ascii=False,
)
```

- [ ] **Step 4: Run the MCP tests and verify GREEN**

Run `uv run pytest tests/unit/servers/test_jira_server.py -q` and expect all
tests to pass.

### Task 3: Documentation and verification

**Files:**
- Modify: `docs/tools/jira-attachments.mdx`
- Modify: `docs/tools-reference.mdx`

**Interfaces:**
- Consumes: registered FastMCP tool metadata
- Produces: generated attachment reference and manual toolset index entries

- [ ] **Step 1: Regenerate tool documentation**

Run:

```bash
uv run python scripts/generate_tool_docs.py
```

Confirm `docs/tools/jira-attachments.mdx` documents the permanent deletion
warning and exact-ID requirement.

- [ ] **Step 2: Update the manual toolset index**

Add `jira_delete_attachment` to the `jira_attachments` row in
`docs/tools-reference.mdx` and update generated tool totals if the generator or
index tracks them.

- [ ] **Step 3: Run proportional verification**

Run:

```bash
uv run pytest tests/unit/jira/test_attachments.py tests/unit/servers/test_jira_server.py tests/unit/utils/test_toolsets.py -q
uv run ruff format --check src/mcp_atlassian/jira/attachments.py src/mcp_atlassian/jira/protocols.py src/mcp_atlassian/servers/jira.py tests/unit/jira/test_attachments.py tests/unit/servers/test_jira_server.py
uv run ruff check src/mcp_atlassian/jira/attachments.py src/mcp_atlassian/jira/protocols.py src/mcp_atlassian/servers/jira.py tests/unit/jira/test_attachments.py tests/unit/servers/test_jira_server.py
uv run mypy src/mcp_atlassian/jira/attachments.py src/mcp_atlassian/jira/protocols.py src/mcp_atlassian/servers/jira.py
```

Expected: all commands pass without errors.

- [ ] **Step 4: Commit the feature**

Stage only the planned source, test, and documentation files and commit with:

```bash
git commit -m "feat(jira): add attachment deletion tool"
```
