# Phase 1: Event Storming - Core Architecture

## Context
The core mechanism of `effect-testinfra`: connecting to a target backend (Host) and executing commands on it. This is the foundation for all other resources (File, Service, etc.).

## Commands (User Intents)
1.  **Connect**: The user (or test runner) wants to establish a session with a target (Local, SSH, Docker, etc.).
2.  **RunCommand**: The user wants to execute a shell statement on the connected target.

## Events (Domain Outcomes)

### Connection Lifecycle
*   **ConnectionEstablished**: The backend is ready to accept commands.
*   **ConnectionFailed**: The backend could not be reached (Auth failure, Network unreachable, Container not found).

### Command Execution
*   **CommandExecuted**: The command ran to completion (even if the exit code was non-zero).
    *   *Payload:* `stdout`, `stderr`, `exitCode`, `duration`, `command`.
*   **ExecutionFailed**: The command could not be invoked or the connection died mid-stream.
    *   *Payload:* `reason` (Timeout, TransportError).

## Domain Questions & Answers

**Q: How do we handle non-zero exit codes?**
A: In the Domain, a non-zero exit code is a **CommandExecuted** event (Success path of execution), not a system failure. The *result* contains the code. The *user* (asserting in a test) decides if that is a failure.

**Q: Stateful vs Stateless?**
A: We are modeling **Stateful Connections**. The `Host` object implies an active session or handle that needs to be maintained (and potentially closed).

**Q: Return Types?**
A: We will not return raw `int` or `string`. We will return a `CommandResult` aggregate containing branded types (`ExitCode`, `Output`, etc.).
