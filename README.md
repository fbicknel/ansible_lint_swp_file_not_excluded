# Ansible Collection - tests.swp_file_not_excluded

This is an MVE (Minimum Viable Example) to demonstrate a possible bug
in ansible-lint.

## TL;DR (or Exectutive Summary, if you prefer)

One would expect that a file specified in `.gitignore` should be
ignored. Some are, but only if they are in the project root.
Files not in the project root are _not_ ignored.

This presents a corner case that demonstrates the danger of not honoring
the `.gitignore` directives correctly.

We also present a workaround for this particular corner case.

## Background

We're picking on vim here because it can demonstrate a file for which
`ansible-lint` produces an error. We have some control over these files
as we shall soon see. The point of our adventure today is just to prove
that files that match patterns in `.gitignore` are not being ignored.

When using the vim editor, files named `.*.swp` are created. These files
are used by the editor internally. They are temporal; they disappear
when the editor exits. Vim calls these _swapfiles_.

The files are hidden (start with a `.` character) and generally end in
`.swp`, though it is possible to have other extentions such as `.swo`,
etc. For simplicity, we will just conceern ourselves with `.*.swp`,
as that is the most common that I have seen.

The default location for swapfiles is the same directory as the file being
edited. As is common with vim, this can be changed. This ability will
form the basis for our workaround for this problem. For now, however,
if you would like to demonstrate the issue, be sure vim has the default
setting for `swapfile`:

    :set swapfile?

This should return simply `swapfile`.

    :set dir?

If you haven't altered it, you should see something like
`directory=.,~/tmp,/var/tmp,/tmp`. The important part here is that the
first entry is `.`, which tells vim to use the current directory. The
workaround will change this, but we'll get to that later. If you don't
see these results, then use this:

    :set swapfile
    :set dir=.

This will allow you to replicate the situation using the files in
this MVE.

## Impact with ansible-lint

When `ansible-lint` runs, there are provisions to ignore certain
files. Notably, files mentioned in `.gitignore` should be ignored. This
appears to be the case when `ansible-lint` runs as demonstrated by
this output:

    INFO     Loading ignores from .gitignore
    INFO     Excluded: .README.md.swp
    INFO     Excluded: .git
    INFO     Excluded: .gitignore
    INFO     Loading ignores from .gitignore
    INFO     Excluded: .README.md.swp
    INFO     Excluded: .git
    INFO     Excluded: .gitignore

This is the model `.gitignore` in this example:

    .*
    **/.*

You can already see the `.swp` files being ignored. I don't know why
the output is repeated, but this should not impact the issue at hand.

## Demonstrating the Problem

I stumbled upon a file that begins with:

    {{ ansible_managed | comment }}

I didn't look up what vim was doing in its swapfile, but if the file
is named <sup>`*`</sup>`roles/another_role/templates/yaap.txt.j2` (I know, corner
case), then when you run `ansible-lint -v`, you get:

    INFO     Loading ignores from .gitignore
    INFO     Excluded: .git
    INFO     Excluded: .gitignore
    INFO     Loading ignores from .gitignore
    INFO     Excluded: .git
    INFO     Excluded: .gitignore

Note that I restarted my editor session, so the previous
`.README.md.swp` files have disappeared. However, I am editing a file
`roles/another_role/templates/yaap.txt.j2`. I should see its `.*.swp`
file excluded in the output above. The file _is_ there, but the exclusion
is missing from the output above.

But most agregious, this new error appears:

    WARNING  Listing 1 violation(s) that are fatal
    load-failure[unicodedecodeerror]: 'utf-8' codec can't decode byte 0xda in position 17: invalid continuation byte
    roles/another_role/templates/.yaap.txt.j2.swp:1


                       Rule Violation Summary
     count tag                              profile rule associated tags
     1 load-failure[unicodedecodeerror] min     core, unskippable

    Failed: 1 failure(s), 0 warning(s) on 11 files.

### What vim is up to

As I mentioned, I didn't look up the purpose of every byte of the
vim swapfile header. I'm pretty sure we should not be linting such
a file and that if the file were ignored, we wouldn't be having this
conversation. However, the error seems to be pointing to a byte with value
`0xda` at position 17. Let's look there:

    $ od -A x -t x1z -v  roles/another_role/templates/.yaap.txt.j2.swp |head -2
    000000 62 30 56 49 4d 20 38 2e 30 00 00 00 00 10 00 00  >b0VIM 8.0.......<
    000010 1e da 37 65 f5 fe 00 03 fa 4e 03 00 6a 62 69 63  >..7e.....N..jbic<

There it is, `0xda` at position 17. 

Turns out, another file that starts out with

    {{ ansible_managed | comment }}

... but named something else, does not turn up the error. Vim also sees
fit not to put the offending `0xda` at position 17:

    $ od -A x -t x1z -v  roles/another_role/files/.test_file.txt.swp |head -2
    000000 62 30 56 49 4d 20 38 2e 30 00 00 00 00 10 00 00  >b0VIM 8.0.......<
    000010 bf e1 37 65 36 05 18 00 fa 4e 03 00 6a 62 69 63  >..7e6....N..jbic<

Again, this is vim's swapfile and it can do what it wants with it, but it
was at least interesting to see what `ansible-lint` has found offensive.

Ultimately, we just need these files to be ignored.

### The Workaround

As with most things, there's a workaround.

Vim can write its swapfile to another location and we solve this
problem. Before opening files, use this to set your swapfile directory:

    :set dir=~/tmp

You can also set it to `/tmp`, if you don't want to create a directory
under your home directory.

This completely solves this problem. Put this in your `~/.vimrc` file
and you'll never have to think about it.

Of course, it does not solve the problem of ignoring files. If you put
something in `.gitignore`, and if `ansible-lint` is _supposed_ to ignore
those files, then expectations are not met.

And as proven by this example, when expectations are not ment, new
problems to solve crop up. Fortunately, we had an acceptable workaround
in this case.

<sup>`*`</sup>The name of the file seemed to be important. However,
being able to create such a file is not important. As I have stressed
throughout, it's not that we can create a file that causes this error that
should be the focus. It's that this file gets linted at all. If the file
exclusion patterns were working, we wouldn't see this phenomenon at all.
