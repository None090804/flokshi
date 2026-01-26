# A Fun Walkthrough on How I Wrote My Own fshell (fsh)

Hello, nerds from the internet. Hope you enjoy my walkthrought. A quick notice: The idea and the code from this project was taken from the following blog post: https://brennan.io/2015/01/16/write-a-shell-in-c/. Credit to the original. Have fun!

---

## Table of Contents

1. [But What is a Computer Shell?](#but-what-is-a-computer-shell)
2. [Writing a Shell in C (FLokshi's Shell)](#writing-a-shell-in-c-flokhsis-shell)
3. [Getting Our Hands Dirty](#getting-our-hands-dirty)
4. [The Main Loop](#the-main-loop)
5. [Reading Lines](#reading-lines)
6. [Parsing Tokens](#parsing-tokens)
7. [Launching Processes](#launching-processes)
8. [Shell Built-ins](#shell-built-ins)
9. [Running It](#running-it)

---

## But What is a Computer Shell?

"An operating system shell is a computer program that provides relatively broad and direct access to the system on which it runs. The term shell refers to how it is a relatively thin layer around an operating system." - Our most trustworthy source of information (Wikipedia)

Think of the computer shell as a thin layer which takes the user's commands and executes them.

---

## Writing a Shell in C (FLokshi's Shell)


I remember a week ago reading about processes, and came across the concept of `fork()`, `exec()` and `wait()`. Doing some random scripts, I thought to myselfwait, so if we can invoke processes within other processes using the `fork()` system call in UNIX, then that must be the core idea behind how shells work.

*Note: I am talking about the OS shell, not the shell that we eat :>*

Cool, so the shell isn't anything else than just a program which allows us to interact with the operating system (in a sense).

The idea around the shell seems intimidating, and I am not going to lie to you, I also thought of them as hard. Because if you think about it, anything around the computer, especially low-level stuff seems scary, but at the same time amazing. But fear you not, the whole idea behind the shell can be summarized in simple parts:

1. **Configuration Files** - Setup and initialization
2. **Interpreting** - Read, parse, and execute commands
3. **Exit and free up extra memory** - (because in the end, there is a global crisis about the RAM shortage, so better be safe)

### Configuration Files

A configuration file (often called a config file) is a text file that defines how a software program, system service, or application should behave. Instead of hardcoding settings inside the program, developers use config files so the user or administrator can easily change parameters without touching the code.

Configuration files control things like:

1. User preferences (themes, keyboard shortcuts, default behaviors)
2. System settings (network configurations, hardware settings)
3. Application behavior (startup options, feature toggles)
4. Environment variables and paths

An example in Linux would be: `/etc/ssh/sshd_config` - the configuration file for the SSH protocol. Try it out, it has some really pretty cool stuff to it.

### Interpreter

I remember my professor in university asking us what is the difference between an interpreted programming language, and half of us sucked really bad at the question. I didn't :)

An interpreted language such as JavaScript—the interpreter reads the source code and interprets it line by line. A compiled language on the other hand, such as our amazing language C, the compiler (the GNU C compiler, gcc or any other compiler whatsoever) would first transform the .c program to an object file, rather than reading it line by line.

Just for notice, the flow of how a C program is compiled is as follows:

```
.c file → (preprocessor) → .i file → (compiler) → .s file → (assembler) → .o/.obj file → (linker) → executable
```
So the way our shell works is that of an interpreter where it reads every line that the user enters.

Now I want also to emphasize the following, which was pretty cool, and I highly recommend the read:

> By default your terminal (shell) starts in **canonical** mode, also called **cooked mode** (chat we are cooked). In this mode, keyboard input is only sent to your program when the user presses `Enter`. This is useful for many programs: it lets the user type in a line of text, use Backspace to fix errors until they get their input exactly the way they want it, and finally press Enter to send it to the program. This is the mode that we will be using.
> 
> The other mode is also very familiar to us. You know when we try to do something that requires root privileges, the password that we can't see? That is because we are operating in **raw mode**. Here the key inputs are being received by the program as they are being typed.

*So the next time somebody asks if you are cooked, tell them I ain't but my terminal is :). You got the joke right, right?*

### Exit and Free the Memory That We Acquire

Since we are apparently in a RAM shortage, making buying a computer today feel like paying an arm and a leg, it is important for us to use our memory safely.

The memory in C doesn't forgive and that is why we have to be really cautious when using it. Memory that is acquired shouldn't be kept occupied if not used. C isn't considered a memory-safe language, meaning if not careful we can have memory leaks, which will bring us to potential loss of data.

---

## Getting Our Hands Dirty

The following is the full code of our overly simplified shell code.

*Notice: The code is reproduced from the following blog: https://brennan.io/2015/01/16/write-a-shell-in-c/. Here is my simplification, and how I, using the tutorial, adapted to my likings*

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <wait.h>
#include <unistd.h>
#include <sys/types.h>

#define FSH_READ_BUFFER 1024 
#define FSH_BUFF_TOKEN 64
#define FSH_DELIMETER " \t\n\r\a" 

// Function declarations
int fsh_cd(char **args);
int fsh_help(char **args);
int fsh_exit(char **args);

char* read_line();
char** parser(char* line);
int launch(char** args);
int fsh_execute(char** args);

void fsh_loop();

// Built-in command names
char *builtin_str[] = {
  "cd",
  "help",
  "exit"
};

// Built-in command function pointers
int (*builtin_func[]) (char **) = {
  &fsh_cd,
  &fsh_help,
  &fsh_exit
};

int fsh_num_builtins() {
  return sizeof(builtin_str) / sizeof(char *);
}

// Built-in function implementations
int fsh_cd(char **args)
{
  if (args[1] == NULL) {
    fprintf(stderr, "fsh: expected argument to \"cd\"\n");
  } else {
    if (chdir(args[1]) != 0) {
      perror(":)>");
    }
  }
  return 1;
}

int fsh_help(char **args) {
  int i;
  printf("Flokshi's fsh\n");
  printf("Type program names and arguments, and hit enter.\n");
  printf("The following are built in:\n");

  for (i = 0; i < fsh_num_builtins(); i++) {
    printf("  %s\n", builtin_str[i]);
  }

  printf("Use the man command for information on other programs.\n");
  return 1;
}

int fsh_exit(char **args)
{
  return 0;
}

// Execute either built-in or launch external program
int fsh_execute(char** args)
{
  int i;

  if (args[0] == NULL) {
    // An empty command was entered
    return 1;
  }

  // Check if command is a built-in
  for (i = 0; i < fsh_num_builtins(); i++) {
    if (strcmp(args[0], builtin_str[i]) == 0) {
      return (*builtin_func[i])(args);
    }
  }

  // Not a built-in, launch external program
  return launch(args);
}

int main(int argc, char*argv[]){
  // The whole idea is that the shell is nothing more than just a loop, looping around
  fsh_loop(); 
  return 0;
}

void fsh_loop(){
  char* line;
  char** args;
  int status;

  do {
    printf(":)>");
    line = read_line();
    args = parser(line);
    status = fsh_execute(args);

    // Free anything that shouldn't be used
    free(line);
    free(args);
  } while (status);
}

char* read_line(){
  // Note: EOF is an integer and to check it you should also compare it with an integer
  int buffer_size = FSH_READ_BUFFER;
  int position = 0;
  // A buffer of characters 
  char* buffer = malloc(sizeof(char) * FSH_READ_BUFFER); // one byte * 1024 bytes 
  int c;
  
  // Checking if malloc failed
  if(!buffer){
    fprintf(stderr, "Malloc Failed\n");
    exit(EXIT_FAILURE);
  }

  while (1) {
    // getchar is of type int and it is going to return an integer (man getchar)
    c = getchar();

    // EOF - if we are reading from the file, else if we are reading from the standard input 
    // If we hit EOF, replace it with a null character and return
    if (c == EOF || c == '\n') {
      buffer[position] = '\0'; // strings by default in C have the null character 
      return buffer;
    } else {
      // Continue feeding our buffer
      buffer[position] = c;
    }
    position++;

    // Sadly we do not know the length of the line that the user is going to be providing for us
    // If we have exceeded the buffer, reallocate
    if (position >= buffer_size) {
      buffer_size += FSH_READ_BUFFER;
      buffer = realloc(buffer, buffer_size);
      if (!buffer) {
        fprintf(stderr, "fsh: allocation error\n");
        exit(EXIT_FAILURE);
      }
    }
  }

  return buffer;
}

char** parser(char* line){
  // All of this can be eliminated using the strtok() function in the standard lib
  int bufsize = FSH_BUFF_TOKEN, position = 0;
  char** tokens = malloc(bufsize * sizeof(char*)); // the size of a char* (8*64) = 512 bytes 
  char* token; // inside the buffer we are going to keep our pointer tokens

  if (!tokens) {
    fprintf(stderr, "fsh: allocation error\n");
    exit(EXIT_FAILURE);
  }

  // Here the usage of strtok
  token = strtok(line, FSH_DELIMETER);
  while (token != NULL) {
    tokens[position] = token;
    position++;

    // We have more tokens than our buffer can fit, resize it 
    if (position >= bufsize) {
      bufsize += FSH_BUFF_TOKEN;
      tokens = realloc(tokens, bufsize * sizeof(char*));
      if (!tokens) {
        fprintf(stderr, "fsh: allocation error\n");
        exit(EXIT_FAILURE);
      }
    }

    token = strtok(NULL, FSH_DELIMETER);
  }
  
  // The last position of the token is going to be NULL
  // This is important for execvp() which expects a NULL-terminated array
  tokens[position] = NULL;

  // Returning tokens 
  return tokens;
}

int launch(char** args){
  pid_t pid, wpid;
  int status;

  pid = fork();
  if (pid == 0) {
    // Child process
    // Here we load the program that a process would need to execute
    // The name of the file and the name of the program
    // Variants of execvp exist
    if (execvp(args[0], args) == -1) {
      perror(":)>");
    }
    exit(EXIT_FAILURE);
  } else if (pid < 0) {
    // Error forking
    perror(":)>");
  } else {
    // Parent process
    do {
      wpid = waitpid(pid, &status, WUNTRACED);
    } while (!WIFEXITED(status) && !WIFSIGNALED(status));
  }

  return 1;
}
```

---

## The Main Loop

We start of with our initial loop `fsh_loop`, this is going to serve as the main logic of our program.

```c
void fsh_loop(){
  char* line;
  char** args;
  int status;

  do {
    printf(":)>");
    line = read_line();
    args = parser(line);
    status = fsh_execute(args);

    // Free anything that shouldn't be used
    free(line);
    free(args);
  } while (status);
}
```

Inside the function we have declared 3 variables:
- **`char* line`** - A pointer to a character (will hold our input line)
- **`char* args`** - A pointer to a pointer to a character I know C pointers can be really confusing but please bear with me (will hold our parsed tokens)
- **`int status`** - An integer that is going to return the status of execution

The loop of our program is implemented using a `do-while` loop, rather than just a while loop. * This is done so even on failure, we get the chance to run theprogram once.*

Inside the loop, the first line is a standard printf function which prints the prompt symbol of our shell.

Next we read the line from the `read_line()` function. To interact with the shell we can write either a single word such as `ls` or a full line such as `gcc file.c -o file`. This is the idea behind the line. *Another fun fact is that every input from stdin is actually text, a string.*

We read args (arguments) from the parser function, which takes a line as input.

**What is parsing?** - Parsing is the process of taking a string (line) and breaking it down into smaller parts called tokens, which can then be interpreted into logical syntax. To analyze (a string or text) into logical syntactic components - from Wikipedia.

**Why would we need to parse a string?** - Good question, and I have an even better answer for you. Referring to the above idea of a line, have you ever thought why when we type `ls -a` we have a listing of all our files/directories even the hidden ones? How does the shell understand that `ls -a` is one thing and `ls -l` is another thing? That is when parsing comes into play.

Parsing separates the line argument that was given to us and breaks it down into smaller logically syntactically correct parts. *More when I actually write on a parser :).*

And finally we get the status from the `fsh_execute` function. The status will indicate whether or not our shell will continue running, as seen in `while(status)`.

Since both line and args are pointers we would need to free them, that is why we are using free. *You know that it is quite simple to implement malloc using sbrk() but free is a pain to do, since you would need to keep track of the memory allocated.*

---

## Reading Lines

We continue with the implementation of the three functions that we discussed earlier. Firstly `read_line()`.

This function is going to contain the logic on how we will be reading the lines (commands). Now it is quite important to do a distinction we can read the commands from stdin or from a file. In our function, we take both into consideration.

```c
char* read_line(){
  int buffer_size = FSH_READ_BUFFER;
  int position = 0;
  char* buffer = malloc(sizeof(char) * FSH_READ_BUFFER);
  int c;
  
  if(!buffer){
    fprintf(stderr, "Malloc Failed\n");
    exit(EXIT_FAILURE);
  }

  while (1) {
    c = getchar();

    if (c == EOF || c == '\n') {
      buffer[position] = '\0';
      return buffer;
    } else {
      buffer[position] = c;
    }
    position++;

    if (position >= buffer_size) {
      buffer_size += FSH_READ_BUFFER;
      buffer = realloc(buffer, buffer_size);
      if (!buffer) {
        fprintf(stderr, "fsh: allocation error\n");
        exit(EXIT_FAILURE);
      }
    }
  }

  return buffer;
}
```

The function will return a pointer to a character and takes no arguments. We will be storing the characters from either stdin or a file using a buffer, which contains the size of `FSH_READ_BUFFER` macro which we defined at the beginning.

We acquire memory for the buffer using `malloc`. Upon malloc failure we print standard error that the program failed. Upon malloc success, we will be reading the characters from either stdin or a file, and to get each character we will be using the `getchar()` function.

*Something that was quite interesting for me when I was reading the blog post was that EOF is actually an integer rather than a character.*

While the loop runs we are going to check each character. If we have arrived at the end of the file or a new line, we are, at that buffer position, putting a `\0`. To mark the end of a string in C we use the null byte :). *I got roasted pretty bad on Stack Overflow for this, so be grateful* and then we return the buffer. Otherwise we feed the buffer at that position with the character and increment the position.

Unfortunately we do not know how long the line that our program would be reading is, so that is why we need to be careful, because malloc can fail. Upon exceeding buffer size we will be reallocating more memory to the buffer using realloc. If that fails, we print the error and exit on `EXIT_FAILURE`. Once the line has all been read, we return our buffer.

---

## Parsing Tokens

Now it is time to parse each of the characters that we read from the line. As mentioned above, parsing is the process of converting a string into syntactically logical tokens.

Our parsing function is going to return a pointer to a char pointer and as a parameter is going to accept a pointer to a character. The logic of our parsing function is somewhat the same as the one that we used for the read_line function.

Here we also define a buffer and we set its size equal to the `FSH_BUFF_TOKEN` which we defined above.

```c
char** parser(char* line){
  int bufsize = FSH_BUFF_TOKEN, position = 0;
  char** tokens = malloc(bufsize * sizeof(char*));
  char* token;

  if (!tokens) {
    fprintf(stderr, "fsh: allocation error\n");
    exit(EXIT_FAILURE);
  }

  token = strtok(line, FSH_DELIMETER);
  while (token != NULL) {
    tokens[position] = token;
    position++;

    if (position >= bufsize) {
      bufsize += FSH_BUFF_TOKEN;
      tokens = realloc(tokens, bufsize * sizeof(char*));
      if (!tokens) {
        fprintf(stderr, "fsh: allocation error\n");
        exit(EXIT_FAILURE);
      }
    }

    token = strtok(NULL, FSH_DELIMETER);
  }
  
  tokens[position] = NULL;
  return tokens;
}
```

As previously mentioned, we are going to separate the line that we read from stdin into tokens (syntactically correct tokens). To do this we are going to be using the `strtok()` function which, based on `man`, is defined as: `strtok() function breaks a string into a sequence of zero or nonempty tokens`.

The function accepts two parameters: the first one is the string that we are going to separate the tokens from and the second is the DELIMITER. For our example, we have defined our delimiters in the `FSH_DELIMETER` macro, including a space, a tab, a new line, carriage return, and an alert.

**Important:** The last element of our tokens array is set to NULL (`tokens[position] = NULL`). This is crucial because `execvp()` expects a NULL-terminated array of arguments. Without this, the program would not know where the argument list ends.

Everything else is similar to what we did in the read_line function. Upon completion we return tokens.

---

## Launching Processes

Now we come to the part that was the whole purpose of writing this project and learning about the shell: **processes**. Reading about processes and the way they were invoked made me want to explore this project.

For each line of command that we are writing in stdin, a new process is going to be created. We create a process using the `fork()` system call. By man definition: **fork() creates a new process by duplicating the calling process. The new process is referred to as the child process. The calling process is referred to as the parent process**.

```c
int launch(char** args){
  pid_t pid, wpid;
  int status;

  pid = fork();
  if (pid == 0) {
    // Child process
    if (execvp(args[0], args) == -1) {
      perror(":)>");
    }
    exit(EXIT_FAILURE);
  } else if (pid < 0) {
    // Error forking
    perror(":)>");
  } else {
    // Parent process
    do {
      wpid = waitpid(pid, &status, WUNTRACED);
    } while (!WIFEXITED(status) && !WIFSIGNALED(status));
  }

  return 1;
}
```

In our example of the launch function, which accepts as a parameter the token buffer, the parent process is our shell (fsh) while the child is going to be the process which is created by invoking the fork() API.

**fork() return values:**
- On success: returns the PID of the child process in the parent, and 0 in the child
- On failure: -1 is returned in the parent, no child process is created, and errno is set to indicate the error

For this reason, after the creation of the child using fork(), we check if `pid == 0`. Upon success we use another system call which is `execvp()`, which belongs to a large family of executable functions, but in this example we use execvp().

The execvp/exec() functions return only if an error has occurred, that is why if the return value is -1 we return a `perror(":)>")` and exit on EXIT_FAILURE.

### Avoiding Zombie Processes

Something that we also need to be cautious about when using processes is that a process after finishing its execution and not being terminated by the parent becomes a zombie.

*The definition from the dinosaur OS book would be as follows: A process whose parent has not invoked wait() is treated as a zombie, staying inside the OS without releasing its resources.*

That is why to omit that we invoke the `waitpid()` system call, which suspends execution of the calling thread until a child specified by the pid argument has changed status. By default waitpid() waits only for terminated children, but since we have used **WUNTRACED** it also returns if a child has stopped, and we continue to do this until it has read all the commands given using stdin.

---

## Shell Built-ins

If we were to do a quick Google search about the shell we would come to know that there are some built-in features inside the shell which we can use without the need to install or write ourselves—they come with the shell. For example `ls, man, cd, mkdir, nano, touch`, etc., are built-in inside the shell. This is the same approach we will also be using with our shell.

The built-ins that we will be using would be some standard ones like `cd, help, and exit`. The idea of the code is quite self-explanatory.

### How Built-ins Work

We have two parallel arrays:
- `builtin_str[]` - contains the names of built-in commands
- `builtin_func[]` - contains function pointers to the implementations

```c
char *builtin_str[] = {
  "cd",
  "help",
  "exit"
};

int (*builtin_func[]) (char **) = {
  &fsh_cd,
  &fsh_help,
  &fsh_exit
};
```

### The Execute Function

We launch everything in the `fsh_execute()` function:

```c
int fsh_execute(char** args)
{
  int i;

  if (args[0] == NULL) {
    // An empty command was entered
    return 1;
  }

  // Check if command is a built-in
  for (i = 0; i < fsh_num_builtins(); i++) {
    if (strcmp(args[0], builtin_str[i]) == 0) {
      return (*builtin_func[i])(args);
    }
  }

  // Not a built-in, launch external program
  return launch(args);
}
```

This function:
1. Checks if the command is empty (returns 1 to continue the shell loop)
2. Loops through built-in commands to see if there's a match
3. If a built-in is found, calls its corresponding function
4. If not a built-in, calls `launch()` to execute it as an external program

The return value determines whether the shell continues running (1) or exits (0).

---

## Running It

Now it is time for magic. We press `esc :wq` to save the file, compile and run it:

```bash
gcc fsh.c -o fsh
./fsh
```

And the following is shown to us:

```
:)>
```

Okay, till now without any errors. Let us try to enter one of those built-in functions that we specified earlier. Let us try help:

```
:)> help
Flokshi's fsh
Type program names and arguments, and hit enter.
The following are built in:
  cd
  help
  exit
Use the man command for information on other programs.
:)>
```

Let us try something that is not a built-in function, for example mkdir:

```
:)> mkdir test_directory
:)> ls
test_directory
:)>
```

It works! Now let's try a typo:

```
:)> mkidr
:)>: No such file or directory
:)>
```

Let us try to exit now:

```
:)> exit
```

We exit the shell successfully!

---

## Conclusion

So yeah, this is pretty much everything that I built in my very simple shell, using the tutorial specified in the beginning. Somewhere in the future I would have to also include other functionalities as well, and why not make it a good usable shell. Till then bye!

**Flokshi :)**

---

### Fun Fact

*Something really interesting that I learned on the Internet by @rebane2001: Every file falls inside one of two categories: A text file or a .zip file, and I thought it was pretty cool I guess.*

---

## Key Takeaways

1. **A shell is just a loop** - Read, parse, execute, repeat
2. **Processes are created with fork()** - The child inherits from the parent
3. **execvp() replaces the process** - The child becomes the new program
4. **Built-ins are handled differently** - They run in the shell process, not in a child
5. **Memory management is crucial** - Always free what you malloc
6. **NULL-terminated arrays matter** - Functions like execvp() expect them

This project taught me the fundamentals of:
- Process management in Unix
- System calls (fork, exec, wait)
- Memory management in C
- Parsing and tokenization
- The architecture of command-line interfaces

Happy shell hacking! 🐚
