# Linux Commands

**Calendar View**

```
cal oct 2030
```

**Date formatting**
```
date '+%F'
```

**Print numbered lines from a file**
```
nl file.txt
```

**Extract fields** *(-d delimeter -f field/column)*
```
cut -d : -f 1 /etc/passwd
```

**Find files and its type bigger than 10MB modified in last 24 hrs**
```
find / -type f -size +10M -mtime -1 -exec stat {} \;
```

**To manipulate history and run commands from history**
```
!42
!!
!export
or use Ctrl+R for Reverse Search
```

**Create quick backup file**
```
cp file.txt{,.bak}
```

**Find top of something by frequency** *(-c uniq count -n sort numeric -r sort reverse)*
```
awk '{print $1}' something.log | sort | uniq -c | sort -nr | head 
```

**Most common words** *(-c translate complement -s squeeze repeats)*

Read from stdin(file) and sqeeze (multiple occurrences to 1) all non alpha numeric chars and replace with newline. Output: Every group of words/numbers on a new line. 
```
tr -cs '[:alnum:]' '\n' < file | tr A-Z a-z | sort | uniq -c | sort -nr | head
```

**Find trailing whitespaces**
```
grep -R "[[:blank:]]$" .
```

**Make multiple subdirectories or files**
```
mkdir -p labs/{logs,data,scritps}
touch labs/{Readme.md,.env}
```



#### test.log format
*DATE TIME ERROR_LEVEL USER IP MSG*

**Print specific lines of a file** *(NR line number, $0 whole line)*
```
awk 'NR>=2 && NR<=5 {print NR, $0}' test.log
```

**Replace string on line range 2-4**
```
sed '2,4/s/old/new/g' test.log
```

**Count unique lines**
```
uniq -c test.log
```

**Sort based on time (2nd column) numerically in reverse order**
```
sort -k2nr test.log
```

**Count no. of lines per log level**
```
awk '{count[$3]++} END {for (level in count) print level, count[level]}' test.log
```

**Change output field separator in awk**
```
awk -F ' ' 'OFS=" - " {print $4, $5}' test.log
```

**Insert line before matching pattern**
```
sed '/ERROR/i\---ERROR ALERT---' test.log
```

**In-place replace and backup file**
sed -i.bak 's/old/new/g' test.log

**Find 'ERROR/error' occurrences in all .log files** *(-print0 -0 null termination i.e. filenames with spaces, + for batch operations)*
```
find . -name '*.log' -print0 | xargs -0 -I {} | grep -c -i 'ERROR' {}
find . -name '*.log' -exec grep -c -i 'ERROR' {} +
```

**Count unique IP Addresses** *(-o only matching -e Expanded Regex)*
```
awk '/[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+/ {print $1}' access.log | sort | uniq -c
grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}' access.log | sort | uniq -c
```

**Remove vowels from usernames (4th column)** 
```
awk 'gsub(/[aeiou]/,"",$4); {print $4}' test.log
```