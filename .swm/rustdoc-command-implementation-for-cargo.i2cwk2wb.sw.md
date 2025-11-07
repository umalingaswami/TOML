---
title: Rustdoc Command Implementation for Cargo
---
# introduction

This document explains the implementation of the rustdoc command in Cargo. It answers:

1. How the CLI interface for the rustdoc command is structured.
2. What options and arguments are exposed to the user.
3. How the command processes input and triggers documentation generation.

# defining the CLI interface and arguments

The rustdoc command is defined in <SwmPath>[src/…/commands/rustdoc.rs](src/bin/cargo/commands/rustdoc.rs)</SwmPath>. The cli() function sets up the command's interface using clap. It declares the command name, description, and a variety of arguments that control what and how documentation is built. These include:

- A catch-all "args" argument to pass custom flags directly to rustdoc.
- Flags like --open to open docs in a browser after building.
- Package selection and target specification options.
- Build mode (release/debug), features, and target triple.
- Output directory and manifest path.

<SwmSnippet path="/src/bin/cargo/commands/rustdoc.rs" line="1">

---

This setup ensures users can customize documentation builds extensively while integrating with Cargo's existing package and build system options.

```rust
use command_prelude::*;

use cargo::ops::{self, DocOptions};

pub fn cli() -> App {
    subcommand("rustdoc")
        .setting(AppSettings::TrailingVarArg)
        .about("Build a package's documentation, using specified custom flags.")
        .arg(Arg::with_name("args").multiple(true))
        .arg(opt(
            "open",
            "Opens the docs in a browser after the operation",
        ))
        .arg_package("Package to document")
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
        .arg_features()
        .arg_target_triple("Build for the target triple")
        .arg_target_dir()
        .arg_manifest_path()
        .arg_message_format()
        .after_help(
            "\
```

---

</SwmSnippet>

# documenting usage details

The cli() function also appends detailed help text explaining the command's behavior. It clarifies that only the specified package target is documented, dependencies are excluded, and that the custom flags are appended to rustdoc's invocation. It also explains how the --package argument affects which package is documented.

<SwmSnippet path="/src/bin/cargo/commands/rustdoc.rs" line="36">

---

This text guides users on the command's scope and how to specify packages, reducing confusion about what gets documented.

```rust
The specified target for the current package (or package specified by SPEC if
provided) will be documented with the specified <opts>... being passed to the
final rustdoc invocation. Dependencies will not be documented as part of this
command.  Note that rustdoc will still unconditionally receive arguments such
as -L, --extern, and --crate-type, and the specified <opts>...  will simply be
added to the rustdoc invocation.

If the --package argument is given, then SPEC is a package id specification
which indicates which package should be documented. If it is not given, then the
current package is documented. For more information on SPEC and its format, see
the `cargo help pkgid` command.
",
        )
}
```

---

</SwmSnippet>

# executing the rustdoc command

The exec() function handles the command's runtime logic. It first resolves the workspace and compiles options for a single package in documentation mode, explicitly disabling dependency docs. It then collects any extra rustdoc flags passed via the "args" argument and attaches them to the compile options.

Finally, it constructs a <SwmToken path="src/bin/cargo/commands/rustdoc.rs" pos="3:10:10" line-data="use cargo::ops::{self, DocOptions};">`DocOptions`</SwmToken> struct with the open flag and compile options, and calls ops::doc to perform the actual documentation build.

<SwmSnippet path="/src/bin/cargo/commands/rustdoc.rs" line="51">

---

This flow cleanly separates argument parsing, option preparation, and execution, leveraging Cargo's existing compilation and documentation infrastructure.

```rust
pub fn exec(config: &mut Config, args: &ArgMatches) -> CliResult {
    let ws = args.workspace(config)?;
    let mut compile_opts =
        args.compile_options_for_single_package(config, CompileMode::Doc { deps: false })?;
    let target_args = values(args, "args");
    compile_opts.target_rustdoc_args = if target_args.is_empty() {
        None
    } else {
        Some(target_args)
    };
    let doc_opts = DocOptions {
        open_result: args.is_present("open"),
        compile_opts,
    };
    ops::doc(&ws, &doc_opts)?;
    Ok(())
}
```

---

</SwmSnippet>

&nbsp;

*This is an auto-generated document by Swimm 🌊 and has not yet been verified by a human*

<SwmMeta version="3.0.0" repo-id="Z2l0aHViJTNBJTNBVE9NTCUzQSUzQXVtYWxpbmdhc3dhbWk=" repo-name="TOML"><sup>Powered by [Swimm](https://app.swimm.io/)</sup></SwmMeta>
