# Assignment 3: Configuring Nginx & UFW 

## Introduction

This assignment will teach you how to set up a Bash script to generate a static index.html file containing system information. The script will run automatically every day at 5:00 AM using a systemd service and timer. The HTML document created by this script will be hosted by an nginx web server on your Arch Linux droplet, which will aslo be using a ufw firewall setup to help secure your server.

## Table of Contents

1. [Setup Instructions for New Server](#setup-instructions-for-new-server)
    1. [Create the System User and Directory Structure](#task-1---create-the-system-user-and-directory-structure)
    2. [Create the generate-index.service and generate-index.timer scripts](#task-2---create-the-generate-indexservice-and-generate-indextimer-scripts)
    3. [Modify nginx.conf and Create Server Blocks](#task-3---modify-nginxconf-and-create-server-blocks)
    4. [Configure UFW for SSH and HTTP](#task-4---configure-ufw-for-ssh-and-http)
    5. [Verify Your System Information Page Works](#task-5---verify-your-system-information-page-works)
2. [References](#references)


# Setup Instructions for New Server

## Task 1 - Create the System User and Directory Structure

Before we start, we need to create a system user with a home directory and a login shell for a non-login user. The reason we would want to create a system user is because it enhances security by limiting permissions. It prevents direct logins to the system user, which reduces the risk of unauthorized access and accidental changes.[^1]

**1. Creating the system user**

Type the following command to create a system user with a specified custom home directory[^1]:

```
sudo useradd -r -d /var/lib/webgen -s /usr/sbin/nologin webgen
```

**-r:** Creates a system account.[^1][^2]

**-d:** Specifies the home directory.

**-s /usr/sbin/nologin webgen:** Specifies a non-login shell to prevent interactive logins.[^1]

>[!NOTE]
>The reason the `-d` option was used is to specify a custom home directory for the system user. Although system users don't require a home directory, in this case, it helps in organizing and managing files related to the user's tasks. 

**2. Copy and Paste the following command to create the home directory, as it doesn't exist yet:**

```
sudo mkdir -p /var/lib/webgen
```

**3. Create the Necessary Directory Structure and Files**

Type the following to create the sub-directories `bin` and `HTML` for the `webgen` directory:
```
sudo mkdir -p /var/lib/webgen/bin /var/lib/webgen/HTML
```

**4. Git Clone generate_index File**

>[!IMPORTANT]
> If working on part 2, click [here](#assignment-3---part-2-deploying-and-configuring-digitalocean-droplets-with-load-balancing-and-nginx-file-server) for different instructions.

To clone a directory that includes the `generate_index` file script from this **sourcehut** repository, copy and paste the following command into your **home directory**:

```
git clone https://git.sr.ht/~nathan_climbs/2420-as2-start
```

**5. Move generate_index file to the webgen/bin directory**

Copy and paste the following commany to move the `generate_index` file into `/var/lib/webgen/bin` directory:
```
sudo mv 2420-as2-start/generate_index /var/lib/webgen/bin/
```

**6. Give generate_index executable permission**

Copy and paste the following command to give `generate_index` executable permission:
```
sudo chmod +x /var/lib/webgen/bin/generate_index
```

>[!NOTE]
>When executed, the file will generate a HTML structure into your `index.html` file.


**7. Create `index.html` File in the /webgen/HTML directory**
```
sudo nvim /var/lib/webgen/HTML/index.html
```

**8. Set Ownership**

Type the following commmand to set ownership of the home directory and its contents to webgen:

```
sudo chown -R webgen:webgen /var/lib/webgen
```

>[!NOTE] 
>Running this command changes the ownership to the webgen user, ensuring that only the webgen user has the necessary permissions to access and manage its home directory and all files within it.


## Task 2 - Create the generate-index.service and generate-index.timer Scripts


**1. Create the generate-index.service File**

Type the following to create and enter a service file named `generate-index.service` in your system directory:

```
sudo nvim /etc/systemd/system/generate-index.service
```

Type the following into the `generate-index.service` file:

```ini
[Unit]
Description=Generate Index Service
Wants=network-online.target
After=network-online.target


[Service]
User=webgen
Group=webgen
ExecStart=/var/lib/webgen/bin/generate_index
```

>[!NOTE]
>The `network-online.target` ensures the network is fully operational. The `Wants` directive suggests that a service prefers to wait for the network to be ready but doesn't strictly require it. The `After` directive ensures the service starts only after the network is confirmed to be online, ensuring proper functionality. This helps prevent network-dependent services from failing due to network-related issues[^6].

**2. Create the generate-index.timer File**

Type the following to create and enter a service file named `generate-index.timer` in your system directory:

```
sudo nvim /etc/systemd/system/generate-index.timer
```

Type the following into the `generate-index.service` file:

```ini
[Unit]
Description=Timer for Generate Index Service

[Timer]
OnCalendar=*-*-* 05:00:00
Unit=generate-index.service
Persistent=true

[Install]
WantedBy=timers.target
```

**3. Enable and Start the Timer**

Type the following command to set the timer to start automatically at boot:

```
sudo systemctl enable generate-index.timer
```

Type the following command to iniate the timer immediately:

```
sudo systemctl start generate-index.timer
```

>[!IMPORTANT]
>Check timer status is **active** by typing `sudo systemctl status generate-index.timer`

**4. Manual Trigger Service for Testing**
>[!NOTE]
>Do not enable service. We just want to start it to check if the `generate-index.service` will work when active.

Type the following command to start the service:
```
sudo systemctl start generate-index.service
```


>[!NOTE]
>When you start `generate-index.service` , index.html's structure will be successfully written if the service output message shows that it has started, executed, and deactivated successfully.

5. Check if Status is Active and if the Service Executed

Type the following command to check the status of the service:

```
sudo systemctl status generate-index.service
```

Type the folowing to view the logs of the service:

```
sudo journalctl -u generate-index.service
```

>[!NOTE]
>When you check the service status, Active will say **inactive (dead)** because the `generate-index.service` successfully completed its task and then stopped, as it only executes the `generate_index` script once. However, when you check the logs, you should see detailed entries confirming the script's successful execution and any relevant output generated during its run.


## Task 3 - Modify nginx.conf and Create Server Blocks

**1. Install nginx**
```
sudo pacman -S nginx
```

**2. Modify the Main `nginx.conf` File**

Type the following command to access and edit the `nginx.conf` file[^3]:
```
sudo nvim /etc/nginx/nginx.conf
```

**3. Locate `user` in nginx.conf and change it to `webgen webgen`**
```
user webgen webgen;
```
>[!NOTE]
>The first user indicates the username and the second user indicates the usergroup. This allows nginx.conf the correct permissions for managing files and directories associated with the webgen user and group.


**4. Create the `sites-available` and `sites-enabled` directory**

Creates the `sites-available` directory[^3]:
```
sudo mkdir -p /etc/nginx/sites-available
```

Creates the `sites-enabled` directory[^3]:
```
sudo mkdir -p /etc/nginx/sites-enabled
```


**5. Create a Separate Server Block File into `sites-available`**

Type the following command to create a new server block file that'll configure Nginx to host index.html[^3]:

```
sudo nvim /etc/nginx/sites-available/webgen.conf
```

>[!NOTE]
>We created a separate server block file instead of modifying the main nginx.conf file because it helps manage configurations more easily[^3]. Through this approach, it allows you to split a large configuration into smaller, manageable files.Therefore, you can enable or disable specific parts quickly without affecting the entire setup, keeping your configuration organized and easier to maintain.

Copy and paste the following server block into your `webgen.conf` file:

>[!IMPORTANT]
>If working on part 2, go to [Task 4 - Modify nginx.conf and Create the Server Block to Include a File Server](#task-4---modify-nginxconf-and-create-the-server-block-to-include-a-file-server). The new configuration adds a file server to Nginx, allowing access to documents via 'your-ip/documents'.

```
server {
   listen 80;
   listen [::]:80;

   server_name localhost.webgen;

   root /var/lib/webgen/HTML;
   index index.html;

        location / {
        try_files $uri $uri/ =404;
    }
}
```

>[!NOTE]
>Starting with `listen 80;` `and listen [::]:80;` , this server block is set to listen for both ipv4 and ipv6 HTTP requests on port 80, which is the default port for http traffic[^4]. For the server name, we named it localhost.webgen. **Make sure the `server_name` is always unique.** The root directory `/var/lib/webgen/HTML` specifies where Nginx will look for files to host, and the default file `index.html` ensures that users can access the website's main content correctly. Finally, the `location /` block handles requests to your website's main root page, and the `try_files $uri $uri/ =404;` directive checks for the requested file or directory, hosting them if found, and returns a 404 error if not[^4].


**6.Enable the Server Block**

Type the following command to symlink and enable the server block[^3]:

```
sudo ln -s /etc/nginx/sites-available/webgen.conf /etc/nginx/sites-enabled/
```

**7. Add the `sites-enabled` Directory in nginx.conf file**

```
http {
    ...
    include /etc/nginx/sites-enabled/*;
}
```

>[!NOTE]
>Type `sudo nginx -t` to verify if `nginx.conf` has any errors

**8. Restart Nginx**

Copy and paste the following command to restart Nginx:

```
sudo systemctl restart nginx
```

**9. Start Nginx**

Copy and paste the following command to start Nginx:

```
sudo systemctl start nginx
```

>[!IMPORTANT]
> Type `sudo systemctl status nginx` to check if Nginx is **active** and **running**


## Task 4 - Configure UFW for SSH and HTTP

**1. Install UFW**

Copy and paste the following code to install UFW[^5]:

```
sudo pacman -S ufw
```

>[!CAUTION]
>It is very important that after downloading UFW, you **do not enable UFW** right away. You will be locked out of your SSH droplet. Follow the next steps carefully to succesfully implement UFW[^4]. 

**2. Allow SSH and HTTP from anywhere**

To allow SSH connections from anywhere copy and paste the following command[^5]:

```
sudo ufw allow ssh
```

To allow HTTP traffic from anywhere copy and paste the following command:

```
sudo ufw allow http
```

>[!IMPORTANT]
>If you're getting an `iptables` error after allowing, read the following steps below. If not, you can skip to step 3 on enabling rate limiting.

**Possible error when trying to allow:**
```
[Errno 2] iptables v1.8.10 (legacy): can't initialize iptables table 'filter': Table does not od?)
Perhaps iptables or your kernel needs to be upgraded.
```

>[!IMPORTANT]
>If doing part 2, type `sudo pacman -Syu linux` to update your kernel to ensure you have the latest modules. Then continue the steps below to fix the iptables error. 

**Steps to fix iptables error:**
1. `sudo pacman -Syu` -- Update your system
2. `sudo pacman -S iptables` -- Update outdated iptables version
3. `sudo systemctl restart iptables` -- Restart iptables
4. `sudo reboot` - Reboot droplet





**3. Enable SSH rate limiting**

To limit the rate of SSH connections to protect against multiple unauthorized access attempts, copy and paste the following command:

```
sudo ufw limit ssh
```

>[!NOTE]
>Setting the `limit` on SSH will block any IP address after 6 connection attempts within 30 seconds[^5].

>[!IMPORTANT]
>After entering the commands, make sure it said `Rules updated` and `Rules updated (v6)` for steps 2 and 3.


**4. Enable UFW**

Copy and paste the following command to enable UFW[^5]:

```
sudo ufw enable
```

**5.Check UFW status**

Copy and paste the following command to check if UFW is active and the correct ports allowed are listed[^5]:

```
sudo ufw status 
```

If done correctly, you should see the following UFW status:
```
Status: active

To                         Action      From
--                         ------      ----
22                         LIMIT       Anywhere
80                         ALLOW       Anywhere
22 (v6)                    LIMIT       Anywhere (v6)
80 (v6)                    ALLOW       Anywhere (v6)
```

## Task 5 - Verify Your System Information Page Works

**1. Find Your Droplet’s IP Address**

You can find your droplet's IP address by logging into your DigitalOcean account and clicking on the **Droplets** section. From there, locate the droplet you've been working with and copy its IP address.


**2. Visit the IP Address**

Open a web browser and type `http://` followed by your `droplet ip` into the address bar

```
http://your-droplet-ip
```

**If all done correctly, your system information page should look like this:**
![Screenshot of my system information page](/System%20Information%20Screenshot.png)


**Congratulations! You have successfully learned how to set up a Bash script, automate tasks with systemd, configure an nginx web server, and secure your server using a firewall with UFW!**

---
# Assignment 3 - Part 2: Deploying and Configuring DigitalOcean Droplets with Load Balancing and Nginx File Server

## Set Up Two Droplets and Configure Load Balancing


## Task 3 - Set Up and Configure the New Directory Structure for Your Server 

**1. Git Clone new generate_index File**

```
git clone https://git.sr.ht/~nathan_climbs/2420-as3-p2-start
```

**2. Move generate_index file to the webgen/bin directory**

Copy and paste the following commany to move the `generate_index` file into `/var/lib/webgen/bin` directory:
```
sudo mv 2420-as3-p2-start/generate_index /var/lib/webgen/bin/
```
**3. Give generate_index executable permission**

Copy and paste the following command to give `generate_index` executable permission:
```
sudo chmod +x /var/lib/webgen/bin/generate_index
```

**4. Create a `documents` directory in `/webgen/ directory`**

```
sudo mkdir -p /var/lib/webgen/documents
```

**5. Create 2 files, `file-one` and `file-two` in the `/webgen/documents/` directory**

```
sudo touch /var/lib/webgen/documents/file-one /var/lib/webgen/documents/file-two
```

**6. Create `index.html` File in the /webgen/HTML directory**
```
sudo touch /var/lib/webgen/HTML/index.html
```

**7. Set Ownership**

Type the following commmand to set ownership of the home directory and its contents to webgen:

```
sudo chown -R webgen:webgen /var/lib/webgen
```

**8. Go to [task 2](#task-2---create-the-generate-indexservice-and-generate-indextimer-scripts) in part 1 to continue your server set up**

Click the following to go continue the steps for the server setup: [Create the generate-index.service and generate-index.timer scripts](#task-2---create-the-generate-indexservice-and-generate-indextimer-scripts)



## Task 4 - Modify nginx.conf and Create the Server Block to include a file server

**1. Configure the server block to incorporate the location block for handling the documents requests:**


Copy and paste the following server block into your `webgen.conf` file:

```
server {
   listen 80;
   listen [::]:80;

   server_name localhost.webgen;

    location / {
       root /var/lib/webgen/HTML;
       index index.html;
       try_files $uri $uri/ =404;
   }

   # Handle /documents/ requests
   location /documents {
       alias /var/lib/webgen/documents/;
       autoindex on;
       autoindex_exact_size off;
       autoindex_localtime on;
       try_files $uri $uri/ =404;
   }
}
```

- `alias` : Directly maps a URL path to a specific directory, replacing the URL path entirely with the alias path for file retrieval.alias path.[^7].

- `autoindex on;` : Enables directory listing, showing all files and directories within the specified location[^7].

- `autoindex_exact_size off;` : Displays file sizes in a simplified, human-readable format like KB or MB, instead of bytes[^7].

- `autoindex_localtime on;` : Shows file modification timestamps using the server's local timezone[^7].


**8. Continue from part 1 of step 6, [task 3](#task-3---modify-nginxconf-and-create-server-blocks) to proceed with the setup of your Nginx server block**

Click the following to go continue the steps for the nginx server block setup: [Task 3 - Modify nginx.conf and Create Server Blocks](#task-3---modify-nginxconf-and-create-server-blocks)



# References
[^1]: "Users and groups - ArchWiki." Arch Linux, 23 Nov. 2024. [Online]. https://wiki.archlinux.org/title/Users_and_groups#Example_adding_a_system_user. [Accessed: 19-Nov-2024].

[^2]: `man useradd` - Use `-r` options for creating a system user.

[^3]: "nginx - ArchWiki." Arch Linux, 7 Nov. 2024. [Online]. https://wiki.archlinux.org/title/Nginx. [Accessed: 19-Nov-2024].

[^4]: N.McNinch, "Week Twelve Notes," CIT2420 Notes, 2024. [Online]https://gitlab.com/cit2420/2420-notes-f24/-/blob/main/2420-notes/week-twelve.md. [Accessed: Nov. 19, 2024].

[^5]: "Uncomplicated Firewall - ArchWiki." Arch Linux, 1 Nov. 2024. [Online]. https://wiki.archlinux.org/title/Uncomplicated_Firewall. [Accessed: 19-Nov-2024].

[^6]: "Systemd - ArchWiki." Arch Linux, 1 Nov. 2024. [Online]. https://wiki.archlinux.org/title/Systemd. [Accessed: 23-Nov-2024].

[^7]: N.Mcninch, "Week Thirteen Notes," CIT2420 Notes, 2024. [Online]https://gitlab.com/cit2420/2420-notes-f24/-/blob/main/2420-notes/week-thirteen.md. [Accessed: Nov. 26, 2024].










