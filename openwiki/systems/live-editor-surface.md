---
type: system
title: Live Editor Surface (File Browser, Editor, Diff, Terminal, LSP)
description: The client-side components and server-side API routes that make up the in-sandbox editor experience — file-browser, file-editor (Monaco), file-diff-viewer, terminal, autocomplete, and the LSP bridge — and how every server route reconnects to the kept-alive sandbox via Sandbox.get().
tags: [monaco, git-diff-view, file-browser, file-editor, lsp, terminal, autocomplete, sandbox, vscode-jsonrpc, typescript-language-server, sandbox-reconnect]
verified:
  - by: openwiki/0.5.1
    at: 2026-09-14T13:12:47.107Z
sources:
  - id: openwiki-source-c9658dc3e15988bffd23d276
    resource: repo://app/api/tasks/%5BtaskId%5D/autocomplete/route.ts
  - id: openwiki-source-4be4f915c18a1f150301ecd1
    resource: repo://app/api/tasks/%5BtaskId%5D/create-file/route.ts
  - id: openwiki-source-28d8b21090bdaffd83b9f038
    resource: repo://app/api/tasks/%5BtaskId%5D/create-folder/route.ts
  - id: openwiki-source-a901e33ab320201622ac1d61
    resource: repo://app/api/tasks/%5BtaskId%5D/delete-file/route.ts
  - id: openwiki-source-cc28ff8dbcc824fed3d0880f
    resource: repo://app/api/tasks/%5BtaskId%5D/diff/route.ts
  - id: openwiki-source-295405996c095885eca11b3e
    resource: repo://app/api/tasks/%5BtaskId%5D/discard-file-changes/route.ts
  - id: openwiki-source-07a2f4893bb9e626c14b8eb3
    resource: repo://app/api/tasks/%5BtaskId%5D/file-content/route.ts
  - id: openwiki-source-ccfb27586ee03634de416ffa
    resource: repo://app/api/tasks/%5BtaskId%5D/file-operation/route.ts
  - id: openwiki-source-fcc27bd5237d574a48c61e83
    resource: repo://app/api/tasks/%5BtaskId%5D/files/route.ts
  - id: openwiki-source-a7d18c322c6e60bfd828ce4c
    resource: repo://app/api/tasks/%5BtaskId%5D/lsp/route.ts
  - id: openwiki-source-0520d7782d77add9cecda0f4
    resource: repo://app/api/tasks/%5BtaskId%5D/save-file/route.ts
  - id: openwiki-source-363e17e53a8fa86bd5a0ca85
    resource: repo://app/api/tasks/%5BtaskId%5D/terminal/route.ts
  - id: openwiki-source-a82f1c53f0e14baa391baaad
    resource: repo://components/file-browser.tsx
  - id: openwiki-source-803a2d6930aac82c78f2c823
    resource: repo://components/file-diff-viewer.tsx
  - id: openwiki-source-763dab4d9193e3d4472e2937
    resource: repo://components/file-editor.tsx
  - id: openwiki-source-768977d69a8ca29ff9eeaca0
    resource: repo://components/terminal.tsx
  - id: openwiki-source-dcf3fe9f5d8eb342de46c68a
    resource: repo://lib/atoms/file-browser.ts
  - id: openwiki-source-5b54a58d1b51cd490b0e7162
    resource: repo://package.json
generated: { by: "openwiki/0.5.1", at: "2026-09-14T13:12:47.107Z" }
---

# Live Editor Surface

