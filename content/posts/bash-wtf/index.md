---
date: "2025-07-15T20:22:23+07:00"
draft: false
title: "Bash WTF?"
summary: "Dive into some fascinating Bash commands and features."
categories:
  - Code
tags:
  - bash
  - wtf
---

Let's explore some cool bash commands. ok?

### Command Substitution

```sh
# Standard command substitution
echo $(ls)

# Backtick variant (older and not recommended)
echo `ls`
```

Learn more: [GNU Bash Manual - Command Substitution](https://www.gnu.org/software/bash/manual/html_node/Command-Substitution.html)

### Brace Expansion

```sh
echo {a,b,c}
```

Learn more: [GNU Bash Manual - Brace Expansion](https://www.gnu.org/software/bash/manual/html_node/Brace-Expansion.html)

### Process Substitution

```sh
# Compare directory contents
diff <(ls dir1) <(ls dir2)

# Example with a command
grep foo <(echo foo)

# Example with a file
grep foo <(cat file)

# Input redirection
echo "foo" > >(cat)
```

Learn more: [GNU Bash Manual - Process Substitution](https://www.gnu.org/software/bash/manual/html_node/Process-Substitution.html)

### Associative Arrays (Dictionaries)

```sh
#!/bin/bash

# Declare an associative array
declare -A fruits

# Add key-value pairs
fruits[apple]="green"
fruits[banana]="yellow"
fruits[cherry]="red"

# Access and print values
echo "The color of an apple is ${fruits[apple]}"
echo "The color of a banana is ${fruits[banana]}"
echo "The color of a cherry is ${fruits[cherry]}"

# Iterate over keys and values
for fruit in "${!fruits[@]}"; do
    echo "$fruit is ${fruits[$fruit]}"
done
```

Learn more: [GNU Bash Manual - Arrays](https://www.gnu.org/software/bash/manual/html_node/Arrays.html)

### Indexed Arrays

```sh
# Declare an indexed array
arr=(a b c)

# Access elements
echo "First element: ${arr[0]}"
echo "Second element: ${arr[1]}"
echo "Third element: ${arr[2]}"

# Iterate over elements
for element in "${arr[@]}"; do
    echo "Element: $element"
done
```

Learn more: [GNU Bash Manual - Arrays](https://www.gnu.org/software/bash/manual/html_node/Arrays.html)

### Here Documents (Heredoc)

```sh
cat <<EOF
This is a here document.
It will continue until it sees EOF.
EOF
```

```sh
while read -r id; do
  # some commands using $id
done <<< "$ids"
```

Learn more: [GNU Bash Manual - Here Documents](https://www.gnu.org/software/bash/manual/html_node/Here-Documents.html)

### Grouping Commands with \{\}

```sh
{ echo hi; echo there; } > test.txt
```

### Truncates, create empty file

```sh
: > readme.txt
```

---

That's it for now. Happy scripting!
