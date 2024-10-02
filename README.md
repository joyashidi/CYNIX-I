# CYNIX-I
This is a project to capture two (2) flags. I had to find and read the flags (user and root) which was present in user.txt and root.txt respectively.

 During the testing I discovered the IP address of the target machine; 192.168.56.104 

 Vulnerability Exploited: Local File Inclusion (LFI), Google lxb

Information Gathering 
 
Making use of nmap, I discovered the ip address of the target virtual machine. 

Target ip address: 192.168.56.104 

Host ip address: 192.168.56.101 

I went on further to scan the target machine, making use of nmap and discovered 2 open ports which are ports 80 and 6688 on this target machine. Port 80 runs HTTP but port 6688 is running SSH, we can say it is making use of an unallocated port. 

 └─$ nmap –Pn –p- -sVC 192.168.56.104  

-Pn stands for port scanning 

-p- stands for entire port range 

-sVC is the same as –sV (services operating on the ports) -sC (script scanning) 

Service Enumeration 

This is a process of identifying and gathering information about the services running on a computer system or network. In this test the service HTTP was further investigated. 

To make use of SSH (Secure Shell) I have to be able to gain access to the target machine of which I haven’t had yet so this option was not explored.  

Port 80 is the standard port for web traffic i.e HTTP, the fact that it is open shows that there is access to the web server that is being run on the target machine and can be exploited. In this case, Apache default installation page. 

I went on to further explore making use of port 80 on a browser, 

192.168.56.104:80 

Accessed the web page on port 80 


- I ran gobuster to brute force directories to help discover hidden directories and files on the web server that may not be directly accessible via the web application interface. 

gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://192.168.56.104 -o gobuster_results.txt 

I navigated to the url http://192.168.56.104/lavalamp 

Arrived at a different webpage, studied the page and it has a form. Looking through its source code with developer tools (f12) and navigating to debugger. The JavaScript codes are displayed there from 94 and 95 I can find where the results of the form link to which is canyoubypassme.php. I navigated to that to that page. 
 
http://192.16.104/lavalamp/canyoubypassme.php 

http://192.16.104/lavalamp/canyoubypassme.php 

Looking into the source codes of this page (f12), I saw that a form (on line 139) was created on this page, but I cannot see it displayed. I could have decided to reduce the opacity of the page or interact with the source code using curl. 

 
I decided to use curl since it can be used to retrieve the HTML source code of a web page, including vulnerable web applications. 

<form method=post action="/lavalamp/canyoubypassme.php">Specify a number: <input type=text name=file placeholder=integer><br><br><input type=submit name=read value="Download the number"></form> 

I saw that the page was requesting for a number and its placeholder is a file so I added a number to the file to see the outcome. 

curl -X POST -v http://192.168.56.104/lavalamp/canyoubypassme.php -d "file=42" 

I was able to interact with it and it successfully sent a POST request to the /lavalamp/canyoubypassme.php endpoint with the data file=42, and the server responded with a 200 OK status. This looks like a Local File Inclusion (LFI). The server is running Apache/2.4.29 on Ubuntu. 
 
To verify LFI, I looked for /etc/passwd file by injecting “../../../etc/passwd” in the parameter “file=6” and checked its response. I found the application is vulnerable to LFI.  

curl -X POST -d "file=6../../../../etc/passwd&read=Download+the+number" http://192.168.56.104/lavalamp/canyoubypassme.php 

 
Exploitation 

 SSH is installed on the host machine, so I also checked for ssh rsa key by injecting “../../../home/ford/.ssh/id_rsa” in parameter ‘’file=6” as a result I found id_rsa key for ssh.  

curl -X POST -d "file=6../../../../home/ford/.ssh/id_rsa&read=Download+the+number" http://192.168.56.104/lavalamp/canyoubypassme.php 


Gotten a key which I copied and saved in a file called sshkey1 and grant permission 600, and finally get logged in and spawn the ssh shell of the machine. I was able to get into the user ford’s account. Listing out his directory I can see the user.txt file, which was the first flag to be captured, I used cat to display the content of the file. 

nano sshkey1 

chmod 600 sshkey1  

ssh ford@192.168.56.104 -p 6688 -i sshkey1 

 
 Privilege Escalation 

As the name implies this is gaining more access than a typical user is authorised to. I Checked the directory and found it only has a user's directory so I decided to check the group memberships.  
 

I did research on what to do with this information I got, I found these: 

  ford (gid 1000): This is the primary group for the user. 

  cdrom (gid 24): Typically, this group gives users permission to access CD-ROM devices. It's not usually a privilege escalation vector but could be useful for accessing media. 

  dip (gid 30): This group typically allows users to manage network interfaces and might be useful if you can manipulate network configurations, but it usually doesn’t provide significant privilege. 

  plugdev (gid 46): Members of this group can access USB devices. If there are any USB-related scripts or binaries that can be exploited, this could potentially lead to privilege escalation. 

 lpadmin (gid 111): This group is related to printer administration. If there are any misconfigurations or SUID binaries related to printing, you might find an escalation path here. 

  sambashare (gid 112): Users in this group can access Samba shares. If there are insecure Samba shares or configurations, this might be an avenue for further exploitation. 

 lxd (gid 113): This group allows for managing Linux containers using LXD. This can be a powerful group, as it may give you access to create containers that can run with elevated privileges. 

LXD gives access to create containers and can run with elevated privileges, so I decided to use this group. Further research says that with our rights, we can create a privileged container and mount the full disk into the container. Making us root. 

Creating container and mounting the full disk into the container, called fordContainer 
 
I was able to start and execute the bash in the container. 

Gained access to the root, making me able to view files present in the root system. 

I eventually got the root.txt flag of the target virtual machine. 


Recommendations 

I recommend validating and sanitizing user input, using absolute file paths, and implementing strict access controls to prevent unauthorized access to sensitive files. Disable directory listing and ensure proper error handling to avoid information leakage. Restrict container privileges by running them unprivileged, control device access, and keep both LXD and the host kernel updated. Use security profiles to isolate containers from the host system and reduce attack surfaces. Regularly audit container configurations and limit network exposure to trusted IPs. 

 
