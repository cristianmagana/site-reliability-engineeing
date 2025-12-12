# Bash File Processing - Quick Reference

## For Loops

### Basic For Loops
```bash
# Loop over numbers
for i in 1 2 3 4 5; do
  echo "$i"
done

# Loop with range (brace expansion)
for i in {1..10}; do
  echo "Number: $i"
done

# Loop with step
for i in {0..100..10}; do
  echo "$i"  # 0, 10, 20, 30...
done

# C-style for loop
for ((i=0; i<10; i++)); do
  echo "$i"
done

# Loop over array
servers=("web1" "web2" "db1")
for server in "${servers[@]}"; do
  echo "Connecting to $server"
done

# Loop over files in directory
for file in /var/log/*.log; do
  echo "Processing $file"
done

# Loop over command output (use with caution - prefer while read for files)
for user in $(cut -d: -f1 /etc/passwd); do
  echo "User: $user"
done

# Loop over glob pattern
for yaml in *.yaml; do
  kubectl apply -f "$yaml"
done
```

### Nested Loops
```bash
# Nested for loops
for i in {1..3}; do
  for j in {a..c}; do
    echo "$i$j"
  done
done

# Loop over 2D array
for row in "1,2,3" "4,5,6" "7,8,9"; do
  IFS=, read -ra cols <<< "$row"
  for col in "${cols[@]}"; do
    echo -n "$col "
  done
  echo
done
```

### Break and Continue
```bash
# Break out of loop
for i in {1..10}; do
  if [[ $i -eq 5 ]]; then
    break
  fi
  echo "$i"
done

# Skip iteration
for i in {1..10}; do
  if [[ $i -eq 5 ]]; then
    continue
  fi
  echo "$i"  # Prints all except 5
done
```

---

## Loop Through File Lines

### The Correct Way
```bash
# Basic loop - preserves whitespace, handles special characters
while IFS= read -r line; do
  echo "$line"
done < file.txt

# With line numbers
while IFS= read -r line || [[ -n "$line" ]]; do
  echo "$((++count)): $line"
done < file.txt

# Parse CSV
while IFS=, read -r col1 col2 col3; do
  echo "Name: $col1, Age: $col2, City: $col3"
done < data.csv

# Skip header line
while IFS=, read -r name age city; do
  echo "$name is $age years old"
done < <(tail -n +2 data.csv)
```

### Common Patterns
```bash
# Count lines matching pattern
count=0
while IFS= read -r line; do
  [[ $line =~ ERROR ]] && ((count++))
done < log.txt
echo "Errors: $count"

# Filter lines
while IFS= read -r line; do
  [[ $line =~ ^# ]] && continue  # Skip comments
  echo "$line"
done < config.txt

# Process in parallel (limit 10 concurrent)
jobs=0
while IFS= read -r url; do
  curl -s "$url" &
  ((++jobs % 10 == 0)) && wait
done < urls.txt
wait
```

---

## sed - Stream Editor

### Basic Substitution
```bash
# Replace first occurrence per line
sed 's/old/new/' file.txt

# Replace all occurrences (global)
sed 's/old/new/g' file.txt

# Case-insensitive
sed 's/old/new/gi' file.txt

# Different delimiter (useful for paths)
sed 's|/home/user|/data/user|g' file.txt

# In-place edit (with backup)
sed -i.bak 's/old/new/g' file.txt
```

### Line Addressing
```bash
# Specific line number
sed -n '5p' file.txt           # Print line 5
sed '5d' file.txt              # Delete line 5
sed '5s/old/new/' file.txt     # Replace only on line 5

# Line range
sed -n '10,20p' file.txt       # Print lines 10-20
sed '10,20d' file.txt          # Delete lines 10-20

# Pattern matching
sed -n '/ERROR/p' file.txt     # Print lines with ERROR (like grep)
sed '/^#/d' file.txt           # Delete comment lines
sed '/^$/d' file.txt           # Delete empty lines

# Pattern range
sed -n '/START/,/END/p' file.txt   # From START to END
```

