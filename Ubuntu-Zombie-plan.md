# Ubuntu Zombie Chat Replacement Plan

## Objective

Replace Ubuntu Zombie's primitive chat experience with the standalone llama.cpp WebUI while preserving Ubuntu Zombie's product shell, authentication and deployment model, and existing model access. The replacement should deliver streaming chat, local conversation history, branching, attachments, model settings, reasoning display, and optional MCP tools without exposing backend credentials in the browser.

## Current WebUI Baseline

The replacement UI is a client-only SvelteKit 2/Svelte 5 application with a static adapter and hash routing. Its main integration surfaces are:

- `src/routes/+layout.svelte`: global sidebar, settings, theme, server initialization, keyboard shortcuts, and route shell.
- `src/routes/+page.svelte` and `src/routes/chat/[id]/`: new and existing conversation entry points.
- `src/lib/components/app/chat/`: chat screen, composer, messages, attachments, sidebar, and settings.
- `src/lib/stores/chat.svelte.ts`: streaming and agentic-loop orchestration.
- `src/lib/stores/conversations.svelte.ts`: conversation lifecycle, branching, import, and export.
- `src/lib/services/chat.ts`: OpenAI-compatible chat and image requests, streaming parsing, reasoning, and tool calls.
- `src/lib/services/database.service.ts`: browser-local Dexie persistence for conversations and message trees.
- `src/lib/stores/settings.svelte.ts`: browser-local API and generation settings.
- `src/lib/stores/mcp.svelte.ts` and `src/lib/services/mcp.service.ts`: optional browser-to-MCP connectivity.
- `svelte.config.js`: static fallback, relative paths, hash router, and inline bundle strategy.
- `vite.config.ts`: standalone llama.cpp packaging, including generation of `public/index.html.gz`.

The source WebUI currently assumes that users may enter an API base URL and API key in browser settings. That behavior must not be carried into Ubuntu Zombie unchanged if Ubuntu Zombie owns or brokers credentials.

## Recommended Integration Shape

Use the WebUI as a separately bounded frontend module or route inside Ubuntu Zombie, backed by an Ubuntu Zombie same-origin OpenAI-compatible gateway. Do not immediately rewrite the WebUI into Ubuntu Zombie's existing chat component model. Keeping the chat domain intact reduces regressions in streaming, message branching, attachments, and tool execution.

Prefer this order:

1. **Same-framework route integration:** If Ubuntu Zombie is already compatible with SvelteKit/Vite, import the chat feature into its application shell and share navigation, authentication, and design tokens.
2. **Mounted static application:** If frameworks differ, build the WebUI as a static asset mounted at an authenticated path such as `/chat/`, with Ubuntu Zombie providing navigation and a same-origin API gateway.
3. **Iframe isolation:** Use only as a short-lived migration path when dependency or CSS collisions prevent direct mounting. Define explicit theme, navigation, and authentication messaging, then retire the iframe after parity is reached.

Avoid copying only visual components while retaining the primitive chat state and request code. The UI depends on coordinated stores, services, database types, attachment processing, and streaming behavior.

## Phase 0: Resolve Product and Architecture Decisions

Before implementation, inspect Ubuntu Zombie's repository and record:

- Frontend framework, package manager, build output, router, styling system, and supported browser targets.
- Files and routes implementing the current chat, including message types, streaming parser, cancellation, retries, model selection, and error states.
- Authentication mechanism and whether the browser receives provider credentials.
- Current model API shape, including endpoint paths, streaming protocol, supported OpenAI fields, file handling, tool calls, and image generation.
- Conversation ownership and storage: browser-only, Ubuntu Zombie backend, or another service.
- Deployment topology, base path, CSP, reverse proxy, offline behavior, and desktop/mobile packaging.
- Existing tests, accessibility requirements, telemetry, and release controls for chat.

Create an explicit integration decision record choosing direct route integration, mounted static application, or temporary iframe. Include the ownership boundary for conversation data and settings.

## Phase 1: Define the Contract

### API gateway

Provide an authenticated, same-origin Ubuntu Zombie gateway that exposes only the capabilities required by the UI:

- Chat completions with streaming and non-streaming responses.
- Model discovery when model selection is enabled.
- Server capability/default information, replacing or adapting llama.cpp-specific `/props`.
- Optional image generation.
- Optional MCP/tool access through trusted server-side connections.

Normalize Ubuntu Zombie's existing inference API to the OpenAI-compatible request and stream shapes consumed by `ChatService`, rather than scattering Ubuntu Zombie-specific conditions through UI components. Document supported fields and deliberately omit unsupported controls from settings.

### Authentication and secrets

