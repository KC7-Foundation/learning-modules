### **Difficulty Level: Advanced** 

##### **What you will learn**
-  What JARM Fingerprinting is and how it can be used to track malware infrastructure 
-  Use JARM to identify a fingerprint and search across Censys.io and Shodan.io for related infrastructure 

##### **Requirements** 
- Linux Operating System 
- Python (version 3)
- Git 

##### **Additional Resources** 
- JARM Blog Post: https://engineering.salesforce.com/easily-identify-malicious-servers-on-the-internet-with-jarm-e095edac525a/
- JARM GitHub: https://github.com/salesforce/jarm
- Censys Documentation on JARM: https://support.censys.io/hc/en-us/articles/4409122252692-JARM-in-Censys-Search-2-0 
- Michaell Koczwara's blog post on hunting for Cobalt C2's with Shodan & other tools: https://michaelkoczwara.medium.com/cobalt-strike-c2-hunting-with-shodan-c448d501a6e2


#### ⚠️ DISCLAIMER ⚠️
<hr> 

This content contains the use of a tool that will connect directly to targeted infrastructure, and as such must be treated with caution. 

Proceed at your own risk and with caution. This is meant for an advanced threat intelligence audience. 

<hr> 



## 🗺 **Overview**

JARM is a server fingerprinting tool developed by Salesforce that is designed to actively scan for Transport Layer Security (TLS) configurations on HTTPS websites, typically over port 443. It generates a unique hash based on these TLS configurations. Threat actors, who often set up Command and Control (C2) servers, commonly reuse the same configurations and automated scripts across multiple servers and hosting providers to quickly establish their infrastructure. Notably, tools such as Cobalt Strike and Metasploit are also set up their infrastructure in a similar manner.

JARM has various use cases, including identifying clustered infrastructure for legitimate services like Google and Yahoo. However, it's important to note that there may be potential false positives due to the vast number of servers on the Internet. Therefore, additional triage and investigation should be conducted when interpreting JARM results. In this guide, we will outline how to generate your own JARM fingerprint for a hosted server and how to use Censys.io and Shodan.io to search for similar infrastructure.

I highly recommend reading the material from the **Additional Resources** section for further documentation on JARM.

## **📌Getting Started**

We'll be using a Linux environment with Python. I highly recommend using a Virtual Machine for this.

