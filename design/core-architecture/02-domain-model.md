# Phase 2: Domain Model (F# Types)

## 1. Primitives (Branded Types)

```fsharp
// Time & Duration
type Milliseconds = int
type Duration = Milliseconds

// Connection Identifiers (Branded Strings)
type SshUrl = string        // "ssh://user@host:22"
type LocalUrl = string      // "local://"
type DockerUrl = string     // "docker://container-id"
type KubectlUrl = string    // "k8s://pod-name"

type ConnectionUrl = 
    | Ssh of SshUrl
    | Local of LocalUrl
    | Docker of DockerUrl
    | Kubectl of KubectlUrl

// Command Components
type BinaryPath = string    // "/usr/bin/git"
type Argument = string      // "--version"
type ShellFragment = string // "git --version | grep 2.0" (Legacy/Unsafe)

type CommandInput = 
    | Exec of BinaryPath * Argument list // Safer, no shell expansion
    | Shell of ShellFragment             // Raw shell string

// Outputs
type Stdout = string
type Stderr = string
type ExitCode = int // Constrained: 0 is distinct from non-zero in logic
```

## 2. Compounds (Value Objects)

The **Happy Path** (Exit Code 0).
```fsharp
type CommandSuccess = {
    Stdout: Stdout
    Stderr: Stderr
    Duration: Duration
}
```

## 3. Errors (Discriminated Union)

We distinguish between "The command ran and failed" vs "The infrastructure failed".

```fsharp
type CommandError = 
    // 1. The command ran, but returned non-zero (Domain Logic Failure)
    | ExitCodeFailure of code: ExitCode * stdout: Stdout * stderr: Stderr * duration: Duration
    
    // 2. The command could not be run (Infrastructure Failure)
    | ConnectionLost of reason: string
    | TimedOut of duration: Duration
    | NotFound of binary: BinaryPath
    | PermissionDenied of reason: string
```

## 4. The Host Interface (Capabilities)

Notice: `CommandError` is in the *Error* channel (Left side of Result).

```fsharp
type Host = {
    // Effect<Success, Error>
    Run: CommandInput -> Async<Result<CommandSuccess, CommandError>>
    
    // Metadata
    Url: ConnectionUrl
    SystemInfo: SystemInfo
}
```

## 5. Workflows (Examples)

**Scenario 1: Standard Install (Happy Path)**
If `apt-get` fails (Exit 100), the Effect short-circuits. The user doesn't need to check the code.
```fsharp
let installNginx (host: Host) =
    host.Run (Shell "apt-get install -y nginx")
    // Returns Success or short-circuits with ExitCodeFailure
```

**Scenario 2: Asserting Failure (Negative Test)**
We expect failure, so we flip the error to value.
```fsharp
let ensureRootLocked (host: Host) =
    host.Run (Shell "ssh root@localhost")
    |> Effect.flip // Expect Error
    |> Effect.map (function
        | ExitCodeFailure(code, _, _, _) when code = 255 -> "Passed"
        | _ -> "Failed: Expected SSH access denied"
    )
```
