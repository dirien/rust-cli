# Let's build a CLI in Rust 🦀 with Clap

## Introduction

This article is heavily inspired
by [Let's build a CLI in Go with Cobra](https://www.thorsten-hans.com/lets-build-a-cli-in-go-with-cobra/)
article from Thorsten Hans. But instead of Go and Cobra, we will use `Rust` 🦀 and `Clap`.

We are going to build the `Rust` 🦀 version of `stringer`, a very simple CLI application that takes a string as input
and, depending on the command, reverse or inspect it. The feature set is big enough to show the power of `Clap` and gives
us a good starting point to build more complex CLI applications in the future.

## Prerequisites

Before we start, we need to make sure we have the following tools installed:

- [Rust](https://www.rust-lang.org)
- An IDE or text editor of your choice

## Building a `Rust` 🦀 CLI with CLap

### What is `Clap`?

[Clap](https://github.com/clap-rs/clap) is a command line argument parser for `Rust` 🦀. It provides a macro to declare the
application's arguments and subcommands. It also generates help and version messages and provides autocompletion
for `bash`, `zsh`, and `fish`
via `clap_complete`

In the recent iteration (4.0.0) of `Clap`, the team did great job on reducing the amount of dependencies and improving
the performance. Check the [clap 4.0, a Rust CLI argument parser](https://epage.github.io/blog/2022/09/clap4/) blog post
for more details.

From a statistic point of view: `Clap` has over 10k stars on GitHub and is by over 240k repositories.

### Initialize the project

Let us jump straight into the code. We will create a new `Rust` 🦀 project with the following command:

```bash
cargo init
```

To use `Clap` in a `Rust` 🦀 project, we need add it as dependencies to the `Cargo.toml` file:

```bash
cargo add clap --features derive
```

To derive clap types, we need to enable the `derive` feature flag.

### Define the CLI structure

Now we can start to build our CLI structure. We will start with the `main.rs` file:

```rust
#[derive(Parser)]
#[command(author, version)]
#[command(about = "stringer - a simple CLI to transform and inspect strings", long_about = "stringer is a super fancy CLI (kidding)

One can use stringer to modify or inspect strings straight from the terminal")]
struct Cli {
    #[command(subcommand)]
    command: Option<Commands>,
}

#[derive(Subcommand)]
enum Commands {
    /// Reverses a string
    Reverse(Reverse),
    /// Inspects a string
    Inspect(Inspect),
}

#[derive(Args)]
struct Reverse {
    /// The string to reverse
    string: Option<String>,
}

#[derive(Args)]
struct Inspect {
    /// The string to inspect
    string: Option<String>,
    #[arg(short = 'd', long = "digits")]
    only_digits: bool,
}
```

#### Derive, Derive and Derive

The CLI structure is defined with the `#[derive(Parser)]` macro. The `command` macro is used to define attributes for
the CLI, here I used the `author` and `version` attributes. The `about` attribute is used to define the short
description of the CLI and `long_about` is used to define the long description of the CLI.

```rust
# [command(author, version)]
# [command(about = "stringer - a simple CLI to transform and inspect strings", long_about = "stringer is a super fancy CLI (kidding)...")
```

The `command` macro is used to define the subcommands of the CLI. In our case, we have two subcommands: `reverse`
and `inspect`. The `inspect` subcommand has an additional argument `only_digits` which is a boolean flag. This is
defined in the `Inspect` struct. The `Inspect` struct is derived from the `Args` trait.

```rust
#[derive(Args)]
struct Inspect {
    /// The string to inspect
    string: Option<String>,
    #[arg(short = 'd', long = "digits")]
    only_digits: bool,
}
```

#### Doc comments

You probably noticed the strange way I created comments in the code. This is because I use the doc comment syntax from
`Clap`.

A doc comment consists of three parts:

* Short summary
* A blank line (whitespace only)
* Detailed description, all the rest

So instead of writing:

```rust
#[derive(Parser)]
#[command(about = "I am a program and I work, just pass `-h`", long_about = None)]
struct Foo {
    #[arg(short, help = "Pass `-h` and you'll see me!")]
    bar: String,
}
```

I can write:

```rust
#[derive(Parser)]
/// I am a program and I work, just pass `-h`
///
/// Very long description of the program
struct Foo {
    /// Pass `-h` and you'll see me!
    bar: String,
}
```

> Note: Attributes have still priority over doc comments!

### The business logic

Now that we have defined the structure of our CLI, we can start to implement the business logic. For this, I created a
subfolder in the `src` folder called `api`. In this folder, I will create a `stringer.rs` file and a `mod.rs` file.

The `mod.rs` file will contain the following code:

```rust
pub mod stringer;
```

The `stringer.rs` file will contain two public functions: `reverse` and `inspect`. The `reverse` function will reverse a
given string and the `inspect` function will inspect a given string and return the number of characters or only count
the digits in the string.

```rust
pub fn reverse(input: &String) -> String {
    return input.chars().rev().collect();
}

pub fn inspect(input: &String, digits: bool) -> (i32, String) {
    if !digits {
        return (input.len() as i32, String::from("char"));
    }
    return (inspect_numbers(input), String::from("digit"));
}

fn inspect_numbers(input: &String) -> i32 {
    let mut count = 0;
    for c in input.chars() {
        if c.is_digit(10) {
            count += 1;
        }
    }
    return count;
}
```

### Putting it all together

Now we can put everything together. We will start with the `main.rs` file with adding the `api` module by adding the
following line to the top of the file:

```rust
...
mod api;
```

In the `main` function, we will add the following code:

```rust
fn main() {
    let cli = Cli::parse();

    match &cli.command {
        Some(Commands::Reverse(name)) => {
            match name.string {
                Some(ref _name) => {
                    let reverse = api::stringer::reverse(_name);
                    println!("{}", reverse);
                }
                None => {
                    println!("Please provide a string to reverse");
                }
            }
        }
        Some(Commands::Inspect(name)) => {
            match name.string {
                Some(ref _name) => {
                    let (res, kind) = api::stringer::inspect(_name, name.only_digits);

                    let mut plural_s = "s";
                    if res == 1 {
                        plural_s = "";
                    }

                    println!("{:?} has {} {}{}.", _name, res, kind, plural_s);
                }
                None => {
                    println!("Please provide a string to inspect");
                }
            }
        }
        None => {}
    }
}
```

With `Cli::parse()` we parse the CLI arguments and store them in the `cli` variable. Then we match the `command` field
of the `cli` variable. If the `command` field is `None`, we do nothing. If the `command` field is `Some`, we match the
subcommand and execute the corresponding code from our `api` module.

Let's try it out by using the same examples from Thorsten's blog post:

```bash
cargo run -- reverse foo
oof

cargo run -- reverse bar
rab

cargo run -- inspect lorem
"lorem" has 5 chars.

cargo run -- inspect FooBar
"FooBar" has 6 chars.
```

### Adding flags to commands with `Clap`

Most of the time, you want to add flags to your commands to make them more flexible. In this section, I will show you
how to add flags to your commands with `Clap`. For this, I will add a flag to the `inspect` command to only count the
digits in a string instead of counting all characters.

The flag will be called `--digits` or `-d` and will be added with the `#[arg(short, long)]` attribute. The `short`
and `long` will be set to `d` and `digits` respectively.

```rust
#[derive(Args)]
struct Inspect {
    /// The string to inspect
    string: Option<String>,
    #[arg(short = 'd', long = "digits")]
    only_digits: bool,
}
```

And I pass the `only_digits` flag to the `inspect` function in the `api` module.

```rust
let (res, kind) = api::stringer::inspect(_name, name.only_digits);
```

Now we can try it out, by using the same examples from Thorsten's blog post:

```bash
cargo run -- inspect A1B2C3 --digits
"A1B2C3" has 3 digits.

cargo run -- inspect A1B2C3 -d
"A1B2C3" has 3 digits.

# check command help
cargo run -- inspect --help   

Inspects a string

Usage: stinger inspect [OPTIONS] [STRING]

Arguments:
  [STRING]  The string to inspect

Options:
  -d, --digits  
  -h, --help    Print help information
```

#### The version and help flags

`Clap` automatically adds the `--version` and `--help` flags to your CLI. The `--version` flag will print the version of
your application and the `--help` flag will print the help of your application.

```bash
cargo run -- --version
    Finished dev [unoptimized + debuginfo] target(s) in 0.02s
     Running `target/debug/stinger --version`
stinger 0.1.0
```

This will print the version of your application. The version is defined in the `Cargo.toml` file. But you can also set
the version via the `version` attribute of the `Command` attribute.

```rust
# [command(author, version="1.1.0")]
```

```bash
cargo run -- --help
stringer is a super fancy CLI (kidding)

One can use stringer to modify or inspect strings straight from the terminal

Usage: stinger [COMMAND]

Commands:
  reverse
          Reverses a string
  inspect
          Inspects a string
  help
          Print this message or the help of the given subcommand(s)

Options:
  -h, --help
          Print help information (use `-h` for a summary)

  -V, --version
          Print version information
```

## Conclusion

Building a CLI application with `Rust` 🦀 is not that hard. Similar to languages like Go, there are a lot of libraries that
will help you in this task. `Clap` is one of them, and it is very easy to use.

I hope this tutorial will help you to get started with building your own CLI application with `Rust` 🦀.

## Resources

* [Clap](https://docs.rs/clap/latest/clap/index.html)
* [Thorsten Hans](https://www.thorsten-hans.com/)