---
title: Cargo Rustc Command Implementation
---
# introduction

This document explains the implementation of the <SwmToken path="src/bin/cargo/commands/rustc.rs" pos="6:4:4" line-data="    subcommand(&quot;rustc&quot;)">`rustc`</SwmToken> command in Cargo. It covers:

1. How the CLI interface is defined and what options it exposes.
2. How the help message clarifies the command's behavior and constraints.
3. How the command execution parses arguments, sets compile modes, and invokes compilation.

# defining the CLI interface

The CLI interface is defined in <SwmPath>[src/…/commands/rustc.rs](src/bin/cargo/commands/rustc.rs)</SwmPath> using a builder pattern. It declares the <SwmToken path="src/bin/cargo/commands/rustc.rs" pos="6:4:4" line-data="    subcommand(&quot;rustc&quot;)">`rustc`</SwmToken> subcommand with various flags and options to control compilation. These include:

- Selecting the package and targets to build (library, binaries, examples, tests, benches).
- Setting the number of jobs, release mode, and profile.
- Passing arbitrary arguments directly to the compiler.
- Specifying features, target triples, output directories, manifest paths, and message formats.

<SwmSnippet path="/src/bin/cargo/commands/rustc.rs" line="1">

---

This setup ensures the command exposes all necessary controls for compiling a package and its dependencies while allowing fine-grained customization of the compiler invocation.

```rust
use command_prelude::*;

use cargo::ops;

pub fn cli() -> App {
    subcommand("rustc")
        .setting(AppSettings::TrailingVarArg)
        .about("Compile a package and all of its dependencies")
        .arg(Arg::with_name("args").multiple(true))
        .arg_package("Package to build")
        .arg_jobs()
        .arg_targets_all(
            "Build only this package's library",
            "Build only the specified binary",
            "Build all binaries",
            "Build only the specified example",
            "Build all examples",
            "Build only the specified test target",
            "Build all tests",
            "Build only the specified bench target",
            "Build all benches",
            "Build all targets",
        )
        .arg_release("Build artifacts in release mode, with optimizations")
        .arg(opt("profile", "Profile to build the selected target for").value_name("PROFILE"))
        .arg_features()
        .arg_target_triple("Target triple which compiles will be for")
        .arg_target_dir()
        .arg_manifest_path()
        .arg_message_format()
```

---

</SwmSnippet>

# clarifying command behavior in help text

The help message appended to the CLI definition explains key usage details:

- Only one target can be compiled at a time; if multiple targets exist, filters like <SwmToken path="src/bin/cargo/commands/rustc.rs" pos="41:20:21" line-data="target is available for the current package the filters of --lib, --bin, etc,">`--lib`</SwmToken> or <SwmToken path="src/bin/cargo/commands/rustc.rs" pos="41:24:25" line-data="target is available for the current package the filters of --lib, --bin, etc,">`--bin`</SwmToken> must be used.
- The <SwmToken path="src/bin/cargo/commands/rustc.rs" pos="35:0:2" line-data="&lt;args&gt;... will all be passed to the final compiler invocation, not any of the">`<args>...`</SwmToken> passed are forwarded only to the final compiler invocation, not dependencies.
- Some compiler arguments (<SwmToken path="src/bin/cargo/commands/rustc.rs" pos="37:6:7" line-data="arguments such as -L, --extern, and --crate-type, and the specified &lt;args&gt;...">`-L`</SwmToken>, <SwmToken path="src/bin/cargo/commands/rustc.rs" pos="37:10:11" line-data="arguments such as -L, --extern, and --crate-type, and the specified &lt;args&gt;...">`--extern`</SwmToken>, <SwmToken path="src/bin/cargo/commands/rustc.rs" pos="37:16:19" line-data="arguments such as -L, --extern, and --crate-type, and the specified &lt;args&gt;...">`--crate-type`</SwmToken>) are always included by Cargo.
- To pass flags to all compiler processes, environment variables or config options should be used instead.

<SwmSnippet path="/src/bin/cargo/commands/rustc.rs" line="31">

---

This clarifies the command’s scope and how arguments are handled, preventing user confusion about argument propagation and target selection.

