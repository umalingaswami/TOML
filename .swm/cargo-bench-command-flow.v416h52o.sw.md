---
title: Cargo Bench Command Flow
---
# introduction

This document explains the design and flow of the Cargo bench command implementation. It answers:

1. How the CLI interface for the bench command is structured and what options it supports.
2. How benchmark filtering and argument passing to benchmark binaries are handled.
3. How workspace and package selection is managed.
4. How compilation options and benchmark execution are configured and invoked.

# defining the CLI interface and options

The bench command is defined with a subcommand builder that sets up its name, description, and arguments. It supports filtering benchmarks by name, passing arbitrary arguments to the benchmark binaries, and selecting specific targets or packages to benchmark. It also includes options like `--no-run` to compile without running benchmarks, `--no-fail-fast` to continue running all benchmarks despite failures, and workspace-related flags like `--all` and `--exclude`.

<SwmSnippet path="/src/bin/cargo/commands/bench.rs" line="1">

---

This setup is important because it exposes all the necessary knobs users need to control benchmarking behavior directly from the CLI, while also forwarding unknown arguments to the underlying benchmark binaries for fine-grained control.

```rust
use command_prelude::*;

use cargo::ops::{self, TestOptions};

pub fn cli() -> App {
    subcommand("bench")
        .setting(AppSettings::TrailingVarArg)
        .about("Execute all benchmarks of a local package")
        .arg(
            Arg::with_name("BENCHNAME")
                .help("If specified, only run benches containing this string in their names"),
        )
        .arg(
            Arg::with_name("args")
                .help("Arguments for the bench binary")
                .multiple(true)
                .last(true),
        )
        .arg_targets_all(
            "Benchmark only this package's library",
            "Benchmark only the specified binary",
            "Benchmark all binaries",
            "Benchmark only the specified example",
            "Benchmark all examples",
            "Benchmark only the specified test target",
            "Benchmark all tests",
            "Benchmark only the specified bench target",
            "Benchmark all benches",
            "Benchmark all targets",
        )
        .arg(opt("no-run", "Compile, but don't run benchmarks"))
        .arg_package_spec(
            "Package to run benchmarks for",
            "Benchmark all packages in the workspace",
            "Exclude packages from the benchmark",
        )
        .arg_jobs()
        .arg_features()
        .arg_target_triple("Build for the target triple")
        .arg_target_dir()
        .arg_manifest_path()
        .arg_message_format()
        .arg(opt(
            "no-fail-fast",
            "Run all benchmarks regardless of failure",
        ))
        .after_help(
            "\
```

---

</SwmSnippet>

# explaining argument forwarding and package selection

The benchmark filtering argument <SwmToken path="src/bin/cargo/commands/bench.rs" pos="10:6:6" line-data="            Arg::with_name(&quot;BENCHNAME&quot;)">`BENCHNAME`</SwmToken> and all arguments after `--` are forwarded to the benchmark binaries, which use Rust's built-in libtest framework. This separation clarifies which arguments affect Cargo itself and which affect the benchmark execution. The documentation also clarifies how package selection works: if <SwmToken path="src/bin/cargo/commands/bench.rs" pos="56:4:5" line-data="If the --package argument is given, then SPEC is a package id specification">`--package`</SwmToken> is specified, only that package is benchmarked; otherwise, the current package is used. The `--all` flag benchmarks all workspace packages, with `--exclude` requiring `--all`.

<SwmSnippet path="/src/bin/cargo/commands/bench.rs" line="49">

---

This design cleanly separates concerns between Cargo's orchestration and the benchmark binary's runtime behavior, while providing flexible package targeting.

```rust
The benchmark filtering argument `BENCHNAME` and all the arguments following the
two dashes (`--`) are passed to the benchmark binaries and thus to libtest
(rustc's built in unit-test and micro-benchmarking framework).  If you're
passing arguments to both Cargo and the binary, the ones after `--` go to the
binary, the ones before go to Cargo.  For details about libtest's arguments see
the output of `cargo bench -- --help`.

If the --package argument is given, then SPEC is a package id specification
which indicates which package should be benchmarked. If it is not given, then
the current package is benchmarked. For more information on SPEC and its format,
see the `cargo help pkgid` command.

All packages in the workspace are benchmarked if the `--all` flag is supplied. The
`--all` flag is automatically assumed for a virtual manifest.
Note that `--exclude` has to be specified in conjunction with the `--all` flag.

The --jobs argument affects the building of the benchmark executable but does
not affect how many jobs are used when running the benchmarks.

Compilation can be customized with the `bench` profile in the manifest.
",
        )
}
```

