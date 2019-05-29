# Filenames and Pathnames in Shell: How to do it Correctly

> David A. Wheeler
>
> 2016-05-04 (original version 2010-05-19)

## How to do it wrongly

First, let’s go through some examples that are wrong, because the first step to fixing things is to know what’s broken. These examples assume default settings (e.g., there is no “`set -f`” or “`IFS=...`”):

---

```sh
cat * > ../collection  # WRONG
```

This is wrong. If a filename in the current directory begins with “`-`”, it will be misinterpreted as an option instead of as a filename. For example, if there’s a file named “`-n`”, it will suddenly enable cat’s “`-n`” option instead if it has one (GNU cat does, it numbers the lines). In general you should never have a glob that begins with “`*`” — it should be prefixed with “`./`”. Also, if there are no (unhidden) files in the directory, the glob pattern will return the pattern instead (“`*`”); that means that the command (`cat`) will try to open a file with the improbable name “`*`”.

---

```sh
for file in * ; do  # WRONG
  cat "$file" >> ../collection
done
```

Also wrong, for the same reason; a file named “`-n`” will fool the cat program, and if the pattern does not match, it will loop once with the pattern itself as the value.

---

```sh
cat $(find . -type f) > ../collection  # WRONG
```

Wrong. If any pathname contains a space, newline, or tab, its name will be split (file “`a b`” will be incorrectly parsed as two files, “`a`” and “`b`”). If a pathname contains a globbing character like `*`, the shell will try to expand it, potentially creating additional problems. Also, if the find command matches no files, the command will be run with no parameters; on many commands (like `cat`) this will cause the program to hang on input from standard input (you can fix this by appending pathname `/dev/null`, but many people do not know to do that).

---

```sh
( for file in $(find . -type f) ; do  # WRONG
    cat "$file"
  done ) > ../collection
```

Wrong, for similar reasons. This breaks up pathnames that contain space, newline, or tab, and it incorrectly expands pathnames if the pathnames themselves contain characters like “`*`”.

---

```sh
( find . -type f |   # WRONG
  while read file ; do cat "$file" ; done ) > ../collection
```

Wrong. This works if a pathname has spaces in the middle, but it won’t work correctly if the pathname begins or ends with whitespace (they will get chopped off). Also, if a pathname includes “`\`”, it’ll get corrupted; in particular, if it ends in “`\`”, it will be combined with the next pathname (trashing both). In general, using “`read`” in shell without the “`-r`” option is usually a mistake, and in many cases you should set `IFS=""` just before the read.

---

```sh
( find . -type f | xargs cat ) > ../collection # WRONG
```

Wrong. By default, xargs’ input is parsed, so space characters (as well as newlines) separate arguments, and the backslash, apostrophe, double-quote, and ampersand characters are used for quoting. According to the [POSIX standard], you have to include the option `-E ""` or underscore may have a special meaning too. Note that many of the examples in the [POSIX standard] xargs section are wrong; pathnames with spaces, newlines, or many other characters will cause many of the examples to fail.

---

```sh
( find . -type f |
  while IFS="" read -r file ; do cat "$file" ; done ) \
        > ../collection # WRONG
