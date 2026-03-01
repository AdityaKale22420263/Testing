# Shell Scripting Cheatsheet

## Basic Navigation & File Operations

### Directory Navigation
```bash
pwd                    # Print working directory
cd /path/to/dir       # Change directory
cd ~                  # Go to home directory
cd ..                 # Go up one directory
cd -                  # Go to previous directory
ls                    # List files
ls -la                # List all files with details (including hidden)
ls -lh                # List with human-readable file sizes
```

### File Operations
```bash
touch file.txt        # Create empty file
mkdir dirname         # Create directory
mkdir -p dir1/dir2    # Create nested directories
cp file1 file2        # Copy file
cp -r dir1 dir2       # Copy directory recursively
mv file1 file2        # Move/rename file
rm file               # Remove file
rm -r dirname         # Remove directory recursively
rm -rf dirname        # Force remove (be careful!)
cat file              # Display file contents
less file             # View file with pagination
head -n 10 file       # Show first 10 lines
tail -n 10 file       # Show last 10 lines
tail -f file          # Follow file updates (logs)
wc -l file            # Count lines in file
```

---

## File Searching & Text Processing

### Finding Files
```bash
find . -name "*.txt"              # Find files by name
find . -type f -name "*.log"      # Find files only
find . -type d -name "dirname"    # Find directories only
find . -mtime -7                  # Files modified in last 7 days
find . -size +100M                # Files larger than 100MB
locate filename                   # Quick file search (uses database)
which command                     # Find command location
whereis command                   # Find binary, source, manual
```

### Text Search (grep)
```bash
grep "pattern" file               # Search for pattern
grep -i "pattern" file            # Case-insensitive search
grep -r "pattern" dir/            # Recursive search in directory
grep -v "pattern" file            # Inverse match (exclude)
grep -n "pattern" file            # Show line numbers
grep -c "pattern" file            # Count matches
grep -E "regex" file              # Extended regex
grep -A 3 "pattern" file          # Show 3 lines after match
grep -B 3 "pattern" file          # Show 3 lines before match
```

### Text Manipulation
```bash
# sed (stream editor)
sed 's/old/new/' file             # Replace first occurrence per line
sed 's/old/new/g' file            # Replace all occurrences
sed -i 's/old/new/g' file         # Edit file in-place
sed -n '10,20p' file              # Print lines 10-20
sed '/pattern/d' file             # Delete lines matching pattern

# awk (pattern scanning)
awk '{print $1}' file             # Print first column
awk '{print $1,$3}' file          # Print columns 1 and 3
awk -F: '{print $1}' /etc/passwd  # Custom delimiter
awk '$3 > 100' file               # Print lines where col 3 > 100
awk 'NR==10,NR==20' file          # Print lines 10-20

# cut (column extraction)
cut -d: -f1 /etc/passwd           # Extract first field
cut -c1-10 file                   # Extract characters 1-10

# sort & uniq
sort file                         # Sort lines
sort -r file                      # Reverse sort
sort -n file                      # Numeric sort
sort -k2 file                     # Sort by second column
uniq file                         # Remove duplicate lines
sort file | uniq -c               # Count unique lines
```

---

## File Permissions & Ownership

```bash
# Permission notation: rwxrwxrwx (owner, group, others)
# r=4, w=2, x=1

chmod 755 file        # rwxr-xr-x
chmod 644 file        # rw-r--r--
chmod +x script.sh    # Add execute permission
chmod -w file         # Remove write permission
chmod u+x file        # Add execute for user
chmod g-w file        # Remove write for group
chmod o+r file        # Add read for others

chown user:group file # Change owner and group
chown user file       # Change owner only
chgrp group file      # Change group only
chown -R user:group dir/  # Recursive change

# Special permissions
chmod u+s file        # Set SUID (runs as owner)
chmod g+s dir         # Set SGID (inherit group)
chmod +t dir          # Sticky bit (only owner can delete)
```

---

## System Information

```bash
# System Info
uname -a              # All system information
uname -r              # Kernel version
hostname              # System hostname
uptime                # System uptime
whoami                # Current user
id                    # User ID and groups
who                   # Logged in users
w                     # Who is logged in and what they're doing

# Hardware & Resources
free -h               # Memory usage (human-readable)
df -h                 # Disk space usage
du -sh directory      # Directory size
du -h --max-depth=1   # Size of subdirectories
lsblk                 # List block devices
lscpu                 # CPU information
lspci                 # PCI devices
lsusb                 # USB devices

# OS Information
cat /etc/os-release   # OS details
cat /proc/cpuinfo     # Detailed CPU info
cat /proc/meminfo     # Detailed memory info
```

---

## Process Management

