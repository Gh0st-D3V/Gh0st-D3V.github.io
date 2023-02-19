---
layout: post
author: Gh0st-D3V
tags: [Automation, Pyhton]
---

# GOOSE.py
## Gathering Online User Usernames and Server Entries

Goose is an automation tool used for enumerating specific ftp servers for a file know as (Staff List.csv) and then changing said file into a list of possible Windows login credentials.


How Goose Works:

- Goose requires a .txt file with a list of IP address that have FTP open.
- Goose then trys an anonymous login using a bank password to keep tracking to a minimum
- Once Goose is logged in, it looks for the file (Staff List.csv) in the current directory and if its there it downloads the file
- Once Downloaded Goose opens the File and takes each name inside and Generates 2 possible usernames per entry, FirstInitial,LastName and FirstName,LastInitial (EX: John Doe => JDoe,JohnD)
- Goose will also save the generated names into an internal list and if it comes accross the same name it will ignore it and move on to the next name.
- Goose then saves each set of possible Usernames into a .cvs called usernames.

---

## Code

```
###########################################################################
# Gathering Online User Usernames and Server Entries (GOOSE)              #
# Author: gh0st                                                           #
# Creation Date: 1/26/2023                                                #
# Version: 3.6                                                            #
# About: Takes a .cvs file of first and last names and generates 2        #
# username possibilites, First intial Lastname and Firstname Last initial #
###########################################################################

import ftplib
import csv
a = input("Enter Server List: ")
# Open the file containing the list of FTP servers
with open( a, "r") as servers_file:
    servers = servers_file.readlines()
    # Strip newlines from the server names
    servers = [server.strip() for server in servers]
    # Create a list to store the staff lists
    staff_lists = []
    for server in servers:
        try:
            ftp = ftplib.FTP(server)
            ftp.login('anonymous','')
            
            with open("StaffList.csv", "wb") as local_file:
                ftp.retrbinary("RETR StaffList.csv", local_file.write)
            ftp.quit()
            with open("StaffList.csv", "r") as local_file:
                staff_lists.append(list(csv.reader(local_file)))
        except ftplib.all_errors as e:
            print(e)
            continue
    #flatten the list
    staff_lists = [item for sublist in staff_lists for item in sublist]
    # Create a list to store the usernames
    usernames = []
    # Create a set to store the used usernames
    used_usernames = set()
    for row in staff_lists:
        if not row:
            continue
        if row[0] != "FirstName": #skip the header row
            first_name = row[0]
            last_name = row[1]
            # Generate the possible usernames
            username1 = first_name[0] + last_name
            username2 = first_name + last_name[0]
            if not (username1 in used_usernames or username2 in used_usernames):
                # Add the usernames to the list and the set
                usernames.append([username1, username2])
                used_usernames.add(username1)
                used_usernames.add(username2)
    with open("Usernames.csv", "w") as output_file:
        writer = csv.writer(output_file)
        # Write the usernames
        for username in usernames:
            writer.writerow(username)

```

---
{: data-content="footnotes"}

