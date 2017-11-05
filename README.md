## Multi-Lingual Literate Programming with `mdsh`

`mdsh` is a bash script compiler and interpreter for markdown files.  It can be used either in a `#!` line to make markdown files executable, or it can be used as a standalone tool to generate dependency-free, distributable bash scripts from markdown files.

By default, `mdsh` only considers `shell` code blocks to be bash code, but you can also use `mdsh` blocks to define handlers for other languages.  For example, this script will run `python`-tagged code blocks by piping them to the `python` command:

~~~markdown
#!/usr/bin/env mdsh

# Hello World in Python

​```mdsh
mdsh-lang-python() { python; }
​```

​```python
print("hello world!")
​```
~~~

Running the above markdown file produces the same results as this equivalent bash script:

```bash
#!/usr/bin/env bash
{ python; } <<'```'
print("hello world!")
​```
```

`mdsh` supports processing blocks of any language that you can write a bash code snippet for, and even lets you write "compile-time" code to transform blocks containing metadata or DSL snippets into bash code.  The results can be either executed on the fly for development, or deployed/distributed via `mdsh --compile`.  Compiled scripts do not include any `mdsh` code, nor do they have any hidden runtime dependencies: everything `mdsh --compile` outputs is code or data you gave it, or generated by bash code you gave it!

**Contents**

<!-- toc -->

