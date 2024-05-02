# Helpers for compile fail tests

This crate contains everything needed to set up compile tests for the Bevy repo. It, like all Bevy compile test crates, is excluded from the Bevy workspace. This is done to not fail [`crater` tests](https://github.com/rust-lang/crater) for Bevy. The `CI` workflow executes these tests on the stable rust toolchain see ([tools/ci](../../tools/ci/src/main.rs)).

## Writing new test cases

Test cases are annotated .rs files. These annotations define how the test case should be run and what we're expecting to fail. Please see <https://github.com/oli-obk/ui_test/blob/main/README.md> for more information.

Annotations can roughly be split into global annotations which are prefixed with `//@` and define how tests should be run and error annotations which are prefixed with `//~` and define where errors we expect to happen. From the global annotations, you're only likely to care about `//@check-pass` which will make any compile errors in the test trigger a test failure.

The error annotations are composed of two parts.
An optional location specifier:

- `^` The error happens on the line above.
- `v` The error happens on the line below.
- `|` The error annotation is connected to another one.
- If the location specifier is missing, the error is assumed to happen on the same line as the annotation.

An error matcher:

- `E####` The error we're expecting has the [`####` rustc error code](https://doc.rust-lang.org/error_codes/error-index.html), e.g `E0499`
- `<lint_name>` The given [compiler lint](https://doc.rust-lang.org/rustc/lints/index.html) is triggered, e.g. `dead_code`
- `LEVEL: <substring>` A compiler error of the given level (valid levels are: `ERROR`, `HELP`, `WARN` or `NOTE`) will be raised and it will contain the `substring`. Substrings can contain spaces.
- `LEVEL: /<regex>/` Same as above but a regex is used to match the error message.

An example of an error annotation would be `//~v ERROR: missing trait`. This error annotation will match any error occurring on the line below that contains the substring `missing trait`.

## Adding compile fail tests for a crate that doesn't have them

This will be a rather involved process. You'll have to:

- Create an empty library crate in the [`crates`](..) directory.
- Add this crate as a `dev-dependency`.
- Create a folder called `tests` within the new crate.
- Add a test runner file to this folder. The file should contain a main function calling one of the test functions defined in this crate.
- Add a `[[test]]` table to the `Cargo.toml`. This table will need to contain `harness = false` and `name = <name of the test runner file you defined>`.
- Modify the [`Ci`](../../tools/ci/) tool to run `cargo test` on this crate.
- And finally, write your compile tests.

If you have any questions, don't be scared to ask for help.

## A note about `.stderr` files

We're capable of generating `.stderr` files for all our compile tests. These files contain the error output generated by the test. To create or regenerate them yourself, trigger the tests with the `BLESS` environment variable set to any value (e.g. `BLESS="some symbolic value"`). We currently have to ignore mismatches between these files and the actual stderr output from their corresponding test due to issues with file paths. We attempt to sanitize file paths but for proc-macros, the compiler error messages contain file paths to the current toolchain's copy of the standard library. If we knew of a way to construct a path to the current toolchains folder we could fix this.