---

</SwmSnippet>

# setting up compilation and test options

When executing the bench command, the workspace is resolved first. Then compilation options are constructed with the <SwmToken path="src/bin/cargo/commands/bench.rs" pos="75:16:18" line-data="    let mut compile_opts = args.compile_options(config, CompileMode::Bench)?;">`CompileMode::Bench`</SwmToken> mode, and the release profile is enabled by default for benchmarking. The <SwmToken path="src/bin/cargo/commands/bench.rs" pos="3:10:10" line-data="use cargo::ops::{self, TestOptions};">`TestOptions`</SwmToken> struct is created to hold flags like <SwmToken path="src/bin/cargo/commands/bench.rs" pos="79:1:1" line-data="        no_run: args.is_present(&quot;no-run&quot;),">`no_run`</SwmToken> and <SwmToken path="src/bin/cargo/commands/bench.rs" pos="80:1:1" line-data="        no_fail_fast: args.is_present(&quot;no-fail-fast&quot;),">`no_fail_fast`</SwmToken> along with the compilation options.

<SwmSnippet path="/src/bin/cargo/commands/bench.rs" line="73">

---

This step configures how the benchmarks are built and run, ensuring that the benchmarks are compiled in release mode for performance and that the command respects user flags controlling execution behavior.

```rust
pub fn exec(config: &mut Config, args: &ArgMatches) -> CliResult {
    let ws = args.workspace(config)?;
    let mut compile_opts = args.compile_options(config, CompileMode::Bench)?;
    compile_opts.build_config.release = true;

    let ops = TestOptions {
        no_run: args.is_present("no-run"),
        no_fail_fast: args.is_present("no-fail-fast"),
        compile_opts,
    };
```

---

</SwmSnippet>

# collecting benchmark arguments

The benchmark arguments vector is built by first adding the optional benchmark name filter, then appending any additional arguments passed after `--`. This vector is passed to the benchmark binaries during execution.

<SwmSnippet path="/src/bin/cargo/commands/bench.rs" line="84">

---

This explicit collection ensures that all user-supplied arguments intended for the benchmark binaries are forwarded correctly, enabling users to customize benchmark runs.

```rust
    let mut bench_args = vec![];
    bench_args.extend(
        args.value_of("BENCHNAME")
            .into_iter()
            .map(|s| s.to_string()),
    );
    bench_args.extend(
        args.values_of("args")
            .unwrap_or_default()
            .map(|s| s.to_string()),
    );
```

---

</SwmSnippet>

# running benchmarks and handling errors

Finally, the <SwmToken path="src/bin/cargo/commands/bench.rs" pos="96:9:9" line-data="    let err = ops::run_benches(&amp;ws, &amp;ops, &amp;bench_args)?;">`run_benches`</SwmToken> function is called with the workspace, test options, and benchmark arguments. The result is matched to handle success or failure. If benchmarks fail, the error is converted into a CLI error with an appropriate exit code.

<SwmSnippet path="/src/bin/cargo/commands/bench.rs" line="96">

---

This error handling ensures that failures in benchmarks propagate correctly to the user, with meaningful exit codes for scripting or CI integration.

```rust
    let err = ops::run_benches(&ws, &ops, &bench_args)?;
    match err {
        None => Ok(()),
        Some(err) => Err(match err.exit.as_ref().and_then(|e| e.code()) {
            Some(i) => CliError::new(format_err!("bench failed"), i),
            None => CliError::new(err.into(), 101),
        }),
    }
}
```

---

</SwmSnippet>

&nbsp;

*This is an auto-generated document by Swimm 🌊 and has not yet been verified by a human*

<SwmMeta version="3.0.0" repo-id="Z2l0aHViJTNBJTNBVE9NTCUzQSUzQXVtYWxpbmdhc3dhbWk=" repo-name="TOML"><sup>Powered by [Swimm](https://app.swimm.io/)</sup></SwmMeta>
