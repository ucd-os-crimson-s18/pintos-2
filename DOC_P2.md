# CSCI 3453		    
# PROJECT 2: USER PROGRAMS	
# DESIGN DOCUMENT     	

# GROUP

Eric Holguin Eric.Holguin@ucdenver.edu

Jeff McMillan Jeff.McMillan@ucdenver.edu

Nikki Yesalusky Nikki.Yesalusky@ucdenver.edu


# PRELIMINARIES 

None


# ARGUMENT PASSING

## DATA STRUCTURES

###### A1: Copy here the declaration of each new or changed `struct` or `struct` member, global or static variable, `typedef`, or enumeration.  Identify the purpose of each in 25 words or less.

`char *token` - Used for the strtok_r function to hold the string

`char_count` - Holds the count of characters of the arguments

`int argc` - Holds the count of arguments

`char * argv[128]` - Used to store the argument address


## ALGORITHMS ----

###### A2: Briefly describe how you implemented argument parsing.  How do you arrange for the elements of `argv[]` to be in the right order? How do you avoid overflowing the stack page?

Inside the setup_stack, we set up our argument passing, once the page
is successfully installed.

In order to implement argument passing we needed to extract each argument 
from the file and the add them onto the stack.
We used the already implemented ‘strtok_r’ function inside ‘string.c’ to extract
the arguments, delimited by spaces. Which were then pushed into the kernel stack by 
subtracting, from the stack pointer, the length of each string. Since the stack pointer is set to 
PHYS_BASE, we used a reverse subtraction in the user stack to increment the stack pointer there.

Afterwards using the character count we kept track of, we word aligned the stack pointer 
so it would be 4 bytes exactly and then pushed 4 bytes of null. We then used an
array, which we pushed the stack pointer addresses with earlier, to reference the previously 
pushed arguments and store those onto the stack but in reverse order so the last argument 
would be on top and the executable name on the bottom.

Finally, we pushed the address to argv, the address of each of the arguments, 
and a fake return address.
We kept our argument limit to 128 bytes to avoid overflowing the stack page.


## RATIONALE ----

###### A3: Why does Pintos implement `strtok_r()` but not `strtok()`?

With `strtok_r()` you can call this from multiple threads simultaneously or in nested loops 
because they take the extra argument to store the state betwen calls, while `strtok()` would use
a global variable which might give undefined behavior.

###### A4: In Pintos, the kernel separates commands into a executable name and arguments.  In Unix-like systems, the shell does this separation. Identify at least two advantages of the Unix approach.

The Unix approach is better to keep the kernel free from malicious code and to verify the existence of and to validate said arguments.



# SYSTEM CALLS

## DATA STRUCTURES----
###### B1: Copy here the declaration of each new or changed `struct` or `struct` member, global or static variable, `typedef`, or enumeration. Identify the purpose of each in 25 words or less.

###### Structs
```C
/* INSIDE PROCCESS.H */
struct child_process
    {
      pid_t pid;                          /* pid of child */
      enum process_status status;         /* Process state */
      int exit_status;                    /* Exit code passed from exit()*/
      char *args;                         /* Args passed to thread_create*/
      struct semaphore child_dead;        /* Synch dying of child (wait) */
      struct list_elem child_elem;        /* Parent uses to add to its child list */
      struct thread *parent;              /* Parent of new child */
     };
     
/* INSIDE THREAD.H & THREAD STRUCT */    
    struct thread * parent;          /* Child's parent thread */       
    struct list children_list;       /* List to hold the children */
    struct child_process * cp_ptr;   /* Pointer to child process struct */
    struct semaphore child_load;     /* Synch loading of child */
    struct list files;               /* List to hold files */
    struct file *exe;                /* Pointer to executable file */

/* INSIDE SYSCALL.H */    
    struct thread * parent;          /* Child's parent thread */       
    struct list children_list;       /* List to hold the children */
    struct child_process * cp_ptr;   /* Pointer to child process struct */
    struct semaphore child_load;     /* Synch loading of child */
    struct list files;               /* List to hold files */
    struct file *exe;                /* Pointer to executable file */
    
    struct file_desc
    {
        int fd;                       /* Int to hold the file descriptor */
        tid_t tid;                    /* thread id to get thread */
        struct file * f;              /* Pointer to a file struct */
        struct list_elem fd_elem;     /* file descriptor list element */
        struct list_elem thread_elem; /* thread list element */
    };

struct list open_files;   /* list of open files */
struct lock filesys_lock; /* lock for the file system */
```

###### Functions

``` C
/* INSIDE SYSCALL.H */  

void syscall_halt (void);
void syscall_exit (int);
pid_t syscall_exec (const char *);
int syscall_wait (pid_t);
bool syscall_create (const char *, unsigned);
bool syscall_remove (const char *);
int syscall_open (const char *);
int syscall_filesize (int);
int syscall_read (int, void *, unsigned);
int syscall_write (int, void *, unsigned);
void syscall_seek (int, unsigned);
unsigned syscall_tell (int);
void syscall_close (int);

static bool check_ptr(void *, uint8_t);
static int get_user (const uint8_t *);
static bool put_user (uint8_t *udst, uint8_t byte);
static int increment_fd(void);
struct file_desc *find_open_file(int);
```

