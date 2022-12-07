# Nmap

Usage: nmap [Scan Type(s)] [Options] {target specification}

## Service version detection

### -sV

Probe open ports to determine service/version info

## Script scan

### -sC

Uses the most commons scripts to scna the target.
Equivalent of --script=default

## Host discovery

### -Pn

Treat all hosts as online -- skip host discovery

## Output

### -o _filename.txt

Saves the result of the scan on a file with the specified name