### Capture Groups
```bash
# Extract version number: "Version: 1.2.3" → "1.2.3"
echo "Version: 1.2.3" | sed 's/Version: \([0-9.]*\)/\1/'

# Swap two words
echo "hello world" | sed 's/\(.*\) \(.*\)/\2 \1/'

# Reformat phone: 1234567890 → (123) 456-7890
echo "1234567890" | sed 's/\([0-9]\{3\}\)\([0-9]\{3\}\)\([0-9]\{4\}\)/(\1) \2-\3/'

# Extract IP addresses from logs
sed -n 's/.*\([0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\).*/\1/p' log.txt
```

### Multiple Commands
```bash
# Semicolon separator
sed 's/foo/bar/; s/baz/qux/' file.txt

# Multiple -e flags
sed -e 's/foo/bar/' -e 's/baz/qux/' file.txt

# Script file
cat > script.sed <<'EOF'
s/\t/    /g     # Tabs to spaces
s/ *$//         # Remove trailing whitespace
/^$/d           # Delete blank lines
EOF
sed -f script.sed file.txt
```

---

## awk - Columnar Data Processing

### Field Access
```bash
# Print specific columns
awk '{print $1, $3}' file.txt       # Columns 1 and 3
awk '{print $NF}' file.txt          # Last column
awk '{print $(NF-1)}' file.txt      # Second-to-last

# Print all except first column
awk '{$1=""; print $0}' file.txt

# Custom delimiter
awk -F, '{print $1, $2}' file.csv   # CSV
awk -F: '{print $1}' /etc/passwd    # Colon-separated
awk -F'\t' '{print $1}' file.tsv    # Tab-separated
```

### Pattern Matching
```bash
# Lines matching pattern
awk '/ERROR/' log.txt               # Like grep ERROR

# Field matches value
awk '$2 == "ERROR"' log.txt         # Column 2 equals ERROR
awk '$3 > 100' data.txt             # Column 3 > 100
awk '$2 ~ /^ERROR/' log.txt         # Column 2 starts with ERROR

# Multiple conditions
awk '$2 == "ERROR" && $3 > 100' log.txt
awk '$2 == "ERROR" || $2 == "WARN"' log.txt
```

### Aggregation
```bash
# Sum column 3
awk '{sum += $3} END {print sum}' numbers.txt

# Average
awk '{sum += $3; count++} END {print sum/count}' data.txt

# Count by field value
awk '{count[$2]++} END {for (s in count) print s, count[s]}' log.txt

# Min/Max
awk 'BEGIN {min=999999; max=0} {if ($1<min) min=$1; if ($1>max) max=$1} END {print "Min:", min, "Max:", max}' numbers.txt
```

### Built-in Variables
```bash
NR    # Line number (Number of Records)
NF    # Number of fields in current line
FS    # Field separator (input)
OFS   # Output field separator
FNR   # File line number (resets per file)

# Examples
awk '{print NR, $0}' file.txt                    # Add line numbers
awk '{print NF}' file.txt                        # Count columns per line
awk 'BEGIN {FS=","} {print $1}' file.csv         # Set input delimiter
awk 'BEGIN {OFS="|"} {print $1, $2}' file.txt    # Set output delimiter
```

### BEGIN and END Blocks
```bash
# Print header/footer
awk 'BEGIN {print "Start"} {print $1} END {print "Done"}' file.txt

# Count records
awk 'END {print NR}' file.txt

# Statistics
awk '{sum += $1; count++} END {print "Total:", sum, "Avg:", sum/count}' data.txt
```

---

## Real-World Examples

### Log Analysis
```bash
# Count errors per hour from logs: [2024-12-11 14:30:15] ERROR ...
awk -F'[][ ]' '/ERROR/ {split($2, t, ":"); print t[1]":00"}' log.txt | sort | uniq -c

# Top 10 IP addresses
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10

# Average response time (last column)
awk '{sum += $NF; count++} END {print sum/count}' access.log
```

### Kubernetes Operations
```bash
# Pods with restarts > 0
kubectl get pods -A | awk 'NR>1 && $4 > 0 {print $1, $2, $4}'

# Sum CPU usage
kubectl top pods -A | awk 'NR>1 {gsub(/m/,"",$3); sum+=$3} END {print sum "m"}'

# Group pods by status
kubectl get pods -A | awk 'NR>1 {count[$4]++} END {for (s in count) print s, count[s]}'
```

