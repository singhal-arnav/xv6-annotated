# User Space: The Shell

Welcome to the final piece of the puzzle! We've traced through the entire boot sequence, seen how the kernel initializes and creates the first process, watched as initcode calls exec("/init"), and seen how init sets up file descriptors and starts the shell. Now it's time to dive into the shell itself -- the program that lets users interact with xv6.

The shell might seem like magic at first: you type commands, it runs programs, you can pipe the output of one program into another, redirect output to files, and chain multiple commands together. But as we'll see, it's all built on the system calls we've already covered. The shell is just a user program that happens to provide a convenient interface to the rest of the system!

This post will walk through the entire xv6 shell (sh.c), explaining how it parses commands, handles pipes and redirection, and generally makes Unix awesome. Let's dive in!

## The Big Picture

The shell is essentially a loop that does three things over and over:

1. **Read** a command line from the user (from stdin).
2. **Parse** the command line into a data structure representing what the user wants.
3. **Execute** the command by forking, setting up I/O, and calling exec.

That's it! Everything else is just details. But oh, what beautiful details they are!

The xv6 shell is about 500 lines of C, and it supports:
- Running programs with arguments: `ls -l`
- I/O redirection: `echo hello > file.txt` and `wc < file.txt`
- Pipes: `ls | grep foo`
- Sequential commands: `echo hello; echo world`
- Background jobs: `sleep 100 &`

It doesn't support all the fancy features of Bash or Zsh (no job control, no command history, no tab completion, no scripting features), but it's got all the essential Unix shell functionality.

## Main Loop (main)

Let's start at the top with the `main()` function:

```c
int
main(void)
{
  static char buf[100];
  int fd;

  // Ensure that three file descriptors are open.
  while((fd = open("console", O_RDWR)) >= 0){
    if(fd >= 3){
      close(fd);
      break;
    }
  }

  // Read and run input commands.
  while(getcmd(buf, sizeof(buf)) >= 0){
    if(buf[0] == 'c' && buf[1] == 'd' && buf[2] == ' '){
      // Chdir must be called by the parent, not the child.
      buf[strlen(buf)-1] = 0;  // chop \n
      if(chdir(buf+3) < 0)
        printf(2, "cannot cd %s\n", buf+3);
      continue;
    }
    if(fork1() == 0)
      runcmd(parsecmd(buf));
    wait();
  }
  exit();
}
```

Let's break this down section by section:

### Ensuring File Descriptors are Open

```c
while((fd = open("console", O_RDWR)) >= 0){
  if(fd >= 3){
    close(fd);
    break;
  }
}
```

Wait, didn't init already set up file descriptors 0, 1, and 2 to point to the console? Yes! So why is the shell doing this again?

This is defensive programming. The shell might not always be started by init -- someone could run it manually after closing some file descriptors. This loop ensures that FDs 0, 1, and 2 are all open before the shell starts. It repeatedly opens the console until it gets an FD >= 3, which means FDs 0, 1, and 2 must already be open.

