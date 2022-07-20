# Setting-up-FTP-server-on-AWS


Setup plain simple FTP
So assuming you already have have a working instance of the free tier EC2 instance on AWS, or some other cloud provider, let’s start.

1. Install vsftpd
Just install from apt-get on ubuntu with the following commands

$ sudo apt-get update && sudo apt-get install vsftpd

After the installation, the FTP server service should up and running so just check it with

$ sudo service vsftpd status

So, usually, we would setup a firewall on the server and that would be the best practice, but since this is usually handled by some security group at the cloud provider level, this next step is optional.

2. Configure firewall
Here we are going to allow the following ports to pass through.

22 for SSH (Important! Since without this will lock you out from SSH)
20 and 21 for simple insecure FTP
12000 to 12100 for passive FTP
So, to do that we will use ufw and the commands are as follows

$ sudo ufw allow OpenSSH 
$ sudo ufw allow 20:21/tcp
$ sudo ufw allow 12000:12100/tcp
$ sudo ufw enable

Now that we have enabled, the ufw let’s just check to make sure everything is up and running.

$ sudo ufw status

This should show something similar to the below.

To                         Action      From
--                         ------      ----
OpenSSH                    ALLOW       Anywhere
20/tcp                     ALLOW       Anywhere
21/tcp                     ALLOW       Anywhere
12000:12100/tcp            ALLOW       Anywhere
OpenSSH (v6)               ALLOW       Anywhere (v6)
20/tcp (v6)                ALLOW       Anywhere (v6)
21/tcp (v6)                ALLOW       Anywhere (v6)
12000:12100/tcp (v6)       ALLOW       Anywhere (v6)
Finally, for these ports that you are allowing, remember to add them to your AWS security groups if you’re using AWS as your cloud provider.

3. Create User
Next, we are going to to create a user with the required credentials and access rights, for this example, we will be creating a FTP user with the username ftpuser.

$ sudo adduser ftpuser

Next, we will limit the user to only be allowed to use FTP and not allow the user to access SSH, we’ll make changes to the /etc/ssh/sshd_config by using the following command.

$ sudo nano /etc/ssh/sshd_config

Add the following line to the /etc/ssh/sshd_config file.

DenyUsers ftpuser

Finally, save and restart the SSH service

$ sudo service sshd restart

4. Access Rights
There are different ways to create the access rights, but I will assume we are using the use case where this user will only be able to upload to his own home directory. If you are interested in other use cases such as uploading to a web directory, consider following the links shown at the bottom of the page.

Let’s start by creating the user folder.

$ sudo mkdir /home/ftpuser/ftp

Set the ownership of the ftp directory to nobody:nogroup

$ sudo chown nobody:nogroup /home/ftpuser/ftp

Next, set the permissions so that everyone will not be able to have write permissions. We do this by using the following command, the flag a-w can be read as ‘all/everyone remove write permissions’

$ sudo chmod a-w /home/ftpuser/ftp 

Then, we will create a files sub folder where the user is allowed to upload files to.

$ sudo mkdir /home/ftpuser/ftp/files

And, assign ownership to him.

$ sudo chown ftpuser:ftpuser /home/ftpuser/ftp/files

5. Configure FTP server
This is the part that tripped me up, and mostly, it was because of the passive modes. So just be aware, and I’ll explain where you need to take extra precautions when you’re there.

Configure the vsftpd configuration file located in /etc/vsftpd. But first, create a backup.

$ sudo cp /etc/vsftpd.conf /etc/vsftpd.conf.bak
$ sudo nano /etc/vsftpd.conf

Next, ensure you turn on or have these flags within the configuration file. Remember to change pasv_address=x.x.x.x with the IP address of your server. Also, ensure listen=YES is set to YES. Without both of these, you might face the warning message from your FTP client such as “Server sent passive reply with unroutable address. Using server address instead.”.

I have enabled all of these

listen=YES
listen_ipv6=NO
write_enable=YES
chroot_local_user=YES
local_umask=022
force_dot_files=YES
pasv_enable=YES
pasv_min_port=12000
pasv_max_port=12100
port_enable=YES
pasv_address=x.x.x.x
user_sub_token=$USER
local_root=/home/$USER/ftp

Finally, restart the ftp server and check that everything is up and running properly with the following commands.

$ sudo systemctl restart vsftpd
$ sudo service vsftpd status

A proper working example would be something like.

● vsftpd.service - vsftpd FTP server
   Loaded: loaded (/lib/systemd/system/vsftpd.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2020-03-26 04:16:27 UTC; 3s ago
  Process: 26682 ExecStartPre=/bin/mkdir -p /var/run/vsftpd/empty (code=exited, status=0/SUCCESS)
 Main PID: 26693 (vsftpd)
    Tasks: 1 (limit: 1152)
   CGroup: /system.slice/vsftpd.service
           └─26693 /usr/sbin/vsftpd /etc/vsftpd.conf
If there would be an error, you will know with something like the following.

● vsftpd.service - vsftpd FTP server
   Loaded: loaded (/lib/systemd/system/vsftpd.service; enabled; vendor preset: enabled)
   Active: failed (Result: exit-code) since Thu 2020-03-26 04:13:21 UTC; 1min 1s ago
  Process: 26445 ExecStart=/usr/sbin/vsftpd /etc/vsftpd.conf (code=exited, status=2)
  Process: 26434 ExecStartPre=/bin/mkdir -p /var/run/vsftpd/empty (code=exited, status=0/SUCCESS)
 Main PID: 26445 (code=exited, status=2)
Finally, everything should work as expected, so log in with your favourite ftp client, you can even use your browser if you wish.



Setup FTP with TLS
The better approach would be to use FTP over TLS.

To do this we will follow all the steps above and more. We’ll make the following small changes.

1. Allow TLS ports in Firewall
Add the following ports to ufw.

sudo ufw allow 990/tcp
Remember to add them in Security Group also.

Create new Certificate
Let’s create a certificate with openssl with the following commands. You can hit enter for all the questions to use the default values.

sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/vsftpd.pem -out /etc/ssl/private/vsftpd.pem
Update FTP Configuration
Finally, we just need to add or uncomment the following lines to /etc/vsftpd.conf. To revert to using the insecure FTP, just change ssl_enable=NO.

ssl_enable=YES
rsa_cert_file=/etc/ssl/private/vsftpd.pem
rsa_private_key_file=/etc/ssl/private/vsftpd.pem
allow_anon_ssl=NO
force_local_data_ssl=YES
force_local_logins_ssl=YES
ssl_tlsv1=YES
ssl_sslv2=NO
ssl_sslv3=NO
require_ssl_reuse=NO
ssl_ciphers=HIGH
Finally, restart server and check that it is up.

$ sudo systemctl restart vsftpd
$ sudo service vsftpd status

