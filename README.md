# Pipex - Unix Pipe Implementation

## 📖 Overview

**Pipex** is a C program that recreates the behavior of Unix pipes (`|`) in the shell. It allows you to chain multiple commands together, where the output of one command becomes the input of the next command, and finally writes the result to a file.

## 🎯 What Does It Do?

Pipex mimics the behavior of this shell command:
```bash
< infile cmd1 | cmd2 | cmd3 > outfile
```

But using our program:
```bash
./pipex infile "cmd1" "cmd2" "cmd3" outfile
```

## 🔧 How It Works

### Basic Concept
Think of pipes like a water pipe system:
- **Input file** → **Command 1** → **Command 2** → **Command 3** → **Output file**
- Each command processes the data and passes it to the next command
- The final result is saved in the output file

### Program Flow Diagram

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   infile    │───▶│   cmd1      │───▶│   cmd2      │───▶│   cmd3      │
│  (input)    │     │  (process)  │     │  (process)  │     │  (process)  │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
                                                                   │
                                                                   ▼
                                                            ┌─────────────┐
                                                            │   outfile   │
                                                            │  (output)   │
                                                            └─────────────┘
```

### Process Creation Schema

```
Parent Process
     │
     ├─── fork() ──▶ Child Process 1 (cmd1)
     │              │
     │              ├─── pipe() ──▶ [read][write]
     │              │
     │              ├─── dup2() ──▶ Redirect stdin/stdout
     │              │
     │              └─── execve() ──▶ Execute cmd1
     │
     ├─── fork() ──▶ Child Process 2 (cmd2)
     │              │
     │              ├─── pipe() ──▶ [read][write]
     │              │
     │              ├─── dup2() ──▶ Redirect stdin/stdout
     │              │
     │              └─── execve() ──▶ Execute cmd2
     │
     ├─── fork() ──▶ Child Process 3 (cmd3)
     │              │
     │              ├─── open() ──▶ Open outfile
     │              │
     │              ├─── dup2() ──▶ Redirect stdout to file
     │              │
     │              └─── execve() ──▶ Execute cmd3
     │
     └─── wait() ──▶ Wait for all children to finish
```

## 🚀 Usage

### Basic Usage
```bash
./pipex infile "command1" "command2" outfile
```

### Examples

1. **Simple text processing:**
   ```bash
   ./pipex input.txt "cat" "grep hello" output.txt
   ```
   This reads from `input.txt`, pipes through `cat`, then `grep hello`, and saves to `output.txt`

2. **Multiple commands:**
   ```bash
   ./pipex input.txt "ls -la" "grep .txt" "wc -l" output.txt
   ```
   Lists files, filters for .txt files, counts lines, saves result

3. **With here_doc (heredoc mode):**
   ```bash
   ./pipex here_doc "EOF" "cat" "sort" output.txt
   ```
   Allows interactive input until "EOF" is typed, then processes it

## 🛠️ Installation & Compilation

### Prerequisites
- GCC compiler
- Make utility
- Unix-like system (Linux, macOS)

### Build Instructions
```bash
# Clone or download the project
cd pipex

# Compile the program
make

# The executable 'pipex' will be created
```

### Makefile Commands
- `make` - Compiles the program
- `make clean` - Removes object files
- `make fclean` - Removes object files and executable
- `make re` - Recompiles everything from scratch

## 📁 Project Structure

```
pipex/
├── inc/                    # Header files
│   ├── pipex.h            # Main header with function declarations
│   └── get_next_line.h    # Get next line utility header
├── srcs/                   # Source files
│   ├── pipex.c            # Main program logic
│   ├── manage_command.c   # Command execution management
│   ├── create_command.c   # Command path resolution
│   └── manage_heredoc.c   # Heredoc functionality
├── utils/                  # Utility functions
│   ├── ft_printf/         # Printf implementation
│   ├── get_next_line.c    # Line reading utility
│   ├── ft_split.c         # String splitting
│   └── libft_functions.c  # Basic string operations
├── Makefile               # Build configuration
└── README.md             # This file
```

## 🔍 Key Functions Explained

### Core Functions

1. **`main()`** - Program entry point
   - Validates command line arguments
   - Handles heredoc mode
   - Orchestrates the entire pipe process

2. **`manage_infile()`** - Input file handling
   - Opens the input file
   - Redirects stdin to the file
   - Handles file opening errors gracefully

3. **`manage_command()`** - Command execution
   - Creates pipes between commands
   - Forks child processes
   - Redirects input/output between processes

4. **`create_command()`** - Command path resolution
   - Finds the full path of commands
   - Searches in PATH environment variable
   - Returns executable path (e.g., "/bin/ls" for "ls")

### System Calls Used

- **`pipe()`** - Creates a pipe for inter-process communication
- **`fork()`** - Creates a new child process
- **`execve()`** - Replaces current process with a new program
- **`dup2()`** - Redirects file descriptors
- **`wait()`** - Waits for child processes to finish
- **`open()`** - Opens files for reading/writing

## 🔧 System Functions Deep Dive

### 1. `pipe()` - Inter-Process Communication

**Purpose**: Creates a pipe, which is a unidirectional data channel for communication between processes.

**Function Signature**:
```c
int pipe(int pipefd[2]);
```

**Arguments**:
- `pipefd[2]`: Array of two integers where:
  - `pipefd[0]` = Read end (where you read data from)
  - `pipefd[1]` = Write end (where you write data to)

**Return Value**:
- `0` on success
- `-1` on failure (sets `errno`)

**Example Usage**:
```c
int pipe_fd[2];
if (pipe(pipe_fd) == -1) {
    perror("pipe failed");
    exit(1);
}
// Now pipe_fd[0] is for reading, pipe_fd[1] is for writing
```

**How it works**:
```
Process A                         Process B
     │                               │
     │ writes to pipe_fd[1]          │ reads from pipe_fd[0]
     ▼                               ▼
