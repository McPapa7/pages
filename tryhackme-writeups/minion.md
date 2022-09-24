---
cover: ../.gitbook/assets/minion cover.png
coverY: 0
---

# Minion

## 0. Setup

Add minion.thm to /etc/hosts:&#x20;

```bash
sudo nano /etc/hosts
```

Put `<TARGETIP> minion.thm` as a new line in the file

![](<../.gitbook/assets/Pasted image 20220923065821.png>)

Optional setup:&#x20;

I like to save the target IP as a variable called TGT which can be used in commands and save having to type it out each time. Also makes copying commands from my notes a lot easier

![](<../.gitbook/assets/Pasted image 20220923065906.png>)

## 1. Recon and Enumeration

### 1.1 nmap

Use nmap tool to start to built up a picture of what is running on the machine: Note: $TGT is the target machine IP as described in the setup section

\<html>

<details>

<summary>Q1: What is the best Language in the World?</summary>

A1: JavaScript

</details>



```bash
sudo nmap -A -T4 -p- $TGT
```

Options explained: -A runs Version detection as well as default set of scripts, -T4 is the timing template to use (0: slowest, 5: quickest), -p- scan all ports

Nmap results:&#x20;

<figure><img src="../.gitbook/assets/Pasted image 20220923070212.png" alt=""><figcaption><p>nmap results</p></figcaption></figure>

Points of interest from nmap results:&#x20;

* Web service running on port 80: Apache 2.4.41
* There is a robots.txt file&#x20;
* WordPress V 6.0.2&#x20;
* /wp-admin/ (WordPress admin area)

## 1.2 View website

We can navigate to the website in browser either using the target IP `http://<TGTIP>` or by using the hostname we saved in /etc/hosts `http://minion.thm`

Loading up the website we are shown our first flag&#x20;

<figure><img src="../.gitbook/assets/Pasted image 20220921072832.png" alt=""><figcaption><p>Landing page</p></figcaption></figure>

From here I generally will browse the website as a user would (alongside viewing page sources and keeping our nmap scan results in mind) to get an idea of the websites functionality and purpose before using enumeration tools

Following the link to the Flag 1 post there is a mention of the author "minion"&#x20;

<figure><img src="../.gitbook/assets/Pasted image 20220923070049.png" alt=""><figcaption></figcaption></figure>

