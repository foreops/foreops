---
title: "Using here documents in a Pinch"
description: "Imagine you're working in a Kubernetes environment and need to troubleshoot an issue with a PHP application running in one of your pods. You exec into the pod to investigate, but to your surprise, there are no text editors installed. This is common in container images designed to reduce attack surfaces and resource usage. However, you need to create a PHP file to test some code changes. You can do that with  here documents."
summary: "Imagine you're working in a Kubernetes environment and need to troubleshoot an issue with a PHP application running in one of your pods. You exec into the pod to investigate, but to your surprise, there are no text editors installed. This is common in container images designed to reduce attack surfaces and resource usage. However, you need to create a PHP file to test some code changes. You can do that with  here documents."
lead: "Imagine you're working in a Kubernetes environment and need to troubleshoot an issue with a PHP application running in one of your pods. You exec into the pod to investigate, but to your surprise, there are no text editors installed. This is common in container images designed to reduce attack surfaces and resource usage. However, you need to create a PHP file to test some code changes. You can do that with  here documents."
date: 2024-07-13T21:16:16-04:00
lastmod: 2024-07-13T22:16:16-04:00
draft: false
weight: 50
categories: []
tags: ["here documents","heredocs","shell","devops","bash","commandline","curl"]
contributors: ["Sharjeel Aziz"]
images: ["here-docs.webp"]
pinned: false
homepage: true
seo:
  title: "" # custom title (optional)
  description: "" # custom description (recommended)
  canonical: "" # custom canonical URL (optional)
  noindex: false # false (default) or true
---

```bash
cat << 'EOF' > test.php
<?php
echo "Hello, World!";
phpinfo();
?>
EOF
```

The `cat << 'EOF' > test.php` command starts a here document. `EOF` is the delimiter that indicates the start and end of the document. Using single quotes around `EOF` ensures that any variables within the document are not expanded by the shell. The content between `<< 'EOF'` and `EOF` is written to the file `test.php`.

## Understanding here documents

