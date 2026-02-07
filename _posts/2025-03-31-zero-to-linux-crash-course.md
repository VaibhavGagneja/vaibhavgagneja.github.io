---
title: "Zero to Linux: An Intensive Crash Course"
description: A comprehensive guide to Linux for DevOps engineers, from basics to advanced
author: Vaibhav Gagneja
date: 2025-03-31 09:19:00 +0000
categories: [DevOps, Linux]
tags: [linux, terminal, commands, devops, tutorial]
image:
  path: https://images.unsplash.com/photo-1629654297299-c8506221ca97?q=80&w=774&auto=format&fit=crop&ixlib=rb-4.1.0&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D
---

## Introduction

This is a comprehensive guide to Linux for DevOps engineers, covering topics from basic to advanced.

### Internet Basics

* The internet works through **optical fibers or fiber cables** laid underwater connecting **data centers**.
* **Data centers** are large locations housing numerous computers for data storage and transmission.
* When you request data, it travels through these fiber cables from the data center to your device.

### Servers

A **server** is fundamentally a computer designed **to serve information**:

* **Email Server**: Serves emails
* **File Server**: Stores files and provides access
* **Database Server**: Manages databases
* **Application Server**: Runs applications
* **Web Server**: Serves static content

### Introduction to Linux

**Linux** is an **operating system** that is largely **open source** and **free** to use.

Multiple ways to use Linux:
* **Dual booting** alongside Windows
* **WSL (Windows Subsystem for Linux)**
* **Virtual machine** (e.g., VirtualBox)
* **Cloud platform** (AWS, Azure, GCP)
* **Vagrant** by HashiCorp

### Boot Process

1. Power button pressed
2. CPU and hard disk energized
3. **Boot loader** (GRUB) runs
4. Kernel loads
5. System processes start

### Linux Process States

1. **Running (R)**: Currently executing on CPU
2. **Interruptible Sleep (S)**: Waiting for an event
3. **Uninterruptible Sleep (D)**: Waiting for critical I/O
4. **Stopped (T)**: Suspended by signal
5. **Zombie (Z)**: Terminated but parent hasn't read exit status

---

## Basic Commands

```bash
date              # Current date and time
ls                # List files
ls -l             # Detailed list
mkdir <name>      # Create directory
pwd               # Present working directory
touch <file>      # Create empty file
clear             # Clear terminal
cd <dir>          # Change directory
cd ..             # Go up one level
cd                # Go to home
cd /              # Go to root
```

## File Manipulation

```bash
rm <file>           # Remove file
rm -r <dir>         # Remove directory recursively
rmdir <dir>         # Remove empty directory
cp <src> <dest>     # Copy file
cp -r <src> <dest>  # Copy directory
mv <src> <dest>     # Move/rename
```

## File Content

```bash
cat <file>          # Display content
head <file>         # First 10 lines
tail <file>         # Last 10 lines
tail -f <file>      # Follow in real-time
less <file>         # Page by page view
wc <file>           # Count lines, words, chars
echo "text" > file  # Write to file
```

## Links

```bash
ln -s <src> <link>  # Soft/symbolic link
ln <src> <link>     # Hard link
```

---

## Remote Access

```bash
ssh -i <key.pem> <user>@<host>
```

## Disk Usage

```bash
df -h               # Disk space (human-readable)
du .                # Directory usage
ls -a               # Show hidden files
```

## Processes

```bash
ps                  # Snapshot of processes
top                 # Real-time process view
kill <PID>          # Send signal to process
kill -9 <PID>       # Force kill
fuser <file>        # Find process using file
```

## Memory

```bash
free -h             # Memory usage
```

## Background Processes

```bash
nohup <command> &   # Run in background
```

## System Information

```bash
uname -a            # System info
uptime              # System uptime
who                 # Logged in users
whoami              # Current user
which <cmd>         # Command path
id                  # User/group IDs
```

---

## User Management

```bash
sudo useradd -m <user>    # Add user
sudo passwd <user>        # Set password
su <user>                 # Switch user
sudo userdel -r <user>    # Delete user
```

## Group Management

```bash
sudo groupadd <group>           # Create group
groups                          # List user's groups
cat /etc/group                  # All groups
sudo groupdel <group>           # Delete group
sudo usermod -aG <group> <user> # Add user to group
```

---

## File Permissions

Format: `-rwxrwxrwx` (owner, group, others)

* `r` = 4 (read)
* `w` = 2 (write)
* `x` = 1 (execute)

```bash
chmod 755 <file>    # rwxr-xr-x
chmod 644 <file>    # rw-r--r--
chmod 700 <file>    # rwx------
```

## Ownership

```bash
sudo chown <user> <file>   # Change owner
sudo chgrp <group> <file>  # Change group
```

## Compression

```bash
zip -r <archive.zip> <dir>   # Create zip
unzip <archive.zip>          # Extract zip
tar cvf file.tar *.c         # Create tar
tar xvzf file.tar.gz         # Extract tar.gz
```

## Secure Copy

```bash
scp -i <key> <src> <user>@<host>:<dest>
scp -r -i <key> <dir> <user>@<host>:<dest>  # Recursive
```

## Rsync

```bash
rsync -avz -e "ssh -i <key>" <src> <user>@<host>:<dest>
```

---

## Networking Commands

```bash
sudo apt install net-tools
netstat -tuln       # Network connections
ifconfig            # Network interfaces
ping <host>         # Test connectivity
host <domain>       # DNS lookup
traceroute <host>   # Trace path
mtr <host>          # Combined ping+traceroute
```

## Text Processing

```bash
grep <pattern> <file>     # Search for pattern
awk '{print $1}' <file>   # Process text
sed 's/old/new/g' <file>  # Stream editor
watch -n 2 <command>      # Repeat command
```

---

## Package Management (Debian/Ubuntu)

```bash
sudo apt update           # Update package list
sudo apt upgrade          # Upgrade packages
sudo apt install <pkg>    # Install package
sudo apt remove <pkg>     # Remove package
```

---

## Conclusion

This crash course covered Linux essentials from internet basics to advanced commands. Practice these commands regularly to become proficient in Linux administration!