```rust
        .after_help(
            "\
The specified target for the current package (or package specified by SPEC if
provided) will be compiled along with all of its dependencies. The specified
<args>... will all be passed to the final compiler invocation, not any of the
dependencies. Note that the compiler will still unconditionally receive
arguments such as -L, --extern, and --crate-type, and the specified <args>...
will simply be added to the compiler invocation.

This command requires that only one target is being compiled. If more than one
target is available for the current package the filters of --lib, --bin, etc,
must be used to select which target is compiled. To pass flags to all compiler
processes spawned by Cargo, use the $RUSTFLAGS environment variable or the
`build.rustflags` configuration option.
",
        )
}
```

---

</SwmSnippet>

# parsing arguments and executing compilation

The <SwmToken path="src/bin/cargo/commands/rustc.rs" pos="49:4:4" line-data="pub fn exec(config: &amp;mut Config, args: &amp;ArgMatches) -&gt; CliResult {">`exec`</SwmToken> function handles the command execution:

- It retrieves the workspace from the arguments and config.
- It determines the compile mode based on the <SwmToken path="src/bin/cargo/commands/rustc.rs" pos="25:7:7" line-data="        .arg(opt(&quot;profile&quot;, &quot;Profile to build the selected target for&quot;).value_name(&quot;PROFILE&quot;))">`profile`</SwmToken> argument, supporting <SwmToken path="src/bin/cargo/commands/rustc.rs" pos="52:4:4" line-data="        Some(&quot;dev&quot;) | None =&gt; CompileMode::Build,">`dev`</SwmToken>, <SwmToken path="src/bin/cargo/commands/rustc.rs" pos="18:10:10" line-data="            &quot;Build only the specified test target&quot;,">`test`</SwmToken>, <SwmToken path="src/bin/cargo/commands/rustc.rs" pos="20:10:10" line-data="            &quot;Build only the specified bench target&quot;,">`bench`</SwmToken>, and <SwmToken path="src/bin/cargo/commands/rustc.rs" pos="55:4:4" line-data="        Some(&quot;check&quot;) =&gt; CompileMode::Check { test: false },">`check`</SwmToken>. Unknown profiles cause an error.
- It builds compile options for a single package using helper methods.
- It extracts any extra arguments passed to the compiler and attaches them to the compile options.
- Finally, it calls <SwmToken path="src/bin/cargo/commands/rustc.rs" pos="72:1:3" line-data="    ops::compile(&amp;ws, &amp;compile_opts)?;">`ops::compile`</SwmToken> with the workspace and options to perform the actual compilation.

<SwmSnippet path="/src/bin/cargo/commands/rustc.rs" line="49">

---

This function ties together argument parsing, validation, and invoking the core compilation logic, ensuring the command behaves as expected based on user input.

```rust
pub fn exec(config: &mut Config, args: &ArgMatches) -> CliResult {
    let ws = args.workspace(config)?;
    let mode = match args.value_of("profile") {
        Some("dev") | None => CompileMode::Build,
        Some("test") => CompileMode::Test,
        Some("bench") => CompileMode::Bench,
        Some("check") => CompileMode::Check { test: false },
        Some(mode) => {
            let err = format_err!(
                "unknown profile: `{}`, use dev,
                                   test, or bench",
                mode
            );
            return Err(CliError::new(err, 101));
        }
    };
    let mut compile_opts = args.compile_options_for_single_package(config, mode)?;
    let target_args = values(args, "args");
    compile_opts.target_rustc_args = if target_args.is_empty() {
        None
    } else {
        Some(target_args)
    };
    ops::compile(&ws, &compile_opts)?;
    Ok(())
}
```

---

</SwmSnippet>

&nbsp;

*This is an auto-generated document by Swimm 🌊 and has not yet been verified by a human*

<SwmMeta version="3.0.0" repo-id="Z2l0aHViJTNBJTNBVE9NTCUzQSUzQXVtYWxpbmdhc3dhbWk=" repo-name="TOML"><sup>Powered by [Swimm](https://app.swimm.io/)</sup></SwmMeta>
