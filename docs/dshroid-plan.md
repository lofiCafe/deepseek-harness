---
description: "Proposed Dshroid development plan covering capability reuse, remote access, task semantics, milestone acceptance, and the GLM implementation handoff."
---

# Dshroid development plan

English | [中文](dshroid-plan.zh.md)

## Summary

Dshroid lets an Android tablet start, monitor, and supplement AI tasks on a Mac. The Mac owns execution processes, model credentials, and Session logs; both clients display the same task data. The first version delivers the task-control workflow, an Android APK, and a Mac management app; durable code diffs follow in a later version.

This document specifies a proposed implementation, not shipped functionality. Research is dated 2026-10-01, with upstream tag `dsh-v0.2.0-rc.2` and repository version `0.2.0-rc.2` as the code baseline. Development uses the fork's `dshroid` branch; keep this baseline fixed during implementation and validate upstream upgrades separately.

## Table of Contents

- [Goals and scope](#scope)
- [Existing capabilities and plan corrections](#findings)
- [Community code reuse](#community)
- [Technical design](#architecture)
- [Task and prompt semantics](#semantics)
- [Remote access and device management](#remote)
- [Milestones and estimates](#phases)
- [Acceptance and review evidence](#acceptance)
- [GLM implementation handoff](#handoff)
- [Dev Note](#dev-note)

-----

<a id="scope"></a>
## Goals and scope

The tablet user selects a project registered on the Mac, submits a goal and constraints, follows actual execution, supplements instructions, answers questions or approves operations, and sees why execution ended. Tablet disconnection and backgrounding must not cancel Mac tasks.

| First-version delivery | Explicitly deferred |
| --- | --- |
| One user and one Mac; at most one executing Dshroid task per project | Multiple hosts, team permissions, scheduled tasks, plugin marketplace |
| Project selection, task list, details, and creation | Tablet code editing, a complete IDE, remote desktop |
| Mid-task instructions, follow-up messages, stop, questions, and approval | Automatic Git merging, commits, or pushes |
| HTTPS access inside a private network and revocable device pairing | Anonymous public access, author-operated relays, remote wake |
| Mac background service, management window, and Android APK | System push, app-store distribution, and a complete installer |
| Result summary and recorded tool and test results | Durable diffs available after restart |

The first version permits authorized AI code changes but does not invent a read-only analysis mode: any future mode must enforce tool capabilities and execution permissions rather than rely on instructions alone. Projects are registered locally on the Mac; the tablet cannot supply arbitrary host paths or change model keys.

Acceptance uses the designated real Mac and Android tablet. Milestone 0 records actual macOS, Android, WebView, browser, Node, and Tailscale versions; set the Android minimum version after device and build-tool validation. Initial artifacts support personal installation; production signing, notarization, and store distribution require separate estimates.

-----

<a id="findings"></a>
## Existing capabilities and plan corrections

Reuse official mechanisms before adding the small product layer. The following findings come from static inspection of this checkout's source and READMEs; real model, network, and device behavior still requires milestone 0 validation.

| Capability | Existing evidence | Project treatment |
| --- | --- | --- |
| Creation, history, prompts, queues, cancellation | [Session Controller](../packages/api/session-controller/README.md) | Reuse controllers and generated Remote types |
| Request deduplication | `requestId` / `rpcId` in the [prompt implementation](../packages/api/session-controller/src/commands.ts) | Preserve identity across retries of one submission |
| Live connection | `/api/remote.mux` in [Gateway](../packages/api/gateway/README.md) | Use existing WebSocket multiplexing rather than another SSE protocol |
| Reconnect catch-up | Gateway journal/snapshot streams and Session Client | Reuse sequence, pagination, and connection-generation handling |
| Projects and navigation | [Workspace Controller](../packages/api/workspace-controller/README.md) | Register projects on the Mac; the tablet reads an allowlist |
| Questions and approval | [User Questions](../packages/interaction/user-questions/README.md), [User Approval](../packages/interaction/user-approval/README.md) | Reuse request, answer, and withdrawal semantics |
| Session and process lifecycle | [Agent](../packages/core/agent/README.md), [Agent Loop](../packages/core/agent-loop/README.md) | Host owns execution; distinguish Client references from Host ownership |
| Current-turn diffs | [Workspace Changes](../packages/deliverables/workspace-changes/README.md), [UI Deliverables](../packages/client/ui-deliverables/README.md) | Reuse presentation components without promising history across restart |
| SDK | [TypeScript SDK limitations](../packages/sdk/client/README.md#known-limitations-and-deferred-work) | Do not carry tablet control over the SDK subprocess protocol in v1 |

The draft's direction is reasonable, but remove the separate AI execution engine, second conversation event stream, and complete scheduling database. In the MVP, a task is a Dshroid-managed root Session; turns are execution records and prompts are Inbox messages. Persist only product metadata that existing Sessions cannot reconstruct.

The draft's claim that the next model call adopts an instruction must distinguish queue acceptance from request inclusion; success must be separate from goal verification; cancellation must be separate from clearing future work; Mac persistence requires an independent service lifecycle. The original 100–142 hours does not adequately cover custom authentication, two native apps, and recovery.

Official [Connection](../packages/client/connection/README.md) exchanges a launch token for an authority-bound cookie and checks Host and Origin. Its current cookie lacks `Secure`, and it has no per-device logout; a Host trust entry is not authentication. Changing the listening address or broadening trusted hosts does not complete remote access.

-----

<a id="community"></a>
## Community code reuse

Adoption requires an appropriate license, current interfaces, suitable permissions, and a working device connection. These are static findings dated 2026-10-01; no candidate plugin was installed or tested. A version declaration is not compatibility evidence.

| Candidate | Inspected facts | Decision |
| --- | --- | --- |
| [dsh-web](https://github.com/zhu1090093659/dsh-web/tree/dev/packages/dsh-remote-web-ui) | remote-web-ui `0.4.4`, DSH `>=0.2.0-rc.2`, Apache-2.0; source includes mux forwarding, pairing, and revocation | Prioritize pairing and recovery references; do not install the whole package by default |
| [dsh-mobile-gateway](https://github.com/agent-mobile/dsh-mobile-gateway) | `0.4.0`, MIT; allowlist and WebSocket support, but its mux adapter uses an older `wireStream.open` invocation incompatible with this baseline | Reference route restrictions and authentication tests; migrate before adopting |
| [Smart-DSH](https://github.com/hikarioyama/Smart-DSH) | `dsh-notify-push` declares MIT; no uniform root license was identified; Android Chrome Web Push target | Later notification reference; verify file-specific permission before copying |
| [dsh-telegram](https://github.com/Kevin66Z0/dsh-telegram) | MIT; remote messaging channel with different product interactions | Exclude from MVP dependencies |
| [dsh-lark](https://github.com/sugarforever/dsh-lark) | MIT; Lark messaging bridge | Reference for later notification channels |

The dsh-web pairing channel grants broad Host control and includes optional author-operated relay and telemetry code. Dshroid needs task-scoped access, so pairing usability alone cannot establish acceptable authorization or justify adopting those network services. Earlier descriptions of these plugins as SSE-only or uniformly obsolete are inaccurate; review HTTP, mux, answer replies, and reconnection individually.

For each adopted component, milestone 0 records its version, license file, official entry points used, necessary changes, and validation. Resolve dependencies precisely through the lockfile. Source imports retain copyright, license, applicable NOTICE, origin files, and local modifications. Prefer maintained dependencies over copying entire community repositories and old protocol implementations.

-----

<a id="architecture"></a>
## Technical design

The React task interface reuses official Client connection, Session, and projection capabilities. One Mac Node Host runs models, tools, and persistence; native apps manage the service or display the interface. Every supported Node application continues to launch through [dsh profiles](architecture.md#application-launch).

| Component | Proposed responsibility | Excluded responsibility |
| --- | --- | --- |
| Dshroid profile / bundle | Mount required plugins over official Web/base composition, fix dependencies, disable unnecessary features | Another CLI bin or launcher bypass |
| Dshroid Host plugin | Restricted task control, device authorization, stop intent, minimal metadata | Duplicated Agent Loop, Session log, or queue |
| Dshroid Client plugin | Task list, details, submission receipts, and human-action state | Inferring execution outcomes from frontend timers |
| macOS `launchd` LaunchAgent | Own one Host during the login session; manage start, exit, and crash restart | Execution during logout, shutdown, or sleep |
| macOS management app | SwiftUI menu bar, service state, task-page opening, pairing management; WKWebView if embedding is needed | Starting a second Host |
| Android app | Kotlin WebView displaying the same-origin HTTPS task page; back navigation, foregrounding, and connection errors | Running Node or model tools on the tablet |
| Tailscale Serve | Private-tailnet HTTPS access and proxying to loopback | Replacing Dshroid device authorization |

Default to thin native platform shells rather than maintaining Electron and Tauri runtimes to display the same Web page. Official Electron remains a separate product that owns its Host; it cannot be reused unchanged alongside a LaunchAgent. If milestone 0 demonstrates that a small Electron change can attach to an external Host, replace the proposed Mac shell with documented validation evidence.

Android WebView loads only the paired HTTPS origin, opens external links in the system browser, exposes no arbitrary JavaScript native bridge, respects TLS errors, and loads neither local files nor mixed HTTP content. Use [official Android WebView interfaces](https://developer.android.com/develop/ui/views/layout/webapps/webview); Mac embedding uses [WKWebView](https://developer.apple.com/documentation/webkit/wkwebview). Maintain one React business interface.

Use Node within the repository's supported range. The Mac service uses explicit packaged Node and dsh paths rather than a LaunchAgent shell PATH; source development uses the existing source launcher. A dedicated Dshroid Harness home isolates existing official configuration and Sessions. Configure keys locally on the Mac rather than copying original user data.

Proposed source placement follows existing groups: Host/API functionality in a new feature package under `packages/api/`, UI under `packages/client/`, composition under `packages/bundle/`, and native shells in separate `apps/dshroid-*` directories. Milestone 0 determines the minimum package count, exact names, and profile installation procedure; these paths are not existing interfaces.

Register new packages through the [package cookbook](cookbook/adding-a-package.md) with explicit Host and Client compiler configurations. Validated Config fields own deployment ports, public origin, pairing expiry, device limits, request limits, and optional sleep inhibition. Tools and approvals use official sandbox and permission policies.

-----

<a id="semantics"></a>
## Task and prompt semantics

Represent execution, connectivity, and result verification separately so disconnection, questions, and turn completion do not mislead the user. Reconstruct state from Host observations and durable logs rather than maintaining an independent frontend state-machine database.

### Data and identity

Use existing branded `SessionId`, `WorkspaceId`, `MessageId`, and `SessionRequestId`. Minimal product metadata includes Dshroid Session ownership, creation request identity, project association, stop intent, and optional Git baseline facts. Existing [Storage](../packages/storage/README.md) capabilities hold non-Session data, and [Credentials](../packages/credentials/README.md) capabilities manage device credentials. Official sources remain authoritative for logs, prompt queues, and result text.

Creation first records intent under a stable request identity and binds one stable Session identity, then creates or resumes that Session and submits its initial prompt. A crash at any step repairs only missing registration or admission; an accepted prompt is not resent under a new identity. Reusing an identity with different content returns a conflict rather than another task. A lost response displays that submission confirmation is pending and does not mint a new request.

Serialize Dshroid execution admission per project; the MVP has no unbounded automatic scheduling queue. Reject another task's execution while the project is busy and retain its draft; existing task inputs still use the official Inbox. Reject new instructions during stopping; only explicit continuation may wake a stopped task.

### Prompts and stopping

| Operation | Official semantics or proposed implementation | Visible receipt |
| --- | --- | --- |
| Start / follow-up | `prompt(mode: 'queue')` → `followup`, targeting `next-turn` | Accepted, queued, entered execution |
| Supplement current task | `prompt(mode: 'steer')` → `steer`, targeting `next-step`; wakes idle | Accepted, waiting for next step, included in request |
| Edit / remove pending message | Official queue mutation; reject already-claimed messages | Success or already executing |
| Cancel current turn | Official `session.cancel`, retaining Inbox | Warn that queued messages can continue; not the default stop button |
| Stop task | Dshroid Host serially records stop intent, calls default Inbox-clearing `agent.cancel`, stops owned activity, and awaits settlement | Stopping; stopped only after actual settlement |

In the current [runtime implementation](../packages/core/agent-loop/src/agent.ts), `steer` only calls `send(input, 'next-step', true)`; it does not actively abort a model request or tool. It can miss a request that already claimed inputs, or enter a later turn after cancellation. The Client comment saying “interrupts” cannot guarantee forced interruption; milestone 0 must establish integration evidence for actual behavior.

Show “included in request” only after the recorded message enters actual model-request inputs, not from HTTP success, Inbox claim, or new Assistant output alone. A sent request does not establish that the model obeyed the constraint; stopping that clears a message marks it removed. If current projections lack this evidence, add a projection derived from existing events rather than a receipt that assumes success.

Stopping does not undo file writes or external operations already performed. Stop intent must prevent late Dshroid prompt retries from waking the task; use owner APIs to cancel and await its subagents and background jobs without killing other Sessions' processes. The MVP profile omits automatic scheduling and automatic goal recovery. Unstoppable operations remain visibly blocked with activity details; a timeout must not fabricate stopped status.

### State and recovery

| Dimension | Values and evidence |
| --- | --- |
| Connection | Connecting, online, offline; offline includes the last observation time and labels task state as last observed |
| Execution | Queued, running, awaiting approval, needs reply, stopping, stopped, ended, failed, blocked, interrupted |
| End reason | Project `turn/end.reason`; `completed` means execution ended, while `error`, `aborted`, and `interrupted` remain distinct; token limits and unknown extensions cannot imply completion |
| Goal verification | Unverified, recorded test evidence, unresolved issues; no automatic success score in v1 |

An active approval wait displays awaiting approval; a continued question while execution progresses displays needs reply rather than pretending execution is blocked. Show actual tool operations and logs instead of inventing locate→edit→test stages or percentages from message wording. Distinguish model-reported testing from recorded commands and exit status.

Reconnect reads a replacement control baseline and catches up the durable journal; streamed tokens are transient presentation rather than the only history. Reuse official subscriptions, paging, duplicate handling, and stale-generation rejection. Locking the tablet may close the connection; foregrounding reconnects.

After Host failure or restart, retain history and use official handling of unclosed turns as `interrupted`; ask the user whether to continue. Service restart must not automatically repeat shell operations, external writes, or old approvals. Cold reads and repeated creation confirmation must not trigger execution; recovered Inbox admission remains controlled by stop intent and explicit continuation.

### Code changes and later diffs

The first version permits only one Dshroid write task per project, but this does not isolate edits from the user or other Mac tools. Record Git HEAD, working directory, and initial dirty status as baseline facts only; do not claim a HEAD diff precisely attributes task changes. Non-Git projects explicitly have no Git baseline.

Deliver diffs after adding task-owned worktrees and durable content snapshots. For dirty source directories, explicitly choose a committed baseline or explicitly preserve initial uncommitted content; never silently reset, stash, or copy it. Cover tracked/untracked files, deletion, renames, binary and oversized files, and restart recovery. A commit or path alone cannot reconstruct both changed sides. A worktree isolates code, not process permissions.

-----

<a id="remote"></a>
## Remote access and device management

Default to [Tailscale Serve](https://tailscale.com/docs/features/tailscale-serve) providing HTTPS inside a private tailnet, with the Mac Node service listening on loopback. Both devices install Tailscale and join the same restricted tailnet. Milestone 0 validates port proxying, mux upgrade, and persistent connections with the selected Mac distribution. Public Funnel, automatic tunnels, and third-party stable domains are outside the default design.

The same-origin Dshroid task entry authenticates a device before checking operations and project ownership. Official owner cookies and paired-device credentials are not interchangeable. Local Mac management requires official local authentication; a loopback source IP alone must not authorize requests forwarded by Tailscale.

The Mac creates a short-lived, single-use random pairing credential, accepted by QR or pasted link; local Mac confirmation exchanges it for a revocable device session. Camera scanning is optional in the MVP. HTTPS cookies use `HttpOnly`, `Secure`, and appropriate SameSite / Path settings. Keep long-lived credentials out of URLs, localStorage, and logs.

Check the configured public origin, HTTP Host, browser Origin, WebSocket upgrade Origin, and request sizes exactly. Trust proxy headers only from the designated loopback proxy and validate deployment settings. Unauthenticated page bootstrap contains no workspaces, paths, configuration, or secrets. Prefer one origin for UI, HTTP, and WebSocket to avoid cross-origin and WebView third-party cookies.

| Allowed remote operations | Operations rejected by Host |
| --- | --- |
| Read allowed projects and Dshroid root Sessions, history, and status | Arbitrary directories and unowned Sessions or projects |
| Create tasks, send instructions, edit queues, stop, explicitly continue | Plugin installation, configuration changes, secrets, updates, power control |
| Answer valid task questions and specific approval requests | Arbitrary shell/terminal RPC, unrestricted downloads, forged approval identities |
| Read necessary authorized task results | Unrestricted `$events`, arbitrary scoped receivers, and event replies |

Tasks may still execute shell tools under their permissions; the restriction prevents direct tablet shell API calls that bypass task and approval policies. Filter HTTP unary requests, every logical mux open, uplinks, and answer replies rather than URL paths alone. Authorize durable journals by Session; live events must not broadcast other Sessions or global configuration.

Reuse official Gateway protocol, generated types, and public dispatch capabilities, adding minimal device admission and filtering. Do not copy the protocol server or access private members. Current official Connection treats calls as operator activity; per-device and per-task authorization is new work. Milestone 0 determines whether public extension points suffice; otherwise implement the smallest typed admission extension with tests, never an unconditional `/api` proxy.

Revocation rejects new requests, closes established device mux connections, removes human-response claims, and clears corresponding cookie/native pairing records; it does not cancel Mac tasks. Approval binds request, Session, exact command or file, working directory, and expiry; simultaneous clients can produce only one decision. Stopping, tool cancellation, and Host restart invalidate approval; old answers are not automatically replayed.

-----

<a id="phases"></a>
## Milestones and estimates

Advance through independently reviewable milestones. Estimates cover implementation and validation rather than fixed GLM completion commitments; revise after milestone 0 using measured evidence. Real model costs and the user's device acceptance time are additional.

| Milestone | Work and delivery | Release condition | Estimate |
| --- | --- | --- | --- |
| 0: capability validation | Pin official baseline; real Mac→Android browser experiment; verify prompt, steer, cancel, questions, approval, reconnection, and candidates | Actual connection evidence, license list, profile draft, remote admission design, and identified reuse gaps | 12–18 hours |
| 1: task control | Session ownership, creation deduplication, project admission, prompt receipts, stop intent, recovery | No duplicate execution on retry; stop clears owned work; crash does not redo operations | 24–36 hours |
| 2: restricted access | Tailscale HTTPS, pairing, revocation, HTTP/mux authorization, question and approval bridge | Unauthorized reads/writes denied; connected revoked device loses access; cross-network workflow works | 28–42 hours |
| 3: tablet interface | List, details, creation; reply indicators, drafts, receipts, stopping, error recovery; official projections and primitives | Orientation, Chinese input, keyboard, rotation, lock-screen recovery; truthful state | 24–36 hours |
| 4: native apps | LaunchAgent, Mac menu-bar management, fixed Node/runtime paths; Android WebView APK and lifecycle | Closing windows or management shell retains Host; explicit service quit; device-installable APK | 24–36 hours |
| 5: stability and handoff | End-to-end fault cases, relevant snapshots, recordings, operation/uninstall instructions, review materials | Acceptance below passes; no hidden privilege widening or destructive data handling | 20–28 hours |
| Later version | Task-owned worktrees, durable diffs, tablet review; notification as a separate item | Verifiable content after restart; existing source edits remain distinct | 28–44 hours |

The MVP totals 132–196 hours, or roughly 158–235 hours with 20% integration allowance. At 15 hours per week, allow about 11–16 weeks. Deliver the first browser workflow during milestones 0–3 rather than waiting for APK and Mac packaging to validate the core experience.

Milestone 0 uses real model checks only with existing user API configuration; otherwise mark them unverified and provide executable manual steps. Cover control protocols with keyless fixtures first, then add real network and lifecycle evidence. No public deployment before remote authorization validation.

Commit independently reviewable changes per milestone and complete dependencies before proceeding. Preserve UI mock, keyless integration, and device evidence rather than substituting static checks for user workflows. Do not invent frontend APIs before backend delivery; identify generated Remote entry points in milestones 0/1 before connecting the UI.

-----

<a id="acceptance"></a>
## Acceptance and review evidence

Exercise these scenarios through official entry points or the actual Dshroid profile so direct plugin mounting does not hide launch, permission, or network failures. Record each command, environment, and outcome; retain failures and unverified cases.

| Scenario | Required observation |
| --- | --- |
| Lost creation response, repeated submission, Host exits during creation | One Session and initial prompt per logical request; identity conflict explicitly rejected |
| Instruction during model request, tool execution, and idle | Actual execution semantics retained; nearest admissible request receives instruction; no false adopted receipt |
| Stop with pending messages | Cancel actual activity and clear owned Inbox; late retries do not wake it; other tasks unaffected |
| Live subagent, long command, or job | Stopped only after owner APIs confirm settlement; uncancellable activity remains visible |
| Tablet network change, lock, background, process death, rotation | Mac continues; recovered state and durable log have no duplicates or gaps; drafts follow documented policy |
| Mac window close, management shell quit, Host crash, sleep | First two retain service; latter cases show offline/interrupted; recovery does not replay side effects |
| Missing credentials, revoked device, expired pairing, wrong Origin | HTTP and mux deny; connected device loses access; logs and screenshots contain no credentials |
| Unowned Session, forged path, direct global API or `$events` | Host denies without leaking other projects |
| Simultaneous answers/approvals or stale answer after stop | Only a valid decision accepted; expired identity has no effect |
| Turn end, model error, token limit, testing not run | Actual reason displayed; neither `idle` nor `completed` implies goal success |

Non-trivial model/product-visible changes update keyless recorded-session snapshots under [snapshot ownership](../snapshots/AGENTS.md). Geometry and ARIA expectations remain owner-local; UI changes include browser GIF evidence. Device records cover Android WebView and Mac lifecycle; screenshots do not substitute for reconnect and stop behavior.

Select relevant commands per milestone using [pre-push checks](../.agents/skills/dsh-pre-push-checks/SKILL.md): focused behavior tests, owning output snapshots, typecheck for Remote changes, build/hygiene/built smokes for package and launch changes, and doc-sync for documentation. CI owns exhaustive coverage and platform matrices; do not reflexively repeat the whole suite locally.

Follow root [AGENTS.md](../AGENTS.md), [architecture](architecture.md), [lifecycle rules](defensive-patterns.md), and [testing policy](testing.md). New model inputs must reconstruct from Session logs. Declared persistence-type changes require [type acknowledgement](cookbook/reviewing-persistence-type-changes.md); never alter released Session generations. Record perceptible breaking changes immediately in upgrade guides; retain existing sandbox and approval mechanisms.

Rotate and redact Mac logs. Sleep inhibition is opt-in and holds an assertion only during active tasks; release it on completion or plugin disposal without promising closed-lid execution or remote wake. Service stop and uninstall display active tasks, remove service registration and device access, and retain Session data until the user separately chooses deletion.

Review materials include milestone changes, actual commands and results, snapshots/GIFs, device records, dependency/license origins, known limitations, and install/upgrade/stop/uninstall instructions. Signing keys, API keys, pairing credentials, and complete personal logs never enter commits.

-----

<a id="handoff"></a>
## GLM implementation handoff

Give GLM the following instruction and this document. The plan owns product and acceptance requirements; existing types, generated descriptors, and source own actual interfaces. Resolve implementation conflicts in the specification before submitting behavior and validation evidence.

> Develop Dshroid on the fork's `dshroid` branch. Read `AGENTS.md`, `docs/architecture.md`, and `docs/dshroid-plan.zh.md` first. Complete milestone 0 and commit its evidence before milestones 1–5. Reuse official Sessions, Inbox, Gateway, projections, Workspaces, questions, and approval; do not create another Agent Loop, scheduling engine, or event protocol. Launch all Node services through dsh profiles. Real remote devices require restricted admission; neither a whole `/api` proxy nor hidden settings buttons suffices. Commit minimal reviewable changes per milestone, run scope-appropriate checks, record actual results and unverified cases, and deliver snapshots/GIFs for UI changes. Do not automatically commit or push user-project code or merge the development branch. Update both versions of this plan when implementation differs. Deliver commits, both installable apps, operation docs, and review evidence for Codex review.

Milestone 0 evidence includes actual versions, real profile composition, API/event mapping, acceptance and request-inclusion evidence, stop/queue behavior, authorization entry design, adopted component licenses and reuse decisions, and exact milestone 1 source locations. Without network or API credentials, complete keyless work and identify device gaps rather than claiming full milestone 0 acceptance.

Subsequent Codex review uses this plan's observable results, prioritizing duplicate execution, stop settlement, device authorization, recovery, log reconstruction, and artifact launch paths. Repair and revalidate blocking findings before releasing for personal use.

-----

<a id="dev-note"></a>
## Dev Note

This work only statically verified source/docs and authored the plan. It did not implement Dshroid, install community plugins, run real models, or test an Android device. Estimates, native shell choices, and exact remote admission extension points remain milestone 0 hypotheses. Proposed behavior and milestone acceptance are development requirements rather than claims about the current product.