- Reuse Ubuntu Zombie's authenticated browser session for gateway requests.
- Keep provider API keys, MCP credentials, and privileged headers on the server.
- Remove or hide the API key and arbitrary API base URL controls in managed mode.
- Add CSRF protection where the existing authentication model requires it.
- Restrict allowed upstream origins and validate any user-supplied endpoint or MCP URL to prevent SSRF.
- Define CSP `connect-src`, media, worker, and blob/data URL requirements for streaming and attachments.

### Capability model

Create one capability object that controls visibility and behavior for:

- Models and model selection.
- Vision, audio, PDF, and text attachments.
- Image generation.
- Reasoning output.
- Tool calling and maximum iterations.
- MCP server management.
- llama.cpp slot/server diagnostics.
- Generation parameters and custom JSON.

This replaces UI inference from provider-specific endpoints and prevents users selecting unsupported features.

## Phase 2: Extract the Portable Chat Feature

Move the reusable chat implementation as a cohesive feature, retaining:

- Chat screen, message renderers, composer, attachment previews, branching controls, and conversation sidebar.
- Chat, conversation, agentic, model, MCP, and settings stores needed by enabled capabilities.
- Chat, persistence, attachment, markdown, and tool services.
- Shared UI primitives, icons, styles, constants, enums, and type declarations referenced by the feature.
- Existing client and unit tests for all moved modules.

Separate host-specific behavior behind adapters:

- `InferenceAdapter`: chat, model, capability, image, and cancellation operations.
- `PersistenceAdapter`: local browser storage initially, with a future server-backed option.
- `HostAdapter`: navigation, authentication failure handling, notifications, theme, and product links.
- `ToolAdapter`: built-in tools and optional trusted MCP execution.

Keep route components thin. They should initialize the adapters, load a conversation, and render the feature rather than owning chat logic.

## Phase 3: Integrate with Ubuntu Zombie

### Application shell and routing

- Replace the primitive chat route while keeping its stable public URL when practical.
- Map new-chat and conversation routes into Ubuntu Zombie's router.
- Integrate the chat sidebar with Ubuntu Zombie's navigation, especially on small screens.
- Replace hard-coded hash-route navigation and llama.cpp route assumptions with host navigation.
- Preserve deep links, refresh behavior, browser back/forward navigation, and any existing query-based prompts.
- Add a compatibility redirect from retired primitive-chat URLs if route shapes change.

### Styling and branding

- Scope imported styles to avoid Tailwind reset and component-class collisions.
- Map background, foreground, border, accent, destructive, radius, and typography tokens to Ubuntu Zombie's theme.
- Replace llama.cpp-specific names, titles, icons, server labels, and links.
- Preserve responsive layout, keyboard navigation, focus management, reduced motion, contrast, and screen-reader labels.
- Validate desktop, narrow mobile, touch, and virtual-keyboard behavior.

### Runtime configuration

- Supply managed defaults from Ubuntu Zombie rather than the WebUI's current default external DeepInfra API endpoint.
- Disable arbitrary endpoint, API key, custom JSON, and MCP configuration unless they are intentional administrator/user features.
- Replace `/props` and slot polling with Ubuntu Zombie capability/health data, or remove those displays.
- Ensure all API paths respect Ubuntu Zombie's deployment base path.
- Remove the llama.cpp-specific gzip-copy build plugin from the integrated build unless Ubuntu Zombie serves that artifact format.

## Phase 4: Data and Conversation Migration

Choose and document one storage mode:

### Browser-local mode

Retain Dexie and the existing message-tree schema. Use a Ubuntu Zombie-specific database name and schema version so it cannot collide with another deployment. Treat history as device-local and provide import/export and a visible explanation that it does not synchronize.

### Server-backed mode

Implement the persistence adapter against Ubuntu Zombie's authenticated conversation API. Preserve:

- Conversation metadata and model.
- Parent/child message relationships and active branch.
- Attachments or stable attachment references.
- Reasoning, tool calls, timings, and timestamps.
- Atomic branch creation and cascading deletion.

Do not store large base64 attachments in both IndexedDB and the backend. Define upload limits, accepted MIME types, retention, deletion, malware handling, and authorization for attachment retrieval.

### Hybrid mode

Keep transient drafts and selected preferences in the browser while storing conversations and attachment references in Ubuntu Zombie. Define the source of truth, synchronization behavior, offline limits, conflict handling, and logout cleanup so local data cannot diverge silently from server history.

### Existing primitive-chat history

- Inventory its schema and identify which fields map losslessly.
- Create a versioned, idempotent migration when history is retained.
- Preserve original IDs where safe, or maintain an ID mapping.
- Convert linear conversations into a single message branch.
- Record unsupported data rather than silently dropping it.
- Back up or leave the old data untouched until migration acceptance.
- If migration is not feasible or desired, communicate the cutoff and provide an export path.

## Phase 5: Feature Rollout