┌─────────────┐             ┌─────────────┐
│   PIPE      │───────────▶│   PIPE      │
│  (buffer)   │             │  (buffer)   │
└─────────────┘             └─────────────┘
```

### 2. `fork()` - Process Creation

**Purpose**: Creates a new child process by duplicating the current process.

**Function Signature**:
```c
pid_t fork(void);
```

**Arguments**: None

**Return Value**:
- **In parent process**: Returns the PID (Process ID) of the child
- **In child process**: Returns `0`
- **On failure**: Returns `-1` (sets `errno`)

**Example Usage**:
```c
pid_t pid = fork();
if (pid == -1) {
    perror("fork failed");
    exit(1);
} else if (pid == 0) {
    // Child process code
    printf("I am the child, PID: %d\n", getpid());
} else {
    // Parent process code
    printf("I am the parent, child PID: %d\n", pid);
}
```

**What happens after fork()**:
```
 Before fork():      After fork():
┌─────────────┐     ┌─────────────┐    ┌─────────────┐
│   Parent    │     │   Parent    │    │    Child    │
│   Process   │───▶│   Process   │    │   Process   │
│             │     │             │    │             │
└─────────────┘     └─────────────┘    └─────────────┘
                    (returns child       (returns 0)
                     PID)                
```

### 3. `execve()` - Program Execution

**Purpose**: Replaces the current process with a new program. The current process "dies" and is replaced by the new program.

**Function Signature**:
```c
int execve(const char *pathname, char *const argv[], char *const envp[]);
```

**Arguments**:
- `pathname`: Full path to the executable (e.g., "/bin/ls")
- `argv[]`: Array of command-line arguments (argv[0] is usually the program name)
- `envp[]`: Array of environment variables

**Return Value**:
- **On success**: Never returns (process is replaced)
- **On failure**: Returns `-1` (sets `errno`)

**Example Usage**:
```c
char *args[] = {"ls", "-la", NULL};
char *env[] = {"PATH=/bin:/usr/bin", NULL};
execve("/bin/ls", args, env);
// If execve succeeds, this line is never reached
perror("execve failed");
```

**Important**: After `execve()` succeeds, the current process is completely replaced. Any code after `execve()` in the same process will not execute.

### 4. `dup2()` - File Descriptor Duplication

**Purpose**: Duplicates a file descriptor, making the new descriptor point to the same file/pipe as the old one.

**Function Signature**:
```c
int dup2(int oldfd, int newfd);
```

**Arguments**:
- `oldfd`: The file descriptor to duplicate
- `newfd`: The new file descriptor number

**Return Value**:
- **On success**: Returns the new file descriptor (`newfd`)
- **On failure**: Returns `-1` (sets `errno`)

**Example Usage**:
```c
// Redirect stdout to a file
int file_fd = open("output.txt", O_WRONLY | O_CREAT, 0644);
dup2(file_fd, STDOUT_FILENO);  // Now stdout writes to the file
close(file_fd);  // Close the original file descriptor
printf("This goes to output.txt\n");  // This writes to the file
```

**How it works**:
```
Before dup2():           After dup2(fd, 1):
┌─────────────┐         ┌─────────────┐
│ fd=3        │         │ fd=3        │
│ (file)      │         │ (file)      │
└─────────────┘         └─────────────┘
┌─────────────┐         ┌─────────────┐
│ fd=1        │         │ fd=1        │
│ (stdout)    │         │ (stdout)    │
└─────────────┘         └─────────────┘
                        (now points to same file)
