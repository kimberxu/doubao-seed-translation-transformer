# AGENTS.md - Repository Guidelines for AI Agents

This document defines the operational protocols, coding standards, and workflow procedures for AI agents (and humans) working in the `doubao-seed-translation-transformer` repository.
Strict adherence to these guidelines is required to maintain codebase integrity across the hybrid JavaScript (EdgeOne) and Go environments.

---

## 1. Build, Lint, and Test Commands

### JavaScript (EdgeOne / Cloudflare Workers)
This project does not use a `package.json` or standard Node.js build pipeline. It relies on standalone script files.

*   **Syntax & Runtime Check (Lint-lite)**:
    Use the Node.js watcher to catch syntax errors and runtime crashes during development.
    ```bash
    node --watch edge-function.js
    ```
    *Note: This script will not "build" anything. It simply validates that the file is parseable and valid JavaScript.*

*   **Testing (Manual Smoke Tests)**:
    There is no automated test harness for the JavaScript components. Verification is performed via `curl` against the deployed endpoint.
    
    **Standard Smoke Test:**
    ```bash
    curl -X POST https://<edgeone-endpoint>/v1/chat/completions \
      -H "Authorization: Bearer <token>" \
      -H "Content-Type: application/json" \
      -d '{
        "model": "doubao-seed-translation",
        "messages": [
          {"role": "system", "content": "{\"target_language\": \"ja\"}"},
          {"role": "user", "content": "Hello"}
        ],
        "stream": false
      }'
    ```

### Go Service (`go/` directory)
The Go component follows standard Go tooling conventions. All commands should be run from the repository root or `go/` directory as specified.

*   **Build**:
    Compile the binary to `doubao`.
    ```bash
    cd go && go build -o doubao .
    ```

*   **Lint & Format**:
    Uses `gofmt` and `go vet`.
    ```bash
    make lint
    # Or manually:
    cd go && gofmt -s -w . && go vet ./...
    ```

*   **Run All Tests**:
    Executes all unit tests in the module.
    ```bash
    make test
    # Or manually:
    cd go && go test -v ./...
    ```

*   **Run a Single Test**:
    To target a specific test function (e.g., `TestHandlerLogic`):
    ```bash
    cd go && go test -v -run TestHandlerLogic ./...
    ```
    *Note: Currently, no `_test.go` files exist. When creating tests, place them alongside the source files.*

---

## 2. Code Style & Conventions

### General Principles
*   **Simplicity**: Prefer simple, readable code over clever abstractions.
*   **Consistency**: Follow the patterns of existing files (`edge-function.js`, `main.go`).
*   **No Build Tools**: Do not introduce `npm`, `webpack`, or complex build steps unless explicitly requested.

### JavaScript Style (`edge-function.js`, `cf-workers.js`)
*   **Formatting**:
    *   **Indentation**: 4 spaces.
    *   **Semicolons**: Always use trailing semicolons.
    *   **Quotes**: Prefer double quotes (`"`) for strings to match the existing module style. Single quotes are acceptable for internal implementation details if consistent within the block.
    *   **Max Line Length**: 100 characters where possible.

*   **Naming**:
    *   **Variables/Functions**: `lowerCamelCase` (e.g., `parseTranslationOptions`, `fetchUpstream`).
    *   **Constants**: `SCREAMING_SNAKE_CASE` for top-level configuration (e.g., `DOUBAO_BASE_URL`).
    *   **Files**: `kebab-case` (e.g., `edge-function.js`).

*   **Language Features**:
    *   Use `const` for immutable references and `let` for mutable state. Avoid `var`.
    *   Use Arrow Functions (`() => {}`) for callbacks and simple helpers.
    *   Avoid TypeScript syntax; this is pure JavaScript.

*   **Error Handling**:
    *   Return JSON error responses with standard HTTP status codes (400, 500).
    *   Use `try/catch` blocks around upstream API calls.
    *   Do not crash the worker; always return a valid Response object.

### Go Style (`go/` directory)
*   **Formatting**:
    *   Strictly follow `gofmt` standards.
    *   Run `make fmt` before committing.

*   **Naming**:
    *   **Exported**: `PascalCase`.
    *   **Unexported**: `camelCase`.
    *   **Constants**: `PascalCase` or `SCREAMING_SNAKE_CASE` (if matching external specs).

*   **Error Handling**:
    *   Check all errors. Do not use `_` to ignore errors unless absolutely necessary and commented.
    *   Wrap errors with context: `fmt.Errorf("failed to parse config: %w", err)`.

---

## 3. Project Structure & Organization

*   `edge-function.js`:
    *   **Core Logic**: Holds the complete EdgeOne fetch handler.
    *   **Responsibilities**: Request validation, options parsing, upstream calls, OpenAI-response shaping.
    *   **Constraint**: Must remain self-contained.
    *   **Default Target Language Logic**:
        *   If `target_language` is not specified in `systemPrompt`, the system inspects the user's input text.
        *   Calculates the number of Chinese characters (CJK Unified Ideographs) and English letters.
        *   **If Chinese > English**: Defaults to `target_language: "en"`.
        *   **Otherwise**: Defaults to `target_language: "zh"`.

*   `cf-workers.js`:
    *   **Adapter**: Cloudflare Workers specific implementation.
    *   **Sync**: Keep logic consistent with `edge-function.js`.

*   `go/`:
    *   **Microservice**: Independent Go implementation for containerized environments.
    *   **Dockerfile**: Uses multi-stage build (Distroless or Alpine).

*   `docs/`:
    *   **Source of Truth**: `README.md` is the primary documentation.
    *   **Translations**: `ARCHITECTURE.zh-CN.md`, `USAGE.zh-CN.md`.

---

## 4. Git & Workflow Guidelines

### Commit Messages
Follow Conventional Commits (`type(scope): description`).
*   `feat(edge)`: New feature in edge function.
*   `fix(go)`: Bug fix in Go module.
*   `docs(readme)`: Documentation updates.
*   `refactor`: Code change that neither fixes a bug nor adds a feature.

**Language**: Prefer English for code/commit messages. Chinese descriptions are acceptable for user-facing behavior changes if the audience is primarily Chinese.

### Pull Requests
1.  **Scope**: One logical change per PR.
2.  **Verification**:
    *   For JS: Include `curl` smoke test results in PR description.
    *   For Go: Ensure `make test` passes.
3.  **Review**: Wait for approval before merging.
4.  **No Force Push**: Unless necessary to fix sensitive data leaks.

---

## 5. Deployment

*   **Configuration**:
    *   Update `CONFIG.DOUBAO_BASE_URL` in `edge-function.js` if the upstream provider changes.
    *   **Security**: NEVER commit API keys or Bearer tokens. Use environment variables or secrets management.

*   **EdgeOne**:
    *   Ensure strict HTTPS enforcement.
    *   Verify Bearer token propagation in staging before production.

*   **Go Service**:
    *   Build Docker image: `cd go && docker build -t doubao-service .`
    *   Deploy via standard container orchestration (K8s, Docker Compose).