```bash
# Viewing Processes
ps                    # Current shell processes
ps aux                # All processes (detailed)
ps aux | grep nginx   # Find specific process
top                   # Real-time process monitor
htop                  # Better process monitor (if installed)
pgrep processname     # Find process ID by name
pidof processname     # Get PID of running program

# Managing Processes
kill PID              # Terminate process (SIGTERM)
kill -9 PID           # Force kill (SIGKILL)
killall processname   # Kill all processes by name
pkill processname     # Kill processes by name pattern

# Background Jobs
command &             # Run in background
jobs                  # List background jobs
fg %1                 # Bring job 1 to foreground
bg %1                 # Resume job 1 in background
Ctrl+Z                # Suspend current process
nohup command &       # Run immune to hangups

# Process Priority
nice -n 10 command    # Start with lower priority
renice -n 5 -p PID    # Change priority of running process
```

---

## Networking Commands

```bash
# Network Information
ifconfig              # Network interfaces (older)
ip addr               # Show IP addresses (modern)
ip link               # Show network interfaces
hostname -I           # Get IP address
route -n              # Routing table (older)
ip route              # Routing table (modern)

# Network Testing
ping google.com       # Test connectivity
ping -c 4 google.com  # Ping 4 times only
traceroute google.com # Trace route to host
mtr google.com        # Better traceroute (if installed)

# Port & Connection Info
netstat -tuln         # Active connections (older)
ss -tuln              # Active connections (modern)
lsof -i :80           # What's using port 80
nmap localhost        # Scan ports (if installed)

# File Transfer
scp file user@host:/path     # Secure copy to remote
scp user@host:/path/file .   # Secure copy from remote
rsync -avz source/ dest/     # Sync directories
wget url              # Download file
curl url              # Transfer data from URL
curl -O url           # Download and save file
```

---

## Package Management

```bash
# Debian/Ubuntu (apt)
apt update                    # Update package list
apt upgrade                   # Upgrade packages
apt install package           # Install package
apt remove package            # Remove package
apt search package            # Search for package
apt list --installed          # List installed packages

# RedHat/CentOS (yum/dnf)
yum update                    # Update packages
yum install package           # Install package
yum remove package            # Remove package
dnf install package           # Fedora/newer systems

# Arch Linux (pacman)
pacman -Syu                   # Update system
pacman -S package             # Install package
pacman -R package             # Remove package
```

---

## Compression & Archives

```bash
# tar (tape archive)
tar -czf archive.tar.gz dir/  # Create gzip compressed tar
tar -xzf archive.tar.gz       # Extract gzip tar
tar -cjf archive.tar.bz2 dir/ # Create bzip2 compressed tar
tar -xjf archive.tar.bz2      # Extract bzip2 tar
tar -tf archive.tar.gz        # List contents

# gzip/gunzip
gzip file             # Compress file (removes original)
gunzip file.gz        # Decompress file
gzip -k file          # Compress keeping original

# zip/unzip
zip -r archive.zip dir/       # Create zip archive
unzip archive.zip             # Extract zip archive
unzip -l archive.zip          # List zip contents
```

---

## Shell Scripting Basics

### Shebang & Basics
```bash
#!/bin/bash           # Script interpreter (always first line)

# Comments start with #

# Make script executable
chmod +x script.sh

# Run script
./script.sh
bash script.sh
```

### Variables
```bash
# Declaration
name="John"           # No spaces around =
age=25

# Usage
echo $name            # Print variable
echo ${name}          # Same, clearer syntax
echo "Hello, $name"   # Variable in string

# Command substitution
current_date=$(date)
files=$(ls)
num_files=`ls | wc -l`  # Old style (backticks)

# Special variables
$0                    # Script name
$1, $2, ...          # Positional arguments
$#                    # Number of arguments
$@                    # All arguments as separate words
$*                    # All arguments as single string
$?                    # Exit status of last command
$$                    # Current process ID
$!                    # PID of last background process

# Environment variables
export VAR="value"    # Make variable available to child processes
echo $PATH            # Common env variable
echo $HOME
echo $USER
```

### User Input
```bash
# Read input
read -p "Enter name: " name
echo "Hello, $name"

# Read with timeout
read -t 5 -p "Quick! Enter something: " input

# Read password (hidden)
read -sp "Password: " password
```

