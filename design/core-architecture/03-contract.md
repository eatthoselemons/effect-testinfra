# Phase 3: The Contract (TypeScript/Effect) - REVISED v2

## 1. Domain Models (Pure Data)
**Location:** `src/domain/models/command.ts`
**Rule:** Schema.Struct only. No Methods.

```typescript
import { Schema } from "@effect/schema";
import { Brand } from "effect";

// ... Primitives (ExitCode, Stdout, etc.) ...

// ... ConnectionUrl Types ...

// --- The Host Entity (State) ---
export const Host = Schema.Struct({
    url: ConnectionUrl,
    // systemInfo can be added later
});
export type Host = Schema.Schema.Type<typeof Host>;

// --- Command Input ---
export const ExecCommand = Schema.Struct({
    _tag: Schema.Literal("ExecCommand"),
    binary: Schema.String,
    args: Schema.Array(Schema.String)
});

export const ShellCommand = Schema.Struct({
    _tag: Schema.Literal("ShellCommand"),
    command: Schema.String
});

export const CommandInput = Schema.Union(ExecCommand, ShellCommand);
export type CommandInput = Schema.Schema.Type<typeof CommandInput>;

// --- Command Success ---
export const CommandSuccess = Schema.Struct({
    stdout: Stdout,
    stderr: Stderr,
    duration: Duration
});
export type CommandSuccess = Schema.Schema.Type<typeof CommandSuccess>;
```

## 2. Domain Errors
**Location:** `src/domain/errors.ts`

```typescript
import { Schema } from "@effect/schema";
import type { ConnectionUrl } from "./models/command";

// ... Error Definitions (ExitCodeError, ConnectionError) ...
// Same as before
```

## 3. Service Interface (Stateless Capability)
**Location:** `src/services/HostExecutor.ts`
**Rule:** Service functions take the State (Host) as an argument.

```typescript
import { Effect, Context } from "effect";
import type { Host, CommandInput, CommandSuccess } from "../domain/models/command";
import type { HostError } from "../domain/errors";

export class HostExecutor extends Context.Tag("HostExecutor")<
    HostExecutor,
    {
        /**
         * Execute a command on a specific host.
         * Pure wrt class state (stateless service).
         */
        readonly execute: (
            host: Host, 
            input: CommandInput
        ) => Effect.Effect<CommandSuccess, HostError>
    }
>() {}
```