```

### 5. `wait()` - Process Synchronization

**Purpose**: Waits for a child process to change state (terminate, stop, or continue).

**Function Signature**:
```c
pid_t wait(int *wstatus);
```

**Arguments**:
- `wstatus`: Pointer to store the exit status of the child process (can be NULL)

**Return Value**:
- **On success**: Returns the PID of the terminated child
- **On failure**: Returns `-1` (sets `errno`)

**Example Usage**:
```c
pid_t pid = fork();
if (pid == 0) {
    // Child process
    printf("Child process\n");
    exit(42);
} else {
    // Parent process
    int status;
    pid_t child_pid = wait(&status);
    printf("Child %d finished with status: %d\n", child_pid, WEXITSTATUS(status));
}
```

**Why wait() is important**:
- Prevents "zombie processes" (terminated processes that haven't been cleaned up)
- Allows parent to know when child has finished
- Synchronizes parent and child processes

### 6. `open()` - File Operations

**Purpose**: Opens a file and returns a file descriptor for reading/writing.

**Function Signature**:
```c
int open(const char *pathname, int flags, mode_t mode);
```

**Arguments**:
- `pathname`: Path to the file
- `flags`: How to open the file (O_RDONLY, O_WRONLY, O_RDWR, O_CREAT, O_TRUNC, O_APPEND, etc.)
- `mode`: File permissions (used when O_CREAT is specified)

**Common Flags**:
- `O_RDONLY`: Read only
- `O_WRONLY`: Write only  
- `O_RDWR`: Read and write
- `O_CREAT`: Create file if it doesn't exist
- `O_TRUNC`: Truncate file to zero length
- `O_APPEND`: Append to file instead of overwriting

**Return Value**:
- **On success**: Returns the file descriptor (non-negative integer)
- **On failure**: Returns `-1` (sets `errno`)

**Example Usage**:
```c
// Open for reading
int fd_read = open("input.txt", O_RDONLY);
if (fd_read == -1) {
    perror("Failed to open input.txt");
    exit(1);
}

// Open for writing (create if doesn't exist, truncate if exists)
int fd_write = open("output.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
if (fd_write == -1) {
    perror("Failed to open output.txt");
    exit(1);
}

// Open for appending
int fd_append = open("log.txt", O_WRONLY | O_CREAT | O_APPEND, 0644);
```

**File Descriptor Numbers**:
- `0`: Standard input (stdin)
- `1`: Standard output (stdout)  
- `2`: Standard error (stderr)
- `3+`: Other files/pipes

### How These Functions Work Together in Pipex

```
1. pipe(pipe_fd)     → Creates communication channel
2. fork()            → Creates child process
3. dup2()            → Redirects stdin/stdout to pipe
4. execve()          → Replaces process with new command
5. wait()            → Parent waits for child to finish
6. open()            → Opens input/output files
```

This combination allows pipex to create a pipeline where data flows from one command to the next through pipes, just like the Unix shell does.

## 🧪 Testing

### Basic Tests
```bash
# Create a test input file
echo "Hello World\nThis is a test\nHello again" > test_input.txt

# Test basic functionality
./pipex test_input.txt "cat" "grep Hello" test_output.txt

# Check the result
cat test_output.txt
```

### Advanced Tests
```bash
# Test with multiple commands
./pipex test_input.txt "cat" "grep Hello" "wc -l" result.txt

# Test heredoc
./pipex here_doc "END" "cat" "sort" sorted.txt
# Then type some lines and "END" to finish
```

## ⚠️ Error Handling

The program handles various error scenarios:
- Invalid number of arguments
- File opening failures
- Command not found
- Pipe creation failures
- Process creation failures

## 🎓 Learning Objectives

This project teaches:
- **Process management** with `fork()` and `wait()`
- **Inter-process communication** with pipes
- **File descriptor manipulation** with `dup2()`
- **Command execution** with `execve()`
- **Error handling** in system programming
- **Memory management** in C

## 🔗 Related Concepts

- **Unix Philosophy**: "Do one thing and do it well"
- **Pipeline Pattern**: Chain of processing steps
- **Process Communication**: How processes share data
- **File Descriptors**: How Unix handles I/O

## 📚 Further Reading

- [Unix Pipes](https://en.wikipedia.org/wiki/Pipeline_(Unix))
- [Process Management in Unix](https://en.wikipedia.org/wiki/Process_management_(computing))
- [File Descriptors](https://en.wikipedia.org/wiki/File_descriptor)

---

**Note**: This project is part of the 42 school curriculum and demonstrates fundamental Unix system programming concepts.