###### B2: Describe how file descriptors are associated with open files. Are file descriptors unique within the entire OS or just within a single process?

File descriptors numbered 0 and 1 are reserved for the console: fd 0 (STDIN_FILENO) is
standard input, fd 1 (STDOUT_FILENO) is standard output. Each process has an independent 
set of file descriptors. When a single file is opened more than once, whether by a single 
process or different processes, each open returns a new file descriptor. Different file 
descriptors for a single file are closed independently in separate calls to close and 
they do not share a file position.

## ALGORITHMS 

###### B3: Describe your code for reading and writing user data from the kernel.

We setup a function `static bool check_ptr(void *, uint8_t);` which
returns true if pointer is valid or false if the pointer is invalid.
We pass a `void *` for the address and `uint8_t` for the size.
Inside the function we have a for loop to iterate through the address 
for the size passed in. We check each address if it is in user memory using
the `is_user_vaddr` function inside ‘vaddr.h’ and checking if the address is 
mapped using the `pagedir_get_page` inside ‘pagedir.h’. 

###### B4: Suppose a system call causes a full page (4,096 bytes) of data to be copied from user space into the kernel.  What is the least and the greatest possible number of inspections of the page table (e.g. calls to pagedir_get_page()) that might result?  What about for a system call that only copies 2 bytes of data?  Is there room for improvement in these numbers, and how much?

We can have at least 1024 inspections and at most 4096 inspections for the 4096 bytes and 
for 2 bytes at least 0 at most 1. There can be improvement using the ‘get_user’ and ‘put_user’
functions provided in the Pintos PDF.

###### B5: Briefly describe your implementation of the "wait" system call and how it interacts with process termination.
Inside our `process_wait(tid child_tid)` we iterate through our ‘children_list’ belonging 
to a thread in order to find the corresponding child thread given as an argument. We then down
the semaphore belonging to the child process and proceed to remove the child from the thread's
children list and set the children's status.
This interacts with process termination because this semaphore is upped inside `process_exit()`
so the child can wait until the process is terminated. And inside our `syscall_wait` we do something similar and get the child from the thread and remove it from the list and set the children status.

###### B6: Any access to user program memory at a user-specified address can fail due to a bad pointer value.  Such accesses must cause the process to be terminated.  System calls are fraught with such accesses, e.g. a "write" system call requires reading the system call number from the user stack, then each of the call's three arguments, then an arbitrary amount of user memory, and any of these can fail at any point.  This poses a design and error-handling problem: how do you best avoid obscuring the primary function of code in a morass of error-handling?  Furthermore, when an error is detected, how do you ensure that all temporarily allocated resources (locks, buffers, etc.) are freed?  In a few paragraphs, describe the strategy or strategies you adopted for managing these issues.  Give an example.

Our implementation used the `check_ptr()` function discussed above in order to avoid obscuring the primary function of code and using a switch statement we called the `check_ptr()` function before accessing the memory and inside each’ syscall’ function we also used this function to make sure the memory was valid. This made sure our ‘syscall’ implementations were cleaner. It was in our `process_exit()` a lot of the clean up took place and also assuring anywhere we allocated memory we had a free statement.

## SYNCHRONIZATION 

###### B7: The "exec" system call returns -1 if loading the new executable fails, so it cannot return before the new executable has completed loading.  How does your code ensure this?  How is the load success/failure status passed back to the thread that calls "exec"?

Our code ensures by having a semaphore that allows for synchronization based on load's success. This semaphore is downed inside `process_execute()` and upped inside `proccess_load()`, we then use a boolean variable to pass into `process_execute` if the load was successful or not.

###### B8: Consider parent process P with child process C.  How do you ensure proper synchronization and avoid race conditions when P calls wait(C) before C exits?  After C exits?  How do you ensure that all resources are freed in each case?  How about when P terminates without waiting, before C exits?  After C exits?  Are there any special cases?
If any direct child of process P is called to wait then the kernel must return either the process waiting or a termination and reason. This happens whether the child is alive or dead. All of a process’s resources, including its struct thread, must be freed whether its parent ever waits for it or not, and regardless of whether the child exits before or after its parent, thus free is called wherever there is a resource allocated.

## RATIONALE 

###### B9: Why did you choose to implement access to user memory from the kernel in the way that you did?

We chose to implement accessing user memory using method 1 from the pdf because we found this more intuitive and simpler than method 2, originally we tried to implement method 2 but it didn’t seem to work and we were struggling with it so we then decided to change it to a simpler function call and check if it was within user memory space and whether it was mapped which worked more in our favor.

###### B10: What advantages or disadvantages can you see to your design for file descriptors?
Our design for file descriptors uses a struct to hold the necessary variable to handle the file system calls. We have two lists being used outside of our struct with list elements inside the struct. We can get the file descriptors and corresponding threads from these list elements. The advantage is that it is a simple design and easy to understand, but a disadvantage may be that it is slow which could possibly be implemented using hash tables to provide faster access.

###### B11: The default tid_t to pid_t mapping is the identity mapping. If you changed it, what advantages are there to your approach?
We did not change this.