### Conditionals
```bash
# If statement
if [ condition ]; then
    # commands
elif [ condition ]; then
    # commands
else
    # commands
fi

# Test conditions
if [ -f file ]; then echo "File exists"; fi
if [ -d dir ]; then echo "Directory exists"; fi
if [ -z "$var" ]; then echo "Variable is empty"; fi
if [ -n "$var" ]; then echo "Variable is not empty"; fi

# String comparison
if [ "$str1" = "$str2" ]; then echo "Equal"; fi
if [ "$str1" != "$str2" ]; then echo "Not equal"; fi

# Numeric comparison
if [ $num1 -eq $num2 ]; then echo "Equal"; fi
if [ $num1 -ne $num2 ]; then echo "Not equal"; fi
if [ $num1 -gt $num2 ]; then echo "Greater than"; fi
if [ $num1 -lt $num2 ]; then echo "Less than"; fi
if [ $num1 -ge $num2 ]; then echo "Greater or equal"; fi
if [ $num1 -le $num2 ]; then echo "Less or equal"; fi

# Logical operators
if [ condition1 ] && [ condition2 ]; then echo "AND"; fi
if [ condition1 ] || [ condition2 ]; then echo "OR"; fi
if [ ! condition ]; then echo "NOT"; fi

# Modern test (double brackets)
if [[ $var == pattern* ]]; then echo "Pattern match"; fi
if [[ $num > 10 ]]; then echo "Greater"; fi
```

### Loops
```bash
# For loop
for i in 1 2 3 4 5; do
    echo "Number: $i"
done

# For loop with range
for i in {1..10}; do
    echo $i
done

# For loop C-style
for ((i=0; i<10; i++)); do
    echo $i
done

# For loop over files
for file in *.txt; do
    echo "Processing $file"
done

# While loop
counter=0
while [ $counter -lt 10 ]; do
    echo $counter
    ((counter++))
done

# Until loop (opposite of while)
until [ $counter -eq 10 ]; do
    echo $counter
    ((counter++))
done

# Read file line by line
while IFS= read -r line; do
    echo "Line: $line"
done < file.txt
```

### Functions
```bash
# Function definition
function greet() {
    echo "Hello, $1!"
}

# Alternative syntax
greet() {
    echo "Hello, $1!"
}

# Call function
greet "John"

# Function with return value
add() {
    local sum=$(($1 + $2))
    echo $sum
}

result=$(add 5 3)
echo "Result: $result"

# Return exit status
check_file() {
    if [ -f "$1" ]; then
        return 0  # Success
    else
        return 1  # Failure
    fi
}

if check_file "myfile.txt"; then
    echo "File exists"
fi
```

### Arrays
```bash
# Declaration
arr=(apple banana cherry)
numbers=(1 2 3 4 5)

# Access elements
echo ${arr[0]}        # First element
echo ${arr[@]}        # All elements
echo ${#arr[@]}       # Array length

# Add elements
arr+=("date")

# Iterate array
for item in "${arr[@]}"; do
    echo $item
done

# Iterate with index
for i in "${!arr[@]}"; do
    echo "Index $i: ${arr[$i]}"
done
```

### String Operations
```bash
# Length
str="Hello World"
echo ${#str}          # 11

# Substring
echo ${str:0:5}       # Hello
echo ${str:6}         # World

# Replace
echo ${str/World/Universe}  # Replace first occurrence
echo ${str//l/L}            # Replace all occurrences

# Upper/lowercase (Bash 4+)
echo ${str^^}         # HELLO WORLD
echo ${str,,}         # hello world

# Trim
str="  hello  "
trimmed=$(echo $str | xargs)  # Remove whitespace
```

### Arithmetic
```bash
# Basic arithmetic
num=$((5 + 3))        # 8
num=$((10 - 2))       # 8
num=$((4 * 2))        # 8
num=$((16 / 2))       # 8
num=$((17 % 2))       # 1 (modulo)

# Increment/Decrement
((num++))
((num--))
((num += 5))

# Using let
let "num = 5 + 3"

# Using expr (old style)
num=$(expr 5 + 3)

# Floating point (use bc)
echo "scale=2; 10 / 3" | bc  # 3.33
```

---

## Common Operators & Redirections

### Redirection
```bash
command > file        # Redirect stdout to file (overwrite)
command >> file       # Redirect stdout to file (append)
command 2> file       # Redirect stderr to file
command &> file       # Redirect both stdout and stderr
command 2>&1          # Redirect stderr to stdout
command < file        # Use file as stdin
command1 | command2   # Pipe: output of cmd1 to input of cmd2

# Examples
ls > files.txt
ls >> files.txt
ls nonexistent 2> errors.txt
ls &> all_output.txt
grep "pattern" < input.txt
ls | grep ".txt"
```

### Command Chaining
```bash
command1 ; command2   # Run sequentially (always run both)
command1 && command2  # Run cmd2 only if cmd1 succeeds
command1 || command2  # Run cmd2 only if cmd1 fails

# Examples
mkdir test && cd test
rm file.txt || echo "File not found"
```

---

## Useful Shortcuts & Tips

