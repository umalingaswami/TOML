---
title: Project Manifest Verification Command
---
# introduction

This document explains the implementation of a command that verifies the correctness of a crate manifest file. The main questions answered here are:

1. How is the command integrated into the CLI?
2. How does the command handle errors and report them?
3. How is the manifest file read and validated?
4. How is success communicated back to the user?

# integrating the command into the CLI

<SwmSnippet path="/src/bin/cargo/commands/verify_project.rs" line="10">

---

The command is registered as a subcommand named <SwmToken path="src/bin/cargo/commands/verify_project.rs" pos="13:4:6" line-data="    subcommand(&quot;verify-project&quot;)">`verify-project`</SwmToken> with a description and an argument for the manifest path. This setup allows users to invoke it like any other cargo subcommand, specifying the manifest file to check.

```rust
use cargo::print_json;

pub fn cli() -> App {
    subcommand("verify-project")
        .about("Check correctness of crate manifest")
        .arg_manifest_path()
}
```

---

</SwmSnippet>

# error handling and reporting

<SwmSnippet path="/src/bin/cargo/commands/verify_project.rs" line="18">

---

A helper function <SwmToken path="src/bin/cargo/commands/verify_project.rs" pos="19:3:3" line-data="    fn fail(reason: &amp;str, value: &amp;str) -&gt; ! {">`fail`</SwmToken> is defined to handle errors uniformly. It creates a JSON object with the error reason and value, prints it, and then exits the process with a failure code. This approach ensures that errors are machine-readable and consistent.

```rust
pub fn exec(config: &mut Config, args: &ArgMatches) -> CliResult {
    fn fail(reason: &str, value: &str) -> ! {
        let mut h = HashMap::new();
        h.insert(reason.to_string(), value.to_string());
        print_json(&h);
        process::exit(1)
    }
```

---

</SwmSnippet>

# reading and validating the manifest file

The command first tries to determine the manifest file path from the CLI arguments or configuration. If this fails, it triggers the <SwmToken path="src/bin/cargo/commands/verify_project.rs" pos="19:3:3" line-data="    fn fail(reason: &amp;str, value: &amp;str) -&gt; ! {">`fail`</SwmToken> function with an "invalid" reason.

Next, it attempts to open and read the manifest file into a string. Any I/O errors during this step also cause an immediate failure with a descriptive message.

<SwmSnippet path="/src/bin/cargo/commands/verify_project.rs" line="26">

---

Finally, it parses the file contents as TOML. If parsing fails, it again calls <SwmToken path="src/bin/cargo/commands/verify_project.rs" pos="29:8:8" line-data="        Err(e) =&gt; fail(&quot;invalid&quot;, &amp;e.to_string()),">`fail`</SwmToken> with an <SwmToken path="src/bin/cargo/commands/verify_project.rs" pos="38:9:11" line-data="        fail(&quot;invalid&quot;, &quot;invalid-format&quot;);">`invalid-format`</SwmToken> message. This sequence ensures that only valid, readable TOML manifests pass the check.

```rust
    let mut contents = String::new();
    let filename = match args.root_manifest(config) {
        Ok(filename) => filename,
        Err(e) => fail("invalid", &e.to_string()),
    };

    let file = File::open(&filename);
    match file.and_then(|mut f| f.read_to_string(&mut contents)) {
        Ok(_) => {}
        Err(e) => fail("invalid", &format!("error reading file: {}", e)),
    };
    if contents.parse::<toml::Value>().is_err() {
        fail("invalid", "invalid-format");
    }
```

---

</SwmSnippet>

# reporting success

<SwmSnippet path="/src/bin/cargo/commands/verify_project.rs" line="41">

---

If all previous steps complete without error, the command prints a JSON object indicating success. This confirms to the caller that the manifest is valid.

```rust
    let mut h = HashMap::new();
    h.insert("success".to_string(), "true".to_string());
    print_json(&h);
    Ok(())
}
```

---

</SwmSnippet>

&nbsp;

*This is an auto-generated document by Swimm 🌊 and has not yet been verified by a human*

<SwmMeta version="3.0.0" repo-id="Z2l0aHViJTNBJTNBVE9NTCUzQSUzQXVtYWxpbmdhc3dhbWk=" repo-name="TOML"><sup>Powered by [Swimm](https://app.swimm.io/)</sup></SwmMeta>