{{< img src="here-docs.webp" alt="Image showing a terminal window with a here doc" caption="" class="wide" >}}
<br/>
The concept of "here documents" originates from the early days of UNIX. Ken Thompson developed the first shell for UNIX called the V6 shell in 1971. The shell introduced a compact syntax for redirection (`< >` and `>>`) and piping (`|` or `^`) that has survived into modern shells. You can also find support for invoking sequential commands (with `;`) and asynchronous commands (with `&`). Bash, short for Bourne-Again SHell, developed by Stephen Bourne at Bell Labs, was a replacement for the Thompson shell. This shell incorporated a number of features we use today, including command substitution (using back quotes) and HERE documents to embed preserved string literals within a script. See [Evolution of shells in Linux](https://developer.ibm.com/tutorials/l-linux-shells/#a-history-of-shells0). Early [teletype machines](http://www.k7tty.com/development/teletype/model-32/index.html) had a "here is" key to send RTTY identification (a programmable set of codes) and the term may have originated from that.

**Minimum requirements to create here documents:**
Here documents are created using shell and require the `<<` operator followed by a delimiter. The content of the document is written on the lines that follow, and the document is closed with the delimiter. You can suppress leading tabs by using `<<-`

**Basic syntax:**
The basic syntax for a here document is as follows:

```bash
command << delimiter
here document content
delimiter
```

The delimiter can be any string, commonly `EOF` or `END`, and it indicates where the content of the here document begins and ends. `EOF` (End Of File) is a commonly used marker, but you can use almost any string as a marker, provided it does not appear in the text block.

**Preventing variable expansion:** 
To prevent [variable expansion](https://www.gnu.org/software/bash/manual/html_node/Shell-Parameter-Expansion.html), use single quotes around the delimiter.

```bash
name="John"
cat << 'EOF'
Hello, $name!
This will print $name literally.
EOF
```

**Redirecting to a file:**
You can redirect the output of a here document into a file.

```bash
cat << EOF > output.txt
This is a here document.
It will be written to a file.
EOF
```

The beauty of `here documents` lies in their simplicity and flexibility. They are particularly useful in several scenarios, including but not limited to:

- **Configuration files creation:** Quickly create configuration files for software applications directly from the command line.

- **Scripting:** Pass multi-line strings or scripts to languages like Python, PHP, or SQL directly from the shell.

- **Instructions or notes:** Generate files containing instructions, notes, or even templated text without needing an editor.

## Practical examples

### Create multiple YAML objects from stdin
When working with Kubernetes, you might need to create multiple resources at once. Here documents can help streamline this process:

```bash
# Create multiple YAML objects from stdin
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: busybox-sleep
spec:
  containers:
  - name: busybox
    image: busybox:1.28
    args:
    - sleep
    - "1000000"
---
apiVersion: v1
kind: Pod
metadata:
  name: busybox-sleep-less
spec:
  containers:
  - name: busybox
    image: busybox:1.28
    args:
    - sleep
    - "1000"
EOF
```

The `-` tells kubectl to read from stdin.

### Send email using netcat
Send email using basic command line tools. In the example below, all the commands will be piped into netcat one by one including the here document.

```bash
(sleep 1
echo "HELO localhost"
sleep 1
echo "MAIL FROM:<from@example.com>"
sleep 1
echo "RCPT TO:<to@example.com>"
sleep 1
echo "DATA"
sleep 1
cat << EOF
From: from@example.com
To: to@example.com
Subject: Here Documents

Sent using a here document and SMTP commands. All the command between the () will be piped into netcat one by one including this document.
.
EOF
sleep 1
echo "QUIT") | nc 192.168.85.195 25

```

### Add file extension to a list of files
If you need to process a list of files and add an extension to each one, here documents can help:

```bash
awk '{
  print  $0".gz"
}' << EOF
file1
file2
file3
EOF
```

### Creating a configuration file
Here documents make it easy to generate configuration files without needing a text editor:

```bash
cat << EOF > config.conf
[server]
host = localhost
port = 8080

[database]
user = admin
password = secret
EOF
```

### Creating a configuration file with values from environment variables
In this example of a templated text file, a `webapp.config` will be created with values from the  environment variables.

```bash
cat << EOF > webapp.config
[database]
host = $DB_HOST
port = $DB_PORT
user = $DB_USER
password = $DB_PASS

[server]
host = 0.0.0.0
port = 8080
EOF
```

### Feeding SQL commands to MySQL
You can use here documents to run multiple SQL commands at once:

```bash
mysql -u root -p some_database <<EOF
CREATE TABLE Persons (
    PersonID int,
    LastName varchar(255),
    FirstName varchar(255),
    Address varchar(255),
    City varchar(255)
);
EOF
```

### Writing scripts on the fly
Here documents can write scripts directly into a file or feed them into an interpreter:  

```bash
python3 << EOF
print("Hello from Python!")
for i in range(3):
    print(f"Line {i+1}")
EOF
```

### Passing here documents to a function
You can pass multi-line input to a function using here documents:

```bash
process_input() {
  echo "User List"
  while IFS=: read -r user enpass uid gid desc home shell
  do
    # only display if UID >= 200 
    [ $uid -ge 200 ] && echo "User $user ($uid) assigned \"$home\" home directory with $shell shell."
  done
}

process_input << EOF
lp:*:11:11::/var/spool/lp:/bin/false
invscout:*:200:1::/var/adm/invscout:/usr/bin/ksh
nuucp:*:6:5:uucp login user:/var/spool/uucppublic:/usr/sbin/uucp/uucico
paul:!:201:1::/home/paul:/usr/bin/ksh
jdoe:*:202:1:John Doe:/home/jdoe:/usr/bin/ksh
EOF
```

### Displaying transfer information with curl
You can make curl display detailed transfer information on stdout using the `-w` (write-out) option after a completed transfer. The format is a string that can mix plain text with any number of variables. You can specify this format as a literal string, read it from a file using @filename, or read it from stdin using `@-`. The following command is self contained and does not require one to create a separate text file first.

Here's is an example of a command that fetches the specified URL and prints out various timing details related to the transfer, making it easy to analyze the performance of different stages in the transfer process:

```bash
curl -w "@-" -o NUL -s https://foreops.com/blog/understanding-and-implementing-dora-metrics/ <<'EOF'
namelookup:    %{time_namelookup}s\n
connect:       %{time_connect}s\n
appconnect:    %{time_appconnect}s\n
pretransfer:   %{time_pretransfer}s\n
redirect:      %{time_redirect}s\n
starttransfer: %{time_starttransfer}s\n
———————————————\n
time_total:    %{time_total}s\n
EOF
```

The `here document` is an excellent example  of the Unix's philosophy of making complex tasks manageable with simple commands. Whether you're scripting, managing configuration files, or just need to jot down notes, understanding and utilizing `heredocs` can significantly streamline your workflow. Experiment with them, and you'll soon find them an indispensable part of your command-line toolkit.

**References:**
* [Evolution of shells in Linux by M. Jones](https://developer.ibm.com/tutorials/l-linux-shells/#a-history-of-shells0)
* [Teletype Model 28 KSR](https://kb8ojh.net/station/teletype/)
* [ksh - An Extensible High Level Language" by David G. Korn](https://www.in-ulm.de/~mascheck/bourne/korn.html)
* [Rosetta Code](https://rosettacode.org/wiki/Here_document#UNIX_Shell)
