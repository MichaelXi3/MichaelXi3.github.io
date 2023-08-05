---
layout: post
title: "Unix Security"
subtitle: "A comprehensive introduction to Unix security"
date: 2023-08-04
author: "Michael Xi"
header-img: "img/ubuntu.jpeg"
tags: [Unix, Linux, Security, System]
---

# UNIX Introduction

### Operating Systems and Security

> OS security typically consists of the following **four** items
>

1. **Login and user accounts**
    - Info about users is stored in accounts
    - This includes *privileges* granted to a user
    - *Identification* and *authentication* identify a user
2. **Access control**
    - *Permissions* on resources can be set by the admin
    - Permissions will depend on the user’s identity
3. **Audits**
    - Security involves the *prevention* and *detection*
    - Mechanisms are needed to determine security violations
4. **Configuration management**
    - Updating OS based on security needs

### UNIX Background

UNIX is originally intended for one computer with multiple users

- So that user mistakes can be isolated
- The initial UNIX design didn’t have security in mind, the trust association is: If you connect to the mainframe, then I must trust you



# UNIX System & Security

### UNIX Mechanisms

![截屏2023-08-04 下午9.47.39.png](https://s2.loli.net/2023/08/05/P4bFwjroKt8iGHn.png)

- UNIX program runs as either `kernel` or `user`,  which corresponds to complete or limited control of all resources

### Runlevel

**Runlevel** is a state or mode state of the machine after boot. It defines what the computer does in a general sense.


- There are **Seven** run levels
    ![截屏2023-08-04 下午9.49.26.png](https://s2.loli.net/2023/08/05/VoiCsIcNrbwSQDG.png)

    
    > For example, our AWS VM has a run level equal to 5, so we have a graphical user interface. To check the run level, use `runlevel` in CLI.
    >

    The system’s run level gives you an idea of the computer, e.g. whether it’s a single-user computer, a server, etc.

- Consider UNIX system booting
  
    ![截屏2023-08-04 下午9.51.58.png](https://s2.loli.net/2023/08/05/PCzlqFLayXfRDJn.png)
    
    1. `init` process starts → kick off everything
    2. `init` determines the default run level (set by admin)
    3. `init` runs the scripts associated with the run level, which are located in the`/etc` directory with one directory per run level



### Login and User Accounts

#### Password

- In UNIX, users are identified by **user names**
    - Authenticated by passwords
    - Passwords are ***encrypted*** and stored in `/etc/passwd`
- In `/etc/passwd`, every user’s row looks like this:
  
    ![截屏2023-08-04 下午9.53.31.png](https://s2.loli.net/2023/08/05/7zbVoFAfKcY8BXg.png)
    
    The root user always has `UID = 0`, which provides the highest level of permissions in Unix
    
    - If the password is empty, the password is not required
    - If the password starts with an `*`, the user cannot log in
- `/etc/passwd` is readable by any user (in the good old days), but only root can write to this file
  
    - **Security risk**: user can elevate privileges by `setuid` and modify readonly files
- `/etc/passwd` and **Shell**
    - Shell is the command line environment given to the account
      
        ![截屏2023-08-04 下午9.56.01.png](https://s2.loli.net/2023/08/05/wPTpH8oyb3DLemv.png)
        
    - If the shell is `/usr/sbin/nologin` or `/bin/false` then **Cannot** log in
    - Why have an account you cannot log into?
      
        > Every process has to be run by an owner, so the admin may want to create a user to run certain processes. So that admin can kill these processes by killing the corresponding user.
        >
        1. Security: In some cases, a process or service may require access to certain system resources, such as files or network ports, but does not need a user shell or interactive login. By creating a dedicated system account for the process or service, system administrators can restrict access to the necessary resources while minimizing the risk of unauthorized access.
        2. Process isolation: Sometimes, it is necessary to create a user account to run a particular process, but it is not desirable to allow the user to log in interactively or run other programs. For example, a web server may need a dedicated user account to run its processes, but that user should not have shell access or be able to run other programs on the system.
        3. Backup purposes: Creating system accounts for backup purposes, such as for system-level backups, allows for a clean and organized environment. It also helps to ensure that no user or program can interfere with the backup process.
- **Shadow Files**
    - There is a security risk with a readable password, so people create shadow files to store hashed passwords now
    - If a shadow password file is used
        - An x is placed for the password in `/etc/passwd`
          
            ![截屏2023-08-04 下午9.57.31.png](https://s2.loli.net/2023/08/05/YMiTfOS4IJWeoym.png)
            
        - The actual encrypted password is located in `/etc/shadow` and this file is readable only by the **root**
          
            ![截屏2023-08-04 下午9.58.02.png](https://s2.loli.net/2023/08/05/jQwfHRYKP6SOk7I.png)
        
    - To crack the password, you need to crack these two files and merge them. You also can use **[John the Ripper (JTR)](https://github.com/openwall/john)** to do the cracking automatically.
- **Interpret** `/etc/shadow` file meanings
  
    ![截屏2023-08-04 下午9.59.21.png](https://s2.loli.net/2023/08/05/WZPlvX8Um61Mfn2.png)
    

#### User Types

- **User Types**
    - `Root` is superuser that has complete control of the system
        - Root has `UID = 0` and can do almost everything
        - Root **can** become any other user, change passwd file
        - Root **cannot** change system read-only file
    - `Normal user` can log in to the system with restricted access
    - `System user` cannot log in, and their accounts are used for specific system purposes.
- **UNIX User UID and Group IDs**
    - UNIX represents users by a user name and a user ID. UID is linked to user name in the passwd file
      
        ![截屏2023-08-05 上午12.48.12.png](https://s2.loli.net/2023/08/05/52XyVo8KgJLAeti.png)
        
    - **Group ID**: Collecting users in groups for access control
      
        ![截屏2023-08-05 上午12.49.22.png](https://s2.loli.net/2023/08/05/ipSVWMRvHk6rGaw.png)
        

#### **Discussion about Root Privilege Control**

- `sudo`, a more controlled method to share root. `sudo = superuser do`
  
    - `sudo` can specify the allowed privileged commands → prevent sudo - all or nothing (more granularity)
      
        ![截屏2023-08-05 上午12.50.19.png](https://s2.loli.net/2023/08/05/81sOTAuWfxPNLSI.png)
    
- `sudoers.d` is a directory containing files specifying the permissions per group or user
  
    - The file path is `/etc/sudoers.d`, and the idea is the same as `sudoers`, we just place entries in separate files
    - This change was done to prevent mistakes while updating `sudo` permissions, The file `sudoers` still exist even with this update
    
    ![截屏2023-08-05 上午12.54.10.png](https://s2.loli.net/2023/08/05/1WdZaqhpo3F2ly4.png)
    
- **Controlled Invocation →** a more careful way of giving privilege than sudo
  
    - UNIX may occasionally need superuser privilege to perform certain tasks
    - The solution is **Set User ID** (`SUID`) and **Set Group ID** (`SGID`) **programs**
        - **Programs** that are SUID give a user a different status during their execution, then return to normal once complete
        - The list of programs that has **SUID** bit turned on is: (You can also write your own `setuid()` program)
          
            ![截屏2023-08-05 上午12.56.05.png](https://s2.loli.net/2023/08/05/zhmPWQygtoI86rX.png)
            
            In UNIX, the **`perm -4000`** option is used as the **`find`** command searches for files with the "`setuid`" bit set. This special permission bit allows a program to be executed with the privileges of the file owner, rather than the user executing the program. It is used to give users temporary elevated privileges to perform certain tasks requiring special permissions.
            
        
    - A program that sets the UID looks like this:
      
        ![截屏2023-08-05 上午12.57.24.png](https://s2.loli.net/2023/08/05/VqGQO471SYKAljy.png)
        
        - SUID program is very dangerous in terms of security, because people could hack you during the program is root, such as spawn a shell.

### UNIX File Structure

- UNIX arranges files in a **tree** structure.
    - The base of the tree is called the root `/`
    - The system consists of files and directories
    - Every directory contains two files `.` and `..`
    - Every file has an owner and belongs to a group
- For each file, the representation is:
  
    ![截屏2023-08-05 上午1.00.51.png](https://s2.loli.net/2023/08/05/GybrlBWRQjoKtH4.png)
    
    - First character indicates if the file is a directory `d`
    - `rwx` means read, write, and execute
- If the file is a **SUID** program, then the representation is:
  
    ![截屏2023-08-05 上午1.02.33.png](https://s2.loli.net/2023/08/05/6zWtLcQJ14XlT9p.png)
    
    - The reason why `s` is used in place of `x` when the `setuid` or `setgid` bit is set on a file is to indicate that the execute bit is implicitly set along with the special bit
- To change the permissions of a file, use the command `chmod`
    - Add permission → `chmod who+perm fileName`
    - Remove permission → `chmod who-perm fileName`
- Check detailed information about file via `stat` command
  
    ![截屏2023-08-05 上午1.05.45.png](https://s2.loli.net/2023/08/05/CtxXEsbv6T23UFf.png)
    

### UNIX Command Shell

#### Bash Environment

Bash keeps track of settings and details through an area it maintains called the ***environment***.

- Environment is an area that is built every time that a session is started
- Contains variables that define system properties
- To display the variables, use the command `env` or `printenv`

- Every time a shell session started:
    - A process takes place to gather and compile information that should be available to the shell process and its child processes

#### Environment Variable and Shell Variable

- What is **Environment Variable**?
  
    Environment variables are named dynamic values (often set by OS)
    
    - Should be considered ***hidden*** **input**, and it may affact program behavior
    - Ignoring how they are set is dangerous
    - When executed, a program may use these variables for system access
        - Such as `PATH`, `IFS`, `LD_LIBRARY_PATH`, and `LD_PRELOAD`
    
    	![截屏2023-08-05 上午1.08.32.png](https://s2.loli.net/2023/08/05/vfinLQjZPFEm1Xu.png)
    
    	To check current environment variables, use `printenv` command.
    
- How to create Environment Variable?
  
    - Environment variables can be created from local shell variables by `export`
      
        ![截屏2023-08-05 上午1.10.06.png](https://s2.loli.net/2023/08/05/4MmKeDCUAhrHxGb.png)
        
    - You can also create environment variable using `env` command
      
        ![截屏2023-08-05 上午1.10.36.png](https://s2.loli.net/2023/08/05/e43cTLzJuxw6WZy.png)
    
- What is **Bash Variable** or **Shell Variable** (a more general term)?

    Bash is a command line interface for the OS, and is also a programming/scripting language. Bash allows using `=` to declare variables
    
    - These variables are used to manage **ephemeral** data, like current working directory
    - They are contained within the shell in which they were set or defined
      
        ![截屏2023-08-05 上午1.12.20.png](https://s2.loli.net/2023/08/05/uqJBbvkFLOIpcxS.png)
        
    - What is the **difference** between **Environmental variable and Bash variable**?
      
        **Environment variables** are system-wide variables that are set and used by all programs running on a machine, while **bash variables** are specific to the shell or command being executed. 
        
        Environment variables can be set by the operating system or by users, and they provide information about the system and its configuration. Bash variables, on the other hand, are used to store information that is specific to a particular shell or command, such as the output of a command or a user-defined value. Bash variables are not accessible outside of the shell or command in which they are defined, while environment variables can be accessed by any program running on the machine.
        
    - What is the difference between **Bash and Shell variable**?
      
        A shell variable and a bash variable are essentially the same thing, but they are often used in different contexts. "Shell variable" is a more general term that refers to a variable within any Unix-like shell (e.g., sh, bash, zsh, ksh, etc.), while "bash variable" specifically refers to a variable within the Bash (Bourne Again Shell) environment.
        
        Both types of variables are used to store and manipulate data in the shell environment, and they can be set, accessed, and modified using shell scripting. However, the distinction is mainly based on the context in which they are used.
        
        - Shell variable: A variable used within any Unix-like shell environment. It is a more general term.
        - Bash variable: A variable specifically used within the Bash shell environment.

#### Shell Shock

- Bash environment variables can also **export function definitions**
  
    ![截屏2023-08-05 上午1.17.32.png](https://s2.loli.net/2023/08/05/MxjNPvVZcgTGX3H.png)
    
    - If a variable string starts with `() {`, then this is a function
- The cases of shell shock
  
    ![截屏2023-08-05 上午1.19.09.png](https://s2.loli.net/2023/08/05/YOiKl2JPudNkVq9.png)
    
    - For the third command, `echo bar` should not be executed at all
    - Thus, this bash issue allows arbitrary command execution

#### PATH

- Users interact with the OS through a shell, such as `zsh`, `bash`, etc.
- As a matter of convenience users can type run a program command without specifying the complete path. For example, to run `ls`, you don’t need to type `/bin/ls`.
    - The shell knows where to look using a PATH variable
- The PATH environment variable lists the directories to search and it’s defined in `.cshrc` or `.bashrc` file depending on the shell you use and located at the home directory.
    - The **order matters**! It determines the commands look-up sequence of the shell.
      
        ![截屏2023-08-05 上午1.21.13.png](https://s2.loli.net/2023/08/05/bEhpr8qR5GvWoHu.png)
        

**Path Attack**

> Using path attack to trick system admin to get the root shell. Assume system admin has `.` at the beginning of their PATH, so the shell is looking for commands at local first.
>

1. Create a file called `ls` with the following content
   
    ```bash
    cp /bin/sh ./Stuff/sume
    chmod 4555 ./Stuff/sume
    rm -f $0
    exec /bin/ls $1
    ```
    
    1. **`cp /bin/sh ./Stuff/sume`**: Copies the **`/bin/sh`** file to **`./Stuff/sume`**.
    2. **`chmod 4555 ./Stuff/sume`**: Changes the permissions of the **`./Stuff/sume`** file to allow it to run with elevated privileges.
       
        ![截屏2023-08-05 上午1.23.37.png](https://s2.loli.net/2023/08/05/EU3CeSMgZLuox8F.png)
        
    3. **`rm -f $0`**: Deletes the script file after it has been run.
    4. **`exec /bin/ls $1`**: Runs the **`/bin/ls`** command with the specified argument, if any.
2. Create another unreadable file with whatever content
   
    ```bash
    # create a file named -f, since admin may try rm -f
    chmod 700 .; touch ./-f
    ```
    
3. You tell the admin you cannot erase one of your files (`-f`)
4. Admin becomes root moves to your directory and enters `ls`, then your customized version of `ls` is executed
5. You now have root for `sume`, a root shell interpreter
   ```bash
   > ./Stuff/sume
   ```
   



**`LD_LIBRARY_PATH` and `LD_PRELOAD`**

- `LD_LIBRARY_PATH`: this environment variable specifies a list of directories where the dynamic linker should look for dynamically linked library files (.so files in Unix/Linux), in addition to the standard locations (like /lib and /usr/lib).
  
    - If these libraries are exchanged with malware, attackers can place a fake libc.so to user’s machine and get executed everytime
      
        ```bash
        # /tmp will be searched first for libc.so
        setenv LD_LIBRARY_PATH /tmp:$LD_LIBRARY_PATH
        ```
        
    - Most C runtime libraries have fixed this issue by ignoring the `LD_LIBRARY_PATH` var when the EUID (Process ID) is not equal to the UID (User ID)
- `LD_PRELOAD`: this variable is used to load a specific library (or libraries) before any others when starting a program, even if they are not explicitly requested by the program. This allows you to override functions in other shared libraries.

    - This allows the user to replace standard C library functions, such as the C interfaces to system calls
      
        ![截屏2023-08-05 下午2.41.28.png](https://s2.loli.net/2023/08/06/nwqvFyJuaxdM5rf.png)
- `LD_LIBRARY_PATH` changes where the system looks for libraries, while `LD_PRELOAD` changes the order in which they are loaded.
        

#### IFS Environment Variable

- IFS stands for **Internal Field Separators**, which identifies characters to be interpreted as whitespace
- IFS malicious usage:
  
    ![截屏2023-08-05 下午2.47.39.png](https://s2.loli.net/2023/08/06/KnhP65bOikAgwJS.png)
    

**Vi + PATH _ IFS Attack**

![截屏2023-08-05 下午2.48.15.png](https://s2.loli.net/2023/08/06/O2UrpJ8W4c6u7Xq.png)
    

#### Avoid Environment Variables

In C and C++, the `environ` variable is an external pointer variable that is part of the standard library. It is used to access the environment of the current process, which includes a list of environment variables and their values.


It possible to avoid environment variables in the program by set external pointer variable `environ` to zero (NULL).

![截屏2023-08-05 下午2.49.09.png](https://s2.loli.net/2023/08/06/ukMPT4C9H2dsWt6.png)

### Session Configuration Files

Once a user logs in, a session starts.

- Session is configured using a set of hidden files
- These configuration files are located in the home directory

	![截屏2023-08-05 下午2.50.24.png](https://s2.loli.net/2023/08/06/jMAoWs9cptqIGUB.png)


- Shell environment is configured using `.bashrc` and `.config` files
    - File `.config` executed per login
    - File `.bashrc` executed for each new terminal as new shell starts
- Let take a close look at `.bashrc` file
  
    ![截屏2023-08-05 下午2.51.28.png](https://s2.loli.net/2023/08/06/FqQcahNRwu9M3tA.png)
    
    - `alias` is a place to hide commands

### Root Processes

- Three Processes
    - **Initial boot** uses scripts in `/etc/rc*.d` depends on the system runlevel
        - In this boot script directory, for example, `/etc/rc5.d`, is a directory that contains symbolic links to scripts in the `/etc/init.d` directory
    - `/etc/init.d` is a directory that stores the **init scripts** responsible for managing **system services**. These scripts are typically executed by the system init processes.
    - Network processes `/etc/inetd.conf` sets the list of network services started at boot
- Commands
    - `ps` or `top`: show what is running
      
        ![截屏2023-08-05 下午2.53.10.png](https://s2.loli.net/2023/08/06/9rWKGQjJTuh24Cg.png)
        
    - `ps -ef` will list the Parent Process ID (**PPID**)
      
        ![截屏2023-08-05 下午2.53.23.png](https://s2.loli.net/2023/08/06/rSoHguxk4jeE9G1.png)
        
    - `ps` shows the current process running and we can determine if we started them

- **init** Services
    - Started automatically during the boot or runlevel change, it locates in `/etc/init.d` or `/etc/rc.d/init/d`
    
      ![截屏2023-08-05 下午2.55.48.png](https://s2.loli.net/2023/08/06/EiJULxytp3ZArRa.png)
      
    - To start, restart, or stop a service, use following command
      
      ![截屏2023-08-05 下午2.56.08.png](https://s2.loli.net/2023/08/06/3iPkoDQhpdJAHMg.png)
      
    
- **inetd** Services
    - This service is used for Internet based services and generally started via init scripts
        - `inetd.conf` file contents
          
            ```bash
            seed@ip-10-4-149-82:/etc$ cat inetd.conf 
            # /etc/inetd.conf:  see inetd(8) for further informations.
            #
            # Internet superserver configuration database
            #
            # Lines starting with "#:LABEL:" or "#<off>#" should not
            # be changed unless you know what you are doing!
            #
            # If you want to disable an entry so it isn't touched during
            # package updates just comment it out with a single '#' character.
            #
            # Packages should modify this file by using update-inetd(8)
            #
            # <service_name> <sock_type> <proto> <flags> <user> <server_path> <args>
            #
            #:INTERNAL: Internal services
            #discard		stream	tcp	nowait	root	internal
            #discard		dgram	udp	wait	root	internal
            #daytime		stream	tcp	nowait	root	internal
            #time		stream	tcp	nowait	root	internal
            
            #:STANDARD: These are standard services.
            telnet		stream	tcp	nowait	telnetd	/usr/sbin/tcpd	/usr/sbin/in.telnetd
            
            #:BSD: Shell, login, exec and talk are BSD protocols.
            
            #:MAIL: Mail, news and uucp services.
            
            #:INFO: Info services
            
            #:BOOT: TFTP service is provided primarily for booting.  Most sites
            #       run this only on machines acting as "boot servers."
            
            #:RPC: RPC based services
            
            #:HAM-RADIO: amateur-radio services
            
            #:OTHER: Other services
            #<off># sane-port	stream	tcp	nowait	saned:saned	/usr/sbin/saned saned
            ```
        
    - `netstat` command: stands for network statistics. It displays information about active network connections, including the local and remote addresses, the protocol used (TCP or UDP), the state of the connection (e.g. established, listening, etc.), and the process ID (PID) of the application or service that is associated with the connection.
      
        ```bash
        # shows all active TCP connections 
        netstat -atn
        ```
        
    - Common **options** for the `netstat` command include:
        - **`a`** : displays all active connections, including listening ports
        - **`n`** : displays network addresses as numeric values rather than resolving them to host and domain names
        - **`p`** : displays the process ID and name of the program that is associated with each connection
        - **`r`** : displays the system's routing table

### Cron Job

`cron` is a program that allows Unix users to automatically execute commands or scripts (a cron job) at specific dates and times. Examples of cron jobs include deleting temporary files every hour, restarting ssh, or backing up files every day.

- `cron` is a daemon process, so it’s dormant until it’s required

	![截屏2023-08-05 下午3.01.54.png](https://s2.loli.net/2023/08/06/7quvTpdcarIBjkn.png)

- **How to control cron jobs?**
    - Go to directory `/etc`, it contains `cron` sub-directories
      
        ![截屏2023-08-05 下午3.02.37.png](https://s2.loli.net/2023/08/06/4u2oYC9xiOnJUM3.png)
        
        - Each sub-dir has a repeat time associated with it
        - You can place a script in the sub-dir you want
        - An **Example** of daily script
          
            ![截屏2023-08-05 下午3.02.48.png](https://s2.loli.net/2023/08/06/IGkEKZ2rTV8QjyS.png)
            
        - More script time flexibility is provided in the `/etc/crontab` file
          
            ![截屏2023-08-05 下午3.02.58.png](https://s2.loli.net/2023/08/06/Tx1UKH2pnkXb4mc.png)
            

### What is Running?

> **`systemctl`** is a command-line tool used to manage and control the **`systemd`** system and service manager in modern Linux distributions.

- `systemctl start service-name`: Starts a specific service.
- `systemctl stop service-name`: Stops a specific service.
- `systemctl restart service-name`: Restarts a specific service.
- `systemctl enable service-name`: Enables a specific service to start at boot time.
- `systemctl disable service-name`: Disables a specific service from starting at boot time.
- `systemctl status service-name`: Displays the status of a specific service.
- `systemctl list-units`: Lists all active units (services, targets, sockets, etc.) on the system.
- `systemctl list-unit-files`: Lists all unit files (configuration files) installed on the system.

```bash
# List all active system services
systemctl list-units --type=service --state=running
```

![截屏2023-08-05 下午3.04.48.png](https://s2.loli.net/2023/08/06/EbCh5n2lT1PQag4.png)

---

> **`pstree`** displays the running processes in tree format.

![截屏2023-08-05 下午3.11.23.png](https://s2.loli.net/2023/08/06/15okvlqJnrfShCI.png)

---

> **`netstat -a`** inspects network connectivity.

![截屏2023-08-05 下午3.12.37.png](https://s2.loli.net/2023/08/06/6WdzHS1NIqrhimv.png)

- Unix domain sockets are **interprocess** communication

### Audit Trails

Linux keeps a good audit trail of happening on the system.

- **Things are logged**: crashes, login/logout, su, login failures, dialouts printer use/errors, mail/www, firewall exceptions
- **Things are not logged**: failed file access, some network access (rcp), file changes, password changes, www access failures, and superuser actions

- **`syslog()`**
    - Logging is done by `syslogd` **daemon**, and it has UDP support
    - The file `/etc/rsyslog.conf` indicates log locations
      
        > *selector + action*
        >
        >
        > ![截屏2023-08-05 下午3.14.53.png](https://s2.loli.net/2023/08/06/pIcAv8Q6wHNY7PK.png)
        >

#### Intrusion Detection System

- To check all authentication attempts
  
    ```bash
    sudo grep "sshd" /var/log/auth.log
    ```
    
    ![截屏2023-08-05 下午3.16.59.png](https://s2.loli.net/2023/08/06/tN2wld4XeRChOEx.png)
    

### Make Unix/Linux More Secure

1. Set a password to disallow booting from USB, DVD, etc…
   
    > This is done by setting a BIOS or UEFI password, which requires a password to be entered before the system can boot from removable media such as USB drives or DVDs. This helps prevent unauthorized access to the system or the installation of malware from external sources.
    >
2. Disable special and default accounts
   
    > Unix/Linux systems come with several default accounts, including **`lp`**, **`sync`**, **`shutdown`**, **`operator`**, and others. These accounts are typically used for system processes and are not intended for general user access. It is recommended to disable these accounts or change their passwords to prevent unauthorized access.
    >
    >
    > ```bash
    > # Locks the lp account, preventing anyone from logging in that account.
    > sudo usermod -L lp
    > ```
    >
3. Set a timeout (TMOUT=) in the profile file (users and root)
4. Disable console-equivalent user access to reboot, shutdown
5. Disable unnecessary network services (with caution)
    1. Edit the `/etc/inetd.conf` file
    2. make certain root is the owner of `/etc.inetd.conf`
    3. Change permission of `/etc/inetd.conf` to 600 (read and write permissions for the owner of the file and no permissions for the group and others.)
6. Use `TCP_WRAPPERS` and/or firewall to control access
7. Do not display the system issue information at log in
8. Review the `/etc/host.conf` file
    1. This file resolves IP names to addresses
    2. You can make it ask DNS first or have the file resolve
9. Immunize `/etc/services` file, the file that maps ports to applications
    1. Prevent unauthorized deletion or addition of services
    2. Use the command `chattr +i /etc/services`
10. Make signatures of any setuid programs, such as MD5 signature
11. Prevent users from `su` , should do `sudo` instead
12. Have the shell erase the command history of a user at log out, can be done with `/bash_logout` file
13. Disable the Control-Alt-Delete keyboard shutdown command
14. Fix permissions under `/etc/rc.d/init.d` direcotry for script files
    1. Use the command `chmod -R 700 /etc/rc.d/init.d/*`
15. Turn off the machine and don’t turn it on ever