```

Wrong. Like many programs, this assumes that you can have list of pathnames, with one pathname per line. But since pathnames can internally include newline, all simple line-at-a-time processing of pathnames is wrong! This construct is fine if pathnames can’t include newline, but since many Unix-like systems permit, attackers are happy to use this false assumption as an attack.

---

```sh
cat $file
```

Wrong. If `$file` can contain whitespace, then it could broken up and interpreted as multiple file names, and if `$file` starts with dash, then the name will be interpreted as an option. Also, if `$file` contains metacharacters like “`*`” they will be expanded first, producing the wrong set of filenames.

## Doing it correctly: A quick summary

So, how can you process pathnames correctly in shell? Here’s a quick summary about how to do it correctly, for the impatient who “just want the answer”.

### Basic rules

1. [Double-quote all variable references and command substitutions](https://dwheeler.com/essays/filenames-in-shell.html#doublequote) unless you are certain they can only contain alphanumeric characters or you have specially prepared things (i.e., use `"$variable"` instead of `$variable`). In particular, you should practically always put `$@` inside double-quotes; POSIX defines this to be special (it expands into the positional parameters as separate fields even though it is inside double-quotes).
2. [Set IFS to just newline and tab](https://dwheeler.com/essays/filenames-in-shell.html#ifs), if you can, to reduce the risk of mishandling filenames with spaces. Use newline or tab to separate options stored in a single variable. Set `IFS` with `IFS="$(printf '\n\t')"`
3. [Prefix all pathname globs so they cannot expand to begin with “-”](https://dwheeler.com/essays/filenames-in-shell.html#prefixglobs). In particular, never start a glob with “`?`” or “`*`” (such as “`*.pdf`”); always prepend globs with something (like “`./`”) that cannot expand to a dash. So never use a pattern like “`*.pdf`”; use “`./*.pdf`” instead.
4. [Check if a pathname begins with “-” when accepting pathnames](https://dwheeler.com/essays/filenames-in-shell.html#checkdash), and then prepend “`./`” if it does.
5. [Be careful about displaying or storing pathnames](https://dwheeler.com/essays/filenames-in-shell.html#display-store), since they can include newlines, tabs, terminal control escape sequences, non-UTF-8 characters (or characters not in your locale), and so on. You can strip out control characters and non-UTF-8 characters before display using `printf '%s' "$file" | LC_ALL=POSIX tr -d '[:cntrl:]' | iconv -cs -f UTF-8 -t UTF-8`
6. [Do not depend on always using “--”](https://dwheeler.com/essays/filenames-in-shell.html#dashdash) between options and pathnames as the primary countermeasure against filenames beginning with “`-`”. You have to do it with every command for this to work, but people will not use it consistently (they never have), and many programs (including echo) do not support “`--`”. Feel free to use “`--`” between options and pathnames, but only as an additional optional protective measure.
7. Use a template that is known to work correctly; below are some [tested](https://dwheeler.com/encodef/evil-filenames-test) templates.
8. Use a tool like [shellcheck] to find problems you missed.

### Template: [Using globs]

```sh
# Correct portable glob use: use "for" loop, prefix glob, check for existence:
# (remember that globs normally do NOT include files beginning with "."):
for file in ./* ; do        # Prefix with "./*", NEVER begin with bare "*"
  if [ -e "$file" ] ; then  # Make sure it isn't an empty match
    COMMAND ... "$file" ...
  fi
done
```

```sh
# Correct portable glob use, including hidden files (beginning with "."):
for file in ./* ./.[!.]* ./..?* ; do        # Prefix with "./*"
  if [ -e "$file" ] ; then  # Make sure it isn't an empty match
    COMMAND ... "$file" ...
  fi
done
```

```sh
# Correct glob use, simpler but requires nonstandard bash extension nullglob:
shopt -s nullglob  # Bash extension, so globs with no matches return empty
for file in ./* ; do        # Use "./*", NEVER bare "*"
  COMMAND ... "$file" ...
done
```

```sh
# Correct glob use, simpler but requires nonstandard bash extension nullglob;
# you can do things on one line if you can add /dev/null as an input.
shopt -s nullglob  # Bash extension, so globs with no matches return empty
COMMAND ... ./* /dev/null
```

### Template: [Using find]

The find command is great for recursively processing directories. Typically you would specify other parameters to find (e.g., select only normal files using “`-type f`”). For example, here's an example of using find to walk the filesystem, skipping all "hidden" directories and files (names beginning with "`.`") and processing only files ending in `.c` or `.h`:

```sh
find . \( -path '*/.*' -prune -o ! -name '.*' \) -a -name '*.[ch]'
```

Below are the forms that always work (though some require nonstandard extensions or fail with Cygwin), followed by simpler ones with serious limitations.

#### Always works

```sh
# Simple find -exec; unwieldy if COMMAND is large, and creates 1 process/file:
find . -exec COMMAND... {} \;
```

```sh
# Simple find -exec with +, faster if multiple files are okay for COMMAND:
find . -exec COMMAND... {} \+
```

```sh
# Use find and xargs with \0 separators
# (nonstandard common extensions -print0 and -0. Works on GNU, *BSDs, busybox)
find . -print0 | xargs -0 COMMAND
```

```sh
# Head-busting, but it works portably.  Use '\'' for single-quote in command.
# Runs a subshell, so variable values are lost after each iteration:
find . -exec sh -c '
for file do
    ...  # Use "$file" not $file
done' sh {} +
```

```sh
# find... while loop, requires find (-print0) and shell (read -d) extensions.
# Fails on Cygwin; in while loops, filenames ending in \r \n and \n look =.
# Variable values may be lost unset because loop may run in a subshell.
find . -print0 | while IFS="" read -r -d "" file ; do ...
  COMMAND "$file" # Use quoted "$file", not $file, everywhere.
done
```

```sh
# while + find with process substitution.
# Requires nonstandard read -d (bash ok) and find -print0.
# Rquires nonstandard process redirection <(...); bash, zsh, and ksh 93
# have process redirection, but dash and ksh 88 do not. Also,
# the underlying system must have named pipes (FIFOs) or /dev/fd.
# Fails on Cygwin; in while loops, filenames ending in \r \n and \n look =.
# Variables *do* retain their value after the loop ends, and
# you can read from stdin (change the 4s to another number if fd 4 is needed):
while IFS="" read -r -d "" file <&4 ; do
  COMMAND "$file" # Use quoted "$file", not $file, everywhere.
done 4< <(find . -print0)
```

```sh
# Named pipe version.
# Requires nonstandard extensions to find (-print0) and read -d (bash ok);
# underlying system must inc. named pipes (FIFOs).
# Fails on Cygwin; in while loops, filenames ending in \r \n and \n look =.
# Variables *do* retain their value after the loop ends, and
# you can read from stdin (change the 4s to something else if fd 4 needed).
mkfifo mypipe
find . -print0 > mypipe &
while IFS="" read -r -d "" file <&4 ; do
  COMMAND "$file" # Use quoted "$file", not $file, everywhere.
done 4< mypipe
```

```sh
# Use the author's encodef program.
# Variables *do* retain their value after the loop ends.
# This version is POSIX portable, but you must have encodef. In practice
# you can often use "-print0" instead of POSIX "-exec printf '%s\0' {} \;"
for encoded_pathname in $(find . -exec printf '%s\0' {} \; | encodef ) ; do
  file="$(encodef -dY -- "$encoded_pathname")" ; file="${file%Y}"
  COMMAND "$file" # Use quoted "$file", not $file, everywhere.
done
```

---

[POSIX standard]: http://www.opengroup.org/onlinepubs/009695399/
[shellcheck]: http://www.shellcheck.net/
[Using globs]: https://dwheeler.com/essays/filenames-in-shell.html#globbing
[Using find]: https://dwheeler.com/essays/filenames-in-shell.html#find