### CSV Processing
```bash
# Filter rows where column 3 > 100
awk -F, 'NR>1 && $3 > 100 {print $1, $2, $3}' data.csv

# Sum column 2, grouped by column 1
awk -F, 'NR>1 {sum[$1] += $2} END {for (key in sum) print key, sum[key]}' data.csv

# Calculate statistics
awk -F, 'NR>1 {sum+=$3; if(min=="" || $3<min) min=$3; if($3>max) max=$3}
         END {print "Sum:", sum, "Min:", min, "Max:", max}' data.csv
```

### System Monitoring
```bash
# Memory usage percentage
awk '/MemTotal/ {total=$2} /MemAvailable/ {avail=$2}
     END {print (total-avail)/total*100 "%"}' /proc/meminfo

# Top CPU processes
ps aux | awk 'NR>1 {print $3, $11}' | sort -rn | head -10

# Disk usage over 80%
df -h | awk 'NR>1 && $5+0 > 80 {print $6, $5}'
```

### Text Transformation
```bash
# Remove duplicates (keep first occurrence)
awk '!seen[$0]++' file.txt

# Print unique lines with count
awk '{count[$0]++} END {for (line in count) print count[line], line}' file.txt

# Column alignment
awk '{printf "%-20s %-10s %s\n", $1, $2, $3}' file.txt

# Transpose (rows to columns)
awk '{for(i=1;i<=NF;i++) a[i]=a[i]" "$i} END {for(i=1;i<=NF;i++) print a[i]}' file.txt
```

---

## Pipelines - Combining Tools

```bash
# Top 10 largest files
du -ah /var/log | sort -rh | head -10

# Count unique IPs
awk '{print $1}' access.log | sort -u | wc -l

# Find errors, extract service, count by service
grep ERROR *.log | awk -F: '{print $1}' | sort | uniq -c | sort -rn

# Kubernetes pods using >1000m CPU
kubectl top pods -A | awk 'NR>1 {gsub(/m/,"",$3); if($3>1000) print $1, $2, $3"m"}'

# Parse JSON with jq, then awk
curl -s api.example.com/metrics | jq -r '.[] | "\(.name) \(.value)"' | awk '$2 > 100'

# Monitor live log, count errors per minute
tail -f app.log | grep --line-buffered ERROR | while read line; do
  echo "$(date +%H:%M): $line"
done
```

---

## Quick Decision Guide

**Loop through file lines?** → `while IFS= read -r line; do ... done < file`

**Simple text replacement?** → `sed 's/old/new/g'`

**Extract specific columns?** → `awk '{print $2, $5}'`

**Sum/average/count numbers?** → `awk '{sum += $1} END {print sum}'`

**Complex logic or data structures?** → Use Python, not Bash

**Performance matters (>100K lines)?** → Use `awk` over `while read`

---

## Common Gotchas

```bash
# ❌ WRONG - word splitting, memory issues
for line in $(cat file.txt); do echo "$line"; done

# ✅ CORRECT
while IFS= read -r line; do echo "$line"; done < file.txt

# ❌ WRONG - subshell (variables don't persist)
cat file.txt | while read line; do count=$((count+1)); done
echo $count  # Empty!

# ✅ CORRECT
while read line; do count=$((count+1)); done < file.txt
echo $count  # Works

# ❌ WRONG - modifying file while reading
while read line; do echo "$line" >> file.txt; done < file.txt  # Infinite loop!

# ✅ CORRECT - use temp file
while read line; do echo "$line modified"; done < file.txt > tmp && mv tmp file.txt
```

---

## Cheat Sheet

| Task | Command |
|------|---------|
| Loop file lines | `while IFS= read -r line; do ... done < file` |
| Replace text | `sed 's/old/new/g' file` |
| Delete lines | `sed '/pattern/d' file` |
| Extract column | `awk '{print $2}' file` |
| Sum column | `awk '{sum+=$1} END {print sum}' file` |
| Count by field | `awk '{count[$1]++} END {for(k in count) print k,count[k]}' file` |
| Filter rows | `awk '$3 > 100' file` |
| CSV processing | `awk -F, '{print $1, $2}' file.csv` |
| Add line numbers | `awk '{print NR, $0}' file` |
| Remove duplicates | `awk '!seen[$0]++' file` |
| Top N lines | `sort -rn file | head -10` |
| Unique count | `sort file | uniq -c` |