This is a classic Unix idiom: if you open a file, the kernel always gives you the lowest-numbered unused file descriptor. So if FD 0 were closed, `open()` would return 0. If FD 1 were closed, the next `open()` would return 1. And so on. Once we get FD 3, we know 0-2 are all taken, so we close FD 3 (we don't need it) and continue.

### The Main Command Loop

```c
while(getcmd(buf, sizeof(buf)) >= 0){
```

The shell enters an infinite loop, reading commands with `getcmd()`. This function (which we'll see in a moment) prints a prompt and reads a line from stdin. If it returns -1 (EOF), the loop exits and the shell terminates.

### Special Case: cd

```c
if(buf[0] == 'c' && buf[1] == 'd' && buf[2] == ' '){
  buf[strlen(buf)-1] = 0;  // chop \n
  if(chdir(buf+3) < 0)
    printf(2, "cannot cd %s\n", buf+3);
  continue;
}
```

The `cd` command is special-cased! Why? Because `cd` needs to change the *shell's* current directory, not a child process's directory. If we forked and called `chdir()` in the child, the child would change its directory and then exit, leaving the shell's directory unchanged.

So we check if the command starts with "cd " (note the space after cd). If so, we chop off the trailing newline, call `chdir()` directly in the shell process, and `continue` to the next iteration (skipping the fork/exec logic below).

This is kind of clunky -- we're just comparing the first three bytes to "cd ". A more robust implementation would properly parse the command, but this works for the simple case.

### Running Other Commands

```c
if(fork1() == 0)
  runcmd(parsecmd(buf));
wait();
```

For all other commands, we:
1. Fork a child process with `fork1()` (we'll see this helper soon).
2. In the child, call `parsecmd()` to parse the command line into a command structure.
3. Call `runcmd()` to execute the command structure. This never returns -- it calls exec!
4. In the parent, wait for the child to finish with `wait()`.

This is the fundamental Unix shell pattern: the shell forks, the child execs the command, and the parent waits. Simple and elegant!

## Reading a Command: getcmd()

Let's see how the shell reads input:

```c
int
getcmd(char *buf, int nbuf)
{
  printf(2, "$ ");
  memset(buf, 0, nbuf);
  gets(buf, nbuf);
  if(buf[0] == 0) // EOF
    return -1;
  return 0;
}
```

Super simple! Print a prompt ($ followed by a space), clear the buffer, read a line with `gets()` (from ulib.c), and return -1 if we got EOF (buf[0] == 0).

The `gets()` function reads characters from stdin until it sees a newline or EOF, storing them in the buffer. It's defined in the user library (ulib.c), not in the shell.

## Helper: fork1()

```c
void
panic(char *s)
{
  printf(2, "%s\n", s);
  exit();
}

int
fork1(void)
{
  int pid;

  pid = fork();
  if(pid == -1)
    panic("fork");
  return pid;
}
```

These are simple helper functions. `panic()` prints an error message and exits. `fork1()` calls `fork()` and panics if it fails (returning -1). This makes the code cleaner -- we can just call `fork1()` without checking for errors every time.

## Command Structures

Before we look at parsing and executing, we need to understand how commands are represented. The shell uses a tree structure where each node is a "command" with a type:

```c
struct cmd {
  int type;
};
```

Every command has at least a type. Then there are specific command types:

### Execute Command (EXEC)

```c
#define EXEC  1

struct execcmd {
  int type;              // EXEC
  char *argv[MAXARGS];   // arguments to the command
  char *eargv[MAXARGS];  // end pointers for argv strings
};
```

This represents a simple command like `ls -l /tmp`. The `argv` array contains pointers to the program name and arguments (just like `main(int argc, char *argv[])`). The `eargv` array contains pointers to the end of each argument string (used during parsing).

MAXARGS is 10, so you can have at most 10 arguments (including the program name).

### Redirection Command (REDIR)

```c
#define REDIR 2

struct redircmd {
  int type;          // REDIR
  struct cmd *cmd;   // the command to run
  char *file;        // the file to redirect to/from
  char *efile;       // end pointer for file string
  int mode;          // open mode (O_RDONLY, O_WRONLY|O_CREATE, etc.)
  int fd;            // file descriptor to redirect (0 for <, 1 for >)
};
```

This represents redirection like `ls > output.txt` or `wc < input.txt`. It wraps another command (the `cmd` field) and specifies which file to open and which FD to redirect.

For `<`, we open the file read-only and redirect FD 0 (stdin).
For `>`, we open the file write-only (creating it if needed) and redirect FD 1 (stdout).

### Pipe Command (PIPE)

```c
#define PIPE  3

struct pipecmd {
  int type;          // PIPE
  struct cmd *left;  // left side of pipe
  struct cmd *right; // right side of pipe
};
```

This represents a pipe like `ls | grep foo`. It has two sub-commands: `left` (the command before the |) and `right` (the command after the |).

### List Command (LIST)

```c
#define LIST  4

struct listcmd {
  int type;          // LIST
  struct cmd *left;  // first command
  struct cmd *right; // second command
};
```

This represents sequential commands like `echo hello; echo world`. Both commands are executed one after another (left first, then right).

### Background Command (BACK)

```c
#define BACK  5

struct backcmd {
  int type;          // BACK
  struct cmd *cmd;   // the command to run
};
```

This represents a background command like `sleep 100 &`. The command runs without waiting for it to finish.

## Constructors for Command Structures

The shell has constructor functions for each command type. They all follow the same pattern: allocate memory with `malloc()`, zero it out, set the type, fill in the fields, and return it:

```c
struct cmd*
execcmd(void)
{
  struct execcmd *cmd;

  cmd = malloc(sizeof(*cmd));
  memset(cmd, 0, sizeof(*cmd));
  cmd->type = EXEC;
  return (struct cmd*)cmd;
}
```

Notice the cast: `(struct cmd*)cmd`. All command structures start with the type field, so they can all be cast to `struct cmd*`. This is a common C pattern for polymorphism -- we use the type field to figure out the actual type, then cast to the specific struct.

The other constructors (`redircmd()`, `pipecmd()`, `listcmd()`, `backcmd()`) follow the same pattern but fill in their specific fields:

```c
struct cmd*
redircmd(struct cmd *subcmd, char *file, char *efile, int mode, int fd)
{
  struct redircmd *cmd;

  cmd = malloc(sizeof(*cmd));
  memset(cmd, 0, sizeof(*cmd));
  cmd->type = REDIR;
  cmd->cmd = subcmd;
  cmd->file = file;
  cmd->efile = efile;
  cmd->mode = mode;
  cmd->fd = fd;
  return (struct cmd*)cmd;
}
```

And so on. These are straightforward, so I won't list them all here.

## Parsing Commands

Now for the fun part: parsing! The shell uses a recursive descent parser to build the command tree. This is a classic parsing technique where each function handles one level of the grammar.

The grammar looks roughly like this (in informal notation):

```
line      = pipe [ ';' pipe | '&' pipe ]
pipe      = exec [ '|' exec ]
exec      = program args [ '<' file | '>' file ]
```

Each level of the grammar corresponds to a parsing function:
- `parseline()` handles the top level (sequences and background).
- `parsepipe()` handles pipes.
- `parseexec()` handles simple commands and redirections.

Before we dive into those, we need to understand tokenization.

### Tokenization: gettoken()

The parser needs to break the input string into tokens (words, symbols, etc.). That's what `gettoken()` does:

```c
char whitespace[] = " \t\r\n\v";
char symbols[] = "<|>&;()";

int
gettoken(char **ps, char *es, char **q, char **eq)
{
  char *s;
  int ret;

  s = *ps;
  while(s < es && strchr(whitespace, *s))
    s++;
  if(q)
    *q = s;
  ret = *s;
  switch(*s){
  case 0:
    break;
  case '|':
  case '(':
  case ')':
  case ';':
  case '&':
  case '<':
    s++;
    break;
  case '>':
    s++;
    if(*s == '>'){
      ret = '+';
      s++;
    }
    break;
  default:
    ret = 'a';
    while(s < es && !strchr(whitespace, *s) && !strchr(symbols, *s))
      s++;
    break;
  }
  if(eq)
    *eq = s;

  while(s < es && strchr(whitespace, *s))
    s++;
  *ps = s;
  return ret;
}
```

This function is dense, so let's unpack it:

**Parameters:**
- `ps`: A pointer to the current position in the input string. This gets updated.
- `es`: A pointer to the end of the input string.
- `q`: Output parameter for the start of the token.
- `eq`: Output parameter for the end of the token.

**Returns:** A character representing the token type:
- The actual character for symbols like `|`, `<`, `;`, etc.
- `'+'` for `>>`
 (append redirection).
- `'a'` for words (alphabetic/argument tokens).
- `0` for end of string.

**Algorithm:**
1. Skip leading whitespace.
2. Mark the start of the token in `*q`.
3. Figure out what kind of token we have based on the first character.
4. For symbols, just advance past the character (or two for `>>`).
5. For words, keep advancing until we hit whitespace or a symbol.
6. Mark the end of the token in `*eq`.
7. Skip trailing whitespace and update `*ps` to point to the next token.

This is a simple but effective tokenizer. It handles all the symbols the shell needs to recognize.

### Looking Ahead: peek()

Sometimes we need to look at the next token without consuming it. That's what `peek()` does:

```c
int
peek(char **ps, char *es, char *toks)
{
  char *s;

  s = *ps;
  while(s < es && strchr(whitespace, *s))
    s++;
  *ps = s;
  return *s && strchr(toks, *s);
}
```

This skips whitespace and checks if the current character is one of the characters in `toks`. It returns true if there's a match, false otherwise. It also updates `*ps` to skip the whitespace, so the next call to `gettoken()` will be at the right position.

### Entry Point: parsecmd()

The top-level parsing function is simple:

```c
struct cmd*
parsecmd(char *s)
{
  char *es;
  struct cmd *cmd;

  es = s + strlen(s);
  cmd = parseline(&s, es);
  peek(&s, es, "");
  if(s != es){
    printf(2, "leftovers: %s\n", s);
    panic("syntax");
  }
  nulterminate(cmd);
  return cmd;
}
```

We calculate the end of the string, call `parseline()` to parse it, then check that we've consumed the entire string. If there are leftovers (malformed input), we panic.

Finally, we call `nulterminate()` to null-terminate all the strings in the command structure (we'll see this later).

### Parsing a Line: parseline()

```c
struct cmd*
parseline(char **ps, char *es)
{
  struct cmd *cmd;

  cmd = parsepipe(ps, es);
  while(peek(ps, es, "&")){
    gettoken(ps, es, 0, 0);
    cmd = backcmd(cmd);
  }
  if(peek(ps, es, ";")){
    gettoken(ps, es, 0, 0);
    cmd = listcmd(cmd, parseline(ps, es));
  }
  return cmd;
}
```

This handles the top level of the grammar: sequences and background jobs.

First, we parse a pipe (which might just be a simple command). Then:
- If we see `&`, consume it and wrap the command in a `backcmd`.
- If we see `;`, consume it and create a `listcmd` with the left side being what we've parsed so far and the right side being the rest of the line (parsed recursively).

Note the recursion: if we see `;`, we call `parseline()` again to parse the rest. This handles chains of commands like `a; b; c; d`.

### Parsing a Pipe: parsepipe()

```c
struct cmd*
parsepipe(char **ps, char *es)
{
  struct cmd *cmd;

  cmd = parseexec(ps, es);
  if(peek(ps, es, "|")){
    gettoken(ps, es, 0, 0);
    cmd = pipecmd(cmd, parsepipe(ps, es));
  }
  return cmd;
}
```

Similar structure: parse a simple command, then check for `|`. If we see a pipe, create a `pipecmd` with the left side being what we've parsed and the right side being the rest of the pipe (parsed recursively).

This handles chains like `a | b | c | d`.

### Parsing a Simple Command: parseexec()

```c
struct cmd*
parseexec(char **ps, char *es)
{
  char *q, *eq;
  int tok, argc;
  struct execcmd *cmd;
  struct cmd *ret;

  if(peek(ps, es, "("))
    return parseblock(ps, es);

  ret = execcmd();
  cmd = (struct execcmd*)ret;

  argc = 0;
  ret = parseredirs(ret, ps, es);
  while(!peek(ps, es, "|)&;")){
    if((tok=gettoken(ps, es, &q, &eq)) == 0)
      break;
    if(tok != 'a')
      panic("syntax");
    cmd->argv[argc] = q;
    cmd->eargv[argc] = eq;
    argc++;
    if(argc >= MAXARGS)
      panic("too many args");
    ret = parseredirs(ret, ps, es);
  }
  cmd->argv[argc] = 0;
  cmd->eargv[argc] = 0;
  return ret;
}
```

This is the most complex parsing function. Let's trace through it:

1. If we see `(`, this is a parenthesized subcommand, so call `parseblock()`.
2. Create an `execcmd` structure.
3. Parse any leading redirections with `parseredirs()`.
4. Loop, collecting arguments:
   - Stop if we see `|`, `)`, `&`, or `;` (these mark the end of the simple command).
   - Get the next token. If it's not `'a'` (a word), that's a syntax error.
   - Store the token's start and end pointers in `argv` and `eargv`.
   - Check for redirections after each argument.
5. Null-terminate the argv array.

The result is an `execcmd` with all the arguments filled in, potentially wrapped in one or more `redircmd` structures.

### Parsing Redirections: parseredirs()

```c
struct cmd*
parseredirs(struct cmd *cmd, char **ps, char *es)
{
  int tok;
  char *q, *eq;

  while(peek(ps, es, "<>")){
    tok = gettoken(ps, es, 0, 0);
    if(gettoken(ps, es, &q, &eq) != 'a')
      panic("missing file for redirection");
    switch(tok){
    case '<':
      cmd = redircmd(cmd, q, eq, O_RDONLY, 0);
      break;
    case '>':
      cmd = redircmd(cmd, q, eq, O_WRONLY|O_CREATE, 1);
      break;
    case '+':  // >>
      cmd = redircmd(cmd, q, eq, O_WRONLY|O_CREATE, 1);
      break;
    }
  }
  return cmd;
}
```

This handles redirections. We loop while we see `<` or `>`:
1. Get the redirection symbol.
2. Get the next token (the filename). If it's not a word, that's an error.
3. Wrap the command in a `redircmd` with the appropriate mode and FD.

For `<`, we redirect FD 0 (stdin) in read-only mode.
For `>`, we redirect FD 1 (stdout) in write-only mode, creating the file if needed.
For `>>`, same as `>` but... wait, the mode is the same? Looking at the code, xv6 doesn't actually handle `>>` differently! This is a limitation of the xv6 shell -- it recognizes `>>` but treats it the same as `>`.

(If this were a real shell, `>>` would use O_APPEND to append to the file instead of truncating it.)

### Parsing Parenthesized Commands: parseblock()

```c
struct cmd*
parseblock(char **ps, char *es)
{
  struct cmd *cmd;

  if(!peek(ps, es, "("))
    panic("parseblock");
  gettoken(ps, es, 0, 0);
  cmd = parseline(ps, es);
  if(!peek(ps, es, ")"))
    panic("syntax - missing )");
  gettoken(ps, es, 0, 0);
  cmd = parseredirs(cmd, ps, es);
  return cmd;
}
```

This handles parenthesized subcommands like `(echo hello; echo world) > output.txt`.

We expect a `(`, then parse the contents as a line (which might contain `;`, pipes, etc.), then expect a `)`, then handle any redirections that apply to the whole block.

Parentheses let you group commands and apply redirections to the group. Pretty neat!

### Null-Terminating Strings: nulterminate()

```c
struct cmd*
nulterminate(struct cmd *cmd)
{
  int i;
  struct backcmd *bcmd;
  struct execcmd *ecmd;
  struct listcmd *lcmd;
  struct pipecmd *pcmd;
  struct redircmd *rcmd;

  if(cmd == 0)
    return 0;

  switch(cmd->type){
  case EXEC:
    ecmd = (struct execcmd*)cmd;
    for(i=0; ecmd->argv[i]; i++)
      *ecmd->eargv[i] = 0;
    break;

  case REDIR:
    rcmd = (struct redircmd*)cmd;
    nulterminate(rcmd->cmd);
    *rcmd->efile = 0;
    break;

  case PIPE:
    pcmd = (struct pipecmd*)cmd;
    nulterminate(pcmd->left);
    nulterminate(pcmd->right);
    break;

  case LIST:
    lcmd = (struct listcmd*)cmd;
    nulterminate(lcmd->left);
    nulterminate(lcmd->right);
    break;

  case BACK:
    bcmd = (struct backcmd*)cmd;
    nulterminate(bcmd->cmd);
    break;
  }
  return cmd;
}
```

Remember that during parsing, we stored pointers to the start and end of each token, but we never null-terminated them. This function walks the command tree and adds null terminators at the end of each string.

For `execcmd`, we null-terminate each argument.
For `redircmd`, we recursively process the sub-command and null-terminate the filename.
For compound commands (pipe, list, back), we recursively process the sub-commands.

After this function runs, all the strings in the command tree are proper null-terminated C strings, ready to be used with exec, open, etc.

## Executing Commands: runcmd()

Finally, we get to execute the command! The `runcmd()` function takes a command structure and executes it:

```c
void
runcmd(struct cmd *cmd)
{
  int p[2];
  struct backcmd *bcmd;
  struct execcmd *ecmd;
  struct listcmd *lcmd;
  struct pipecmd *pcmd;
  struct redircmd *rcmd;

  if(cmd == 0)
    exit();

  switch(cmd->type){
  default:
    panic("runcmd");

  case EXEC:
    ecmd = (struct execcmd*)cmd;
    if(ecmd->argv[0] == 0)
      exit();
    exec(ecmd->argv[0], ecmd->argv);
    printf(2, "exec %s failed\n", ecmd->argv[0]);
    break;

  case REDIR:
    rcmd = (struct redircmd*)cmd;
    close(rcmd->fd);
    if(open(rcmd->file, rcmd->mode) < 0){
      printf(2, "open %s failed\n", rcmd->file);
      exit();
    }
    runcmd(rcmd->cmd);
    break;

  case LIST:
    lcmd = (struct listcmd*)cmd;
    if(fork1() == 0)
      runcmd(lcmd->left);
    wait();
    runcmd(lcmd->right);
    break;

  case PIPE:
    pcmd = (struct pipecmd*)cmd;
    if(pipe(p) < 0)
      panic("pipe");
    if(fork1() == 0){
      close(1);
      dup(p[1]);
      close(p[0]);
      close(p[1]);
      runcmd(pcmd->left);
    }
    if(fork1() == 0){
      close(0);
      dup(p[0]);
      close(p[0]);
      close(p[1]);
      runcmd(pcmd->right);
    }
    close(p[0]);
    close(p[1]);
    wait();
    wait();
    break;

  case BACK:
    bcmd = (struct backcmd*)cmd;
    if(fork1() == 0)
      runcmd(bcmd->cmd);
    break;
  }
  exit();
}
```

This is a big function, so let's go through each case:

### EXEC: Simple Command

```c
case EXEC:
  ecmd = (struct execcmd*)cmd;
  if(ecmd->argv[0] == 0)
    exit();
  exec(ecmd->argv[0], ecmd->argv);
  printf(2, "exec %s failed\n", ecmd->argv[0]);
  break;
```

For a simple command like `ls -l`, we just call `exec()` with the program name and arguments. If exec succeeds, it never returns (the process is replaced by the new program). If it fails (program not found, etc.), we print an error message.

Note that we don't explicitly `exit()` here -- execution falls through to the `exit()` at the end of the function.

### REDIR: I/O Redirection

```c
case REDIR:
  rcmd = (struct redircmd*)cmd;
  close(rcmd->fd);
  if(open(rcmd->file, rcmd->mode) < 0){
    printf(2, "open %s failed\n", rcmd->file);
    exit();
  }
  runcmd(rcmd->cmd);
  break;
```

For redirection like `ls > output.txt`, we:
1. Close the file descriptor we're redirecting (0 for `<`, 1 for `>`).
2. Open the file. Since we just closed the FD, `open()` will use that FD (remember, the kernel always uses the lowest unused FD).
3. Recursively call `runcmd()` on the sub-command.

For example, for `ls > output.txt`:
1. Close FD 1 (stdout).
2. Open `output.txt` for writing, which gets FD 1.
3. Call `runcmd()` on the `ls` command.
4. When `ls` calls exec, it will inherit FD 1 pointing to the file, so its output goes to the file!

This is the classic Unix I/O redirection trick.

### LIST: Sequential Commands

```c
case LIST:
  lcmd = (struct listcmd*)cmd;
  if(fork1() == 0)
    runcmd(lcmd->left);
  wait();
  runcmd(lcmd->right);
  break;
```

For sequential commands like `echo hello; echo world`, we:
1. Fork a child and run the left command in the child.
2. Wait for the child to finish.
3. Run the right command in the current process (which might be the shell, or might be a child of the shell).

This ensures the commands run sequentially: left first, then right.

### PIPE: Pipes

```c
case PIPE:
  pcmd = (struct pipecmd*)cmd;
  if(pipe(p) < 0)
    panic("pipe");
  if(fork1() == 0){
    close(1);
    dup(p[1]);
    close(p[0]);
    close(p[1]);
    runcmd(pcmd->left);
  }
  if(fork1() == 0){
    close(0);
    dup(p[0]);
    close(p[0]);
    close(p[1]);
    runcmd(pcmd->right);
  }
  close(p[0]);
  close(p[1]);
  wait();
  wait();
  break;
```

Pipes are the most complex case. For `ls | grep foo`, we:

1. Create a pipe with `pipe(p)`. This gives us two file descriptors: `p[0]` for reading and `p[1]` for writing.

2. Fork a child for the left command (`ls`):
   - Close FD 1 (stdout).
   - Dup `p[1]` to get FD 1. Now stdout is the write end of the pipe.
   - Close both pipe FDs (we don't need the originals anymore -- we've got FD 1).
   - Run the left command. Its output goes to the pipe!

3. Fork another child for the right command (`grep foo`):
   - Close FD 0 (stdin).
   - Dup `p[0]` to get FD 0. Now stdin is the read end of the pipe.
   - Close both pipe FDs.
   - Run the right command. Its input comes from the pipe!

4. In the parent, close both pipe FDs (we don't need them).
5. Wait for both children to finish.

The key insight: after we `dup()` the pipe FDs to 0 or 1, we close the original pipe FDs. This is important! If we didn't close them, the pipe wouldn't close properly and the right command would hang waiting for EOF (which only happens when *all* write ends are closed).

Also notice that the parent closes both pipe FDs too. This is crucial! If the parent kept the pipe FDs open, the right command would never see EOF because the write end would still be open in the parent.

### BACK: Background Jobs

```c
case BACK:
  bcmd = (struct backcmd*)cmd;
  if(fork1() == 0)
    runcmd(bcmd->cmd);
  break;
```

For background jobs like `sleep 100 &`, we simply fork and run the command in the child, but we *don't* wait for it. The command runs in the background while the shell continues accepting commands.

The child will eventually finish and become a zombie. Who reaps it? Remember from the "First Process" post that init waits for all orphaned children. When the shell exits (or even if it doesn't, but the child outlives it), the background job gets reparented to init, which will reap it.

## A Complete Example

Let's trace through a complete example to see how everything fits together. Suppose the user types:

```
ls -l | grep txt > results.txt
```

Here's what happens:

### 1. Reading and Parsing

- `getcmd()` prints `$ ` and reads the line into a buffer.
- `parsecmd()` is called:
  - `parseline()` calls `parsepipe()`.
  - `parsepipe()` calls `parseexec()` to parse `ls -l`.
    - Creates an `execcmd` with `argv = ["ls", "-l", NULL]`.
  - `parsepipe()` sees `|`, so it creates a `pipecmd`:
    - Left side: the `ls -l` command.
    - Right side: recursively call `parsepipe()` on the rest.
  - The recursive `parsepipe()` calls `parseexec()` to parse `grep txt > results.txt`.
    - Creates an `execcmd` with `argv = ["grep", "txt", NULL]`.
    - Sees `>`, so wraps it in a `redircmd`:
      - Sub-command: the `grep txt` command.
      - File: "results.txt"
      - Mode: O_WRONLY|O_CREATE
      - FD: 1 (stdout)
  - Returns the `pipecmd` with left = `ls -l` and right = `grep txt > results.txt`.
- `nulterminate()` adds null terminators to all the strings.

The final command tree looks like:
```
pipecmd
├─ left: execcmd ["ls", "-l"]
└─ right: redircmd (fd=1, file="results.txt")
          └─ sub: execcmd ["grep", "txt"]
```

### 2. Executing

- `main()` forks a child and calls `runcmd()` on the command tree.
- `runcmd()` sees it's a PIPE:
  - Creates a pipe with FDs `p[0]` (read) and `p[1]` (write).
  - Forks child 1 for the left side:
    - Closes FD 1 (stdout).
    - Dups `p[1]` to get FD 1 (stdout → pipe write end).
    - Closes `p[0]` and `p[1]`.
    - Calls `runcmd()` on `ls -l`.
      - Sees it's an EXEC.
      - Calls `exec("ls", ["ls", "-l"])`.
      - The `ls` program runs, writes to FD 1 (the pipe).
  - Forks child 2 for the right side:
    - Closes FD 0 (stdin).
    - Dups `p[0]` to get FD 0 (stdin ← pipe read end).
    - Closes `p[0]` and `p[1]`.
    - Calls `runcmd()` on the redircmd:
      - Closes FD 1 (stdout).
      - Opens "results.txt", which gets FD 1.
      - Calls `runcmd()` on `grep txt`.
        - Sees it's an EXEC.
        - Calls `exec("grep", ["grep", "txt"])`.
        - The `grep` program runs, reads from FD 0 (the pipe), writes to FD 1 (the file).
  - Parent closes `p[0]` and `p[1]`.
  - Parent waits for both children to finish.

### 3. Result

- `ls` lists all files, writing to the pipe.
- `grep` reads from the pipe, filters for lines containing "txt", and writes to results.txt.
- Both programs exit.
- The shell prints a new prompt.

Beautiful! The pipe connects the output of `ls` to the input of `grep`, and the redirection sends `grep`'s output to a file. All using simple fork/exec/pipe/dup/close primitives!

## Design Insights

Let's step back and appreciate some of the design decisions in the xv6 shell:

### 1. Fork + Exec Pattern

The shell always forks before running a command (except for `cd`, which is special-cased). This keeps the shell alive and ready for the next command. The child process is disposable -- it can exec the command, and if exec fails, it can just exit without affecting the shell.

This is the fundamental Unix pattern, and it's incredibly powerful. It means:
- The shell doesn't need to know anything about the programs it runs.
- Programs can't corrupt the shell's memory or state.
- Multiple commands can run simultaneously (in background jobs or pipelines).

### 2. File Descriptors as Abstraction

The beauty of Unix I/O is that everything is a file descriptor. Whether it's a file, a pipe, a device, or a socket (not in xv6, but in real Unix), programs just read and write FDs. The shell can wire up complex I/O connections just by manipulating FDs with `open()`, `close()`, `dup()`, and `pipe()`.

Programs don't need to know if their input is coming from a file, a pipe, or a terminal -- they just read from FD 0. They don't need to know if their output is going to a file, a pipe, or a terminal -- they just write to FD 1. This uniformity is what makes pipes and redirection so powerful.

### 3. Recursive Data Structures

The command tree is a recursive data structure: commands can contain other commands (redirection wraps a command, pipes connect two commands, etc.). This mirrors the recursive structure of the shell syntax.

Parsing builds the tree recursively (with `parseline()` calling `parsepipe()` calling `parseexec()`, etc.), and execution traverses the tree recursively (with `runcmd()` calling itself on sub-commands).

This is a elegant and makes the code surprisingly simple for the amount of functionality it provides.

### 4. Separation of Parsing and Execution

The shell cleanly separates parsing from execution. `parsecmd()` builds a command tree, and `runcmd()` executes it. They're completely independent.

This separation makes the code easier to understand and modify. Want to add a new shell feature? Add a new command type, a parsing function for it, and a case in `runcmd()`. Want to change how commands are executed? Just modify `runcmd()`.

### 5. Defensive Programming

Notice how the shell checks for errors everywhere: fork can fail, exec can fail, open can fail, pipe can fail. The shell handles all these cases gracefully (usually by printing an error and exiting the child process).

This defensive programming makes the shell robust. Even if something goes wrong (out of memory, file not found, etc.), the shell doesn't crash -- it reports the error and continues.

## Limitations and Missing Features

The xv6 shell is simple and educational, but it lacks many features of real shells like Bash:

**No job control**: You can run jobs in the background with `&`, but you can't bring them to the foreground, suspend them, list them, etc.

**No variables**: No environment variables, no shell variables like `$HOME` or `$?`.

**No scripting**: No `if` statements, loops, functions, etc. It's just a command interpreter.

**No command history**: No up-arrow to recall previous commands.

**No tab completion**: You have to type out full filenames and commands.

**No globbing**: No `*.txt` to match multiple files (you'd have to use `ls *.txt` and let `ls` handle the pattern, or it won't work).

**No quotes**: No way to include spaces or special characters in arguments (no `"hello world"` or `'foo; bar'`).

**>> doesn't append**: As we noted earlier, `>>` is recognized but doesn't actually append.

**No exit status checking**: No way to check if a command succeeded or failed.

But despite these limitations, the xv6 shell demonstrates all the core concepts: fork/exec, pipes, redirection, background jobs. It's a fully functional Unix shell in just 500 lines of C!

## How Real Shells Work

Real shells like Bash are much more complex (Bash is tens of thousands of lines of code!), but they use the same fundamental mechanisms:

**Job control**: Shells use process groups and session IDs to manage jobs. They can send signals (SIGSTOP, SIGCONT) to suspend and resume jobs.

**Variables**: Shells maintain an environment (a set of name=value pairs) that gets passed to child processes. They also have internal variables for scripting.

**Scripting**: Shells are full programming languages with conditionals, loops, functions, etc. But at the core, they still fork and exec to run commands.

**Command history**: Shells use the readline library (or similar) to provide line editing, history, and completion.

**Globbing**: Shells expand wildcards before executing commands. For example, `ls *.txt` gets expanded to `ls file1.txt file2.txt` before calling exec.

But underneath all these features, the core is the same: read commands, parse them, fork children, set up I/O, and exec programs. The xv6 shell shows you that core in its purest form.

## Interaction with the System

Let's trace one more example to see how the shell interacts with the rest of xv6. Suppose the user types:

```
echo hello
```

Here's the full system trace:

1. **Shell waits for input**: The shell is blocked in `gets()` (called from `getcmd()`), which is blocked in a `read()` system call waiting for input from the console.

2. **User types**: When the user types 'e', the keyboard driver handles the interrupt, adds the character to the console buffer, and wakes up the shell. The shell reads the character and stores it in its buffer. This repeats for each character.

3. **User presses Enter**: When the user presses Enter, the shell sees a newline, and `gets()` returns.

4. **Parsing**: The shell calls `parsecmd()`, which parses "echo hello" into an `execcmd` with `argv = ["echo", "hello", NULL]`.

5. **Forking**: The shell calls `fork()`, which:
   - Allocates a new proc structure.
   - Copies the parent's page table and memory.
   - Copies the parent's file descriptors (so the child inherits FDs 0, 1, 2 pointing to the console).
   - Marks the child as RUNNABLE.

6. **Parent waits**: The parent shell calls `wait()`, which puts it to sleep until a child exits.

7. **Child execs**: The child calls `exec("echo", ["echo", "hello"])`, which:
   - Opens the "echo" file from the file system.
   - Reads and parses the ELF header.
   - Allocates new memory for the program.
   - Loads the program code and data from disk.
   - Sets up the user stack with the arguments.
   - Frees the old memory.
   - Returns to user space at the program's entry point.

8. **Echo runs**: The echo program runs. It writes "hello\n" to FD 1 (stdout), which goes to the console driver, which displays it on the screen.

9. **Echo exits**: The echo program calls `exit()`, which:
   - Closes all open file descriptors.
   - Marks the process as ZOMBIE.
   - Wakes up the parent (the shell).

10. **Shell reaps**: The shell wakes up from `wait()`, which:
    - Finds the zombie child.
    - Frees the child's memory and proc structure.
    - Returns the child's PID to the shell.

11. **Loop continues**: The shell goes back to the top of its loop and calls `getcmd()` again, printing a new prompt.

That's the complete lifecycle! Every command goes through this same process: read, parse, fork, exec, wait. The shell orchestrates it all using the system calls we've studied throughout these posts.

## Wrapping Up

The shell is the crown jewel of Unix: it ties together all the system's components into a coherent, usable interface. Let's recap what we learned:

1. **Main loop**: Read commands, parse them, fork and execute them, wait for completion.

2. **Special case: cd**: Must run in the shell itself, not a child, to change the shell's working directory.

3. **Command structures**: A tree-based representation with types for EXEC, REDIR, PIPE, LIST, and BACK.

4. **Parsing**: Recursive descent parser with `parseline()`, `parsepipe()`, and `parseexec()`. Tokenization with `gettoken()`.

5. **Execution**: Recursive traversal of the command tree with `runcmd()`. Different cases for each command type:
   - EXEC: Just call `exec()`.
   - REDIR: Close and reopen FDs, then recurse.
   - LIST: Fork, run left, wait, then run right.
   - PIPE: Create pipe, fork twice (one for each side), set up FDs, recurse on both sides.
   - BACK: Fork and run in background without waiting.

6. **I/O redirection**: Close the FD you want to redirect, open the file (which gets that FD), then exec.

7. **Pipes**: Create a pipe, fork two children, connect stdout of the left to the write end and stdin of the right to the read end, close all unnecessary FDs.

8. **Design principles**: Fork/exec pattern, FDs as abstraction, recursive data structures, separation of parsing and execution, defensive programming.

The xv6 shell is a masterpiece of simple, elegant design. In just 500 lines, it provides all the essential functionality of a Unix shell. And now you understand every line of it!

And that's the shell! We've seen how it reads commands, parses them into tree structures, and executes them using the fork/exec pattern. The shell is the glue that makes Unix usable -- it turns system calls into a human-friendly interface.

But the shell isn't the whole story of user space. There's more to explore! In the next post, we'll look at the C library that user programs rely on -- functions like `printf()`, `malloc()`, `gets()`, and others that we've been using throughout the shell code. These library functions provide a convenient layer on top of the raw system calls, making it easier to write user programs.

After that, we'll take a tour of the various utility programs that come with xv6 (ls, cat, echo, grep, etc.) to see how they use the system calls and library functions we've learned about.

So stick around -- we're not done with user space yet! See you in the next post!