### Command Line Shortcuts
```bash
Ctrl+A          # Move to beginning of line
Ctrl+E          # Move to end of line
Ctrl+U          # Delete from cursor to beginning
Ctrl+K          # Delete from cursor to end
Ctrl+W          # Delete word before cursor
Ctrl+L          # Clear screen
Ctrl+R          # Search command history
Ctrl+C          # Cancel current command
Ctrl+Z          # Suspend current process
Ctrl+D          # Exit shell / EOF
!!              # Repeat last command
!$              # Last argument of previous command
!n              # Execute command number n from history
history         # Show command history
```

### Common Patterns
```bash
# Find and delete files
find . -name "*.log" -delete

# Find and execute command on each file
find . -name "*.txt" -exec grep "pattern" {} \;

# Count files in directory
ls -1 | wc -l

# Disk usage sorted
du -sh */ | sort -h

# Find large files
find . -type f -size +100M -exec ls -lh {} \;

# Monitor log file
tail -f /var/log/syslog

# Replace text in multiple files
find . -name "*.txt" -exec sed -i 's/old/new/g' {} \;

# Create backup with timestamp
cp file.txt file.txt.$(date +%Y%m%d_%H%M%S)

# Quick HTTP server (Python)
python3 -m http.server 8000

# Check if port is open
nc -zv localhost 8080

# Generate random password
openssl rand -base64 12

# Find processes using most memory
ps aux --sort=-%mem | head -10

# Find processes using most CPU
ps aux --sort=-%cpu | head -10
```

---

## Script Best Practices

```bash
#!/bin/bash

# Use set flags for safer scripts
set -e          # Exit on error
set -u          # Exit on undefined variable
set -o pipefail # Exit on pipe failure
set -x          # Debug mode (print commands)

# Check if script is run as root
if [ "$EUID" -ne 0 ]; then
    echo "Please run as root"
    exit 1
fi

# Check number of arguments
if [ $# -lt 1 ]; then
    echo "Usage: $0 <argument>"
    exit 1
fi

# Validate file exists
if [ ! -f "$1" ]; then
    echo "Error: File $1 not found"
    exit 1
fi

# Use quotes around variables
echo "$variable"    # Good
echo $variable      # Bad (word splitting issues)

# Check command success
if command_that_might_fail; then
    echo "Success"
else
    echo "Failed"
    exit 1
fi

# Cleanup on exit
cleanup() {
    rm -f /tmp/tempfile
}
trap cleanup EXIT

# Create temporary file safely
tmpfile=$(mktemp)
```

---

## Quick Reference: File Test Operators

```bash
-e file     # Exists
-f file     # Is regular file
-d file     # Is directory
-L file     # Is symbolic link
-r file     # Is readable
-w file     # Is writable
-x file     # Is executable
-s file     # File has size > 0
-z string   # String is empty
-n string   # String is not empty
```

---

## Environment & Configuration

```bash
# View environment
env                   # All environment variables
printenv              # Same as env
echo $PATH            # View specific variable

# Set temporarily
export VAR="value"

# Set permanently (add to ~/.bashrc or ~/.bash_profile)
echo 'export VAR="value"' >> ~/.bashrc
source ~/.bashrc      # Reload configuration

# Important files
~/.bashrc             # Bash configuration (interactive shells)
~/.bash_profile       # Bash login configuration
~/.bash_history       # Command history
/etc/profile          # System-wide configuration
/etc/environment      # System environment variables
```

---

## Common Real-World Examples

```bash
# Batch rename files
for file in *.txt; do
    mv "$file" "${file%.txt}.md"
done

# Find and archive old log files
find /var/log -name "*.log" -mtime +30 -exec gzip {} \;

# Monitor system resources
watch -n 1 'df -h; free -h'

# Create dated backup directory
backup_dir="backup_$(date +%Y%m%d)"
mkdir "$backup_dir"
cp -r /important/data "$backup_dir/"

# Kill all processes matching pattern
pkill -f "python script.py"

# Find recently modified files
find . -type f -mmin -60  # Last hour

# Count lines of code
find . -name "*.py" -exec wc -l {} + | awk '{total += $1} END {print total}'

# Convert file to lowercase
tr '[:upper:]' '[:lower:]' < input.txt > output.txt

# Remove duplicate lines while preserving order
awk '!seen[$0]++' file.txt

# Check if service is running
if systemctl is-active --quiet nginx; then
    echo "Nginx is running"
fi

# Parallel processing
cat urls.txt | xargs -P 4 -I {} curl -O {}
```

---

## Debugging Scripts

```bash
# Run with debug output
bash -x script.sh

# Add to script
set -x              # Enable debugging
set +x              # Disable debugging

# Check syntax without running
bash -n script.sh

# Use shellcheck (if installed)
shellcheck script.sh
```

This cheatsheet covers the essential commands and concepts you'll need for shell scripting. Practice these regularly!
