# Ruby Internals: Exec

Today I was curious to learn how Ruby implements the `exec` command to
evaluate code in the shell. It didn't take me mouch to find the
definition on `process.c`. Here's the interesting part (which doesn't
apply for windows):

```c
rb_proc_exec_e(const char *str, VALUE envp_str)
{
  while (*str == ' ' || *str == '\t' || *str == '\n')
    str++;

  if (!*str) {
      errno = ENOENT;
      return -1;
  }

  before_exec();
  if (envp_str)
      execle("/bin/sh", "sh", "-c", str, (char *)NULL, (char **)RSTRING_PTR(envp_str));
  else
      execl("/bin/sh", "sh", "-c", str, (char *)NULL);
  preserving_errno(after_exec());

  return -1;
}
```

This function accepts a string containing the shell command and a
process environment variable, which may be null.

The first part of the function starts with sanitizing user input by
removing whitespaces, tabs and newlines from the beginning of the
string, and it then returns -1 if the string is empty.

Before and after exec are routines which help with threading, but the
interesting part is how Ruby uses `execl[e]` in a clever way to execute
commands without having to parse arguments directly. This is helpful
because both `execl` and `execle` accept a variable number of arguments
which are passed as arguments to the program you're running (the first
parameter). So if you wanted to execute `ls -a` in C, you probably do
the following:

```c
execl("/bin/sh", "ls", "-", (char *)NULL);
```

Where `(char *)NULL` tells `execl` that the arguments are finished.
However, parsing user input this way in C is tedious, so Ruby uses the
`-c` flag to tell bash to interpret the next parameter as the command
itself.

To summarize, when you do `exec "ls -a"` in Ruby what you're really doing
is:

```bash
/bin/sh -c "ls -a"
```

As usual, it pays to know how your tools work (in this case bash). This
tip could have definitely saved me some time when programming in C and
Objective-C.

