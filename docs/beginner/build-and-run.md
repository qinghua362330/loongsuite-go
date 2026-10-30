# 3. Build Time and Run Time

**Question:** When does the work done by `otel go build` affect a request?

## Two different moments

You write `.go` text files; the computer runs the executable produced by the build.

```bash
# Plain Go: build, then start
go build -o demo .
./demo

# With otel: let the tool join the build, then start the output
otel go build -o demo .
./demo
```

- **Build time**: source code is checked, compiled and linked into an executable. This is where `otel` works.
- **Run time**: the generated program starts and handles real requests. This is where the observation actions execute.

`otel` doesn't change code while the program runs. It adds observation code at build time, and what runs afterwards is **a program that already contains the observation code**.

[![Animation: build time and run time](/beginner/build-en.svg)](/beginner/build-en.svg)

## What otel does during the build

The [Hacking Guide overview](../hacking/overview.md) describes two extra phases that happen before the actual compilation:

1. **Preprocess**: analyze the project's dependencies, select the rules to apply, and prepare the dependencies those rules need.
2. **Instrument**: generate code from the rules and add it to the source passed to the compiler.

Between the two phases, the tool takes over compilation through Go's built-in `-toolexec` flag. Each time `go build` invokes the compiler, linker or another tool, it calls `otel` first, which decides whether to rewrite the input and then runs the real tool.

## First decide which moment the problem belongs to

| Symptom | Happens at | Check first |
| --- | --- | --- |
| Package not found, compile errors | Build time | Dependencies and rules, see [Debugging](../hacking/debug.md) |
| Program runs but no telemetry appears | Run time | Whether rules matched and data is exported, see [OpenTelemetry Settings](../user/sdk-config.md) |

> [!NOTE]
> Because dependencies must be prepared and a full rebuild is forced, builds are slower with `otel`. See [Compilation Time](../user/compilation-time.md).

**Interactive version**: to step through the animation frame by frame and try the exercises, open [step 4 in the community-maintained LoongSuite Go Guide](https://qinghua362330.github.io/loongsuite-go-guide/#/start/4) (Chinese).

Next: [Rules and Hooks](./rules-and-hooks.md)