Introduce capabilities in controlled slices:

1. Text chat, streaming, stop generation, errors, and regeneration.
2. Conversation creation, rename, deletion, persistence, and branching.
3. Markdown/code rendering and copy actions.
4. Model selection and supported generation parameters.
5. Attachments, with each modality enabled only after backend validation.
6. Reasoning display and agentic tool-call rendering.
7. Built-in tools and trusted MCP tools.
8. Search mode and image generation, only if they are Ubuntu Zombie requirements.

Keep unavailable controls hidden rather than disabled without explanation. Add a server-side feature flag that can switch users back to the primitive chat during rollout.

## Phase 6: Testing and Validation

### Automated coverage

- Unit-test request normalization, stream parsing, reasoning extraction, tool-call assembly, retries, aborts, and capability gating.
- Unit-test migration of empty, long, malformed, and branched histories.
- Component-test composer submission, stop behavior, attachment validation, branch controls, settings visibility, and errors.
- Integration-test the Ubuntu Zombie gateway for authentication, OpenAI translation, disconnects, provider errors, timeouts, and cancellation.
- End-to-end test new chat, reload, history, rename/delete, regeneration, branching, mobile layout, keyboard-only use, and feature-flag rollback.
- Add contract fixtures for every supported provider response variant.

### Security and privacy validation

- Verify no provider or MCP secret appears in HTML, JavaScript bundles, localStorage, IndexedDB, logs, or browser network requests.
- Validate authorization for every conversation and attachment operation.
- Test prompt/file size limits, MIME spoofing, malicious filenames, unsafe links, rendered HTML, and markdown/code sanitization.
- Verify tool arguments and results are treated as untrusted data.
- Confirm external endpoints and MCP connections cannot bypass allowlists.
- Review telemetry and crash reports for message, prompt, attachment, and credential leakage.

### Repository checks

Run the package-manager equivalents of the WebUI's existing checks after integration:

- Formatting and linting.
- Svelte/TypeScript checks.
- Production build.
- Unit, client/component, and server tests.
- Playwright end-to-end tests.

Also run Ubuntu Zombie's existing full validation suite and inspect the production artifact for route, asset-path, and bundle-size regressions.

## Phase 7: Deployment and Cleanup

- Deploy behind a disabled-by-default feature flag.
- Enable for maintainers, then a small user cohort, while monitoring request failures, stream completion, latency, cancellation, storage errors, and frontend exceptions.
- Verify rollback does not corrupt new or migrated conversations.
- Expand rollout only after accessibility, security, and migration acceptance.
- Remove the primitive UI, obsolete API code, stale dependencies, old routes, and migration-only compatibility code after the rollback window.
- Update Ubuntu Zombie's user documentation, deployment configuration, privacy statement, and contributor setup.
- Define how future standalone WebUI updates are imported and tested to avoid an unmaintainable fork.

## Acceptance Criteria

- The Ubuntu Zombie chat route uses the new interface with no primitive chat remaining after rollout.
- Authenticated users can stream, stop, retry, reload, and manage conversations without exposing provider credentials.
- Supported features match the server capability contract; unsupported features are absent.
- Conversation and attachment access is isolated per user in server-backed mode.
- Existing history is either migrated correctly or handled according to an approved cutoff/export policy.
- Desktop and mobile flows meet accessibility and responsive-layout requirements.
- Refreshes and deep links work under the production base path.
- Build, type, lint, unit, integration, and end-to-end checks pass.
- Security review finds no new secret exposure, cross-user access, unsafe endpoint access, or rendering vulnerability.
- The feature flag provides a tested rollback until the old implementation is retired.

## Clarifications Required Before Implementation

1. Which Phase 4 storage mode should the replacement use: browser-local, server-backed, or hybrid? Should Ubuntu Zombie's existing backend, authentication, and history remain authoritative?
2. What frontend framework and build system does the current Ubuntu Zombie version use, and which exact route/component is the primitive chat?
3. Is Ubuntu Zombie's model endpoint already OpenAI-compatible, including streaming and tool calls?
4. Which features are in scope for the first release: text only, attachments, branching, reasoning, MCP tools, search mode, image generation, and advanced model parameters?
5. May users configure arbitrary providers and MCP servers, or must all traffic pass through administrator-managed Ubuntu Zombie services?
6. Must existing primitive-chat conversations be migrated?
7. Does Ubuntu Zombie need its current chat URL and visual shell preserved?
8. Is direct source integration preferred, or is a separately built/mounted chat application acceptable?

These answers are blocking inputs to the final architecture. Until they are available, the recommended default is a directly linked or mounted chat route using a same-origin authenticated gateway, managed capabilities, no browser-visible provider secrets, and browser-local history only as an explicitly accepted interim state.