For a guide on how to create a virtual machine using VirtualBox visit this link: [https://kc7cyber.com/learning-library/setting-up-a-linux-virtual-machine/](https://kc7cyber.com/learning-library/setting-up-a-linux-virtual-machine/)

### **Step 1: Install the required Linux packages**

Let's make sure our environment is set up correctly. Open up a Linux terminal and run the following commands:

```
$ sudo apt-get update
$ sudo apt-get upgrade 
```

Now let's install our required Python packages:

```
$ sudo apt install python3-pip git
$ sudo apt install build-essential libssl-dev libffi-dev python3-dev
$ sudo apt install python3-venv
```

### **Step 2: Set up your virtual environment and install the required Python packages**

Navigate to where you want to save your Python environment and Jupyter notebook. In this case, I'm going to use my Desktop folder.

```
$ cd ~/Desktop
$ mkdir kc7-projects
$ cd kc7-projects/
```

Now let's set up our Python 3 virtual environment and activate it:

```
$ python3 -m venv jarm
$ cd jarm
$ source bin/activate
```

You should see **"(jarm)"** displayed on your terminal on the left side:

```
(jarm) kc7cyber@kc7cyber-dev:~/Desktop/kc7-projects/jarm$
```

Now let's clone the git repository:

```
$ git clone https://github.com/salesforce/jarm.git
Cloning into 'jarm'...
remote: Enumerating objects: 102, done.
remote: Counting objects: 100% (62/62), done.
remote: Compressing objects: 100% (23/23), done.
remote: Total 102 (delta 53), reused 39 (delta 39), pack-reused 40
Receiving objects: 100% (102/102), 38.17 KiB | 2.73 MiB/s, done.
Resolving deltas: 100% (54/54), done.
```

Let's install the required modules:

-   **Note**: you will need to edit the requirements.txt and set requests version from 2.26.0 to 2.28.0.

```
$ cd jarm/
$ pip3 install -r requirements.txt
```

## ⌨️ JARM Fingerprint Generation

#### ⚠️ DISCLAIMER ⚠️

---

This content contains the use of a tool that will connect directly to targeted infrastructure, and as such must be treated with caution.

Proceed at your own risk and with caution. This is meant for an advanced threat intelligence audience. If you're investigating malicious infrastructure, it is highly recommend you use a VPN or Tor connection when generating a JARM fingerprint.

---

With our environment set up, let's test creating a jarm.py fingerprint on [https://www.kc7cyber.com](https://www.kc7cyber.com), a website we know and love:

```
$ python3 jarm.py www.kc7cyber.com
Domain: www.kc7cyber.com
Resolved IP: 34.149.120.3
JARM: 3fd3fd07d3fd3fd00042d42d0000008fe5654c9239cdb4052d3ab65a579afa
```

At the time of this guide the JARM fingerprint is **3fd3fd07d3fd3fd00042d42d0000008fe5654c9239cdb4052d3ab65a579afa**. It's important to note that this may be different when you do this, because TLS configurations and application versions change over time.

Now that we know it works, we'll keep this JARM fingerprint handy for when we search them using Censys and Shodan.

Let's generate a couple more for different, safe websites:

```
$ python3 jarm.py www.yahoo.com
Domain: www.yahoo.com
Resolved IP: 98.137.11.164
JARM: 27d27d27d3fd27d1dc41d41d000000937221baefa0b90420c8e8e41903f1d5
```

```
$ python3 jarm.py www.twitter.com
Domain: www.twitter.com
Resolved IP: 104.244.42.129
JARM: 29d29d00029d29d00042d43d00041d598ac0c1012db967bb1ad0ff2491b3ae
```

```
$ python3 jarm.py www.sans.org
Domain: www.sans.org
Resolved IP: 45.60.31.34
JARM: 29d29d00029d29d00041d41d0000005d86ccb1a0567e012264097a0315d7a7
```

```
$ python3 jarm.py www.tryhackme.com
Domain: www.tryhackme.com
Resolved IP: 172.67.27.10
JARM: 27d3ed3ed0003ed1dc42d43d00041d6183ff1bfae51ebd88d70384363d525c
```

Alright, we've generated a few JARM fingerprints. Let's go search them on Censys.io!

## 📀Censys.io JARM Searching

Let's start with Censys.

If you don't have an account, visit [https://accounts.censys.io/register](https://accounts.censys.io/register) to sign up for a Censys.io account. You will need to use a valid email account to sign up and verify with. If you're a student, feel free to put 'Student' or your college as your organization.

Once you've signed up and verified your email, log in to your account and head over to **Censys Search** at [https://search.censys.io/](https://search.censys.io/) and sign in.

-   Let's start with the kc7cyber JARM fingerprint: **3fd3fd07d3fd3fd00042d42d0000008fe5654c9239cdb4052d3ab65a579afa**. You can use the parameter **"services.jarm.fingerprint:"** to search for JARM fingerprints.

![](https://kc7cyber.com/wp-content/uploads/2023/04/jarm-1.png)

![](https://kc7cyber.com/wp-content/uploads/2023/04/jarm-2-1024x493.png)

-   There's quite a bit of results! There's a reason why. It's because the way the website is set up is similar to many websites. We'll leave that as a mystery as to why that is :).

Well, that fingerprint didn't seem that promising. Let's try yahoo.com instead using the JARM fingerprint: **27d27d27d3fd27d1dc41d41d000000937221baefa0b90420c8e8e41903f1d5**.

![](https://kc7cyber.com/wp-content/uploads/2023/04/jarm-3-1024x553.png)

-   This seems to have better results. If you browse around Censys, you'll find a large majority of the servers are Yahoo-specific infrastructure. This means that this JARM fingerprint allowed me to find other Yahoo related servers.

I'll leave the other JARM fingerprints we've generated for you to play around with.

Other well known C2 JARM fingerprints can be found here: [https://github.com/cedowens/C2-JARM](https://github.com/cedowens/C2-JARM). Some are outdated, so you may need to find a C2 framework (either commercial or custom to your threat actor) and generate your own JARM fingerprint to pivot on. For the purposes of this guide, we won't be investigating any specific malicious infrastructure.

## 🐦 Shodan.io JARM Searching

Shodan has a similar functionality to Censys' JARM fingerprint search, except you run with the query:

```
ssl.jarm:<fingerprint-here>
```

to search for hosts. The only downside is that it requires a membership to query on Shodan.

![](https://kc7cyber.com/wp-content/uploads/2023/04/jarm-4-1024x481.png)

## 🎉🎉 Hooray! 🎉🎉

We've touched upon how to generate your own fingerprint using JARM, and how to search for it on Censys and Shodan. There's quite a bit of opportunities to pivot on JARM fingerprints to newly created infrastructure. Some things to keep in mind when filtering for false positives and finding new C2 domains:

-   Look for low hanging fruit first, such as servers hosted at common VPS hosting providers.
-   Look for weirdly named or self-signed SSL certificates.
-   Investigate other open ports on the server, as it may give you additional clues to find the threat actor.

## Need additional help?

Join our Slack channel at https://kc7cyber.com/slack and ask us a question!