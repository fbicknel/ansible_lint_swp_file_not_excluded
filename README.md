# Ansible Collection - tests.swp_file_not_excluded

This is an MVE (Minimum Viable Example) to demonstrate a possible bug
in ansible-lint.

## Background

When using the vim editor, files named `.*.swp` are created. These files are used by the editor internally. They are temporal; they disappear when the editor exits.

The files are hidden (start with a `.` character) and generally end in `.swp`, though it is possible to have other extentions such as `.swo`, etc. For simplicity, we will just conceern ourselves with `.*.swp`, as that is the most common that I have seen.

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

If you edit a file that starts with this as the first line:

    {{ ansible_managed | comment }}

Indeed, if you edit a file with this as the only line, then re-run
`ansible-lint -v`, you see this in the output:

    INFO     Loading ignores from .gitignore
    INFO     Excluded: .git
    INFO     Excluded: .gitignore
    INFO     Loading ignores from .gitignore
    INFO     Excluded: .git
    INFO     Excluded: .gitignore

Note that I restarted my editor session, so the previous
`.README.md.swp` files have disappeared. However, I am editing a file
`roles/another_role/templates/yaap.txt.j2`. I should see its `.*.swp`
file excluded in the output above.

And furthermore, this new error appears:

    WARNING  Listing 1 violation(s) that are fatal
    load-failure[unicodedecodeerror]: 'utf-8' codec can't decode byte 0xda in position 17: invalid continuation byte
    roles/another_role/templates/.yaap.txt.j2.swp:1


                       Rule Violation Summary
     count tag                              profile rule associated tags
     1 load-failure[unicodedecodeerror] min     core, unskippable

    Failed: 1 failure(s), 0 warning(s) on 11 files.

# Summary

One would expect that a file specified in `.gitignore` should be
ignored. As it is, files not in the project root are _not_ ignored. This
forces one to close the editor before running `ansible-lint`.