The "live editor surface" is the in-sandbox coding workspace that lives on every task page after the agent has produced a branch: the file tree on the left, the editor or diff on the right, and the terminal pane along the bottom. It is built from three client components — [`components/file-browser.tsx`](repo://components/file-browser.tsx), [`components/file-editor.tsx`](repo://components/file-editor.tsx), and [`components/file-diff-viewer.tsx`](repo://components/file-diff-viewer.tsx) — plus the standalone [`components/terminal.tsx`](repo://components/terminal.tsx), all of which talk to a dedicated set of server routes under [`app/api/tasks/[taskId]/`](repo://app/api/tasks/[taskId]/). All twelve of those routes share the same preamble: load the task row, scope by `userId`, look the sandbox up in the in-memory registry, fall back to `Sandbox.get({ sandboxId, teamId, projectId, token })`, then run a command or read a file.

<!-- openwiki: broken internal link [../integrations/vercel-sandbox.md#reconnect-pattern-sandboxget] heading anchor "reconnect-pattern-sandboxget" does not exist in "../integrations/vercel-sandbox.md". Fix the href or restore the target, then delete this comment. -->
For the sandbox phases that put the VM in place (`Sandbox.create` → clone → install → dev server → push → shutdown), see [Sandbox Lifecycle](../concepts/sandbox-lifecycle.md). For the cross-execution reconnect pattern (`getSandbox(taskId) ?? Sandbox.get(...)`) and why it works after the worker has returned, see [Integration: Vercel Sandbox → Reconnect pattern](../integrations/vercel-sandbox.md#reconnect-pattern-sandboxget) and the same section in [Sandbox Lifecycle → Phase 5 — Reconnect](../concepts/sandbox-lifecycle.md#phase-5--reconnect-sandboxget). For the per-task status model that the editor surface assumes is `processing | completed | error | stopped`, see [Task Lifecycle & Status Model](../concepts/tasks-lifecycle.md).

## Component layout and the four view modes

`task-details.tsx` mounts two copies of `<FileBrowser>` (one in the desktop resizable pane, one in the mobile drawer) and two copies of `<FileDiffViewer>` (one in the desktop code pane, one in the mobile tab strip). Both surface the same `viewMode` prop, which is one of four strings:

| Mode | Source | Diff endpoint | Diff shape |
| --- | --- | --- | --- |
| `'remote'` | `GET /api/tasks/:taskId/files?mode=remote` (PR comparison via Octokit `repos.compareCommits` against `main`/`master`) | `GET /api/tasks/:taskId/diff` (base = `main` SHA or `master`, head = `task.branchName`, or PR `base.sha` / `head.sha` when `task.prNumber` is set) | Branch-vs-PR diff |
| `'local'` | `GET /api/tasks/:taskId/files?mode=local` (`git status --porcelain` + `git diff --numstat` inside the sandbox) | `GET /api/tasks/:taskId/diff?mode=local` (`git diff origin/<branch>` inside the sandbox; falls back to `HEAD` if the remote branch does not exist yet) | Sandbox working tree vs `origin/<branch>` |
| `'all'` | `GET /api/tasks/:taskId/files?mode=all` (`git.getTree` recursive on `task.branchName`) | n/a — `FileDiffViewer` short-circuits and delegates to `FileEditor` for plain text render | File listing (no diff) |
| `'all-local'` | `GET /api/tasks/:taskId/files?mode=all-local` (`find` from the sandbox project root, joined with `git status --porcelain` to color new/modified entries) | n/a — also delegated to `FileEditor` | File listing (no diff) |

The `'all'` and `'all-local'` rows in `FileDiffViewer` skip the diff entirely and call `FileEditor` with the same content ([file-diff-viewer.tsx:330-344](repo://components/file-diff-viewer.tsx#L330-L344)). `'remote'` and `'local'` are the only modes that produce a `DiffFile` from `generateDiffFile(...)` and feed it to `<DiffView>`.

The mode is toggled in the file-browser header — there are two `Changes` / `Files` buttons and a `Remote` / `Sandbox` segmented control — and a `viewMode` prop is forwarded to every other editor component. Both `Files` modes render `FileEditor` (read-only for `'remote'`/`'all'` because the content comes from GitHub, editable for `'local'`/`'all-local'`), while both `Changes` modes render the unified `<DiffView>` from `@git-diff-view/react` with `DiffModeEnum.Unified`.

<!-- openwiki: mermaid parse failed and this diagram was converted to a text fence so it does not break rendering. Fix the diagram source and restore the mermaid fence. Parser error: Heuristic: an unescaped angle bracket inside a label breaks rendering; rephrase the label. -->
```text
flowchart LR
  FB["FileBrowser<br/>viewMode = r l a al"]
  FDV["FileDiffViewer<br/>viewMode = r l a al"]
  FE["FileEditor<br/>read-only or editable"]
  DV["DiffView<br/>@git-diff-view/react"]
  T["Terminal<br/>cwd tracker"]
  FILES["/files?mode="]
  DIFF["/diff?mode="]
  FC["/file-content"]
  TR["/terminal"]
  SB[("Sandbox<br/>via getSandbox<br/>or Sandbox.get")]
  GH[("GitHub<br/>via Octokit")]
  FB -- "list files" --> FILES
  FILES -- "mode=remote all" --> GH
  FILES -- "mode=local all-local" --> SB
  FDV -- "changes" --> DIFF
  FDV -- "files" --> FC
  DIFF -- "default" --> GH
  DIFF -- "mode=local" --> SB
  FC -- "default" --> GH
  FC -- "mode=local" --> SB
  FDV -- "r l: diff data" --> DV
  FDV -- "a al: text content" --> FE
  T -- "POST command" --> TR
  TR -- "sh -c" --> SB
```

*Diagram: how the four view modes select among the three API endpoints, and how `FileDiffViewer` switches between `DiffView` and `FileEditor` based on whether the user is in `Changes` or `Files`.*

## FileBrowser

[`components/file-browser.tsx`](repo://components/file-browser.tsx) is the resizable pane on the left of the task page. It holds a Jotai atom keyed by `taskId` (defined in [`lib/atoms/file-browser.ts`](repo://lib/atoms/file-browser.ts)) that stores one `ViewModeData` per mode — `files`, `fileTree`, `expandedFolders`, `fetchAttempted`, `error` — plus shared `loading` and `error` flags. The atom factory (`getTaskFileBrowserState(taskId)`) is created with `useMemo` so each task gets its own independent state, and the four mode slots are initialized to a fresh `structuredClone(emptyViewModeData)` so that switching modes never leaks one mode's tree into another.

The browser fetches once per mode on initial mount, with a `refreshKey` prop and a manual refresh button that force a refetch by clearing `fetchAttempted`. Each fetch is a `GET /api/tasks/:taskId/files?mode=<viewMode>` call; the route returns `{ files, fileTree, branchName, message? }`. On success the response is written into the matching slot. On the `'all-local'` mode the server returns files with `status` derived from `git status --porcelain` (added / modified / deleted / renamed), and the renderer colors the file name green for added, yellow for modified ([file-browser.tsx:1205-1212](repo://components/file-browser.tsx#L1205-L1212)). The `'all'` mode appends a `<Lock>` icon next to every entry because remote files are read-only.

The browser renders an `AlertDialog` ("Sandbox is not running" + a `Start Sandbox` button) when the response is a 410 Gone or the error text matches `Sandbox is not running`. That button calls `POST /api/tasks/:taskId/start-sandbox` and then re-issues the `files` request after a 6-second wait for the keep-alive rehydration to land.

### Mutations exposed in the context menu

`FileBrowser` is the integration point for every file-system mutation that originates in the UI:

| Action | Server route | Trigger |
| --- | --- | --- |
| New file | `POST /api/tasks/:taskId/create-file` | "New File" dialog or right-click → New File on a folder in `'all-local'` mode |
| New folder | `POST /api/tasks/:taskId/create-folder` | "New Folder" dialog or right-click → New Folder on a folder in `'all-local'` mode |
| Delete file | `DELETE /api/tasks/:taskId/delete-file` | Right-click → Delete in `'all-local'` mode (with an `AlertDialog` confirmation) |
| Cut / Copy / Paste | `POST /api/tasks/:taskId/file-operation` with `operation: 'cut' | 'copy'` | Right-click menu or `Cmd/Ctrl+X/C/V` shortcuts; paste dispatches `cp -r` or `mv` in the sandbox |
| Drag-and-drop move | `POST /api/tasks/:taskId/file-operation` with `operation: 'cut'` | Only enabled in `'all-local'` mode (the source tree is the sandbox) |
| Discard changes | `POST /api/tasks/:taskId/discard-file-changes` | Right-click → Discard Changes in `'local'` mode (with `AlertDialog`) |
| Sync changes / Reset | `POST /api/tasks/:taskId/sync-changes` / `POST /api/tasks/:taskId/reset-changes` | Bottom-bar buttons in `'local'` mode |

Cut / copy / paste / drag-and-drop all funnel through [`app/api/tasks/[taskId]/file-operation/route.ts`](repo://app/api/tasks/[taskId]/file-operation/route.ts), which accepts `{ operation, sourceFile, targetPath }` and dispatches `cp -r` or `mv` in the sandbox at `PROJECT_DIR = /vercel/sandbox/project`. The browser maintains an in-memory `clipboardFile` for the cut/copy state and a `draggedItem` for the drag state; both are cleared after a successful paste. The keyboard shortcuts are scoped to `viewMode === 'local' | 'all-local'` and bail out when the focus target is an `INPUT` / `TEXTAREA` / `contentEditable` element, so they do not fight with the editor ([file-browser.tsx:1010-1045](repo://components/file-browser.tsx#L1010-L1045)).

Discard changes routes through [`app/api/tasks/[taskId]/discard-file-changes/route.ts`](repo://app/api/tasks/[taskId]/discard-file-changes/route.ts#L64-L98), which probes `git ls-files <path>` and either runs `git checkout HEAD -- <path>` (tracked file → revert to last commit) or `rm <path>` (untracked new file). Either path runs in `cwd: PROJECT_DIR`.

## FileEditor

[`components/file-editor.tsx`](repo://components/file-editor.tsx) wraps `@monaco-editor/react` (`"@monaco-editor/react": "^4.7.0"`, [`package.json:22`](repo://package.json#L22)) with two Vercel/Geist themes defined in `handleBeforeMount`. The editor is mounted by `FileDiffViewer` for `'all'` / `'all-local'` modes; it is never mounted by `FileBrowser` directly. Three flags control whether editing is enabled:

- `viewMode === 'remote' | 'all'` → `isRemoteFile = true` → `isReadOnly = true` (file came from GitHub, not from the sandbox).
- `filename` contains `/node_modules/` → `isNodeModulesFile = true` → `isReadOnly = true` and the top of the editor renders an amber "Read-only: node_modules file" banner.
- Otherwise the editor is editable, and any change to `editor.getValue()` flips the parent-side `onUnsavedChanges(true)` callback.

Two keyboard shortcuts are wired up:

- `Cmd/Ctrl + S` is registered with `editor.addCommand(monaco.KeyMod.CtrlCmd | monaco.KeyCode.KeyS, ...)` ([file-editor.tsx:438-446](repo://components/file-editor.tsx#L438-L446)) and also at the `document` level ([file-editor.tsx:685-702](repo://components/file-editor.tsx#L685-L702)). Both call `handleSave`, which `POST`s `{ filename, content }` to `app/api/tasks/[taskId]/save-file/route.ts`. On `success: true` the local `savedContent` is updated so the unsaved-changes indicator clears.
- `F12` and `Cmd/Ctrl + Click` are both rewired to call the in-sandbox LSP bridge ([file-editor.tsx:562-622](repo://components/file-editor.tsx#L562-L622), [file-editor.tsx:626-680](repo://components/file-editor.tsx#L626-L680)). When the LSP returns a definition in a different file, `onOpenFile(filePath, lineNumber)` is fired so the parent can open a new tab at that line; when the target is the same file the editor calls `editor.setPosition(...)` + `editor.revealLineInCenter(...)` to jump to the definition.

Monaco's own TypeScript diagnostics are deliberately disabled — `noSemanticValidation: true`, `noSuggestionDiagnostics: true` ([file-editor.tsx:393-403](repo://components/file-editor.tsx#L393-L403)) — so that the in-browser language service does not double-report errors that the sandbox LSP is the authoritative source for. Syntax validation stays on (`noSyntaxValidation: false`) so missing semicolons are still flagged. The TypeScript compiler options still set `target: ES2020`, `jsx: JsxEmit.React`, `typeRoots: ['node_modules/@types']`, etc., so Monaco's syntax highlighting and bracket matching work.

The model URI is forced to `file:///<absolute-path>` on mount so Monaco's symbol map lines up with what the LSP returns ([file-editor.tsx:341-388](repo://components/file-editor.tsx#L341-L388)). If the model that came with the mount has a different URI, the existing model is disposed and a new one is created with the corrected URI; if a model with the correct URI already exists in Monaco's registry, it is reused.

## FileDiffViewer

[`components/file-diff-viewer.tsx`](repo://components/file-diff-viewer.tsx) is the router between `FileEditor`, `DiffView`, and the four image / binary placeholders. It receives the same `viewMode` prop and either pulls `DiffData` from the parent-provided `diffsCache` (used in `Changes` mode so the parent can pre-warm all diffs after `onFilesLoaded` fires) or issues its own `GET /api/tasks/:taskId/diff?filename=...` (and `mode=local` when applicable) and caches the result in an `internalCacheRef`.

The diff file is built by `generateDiffFile(...)` from `@git-diff-view/file` (`"@git-diff-view/file": "^0.0.32"`, [`package.json:20`](repo://package.json#L20)); the `<DiffView>` component is from `@git-diff-view/react` with `diffViewMode={DiffModeEnum.Unified}`, `diffViewWrap={true}`, `diffViewFontSize={12}`. The theme follows the parent document's `dark` class and listens to the `prefers-color-scheme` media query plus a `MutationObserver` on `<html>` so dark/light flips react immediately ([file-diff-viewer.tsx:57-91](repo://components/file-diff-viewer.tsx#L57-L91)). When `oldContent === newContent` the viewer renders a "No changes detected" panel instead of a diff.

Special cases handled by `FileDiffViewer`:

- `isBinary && !isImage` → "Binary File" placeholder.
- `isImage` → `<img src="data:<mime>;base64,<content>">` with the mime derived from the extension.
- `viewMode === 'all' | 'all-local'` → delegate to `FileEditor` with `viewMode={viewMode}` so the read-only flag is set correctly.
- `error` from the fetch → red error message + "Unable to load diff for ...".
- `diffFile === null` (init failure) → "Error generating diff" panel.

## Diff route semantics

[`app/api/tasks/[taskId]/diff/route.ts`](repo://app/api/tasks/[taskId]/diff/route.ts) is the only route that has both a `mode=local` and a default (PR) branch.

### `mode=local`

1. Reconnects via `Sandbox.get(...)`.
2. Runs `git fetch origin <branchName>` (errors are tolerated — `fetch` can fail if the branch has never been pushed).
3. Probes `git rev-parse --verify origin/<branchName>`. If that fails the route compares against `HEAD` instead — the new-files-only case where the branch has not been pushed yet.
4. With the remote branch present it runs `git diff origin/<branchName> -- <file>` for the diff text and reads `git show origin/<branchName>:<file>` for the old content and `cat <file>` for the new content.
5. With the remote branch absent it reads `git show HEAD:<file>` for the old content (empty if the file is new) and `cat <file>` for the new content (empty if the file has been deleted).

### Default (PR diff)

1. Octokit `repos.getContent` for both the base ref and the head ref. The base ref is normally `main` with a fallback to `master`; when `task.prNumber` is set, the route fetches `pulls.get` and uses `base.sha` and `head.sha` directly so the diff is anchored to PR creation time, not to current `main`. If `prResponse.data.merged_at` and `prResponse.data.merge_commit_sha` are set and `task.prMergeCommitSha` is empty, the route writes `prMergeCommitSha` back to the DB ([diff/route.ts:382-390](repo://app/api/tasks/[taskId]/diff/route.ts#L382-L390)).
2. Returns `{ data: { filename, oldContent, newContent, language, isBinary, isImage, isBase64 } }`. Images come back with `isBase64: true` so `FileDiffViewer` can wrap them in a `data:image/...;base64,...` URL.
3. Binary non-image files short-circuit with `isBinary: true` and empty content.

## File-content and file listing routes

`file-content/route.ts` is the `'all'` / `'all-local'` counterpart to `diff/route.ts`. Both modes follow the same source-of-truth split (`'remote'` reads from GitHub, `'local'` reads from the sandbox) but the contract is simpler: a `{ oldContent, newContent, language, isBinary, isImage }` triple. With `mode=local` the old content is fetched from GitHub (`repos.getContent` on `task.branchName`) and the new content is `cat` in the sandbox; without `mode=local`, both default and `node_modules` paths read from the sandbox (the latter because GitHub never has those files).

`files/route.ts` is the file-tree listing. It picks among four modes:

- `mode=local` → `git status --porcelain` + `git diff --numstat origin/<branch>` inside the sandbox, with a `wc -l` fallback for untracked files so additions are reported.
- `mode=all-local` → `find . -type f` (excluding `.git`, `node_modules`, `.next`, `dist`, `build`, `.vercel`) joined with `git status --porcelain` to color entries.
- `mode=all` → `git.getTree` recursive on `task.branchName` via Octokit.
- `mode=remote` (default) → `repos.compareCommits` against `main` (fallback `master`), then `files: [{ filename, status, additions, deletions, changes }]`.

Both `local` modes map a `410` from the SDK to a clear-the-DB-columns recovery: they update `tasks` to set `sandboxId = null, sandboxUrl = null` and call `unregisterSandbox(taskId)`, then return a 410 to the client so the browser can render its "Sandbox is not running" state with a Start Sandbox button.

All four modes end with a recursive `addToFileTree` pass that builds the nested `{ children: { ... } }` shape consumed by `FileBrowser.renderFileTree`. Responses always set `Cache-Control: no-store, no-cache, must-revalidate` because file listings change frequently.

## Save / create / delete / discard semantics

[`app/api/tasks/[taskId]/save-file/route.ts`](repo://app/api/tasks/[taskId]/save-file/route.ts) writes through `sh -c "echo '<base64>' | base64 -d > '<escaped-filename>'"`. The content is base64-encoded by the route and the filename is shell-escaped with the standard `'\\''` trick so that special characters in either input cannot break out of the command. The base64 path exists specifically because heredocs (`cat > file << EOF`) would let a content like `EOF\nrm -rf /\nEOF` redefine the terminator. The command runs in `cwd: PROJECT_DIR`.

[`app/api/tasks/[taskId]/create-file/route.ts`](repo://app/api/tasks/[taskId]/create-file/route.ts) splits the path on `/`, runs `mkdir -p <parent>` if the path has more than one segment, then `touch <filename>` — both in the sandbox at `PROJECT_DIR`.

[`app/api/tasks/[taskId]/create-folder/route.ts`](repo://app/api/tasks/[taskId]/create-folder/route.ts) is a single `mkdir -p <foldername>` in the sandbox.

[`app/api/tasks/[taskId]/delete-file/route.ts`](repo://app/api/tasks/[taskId]/delete-file/route.ts) is `rm <filename>` in the sandbox. There is no path-traversal guard — the file-browser only ever issues these with a path the user selected from the tree, and the sandbox is per-task — but the sandboxId reconnect check (`if (!task.sandboxId) return 400`) is the gate against calling into a torn-down VM.

All three create/delete routes map a `410` from the SDK to a `410 Gone` response so the browser can render the "Sandbox is not running" alert.

## Terminal

[`components/terminal.tsx`](repo://components/terminal.tsx) is a black-bg / green-text prompt with a per-line history of `{ command | output | error, content, timestamp }`. Every command the user types is `POST`ed to [`app/api/tasks/[taskId]/terminal/route.ts`](repo://app/api/tasks/[taskId]/terminal/route.ts), which reconnects via `Sandbox.get(...)` and runs `sh -c <command>` in `cwd: PROJECT_DIR`. The route returns `{ success, data: { exitCode, stdout, stderr } }`; the terminal pushes a `command` line, then an `output` line for `stdout` and an `error` line for `stderr`, each with its own timestamp.

`Up` / `Down` arrow keys navigate an in-memory `commandHistory` array; the terminal remembers `cwd` by issuing a follow-up `pwd` over the terminal route after any successful `cd ...` command, then caches the result locally ([terminal.tsx:142-159](repo://components/terminal.tsx#L142-L159)). The terminal is a `forwardRef` so the parent (`TaskChat`) can call `clear()` and `getTerminalText()` to dump the transcript into a chat message.

### Tab completion

[`app/api/tasks/[taskId]/autocomplete/route.ts`](repo://app/api/tasks/[taskId]/autocomplete/route.ts) is the terminal's tab-completion engine. Given `{ partial, cwd }`, it:

1. Reconnects via `Sandbox.get(...)` and asks the sandbox for its real `pwd` (the request-side `cwd` is only used if the `pwd` query fails).
2. Splits `partial` on whitespace, takes the last token, and resolves the prefix to complete and the directory to scan. Absolute paths use the path part directly, `~/...` is expanded to `/home/vercel-sandbox/...`, relative paths are joined with the resolved `cwd`.
3. Runs `cd <escaped-dir> 2>/dev/null && ls -1ap 2>/dev/null` in the sandbox. The directory is shell-escaped with the same `'\\''` trick as `save-file`. The `-F` flag turns directories into `name/`, which the client uses to set `isDirectory`.
4. Returns `{ completions: [{ name, isDirectory }], prefix }` filtered by case-insensitive `startsWith(prefix)`.

The terminal then handles three cases ([terminal.tsx:204-272](repo://components/terminal.tsx#L204-L272)): single completion replaces the last token; multiple completions are listed and the longest common prefix is applied; if the common prefix equals the original prefix the completion list is left in the output stream only.

## LSP bridge

[`app/api/tasks/[taskId]/lsp/route.ts`](repo://app/api/tasks/[taskId]/lsp/route.ts) is the only editor route that depends on a different transport — the dependencies `vscode-jsonrpc` (`^8.2.1`), `typescript-language-server` (`^5.1.3`), and `ws` (`^8.19.0`) are declared in [`package.json:67-69`](repo://package.json#L67) for the planned `vscode-jsonrpc` over WebSocket bridge to the in-sandbox language server, but the currently shipped implementation takes a more direct route: it writes a small `.lsp-helper.mjs` helper into the sandbox and runs it with `node`. The helper script uses `ts.createLanguageService` with a host backed by `ts.sys.readFile`/`fileExists`/`directoryExists` and `ts.parseJsonConfigFileContent` (walking up from `process.cwd()` to find the closest `tsconfig.json`), so it has full visibility into the project's `node_modules` and TypeScript configuration — exactly the type-resolution surface Monaco cannot reproduce in the browser.

The route is a switch over the LSP method name:

- `textDocument/definition` — the full pipeline: build the helper script with the requested `filename` and `(line, character)` baked in as string literals, write it to `.lsp-helper.mjs` in the sandbox, `node` it, parse the trailing `JSON.stringify(...)` line from stdout, `rm` the helper, and return `{ definitions: [{ uri: 'file://<path>', range: { start: { line, character }, end: { line, character } } }] }`.
- `textDocument/hover` and `textDocument/completion` — currently return `{ hover: null }` and `{ completions: [] }` respectively, with the same switch shape ready for the jsonrpc-backed implementation.
- Any other method → `400 { error: 'Unsupported LSP method' }`.

The Monaco side ([file-editor.tsx:449-559](repo://components/file-editor.tsx#L449-L559)) calls `POST /api/tasks/:taskId/lsp` with `{ method: 'textDocument/definition', filename, position: { line, character } }` where both coordinates are converted from Monaco's 1-indexed to LSP's 0-indexed scheme. The returned ranges are converted back to 1-indexed before being handed to Monaco. When the target URI differs from the current model URI, `onOpenFile(filePath, lineNumber)` fires; the helper strips a leading `/vercel/sandbox` prefix because the sandbox's `ts` program reports absolute paths inside its working tree ([file-editor.tsx:601-612](repo://components/file-editor.tsx#L601-L612)).

`runtime = 'nodejs'` and `maxDuration = 60` are exported at the top of the route so the LSP bridge gets a generous function budget — definition lookups that touch a large `tsconfig.json` or many `node_modules` files can take several seconds.

## The reconnect preamble shared by every route

All twelve routes — `autocomplete`, `create-file`, `create-folder`, `delete-file`, `diff`, `discard-file-changes`, `file-content`, `file-operation`, `files`, `lsp`, `save-file`, `terminal` — share the same shape, illustrated below by [`save-file/route.ts:42-65`](repo://app/api/tasks/[taskId]/save-file/route.ts#L42-L65):

```ts
let sandbox = getSandbox(taskId)

if (!sandbox) {
  const sandboxToken = process.env.SANDBOX_VERCEL_TOKEN
  const teamId = process.env.SANDBOX_VERCEL_TEAM_ID
  const projectId = process.env.SANDBOX_VERCEL_PROJECT_ID

  if (!sandboxToken || !teamId || !projectId) {
    return NextResponse.json({ error: 'Sandbox credentials not configured' }, { status: 500 })
  }

  sandbox = await Sandbox.get({
    sandboxId: task.sandboxId,
    teamId,
    projectId,
    token: sandboxToken,
  })
}
```

This is the cross-execution reconnect pattern described in [Sandbox Lifecycle → Phase 5 — Reconnect](../concepts/sandbox-lifecycle.md#phase-5--reconnect-sandboxget). The in-memory `Map<taskId, Sandbox>` populated by `registerSandbox` during the original worker execution is the fast path; `Sandbox.get({ sandboxId })` is the cross-execution fallback. Every editor route assumes `keepAlive === true` for the task — without it the worker's shutdown path clears `task.sandboxId` and `task.sandboxUrl` and the routes would all fail to reconnect. The `POST /api/tasks/:taskId/start-sandbox` endpoint is what rehydrates a dropped sandbox when the user has opted into keep-alive.

Each route also shares a small ownership / soft-delete preamble (illustrated again by `save-file/route.ts:26-39`):

```ts
const [task] = await db
  .select()
  .from(tasks)
  .where(and(eq(tasks.id, taskId), eq(tasks.userId, session.user.id), isNull(tasks.deletedAt)))
  .limit(1)

if (!task) return NextResponse.json({ error: 'Task not found' }, { status: 404 })
if (!task.sandboxId) return NextResponse.json({ error: 'Task does not have an active sandbox' }, { status: 400 })
```

`getServerSession()` enforces auth, the `eq(userId)` clause scopes by ownership, `isNull(deletedAt)` excludes soft-deleted tasks, and the absence of `sandboxId` is the signal that the user needs to call `start-sandbox` (the editor surface handles this via the `AlertDialog` in `FileBrowser`). This same preamble shows up in every editor route — `autocomplete`, `create-file`, `create-folder`, `delete-file`, `diff`, `discard-file-changes`, `file-content`, `file-operation`, `files`, `lsp`, `save-file`, and `terminal` — so the editor surface can only be opened on tasks the user owns that have not been soft-deleted and that still have a reachable sandbox.

## Failure modes and invariants

- **`Sandbox.get` stale handle.** When the kept-alive sandbox has been garbage-collected by Vercel's SDK timeout (set at `Sandbox.create` time, independent of `keepAlive`), `Sandbox.get` throws. The editor routes return `500 { error: 'Failed to connect to sandbox' }` (or the route-specific wording) and the file-browser renders the "Sandbox is not running" alert with a Start Sandbox button. There is no automatic retry; the user has to click Start.
- **`sandbox.stop()` on a dead VM.** Not exercised in editor routes — they never stop a sandbox. Only the `killSandbox` helper, the `stop-sandbox` endpoint, and the `merge-pr` endpoint do, and they catch and swallow the rejection.
- **`runCommand` stream exceptions.** Several editor routes call `await result.stdout()` / `await result.stderr()` inside try/catch so a stream that throws does not surface as a 500. `terminal/route.ts:81-93` is the cleanest example.
- **Stale `sandboxId` after a 410.** `files/route.ts:282-319` and `files/route.ts:497-524` are the only two editor routes that proactively clear the DB columns when the SDK reports `status === 410`. The other editor routes leave the cleanup to the next `start-sandbox` call, which has its own `Sandbox.get + echo test` probe that clears the DB before creating fresh.
- **Image / binary files.** `file-content/route.ts` and `diff/route.ts` both return early with `isBinary: true` for non-image binaries so the editor renders a placeholder rather than trying to display binary content. Image files return `isBase64: true` plus the base64 payload, which `FileDiffViewer` wraps in a `data:image/<ext>;base64,...` URL. SVG mime is `image/svg+xml` and ico is `image/x-icon` — see the mime map in [`file-diff-viewer.tsx:284-298`](repo://components/file-diff-viewer.tsx#L284-L298).
- **node_modules files.** `file-content/route.ts:287-325` reads any path containing `/node_modules/` from the sandbox (not from GitHub). `FileEditor` flags the file as read-only and renders an amber banner.
- **shell injection.** `save-file/route.ts` encodes content as base64 and `autocomplete/route.ts` shell-escapes the directory argument with `'\\''`. Both routes comment explicitly that heredoc-style writes are unsafe because a content containing `EOF\n...` could redefine the terminator.
- **Concurrent edits.** There is no optimistic concurrency on save — the editor always wins. `discard-file-changes` is the only sanctioned revert, and it operates on `HEAD` rather than the last edit.
- **`maxDuration` for the LSP bridge.** `lsp/route.ts:8-9` declares `runtime = 'nodejs'` and `maxDuration = 60` so the 60-second Vercel function budget is allocated to a single in-flight LSP request.

## Extension points

- **New editor route.** Add a sibling route under `app/api/tasks/[taskId]/<verb>/route.ts`, copy the `getServerSession + ownership + sandboxId + reconnect preamble` shape, and add the route name to the bullet list at the top of this page. The route should map `410` to a clear-the-DB-columns recovery only if it is a stateful read (like `files`); write routes (`save-file`, `create-*`, `delete-file`) should map `410` to a `410 Gone` response so the client can prompt the user to restart the sandbox.
- **New file-tree action.** Add a `DropdownMenuItem` to either the folder context menu (`FileBrowser.tsx:1107-1169`) or the file context menu (`FileBrowser.tsx:1253-1307`) and wire it to a new server route. The `viewMode` guard pattern is the same — sandbox actions belong in `'local'` / `'all-local'`, remote actions belong in `'remote'` / `'all'`. Drag-and-drop is currently only enabled in `'all-local'` mode.
- **New LSP method.** Add a new `case 'textDocument/...':` to the switch in [`lsp/route.ts:78-230`](repo://app/api/tasks/[taskId]/lsp/route.ts#L78-L230). The `textDocument/definition` case is the reference implementation for the helper-script pattern; the `textDocument/hover` / `textDocument/completion` cases are stubs ready to be filled in. If the new method needs more than `ts.createLanguageService` can give, switch the route to the `vscode-jsonrpc + typescript-language-server over ws` model (the dependencies are already declared) and move the helper-script generation to a one-time `typescript-language-server` spawn that the route talks to over a WebSocket.
- **New Monaco theme / language mapping.** Add the extension to `getLanguageFromPath` ([file-editor.tsx:25-56](repo://components/file-editor.tsx#L25-L56)) and to the theme rule list (`handleBeforeMount`, [file-editor.tsx:237-334](repo://components/file-editor.tsx#L237-L334)). The same `getLanguageFromFilename` map exists on the server side in `file-content/route.ts` and `diff/route.ts` and must be kept in sync, since the server's language ID is what Monaco's syntax highlighter consumes.
- **New view mode.** Add the new string to the union types in `FileBrowser` (`viewMode?: 'local' | 'remote' | 'all' | 'all-local'`), `FileDiffViewer`, and `task-details.tsx`'s `viewMode` state, then add the corresponding `case` to the `files/route.ts` switch and a matching branch to `FileDiffViewer.fetchDiffData` ([file-diff-viewer.tsx:93-158](repo://components/file-diff-viewer.tsx#L93-L158)).
