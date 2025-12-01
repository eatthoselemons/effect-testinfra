# Phase 3: The Contract (TypeScript/Effect) - REVISED

## 1. Domain Models (Pure Data)
**Location:** `src/domain/models/command.ts`
**Rule:** Use `Schema.Struct`, Branded Types, and Data-Last Ops.

```typescript
import { Schema } from "@effect/schema";
import { Brand } from "effect";

// --- Primitives ---
export type ExitCode = number & Brand.Brand<"ExitCode">;
export const ExitCode = Schema.Number.pipe(Schema.brand("ExitCode"));

export type Stdout = string & Brand.Brand<"Stdout">;
export const Stdout = Schema.String.pipe(Schema.brand("Stdout"));

export type Stderr = string & Brand.Brand<"Stderr">;
export const Stderr = Schema.String.pipe(Schema.brand("Stderr"));

export type Duration = number & Brand.Brand<"Duration">;
export const Duration = Schema.Number.pipe(Schema.brand("Duration"));

// --- Connection Types ---
export type SshUrl = string & Brand.Brand<"SshUrl">;
export const SshUrl = Schema.String.pipe(
    Schema.pattern(/^ssh:\/\/.+/), 
    Schema.brand("SshUrl")
);

export type LocalUrl = string & Brand.Brand<"LocalUrl">;
export const LocalUrl = Schema.Literal("local://").pipe(Schema.brand("LocalUrl"));

export const ConnectionUrl = Schema.Union(SshUrl, LocalUrl);
export type ConnectionUrl = Schema.Schema.Type<typeof ConnectionUrl>;

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

## 2. Domain Errors (Discriminated Unions)
**Location:** `src/domain/errors.ts`

```typescript
import { Schema } from "@effect/schema";

// Logic Failure (Exit != 0)
export const ExitCodeError = Schema.TaggedStruct("ExitCodeError", {
    code: ExitCode,
    stdout: Stdout,
    stderr: Stderr,
    duration: Duration
});
export type ExitCodeError = Schema.Schema.Type<typeof ExitCodeError>;

// Infrastructure Failures
export const ConnectionError = Schema.TaggedStruct("ConnectionError", {
    reason: Schema.String,
    host: ConnectionUrl
});

export const TimeoutError = Schema.TaggedStruct("TimeoutError", {
    duration: Duration
});

export const HostError = Schema.Union(ExitCodeError, ConnectionError, TimeoutError);
export type HostError = Schema.Schema.Type<typeof HostError>;
```

## 3. Service Interface (The Host)
**Location:** `src/services/Host.ts`
**Rule:** Interface defines the capability (IO). Implementation is injected.

```typescript
import { Effect, Context } from "effect";
import type { CommandInput, CommandSuccess, ConnectionUrl } from "../domain/models/command";
import type { HostError } from "../domain/errors";

export interface Host {
    readonly url: ConnectionUrl;
    
    // The Capability
    readonly run: (input: CommandInput) => Effect.Effect<CommandSuccess, HostError>;
}

export class HostService extends Context.Tag("HostService")<
    HostService,
    Host
>() {}
```