- [Installation](#installation)
- [Usage](#usage)
  * [Data Blocks](#data-blocks)
  * [Processing Non-`shell` Languages](#processing-non-shell-languages)
  * [Advanced Block Compilation Techniques](#advanced-block-compilation-techniques)
  * [Excluding Blocks From The Generated Script](#excluding-blocks-from-the-generated-script)
  * [Making Executable (and Editable) Markdown Files](#making-executable-and-editable-markdown-files)
  * [Making Sourceable Scripts (and handling $0)](#making-sourceable-scripts-and-handling-0)
  * [Syntax Highlighting of `mdsh` blocks](#syntax-highlighting-of-mdsh-blocks)
- [Metaprogramming and Code Generation](#metaprogramming-and-code-generation)
  * ["Static Linking" for Distribution](#static-linking-for-distribution)
- [Extending `mdsh` or Reusing its Functions](#extending-mdsh-or-reusing-its-functions)
  * [Adding File Headers or Footers](#adding-file-headers-or-footers)
  * [Altering Existing Functions](#altering-existing-functions)
  * [Available Functions](#available-functions)

<!-- tocstop -->

## Installation

If you have [basher](https://github.com/basherpm/basher), you can install `mdsh` by running `basher install bashup/mdsh`.  Otherwise just clone this repo and copy or link the `mdsh` file to a directory on your `PATH`.  (Or just [download the script directly](https://github.com/bashup/mdsh/raw/master/bin/mdsh) to a directory on your `PATH`.)

## Usage

Running `mdsh` *markdownfile args...* will read and translate backquote-fenced code blocks from *markdownfile* into bash code, based on the language listed on the block and any translation rules you've defined.  The resulting translated script is then run, passing in *args* as positional arguments to the script.

Blocks tagged as `shell` are interpreted as bash code, and directly copied to the translated script.  So arguments passed to `mdsh` after the path to the markdown file are available as `$1`, `$2`, etc. within the top-level code of `shell` blocks, just like in a regular bash script.

(Typically, you won't run `mdsh` directly, but will put `#!/usr/bin/env mdsh` on the first line of your markdown file instead, and make it executable with `chmod +x`.  That way, users of your script won't need to do anything special to run it.)

You can also use `mdsh --compile` *file1 file2...* to translate one or more markdown files to bash code, sending the result to stdout.  (A filename of `-` means "read from standard input".)  This can be useful for debugging, or to make a distributable version of your script that does not require its users to have `mdsh`.

(There is also an `mdsh --eval` *filename* option, which is similar to `--compile`, but only takes one, non-stdin file, and emits special code at the end to support markdown files being sourced; see the section below on [Making Sourceable Scripts](#making-sourceable-scripts-and-handling-0) for more details.)

Both `--eval` and `--compile` can be preceded with `--out` *filename*, in which case *filename*'s contents will be replaced with `mdsh`'s output, if and only if the compilation or run succeeds without any errors.  (The output is buffered in-memory, then output all at once upon successful completion.   If the file already existed, its permissions will remain unchanged.)

### Data Blocks

The contents of blocks that are *not* tagged `shell` or `mdsh` are treated as *data* by default: their contents are added to bash arrays named according to the language on the block, e.g.:

~~~markdown
# Data Arrays Example
Blocks without a defined language processor get translated to a variable
assignment like `mdsh_raw_json+=(\#\ block\ 0)` at that point in the
generated script:

​```json
{ "hello": "world" }
​```
​```shell
echo "${mdsh_raw_json[0]}"   # prints '{ "hello": "world" }'
​```
​```json
{ "this is": "great" }
​```
​```shell
echo "${mdsh_raw_json[0]}"   # prints '{ "hello": "world" }'
echo "${mdsh_raw_json[1]}"   # prints '{ "this is": "great" }'
​```

## Naming Rules
Language names are *case sensitive*, and non-identifier
characters in language names become `_` in variable names:

​```C++
// hey
​```
​```shell
echo "${mdsh_raw_C__[0]}"   # prints '// hey'
​```
~~~

Of course, it would be even better if you could automate the processing of these blocks, so you don't have to follow every block with another `shell` block to process it!  Which is why the next section is on...

### Processing Non-`shell` Languages

To automate the handling of non-`shell` language blocks, you can define one or more `mdsh` blocks, containing "hook functions".   `mdsh` blocks are a bit like a Makefile, in that they define *rules* for how to *build* parts of your script, based on the language used.

These build rules are specified by defining specially-named bash functions.  Unlike functions in `shell` blocks, these functions are *not* part of your script and therefore can't be called directly.  Instead, `mdsh` itself invokes them (or copies out their source code), whenever subsequent blocks match the functions' names:

* An `mdsh-lang-X` function is a template for code to be run when a block of language `X` is encountered.  Its function body is copied to the translated script as a bash compound statment (i.e. in curly braces`{...}`) , that will execute with the block contents as a heredoc on its standard input.  (Its standard output is the same as the overall script's.)
* An `mdsh-compile-X` function is invoked *at compile time* with the block contents as `$1`, and must output a bash source code translation of the block on its stdout.
* If neither an `mdsh-lang-X` nor `mdsh-compile-X` function exists, `mdsh-misc` is invoked *at compile time* with the raw language tag as `$1` and the block contents as `$2`.  The output of `mdsh-misc` will be added to the compiled script.  (The default implementation of `mdsh-misc` outputs code to save the block contents in a variable, as described above in the [Data Blocks](#data-blocks) section, above.)
* An `mdsh-after-X` function is a template for code to be run *after* a block of language `X` is encountered.  Its function body is copied to the translated script as a block just after the `mdsh-lang-X` body, `mdsh-compile-X` output, or `mdsh_raw_X+=(contents)` statement.  It does *not* receive the block source, so its standard input and output are those of the script itself.

If both an `mdsh-lang-X` and `mdsh-compile-X` function exist, `mdsh-lang-X` takes precedence.  Defining either one also disables the `$mdsh_raw_X` functionality: only untranslatable "data" blocks are added to the arrays.

If there is no `mdsh-lang-X` or `mdsh-compile-X` however, the `mdsh-after-X` function can read the most recent block's contents from `${mdsh_raw_X[-1]}` (unless you've replaced the default `mdsh-misc` implementation).  If you don't unset the array, it will keep growing as more blocks of that language are encountered.

Note: these function names are **case sensitive**, so a block tagged with an uppercase `C` will not trigger the same functions as a block tagged with a lowercase `c`, or vice-versa.  Also, note that because `mdsh` blocks are executed at compile time, they do **not** have access to the script's arguments or I/O: all you can do in them is define hook functions.

Finally, please remember that you shouldn't have *any* code in an `mdsh` block other than hook function definitions!  `mdsh` blocks are *not* part of the translated script, they are part of the *translation process*.  So any functions you define in them won't be around when the script actually runs, and any changes you make to variables won't be still around when the actual script execution happens.

### Advanced Block Compilation Techniques

Once you've gotten used to doing some `mdsh-lang-X` functions, why not try your hand at some `mdsh-compile` ones?

For example, in the [`jqmd` project](https://github.com/bashup/jqmd), I originally had some code that looked like this:

```bash
YAML() { JSON "$(echo "$1" | yaml2json -)"; }

mdsh-lang-yaml() { YAML "$(cat)"; }
```

Which works pretty well, except, since the YAML is a constant value, why not convert it to JSON during compilation?  That way, we could eliminate the runtime overhead (if we save and rerun the compiled script):

```bash
mdsh-compile-yaml() { printf 'JSON %q\n' "$(echo "$1" | yaml2json)"; }
```

Notice the difference between the two functions: the `lang` function is a code *template*.  `mdsh` copies its body into your script source, resulting in code that looks like:

```bash
{
    YAML "$(cat)"
} <<'```'
... yaml data here ...
​```
```

But the `compile` function simply runs `yaml2json` immediately, and then writes out the translated data, like so:

```bash
JSON ...shell-quoted json here...
```

Notice the use of `printf` with `%q` -- this results in the data being properly escaped to work as a command line argument.  (Take care when you do direct code generation to escape such values properly.  When you need to insert variable data into generated code, always use `printf` with a constant string format, with `%q` placeholders for any standalone arguments.)

Notice too, by the way, that `compile` functions get access to the actual block text, which means that you can do any sort of code generation you like.  For example, I could have taken the output of `yaml2json`, and run `jq` over it, then looped over the output and written bash code to set variables based on the result, or generated code for subcommands based on the specification, or maybe even generated an argument parser from it.  There are all sorts of interesting possibilities for these kinds of code generation techniques!

### Command Blocks and Arguments

Sometimes you have only one block that needs to be processed in a particular way, or each block of a particular language needs unique arguments to compile or execute.  For these scenarios, you can define "command blocks".

A command block is a code block whose language tag's *second word* begins with a `|` or `!`:

* If it's a `|`, the remainder of the language tag is executed at **run** time with the block's contents on standard input (just like an `mdsh-lang-X` function body)
* If it's a `!`, the remainder of the language tag is executed at **compile** time with the block's contents in `$1`, and must output compiled code to standard output (just like an `mdsh-compile-X` function).

(In either case, the word *before* the `|` or `!` is ignored and discarded: it's assumed to be just a syntax highlighting hint.)

So this code as input to mdsh:

~~~markdown
​```json !printf "echo %q\n" "def example: $1;"
{"foo": "bar"}
​```

​```python |python
print("hello world!")
​```
~~~

would compile to the following output:

```bash
echo $'def example: {"foo": "bar"}\n;'
python <<<'```'
print("hello world!")
​```
```

Notice that both forms of command blocks can contain arbitrary bash code, including pipes, substitutions, etc.  Command blocks also override normal language function lookups, so no  `mdsh-after-X` , `mdsh-lang-X`, or `mdsh-compile-X` functions are looked up or executed for command blocks.

### Literate Testing

Documents created with mdsh can be tested using the [cram](https://bitheap.org/cram/) functional testing tool.  Just set cram's indent level to 4 and use 4-space indented blocks for your cram-tested examples, optionally wrapped in `~~~` fenced code blocks like this:

~~~~shell
    $ echo "hello world!"
    hello world!
~~~~

Cram looks for `$` or `>` and a space, indented to a certain level, then runs the command(s) and verifies the output.  So cram needs to know what indentation you're using.

mdsh ignores 4-space indented and `~~~` fenced blocks, so it won't be confused by your examples.  You can get github to syntax-highlight your examples by including a language tag like `~~~bash` or `~~~shell`.

Explaining all the ins and outs of using cram is beyond the scope of this guide, but in the simplest case, using `cram --indent 4 mydocument.md` will run any 4-space indented examples in `mydocument.md`.

(Note that `cram` does not actually understand markdown, so it will try to run anything that begins with `"$ "` or `"> "` at the specified indent.  Non-example code that starts that way can typically be outdented slightly or indented further so that cram will ignore it.)

### Excluding Blocks From The Generated Script

If your script has a lot of documentation examples that contain fenced code blocks, you may want to exclude these from being processed or copied to bash variables.  There are two main ways you can do this.

First, you can change the way you indicate certain code blocks.  All of these are currently ignored by `mdsh` and do not generate any code:

* Code blocks indented with four spaces, instead of fenced
* Code blocks fenced with `~~~X` instead of `` ```X ``
* Code blocks fenced with more than three backquotes, or which are indented
* Code blocks with no language tag

Alternately, you can define empty `mdsh-compile-X` functions in an mdsh block, for each language you want to exclude from the compilation, or define an `mdsh-misc` function that does nothing.  (Which will disable data blocks entirely; see the section on [Metaprogramming and Code Generation](#metaprogramming-and-code-generation) below for more info on `mdsh-misc`.)

### Making Executable (and Editable) Markdown Files

If you want to make a markdown file directly executable, you can `chmod +x` it and give it a shebang line such as `#!/usr/bin/env mdsh`.  This will then let you run `somescript.md args..` without needing to type `mdsh` in front of it.

If you want to get rid of the `.md` extension on your script, you'll probably want to also add a line to tell Github, your editor, etc. that the file is still markdown, e.g.:

```sh
#!/usr/bin/env mdsh
<!-- ex: set syntax=markdown : -->
```

This will tell Github, atom (with the `vim-modeline` package), and other editor/display tools that the file is actually Markdown.

(Alternately, you can keep the `.md` on the file for editing, but use an extensionless symlink to run it without needing to type the `.md`.)

### Making Sourceable Scripts (and handling $0)

It's often good practice to write scripts in such a way that the functions they define can be used by other scripts, usually by `source`-ing them.  This leads to the common pattern of writing code like this at the end of bash scripts, to only run the script's main function if the script was *not* sourced:

```bash
if [[ $0 == $BASH_SOURCE ]]; then
    # Not `source`d: run as script
    my-main "$@"
    exit $?
fi
```

This practice is no different with `mdsh`: whether you execute your script with an `mdsh` `#!` line, or compile it and run it, the variables `$0` and `$BASH_SOURCE` will only be equal if your program was run as a script.

Unfortunately, when `mdsh` is run via the `#!` line, the only way to ensure these values are equal is if they're both *empty*.  That means `$0` will be an empty string, which may interfere with common practices like using it in "usage" messages to denote the program name.

As a workaround, `mdsh` defines an `$MDSH_ZERO` argument for you, if and only if your script was invoked directly.  It contains the value that `$0` and `$BASH_SOURCE`*would* have had, if your script had been compiled.  You can thus use `${0:-$MDSH_ZERO}` to portably retrieve the invoking script name, regardless of whether your script was compiled or interpreted.

But that still doesn't make your script *sourceable*.  It's markdown, not bash, after all.  So if another script tries to `source` it, it'll get all sorts of syntax and other errors.

To fix that, we need a *shelldown* header: two lines of code that execute in bash, but are hidden in most markdown renderings, and still tell our editors (and Github) to ignore the `#!` line and treat the file as markdown, not bash:

~~~markdown
#!/usr/bin/env bash
: '
<!-- ex: set ft=markdown : '; eval "$(mdsh --eval "$BASH_SOURCE")" # -->

# My Awesome Script

...code goes here...
~~~

Now, our script is in `bash` syntax, for just long enough to compile the document and run the result.  Compiling the script with  `mdsh --eval` makes a version that's safe to `eval` and `source`, by adding an extra line to the end of the code that does a `return $?` or `exit $?`, depending on whether the script was sourced.  (This ensures that bash stops processing the file during the `eval`, *before* it can get confused by the markdown content that follows.)

As a result, this file can be either run or `source`-d without any issues.  What's more, you can `mdsh --compile` it to create a plain bash script (that's still runnable and `source`-able), without needing any code changes.

That's because when you directly run or source the script, you're really executing the *same* code that would be compiled, in the same environment, with `$BASH_SOURCE` pointing to the actual file.  (And `$0` matching it, unless it's being sourced.)

So why not use this method all the time?  Well, you certainly *can*.  And if you don't mind copying and pasting it into all your new scripts, then by all means go ahead!  However, an `mdsh` `#!` line is *definitely* the easier choice for one-off scripts that aren't being sourced, where you aren't using the value of `$0`, and you don't care about editor support.

### Syntax Highlighting of `mdsh` blocks

Because `mdsh` is not a widely-recognized language, Github and other markdown editing/processing tools generally don't know how to highlight them properly.  As a workaround, you can give a block tag of `shell mdsh` , which most tools will then interpret as a shell block for highlighting purposes.  e.g.:

~~~markdown
​```shell mdsh
echo 'echo "Most tools will highlight this block as shell script"'
​```
~~~

## Metaprogramming and Code Generation

Any output from a `mdsh`-tagged block becomes part of the generated bash script at the point where the block occurs.  This means that you can simply `cat` other bash files to include them (or use `mdsh-embed`; see the next section), or do anything else you like to generate code there.  This can be a useful alternative to using `source` to load functions, as it means that the resulting script can be `--compile`d to a single file that doesn't need the other modules present.

If you want to programmatically process individual blocks in some fashion (for example, to extract filenames from their language tags), you can define an `mdsh-misc` function.  For each block without an `mdsh-lang-X ` or `mdsh-compile-X` function, `mdsh-misc` is called with the language tag and block contents as arguments, and its output is appended to the compiled script at that point in the file.

So for example, when run, this script outputs the contents of the text block to `file1.txt`:

~~~markdown
​```mdsh
mdsh-misc() {
    if [[ $1 == *'>'* ]]; then echo -n "$2" >"${1#*>}"; fi
}
​```

​```text >file1.txt
Some text goes here!
​```
~~~

Of course, the possible applications of `mdsh-misc` are considerably more varied than just writing blocks to files.  You could, for example:

* Emulate the "tangling" features of other literate programming tools (by having `mdsh-misc` save the contents of blocks to different variables based on their tag information, and then ending your program with an `mdsh` block that outputs the saved blocks in the desired order)
* Interpret tags as arguments for how a block should be processed
* Treat blocks tags starting with `|` as a pipeline to preprocess the block with

...and just about anything else you can imagine.

### "Static Linking" for Distribution

You can use the `mdsh-embed` function to embed the source of other modules into your script.  Calling `mdsh-embed` *modulename* inside an `mdsh` block will search `PATH` for *modulename* (unless *modulename* contains a `/`), and then output its source code, wrapped in a heredoc and `source` command.  (This ensures that the embedded module will know it was sourced, and not via the command line, even if the embedding script *was* run from the command line.)

The net result is that by using `mdsh-embed` in your `mdsh` block(s) to load your modules (instead of `source` inside your `shell` blocks), you gain the ability to `--compile` your script to a "statically-linked executable".  That is, you can create a single file that contains all the modules it needs, so your users don't have to install all your dependencies themselves, and don't need a specific package manager to install your script.

## Extending `mdsh` or Reusing its Functions

Sourcing `mdsh` from a bash script will define all its functions, but not actually run a program.  This allows you to change how command line arguments are processed, or predefine additional language hooks, teardown hooks, etc.   (You can also just do it to make use of the included markdown processing functions.)

(Note that sourcing `mdsh` will set bash to  ["unofficial strict mode"](http://redsymbol.net/articles/unofficial-bash-strict-mode/) , i.e. `-euo pipefail`.  `mdsh` is written with the assumption that these settings are in effect, so changing them may have undesirable results.)

If what you're writing is just "mdsh with more languages", you can do so like this:

```bash
#!/usr/bin/env bash
source "$(command -v mdsh)"

mdsh-compile-somelang() {
    # etc.
}

# ...

[[ $0 == $BASH_SOURCE ]] && mdsh-main "$@"

```

That is, just source mdsh and define your additional language handlers, then run `mdsh-main "$@"`: your script will then have the same command-line interface as `mdsh`, but all its help messages will refer to the name of your script, instead of `mdsh`.

### Adding File Headers or Footers

If your extended version of mdsh needs to add headers or footers to generated files, you can define functions named `mdsh:file-header` and/or `mdsh:file-footer`.  The `--compile` option will call these functions once at the start and end of the compilation process, wrapping the entire output.  `--eval` will works similarly, except that the `--eval` footer will appear *after* `mdsh:file-footer`.

### Altering Existing Functions

In some cases, you may wish to also alter parts of mdsh's behavior, by replacing some of its functions.  You can, however, avoid the need to copy those function into your code by using `mdsh-rewrite`.  `mdsh-rewrite` is a function normally used in the mdsh compiler to rewrite `mdsh-lang-X` and `mdsh-after-X` function bodies, but you can adapt it to do AOP-like editing of bash functions.

For example, [jqmd](https://github.com/bashup/jqmd) adds a header and footer to files compiled with `jqmd --compile` and `jqmd --eval`, by adding a function call to the start and end of the `mdsh.--compile` function:

```bash
eval "mdsh.--compile() $(mdsh-rewrite mdsh.--compile '{ jqmd-header;' 'jqmd-footer; }')"
```

(`mdsh-rewrite` takes a function name and two optional strings that will replace the opening and closing brace lines of the function body.  The result is output to stdout, where it becomes the body of the new function.)

### Available Functions

The following functions are available for your use or alteration in scripts sourcing `mdsh`:

* `run-markdown` *mdfile args...* -- execute the specified markdown file with *args* as its positional arguments (`$1`,  `$2`, etc.)  Use this instead of `mdsh-main` if you just want to interepret some markdown and maybe pass it some arguments: it's really just shorthand for `source <(mdsh-compile mdfile) args...`.
* `mdsh-error` *format args...* --`printf` *format args* to stderr and terminate the process with errorlevel 64 ([EX_USAGE](https://www.freebsd.org/cgi/man.cgi?query=sysexits&sektion=3#DESCRIPTION)) .  (A linefeed is added to the format string automatically.)
* `mdsh-compile` -- accepts markdown on stdin and outputs bash code on stdout.  The compilation takes place in a subshell, so hook functions defined in the code being compiled do **not** affect the caller's environment.  Hook functions *already* defined in the caller's environment, however, will be used to translate blocks of the relevant languages.
* `mdsh-embed` *modulename* -- look for *modulename* on `PATH` (unless it contains a `/`), and dump its contents wrapped in a `source` command and heredoc.  Returns failure if *modulename* isn't found or can't be read.  (Note: unlike Bash's `source` command, this function does **not** fall back to looking for the module in the current directory.  If you want a file in the current directory, use `./modulename`).
* `mdsh-rewrite` *function* *before* *after* -- output the body block of *function* on stdout, optionally replacing the opening and closing braces with *before* and *after*.  (If you're using this to "edit" a function, remember that the replacements must include the opening and closing braces, and the closing brace must be preceded by either a newline or a semicolon or space.)
* `markdown-to-shell` *command language_regexes...* -- **DEPRECATED**: in earlier versions of  `mdsh`, this function was the "compiler", and then a prepropcessor for the compiler, but now it's no longer used, and will be removed in a future version.