There is also a comment section at the bottom of the page. I made note of this incase it could be used in the exploitation phase for possible XSS. (http://minion.thm/2022/09/18/flag1/)

I did not any more pages of notable importance browsing through the site so I chose to move on at this point.

Our nmap results showed from the robots.txt file that /wp-login/ is to be ignored by web crawlers. This is sometimes a clue for where to find sensitive information on websites so it is worth taking note as well as wordpress sites commonly using this as an admin area.

robots.txt can be viewed in the browser or using commands such as curl and is often worth checking out.

![](<../.gitbook/assets/Pasted image 20220921073901.png>)

From viewing robots.txt we can see the site appears to be running PHP as well as giving us a link to a sitemap. Browsing to the sitemap and following the users link we can see another mention of the user "minion" if we had missed it before

![](<../.gitbook/assets/Pasted image 20220923205144.png>)

We can view the disallowed page by navigating to it directly in the browser where we find we get redirected to a login page

<figure><img src="../.gitbook/assets/Pasted image 20220921074117.png" alt=""><figcaption><p>login page</p></figcaption></figure>

## 2. Exploitation

My first process was to try and use XSS in the comments section, however, comments require admin approval before they will be dispalyed properly.

### 2.1 Login page

To see how the page responds to input we can try credentials we do not think will work test:test

![](<../.gitbook/assets/Pasted image 20220921122821.png>)

Error shows no username "test" registered. This is a useful error message compared to a typical "user and/or password incorrect" seen on most site as we can identify if a username is valid without knowing the password.

We can try common usernames (root, admin) and also 'minion' (the author of the flag post)

Entering Minion gives a different error message that the password is incorrect but confirms the username exists.

![](<../.gitbook/assets/Pasted image 20220923070812.png>)

Brute forcing the login can be achieved with a number of tools. I used burpsuite at first but with the delayed response time form the site I changed over to use WPScan

#### WPScan

Using our found valid username "minion" we can try to brute force the login. At first I set max-threads set to 40 but it was timing out and eventually got the command to work at 5, which is the default amount if the options is not used.

Command:

```
wpscan --url http://minion.thm/wp-login.php --usernames minion --passwords /usr/share/wordlists/rockyou.txt --max-threads 5
```

![](<../.gitbook/assets/Pasted image 20220921133836.png>)

Password successfully found so we now have valid login credentials minion : yellow

## 3. Post Compromise Enumeration

### 3.1 Wordpress User Compromise

Once we logon we can see that comments awaiting approval if any were made&#x20;

![](<../.gitbook/assets/Pasted image 20220921133914.png>)

Approving the comments and going back to the post page shows XSS to be working but I did not explore this route any further

![](<../.gitbook/assets/Pasted image 20220921134339.png>)

### 3.2 PHP Reverse Shell

As we know the site runs PHP (Wordpress sites almost always run PHP and robot.txt lists a .php file) my focus turned to finding somewhere that a PHP shell could be upladed to.

Originally I tried in the upload media section but .php extensions are not allowed and file signatures are checked as renaming the extension was still found to be invalid.

From previous experience I have known there to be .php files in the theme files so I looked in here. The theme in use contained .html files, however other themes were available and browsing to TWENTY TWENTY-ONE theme found the desired .php extension files.

![](<../.gitbook/assets/Pasted image 20220923070956.png>)

![](<../.gitbook/assets/Pasted image 20220923071045.png>)

I decided to use the 404 file as it can reliably be called upon by browsing to page that doesn't exist.

I inserted the well known PHP shell from pentestmonkey which can be found at: [https://github.com/pentestmonkey/php-reverse-shell](https://github.com/pentestmonkey/php-reverse-shell) making sure to change the ip and port number to match my workstation and the netcat listener I set up.&#x20;

```bash
nc -nvlp 4444
```

![](<../.gitbook/assets/Pasted image 20220923071246.png>)

![](<../.gitbook/assets/Pasted image 20220923071319.png>)

Once the PHP is pasted in and the port and IP values are set appropriately click the update file button underneath to save the changes&#x20;

![](<../.gitbook/assets/Pasted image 20220923071425.png>)

Now that the PHP shell code exists in the TWENTY TWENTY-ONE theme it needs to be activated which can be done through the dashboard.

![](<../.gitbook/assets/Pasted image 20220923071549.png>)

&#x20;![](<../.gitbook/assets/Pasted image 20220923071607.png>)

Browsing to a URL that doesn't exist (e.g http://minion.thm/NotAPagelllasdbbb) will return the desired 404 response code leading to the themes 404.php being loaded

If e payload is executed properly this will cause the page to "hang" as we now have a web shell established as www-data

![](<../.gitbook/assets/Pasted image 20220923071642.png>)as

### 3.3 Enumeration with established shell

It is not neccessary but it makes life a lot easier if you stabalise your shell:

#### Stabalise the shell

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

background the shell using `Ctrl + Z`

```bash
stty raw -echo; fg
```

This does two things: first, it turns off our own terminal echo (which gives us access to tab autocompletes, the arrow keys, and `Ctrl + C` to kill processes). It then foregrounds the shell, thus completing the process.

If shell dies and cannot see typed stuff use command `reset`

With the shell stabalised I started to gather information:

/etc/passwd&#x20;

<figure><img src="../.gitbook/assets/Pasted image 20220921071813.png" alt=""><figcaption></figcaption></figure>

ls /home&#x20;

![](<../.gitbook/assets/Pasted image 20220923074905.png>)

This shows two users of interest: Gru and Minion.

Looking back at the THM room we are in search of flags so I ran a find to check for accessible flags: find / -type f -name "flag\*" 2>/dev/null which didn't return any useful results so I used -iname to make the search case insensitive picking up the next challenge flag find / -type f -iname "flag\*" 2>/dev/null

The available flag is in /srv/www/wordpress alongside other files from the minion.thm site

Looking into the files gives database information in wp-config.php One line that stands out is define( 'DB\_PASSWORD', 'SuperDuperStrongPasswordThatIsLong' );

<figure><img src="../.gitbook/assets/Pasted image 20220923075130.png" alt=""><figcaption><p>wp-config.php</p></figcaption></figure>

## 4 Priv esc

After not finding any SUID files or cron jobs in /etc/crontab to use for priviliege escalation I decided to try and move across to the minion user hoping that the password we found on the wordpress site had been reused Command:

```
su minion
Password: yellow
```

![](<../.gitbook/assets/Pasted image 20220923075231.png>)

SUCCESS as well as access to a flag there is also a file called "notes"

![](<../.gitbook/assets/Pasted image 20220923075255.png>)

I checked if minion had any sudo rights but no luck here&#x20;

![](<../.gitbook/assets/Pasted image 20220923075325.png>)

So using the info from "notes" it follows to move across to the other user Gru. Trying the database password from wp-config.php:

```
su gru
Password: SuperDuperStrongPasswordThatIsLong
```

![](<../.gitbook/assets/Pasted image 20220923075422.png>)

gives the next challenge flag in Gru's home folder. Checking for sudo rights this time returns the ability to run gawk as sudo.

![](<../.gitbook/assets/Pasted image 20220923075457.png>)

Checking GTFO bins [https://gtfobins.github.io/gtfobins/gawk/#sudo](https://gtfobins.github.io/gtfobins/gawk/#sudo) shows us a command that can be used to elevate to privileged access:

```
sudo gawk 'BEGIN {system("/bin/sh")}'
```

![](<../.gitbook/assets/Pasted image 20220923075604.png>)&#x20;

Now that we have a root shell we can find the rooms final flag in the /root folder.



Thanks for reading my writeup of the Minion room.
