>
> This is no longer needed as replication takes place across all rulebook nodes. 
>

# Rulebook Node Static Data Replication
This repository contains all the necessary tools and scripts to set up a lightweight file repository for Rulebook Node static data replication. 

This project is intended for Ubuntu OS, so that static data from multiple Rulebook Nodes can be written and read by others when Nodes are down for maintenance.

> This is a convenience repository and is optional. You can set up your own SFTP server using your own processes.

We're going to go through the following steps:
- Create an instance of Ubuntu on a Cloud Service of your choice
- Update your DNS settings
- SSH onto the instance
- Hardening the Ubuntu Box
- Create an SFTP user
- Privilege the SFTP account
- Clone this repository onto the box
- Secure public read access with Let's Encrypt and a Reverse Proxy

> :warning: These instructions are for Linux based operating systems.

# Instance

1. Choose a Cloud Provider: Select a cloud provider of your choice that offers virtual machine instances. Popular options include Amazon Web Services (AWS), Google Cloud Platform (GCP), Microsoft Azure, and DigitalOcean.

2. Provision an Instance: Create a new instance on your chosen cloud provider's platform.  as this is a static data repository, you can use one of their smallest instances.

3. Configure SSH Access: During instance creation, configure SSH access to the instance. This typically involves setting up an SSH key pair. Generate an SSH key pair if you don't already have one, and provide the public key when creating the instance. This will allow you to securely connect to the instance using SSH.

4. Set up Tunnelling: To enhance security, set up tunnelling to encrypt and secure the communication between your local machine and the instance. One common approach is to use SSH tunneling, which establishes an encrypted tunnel between your local machine and the instance.

5. Connect to the Instance: Once the instance is provisioned and SSH access is configured, you can connect to the instance using an SSH client. Use the private key associated with the SSH key pair to authenticate the SSH connection. The specific SSH command may vary depending on the cloud provider and the operating system you are using to connect.

6. Set up IPv4 and IPv6 Networking: Ensure that your instance is configured to support both IPv4 and IPv6 networking protocols. This is required by Let's Encrypt to set-up free TLS.

The specific steps and configurations may vary depending on the cloud provider and the operating system of the instance.

# DNS

Set up your DNS `A` (v4) and `AAAA` (v6) records two point at the correct IP addresses. 

We recommend using a subdomain called `static`. E.g. `static.example.com`

# Hardening the Ubuntu Box

The following instructions provide steps to harden an Ubuntu box for improved security.

## Update Package Repositories

Before proceeding with the hardening steps, ensure that your system's package repositories are up to date by running the following commands:

```shell
sudo apt-get update
sudo apt update
```

## Add User

To enhance security, it is recommended to create a non-root user for regular system usage. Follow the steps below to add a new user with a name of your choosing.

```shell
sudo adduser <an unusual username>
sudo groupadd admin
sudo usermod -a -G admin <an unusual username>
```

## Restrict Root Access

It is advisable to disable direct root logins over SSH. Open the SSH server configuration file for editing:

> ⚠️ If you have enabled key access, don't forget to change the key access to the new user. 
  


```shell
sudo pico /etc/ssh/sshd_config
```

Inside the file, find the line containing `PermitRootLogin` and update it to:

```
PermitRootLogin no
```

Save the changes and exit the editor.

## Secure Shared Memory

To improve security, you can secure the shared memory by following these steps:

1. Open the `/etc/fstab` file for editing:

   ```shell
   sudo pico /etc/fstab
   ```

2. Add the following line at the end of the file:

   ```
   tmpfs /run/shm tmpfs defaults,noexec,nosuid 0 0
   ```

   This line ensures that the shared memory is mounted with specific security options.

3. Save the changes and exit the editor.

## Reboot the System

To apply the changes made during the hardening process, it is recommended to reboot the system:

```shell
sudo reboot
```

After the system restarts, the hardening configurations will be in effect.


# SFTP User Set-up

To set up an SFTP user on Ubuntu OS, follow the instructions below:

1. Open a terminal on the Ubuntu server.

2. Create a strong password and put it in the envfile for the rulebook node, `SFTP_PASSWORD`.

    ```
    openssl rand -base64 32
    ```
    > Be careful not to check this into a repo by accident.  If you do, it must be changed. 

3. Run the following command to create a new user named `sftpnode`:
   ```
   sudo adduser sftpnode
   ```

4. You will be prompted to set a password for the new user.


5. Create the upload directory for the SFTP user by running the following command:
   ```
   sudo mkdir -p /var/sftp/uploads
   ```

> N.B. When publishing static data the Rulebook Node will look for `/uploads` if it cannot write to `/`

1. Set the appropriate ownership and permissions for the `/var/sftp` directory and `/var/sftp/uploads` subdirectory by executing the following commands:
   ```
   sudo chown root:root /var/sftp
   sudo chmod 755 /var/sftp
   sudo chown sftpnode:sftpnode /var/sftp/uploads
   ```

2. Open the SSH server configuration file using a text editor. In this example, we'll use the `pico` editor, but you can use any text editor you prefer:
   ```
   sudo pico /etc/ssh/sshd_config
   ```

3. Scroll down to the end of the file and add the following lines:
   ```
   Match User sftpnode
   ForceCommand internal-sftp
   PasswordAuthentication yes
   ChrootDirectory /var/sftp
   PermitTunnel no
   AllowAgentForwarding no
   AllowTcpForwarding no
   X11Forwarding no
   ```

4. Save the changes and exit the text editor.

5. Restart the SSH service to apply the new configuration:
   ```
   sudo service ssh restart
   ```

With these steps, you have set up an SFTP user named `sftpnode` with a strong password and configured the SSH server to restrict the user to SFTP access only, chrooted to the `/var/sftp` directory.

Please note that these instructions assume you have administrative privileges (`sudo`) on the Ubuntu server. Adjust the commands as needed based on your specific setup and requirements.

# Install Docker
To install Docker on Ubuntu, follow these step-by-step instructions:

1. Open a terminal on your Ubuntu system.

2. Update the package repositories by running the following command:

   ```shell
   sudo apt update
   ```

3. Install the necessary packages to allow the use of repositories over HTTPS and manage software properties:

   ```shell
   sudo apt install apt-transport-https ca-certificates curl software-properties-common
   ```

4. Import the Docker GPG key to ensure the authenticity of the Docker repository:

   ```shell
   curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
   ```

5. Add the Docker repository to the system's package sources:

   ```shell
   sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable"
   ```

   Note: Replace "bionic" with your specific Ubuntu version codename if different.

6. Update the package repositories again to include the Docker repository:

   ```shell
   sudo apt update
   ```

7. Check the available Docker package versions:

   ```shell
   apt-cache policy docker-ce
   ```

   This command displays the available Docker versions, allowing you to choose the one you want to install.

8. Install Docker by running the following command:

   ```shell
   sudo apt install docker-ce
   ```

9. Start the Docker service and verify its status:

   ```shell
   sudo systemctl start docker
   docker --version
   ```

This will start the Docker service and display its status, indicating whether it is running successfully.

## Install Local Persist

To install Local Persist for Docker, follow the instructions below:

> ⚠️ You must switch user to `root` as this installation step will not work with `sudo`

1. Open a terminal on your system.

2. Run the following command to download and execute the Local Persist installation script:

   ```shell
   curl -fsSL https://gist.githubusercontent.com/deviantony/bb3ff49aa117ea5294049e3470ef75f5/raw/c2a30b3398d62ddd34ceaee1dee67f184bca9e98/local-persist-install-nosudo.sh | bash
   ```

   This command fetches the installation script from the provided URL and pipes it to the `bash` command for execution. The script will install Local Persist on your system.

3. After the installation completes, update the system's package repositories:

   ```shell
   sudo apt-get update
   ```

   This command ensures that your system's package repositories are up to date.

4. Upgrade installed packages, if necessary, to the latest versions:

   ```shell
   sudo apt-get upgrade -y
   ```

   This command updates installed packages to their latest versions, ensuring your system has the latest bug fixes and security patches.

With these instructions, you can install Local Persist for Docker on your system and ensure that your system is up to date with the latest package upgrades.

# Produce the SFTP Fingerprints
Using `ssh-keyscan` for obtaining and verifying SSH server fingerprints in SFTP offers several benefits. It enables server identification, protecting against man-in-the-middle attacks by comparing fingerprints with trusted sources. 

```
ssh-keyscan static.exonym.io
```

You should get an output as follows:  `...` indicates data has been removed.

```
static.exonym.io ssh-rsa AAAAB3NzaC1yc2EAAAADAQAB...hDkQcnS+Iggzs9+FkuHgoBstJ+MiT69hnFOFhS7zuDveGQ7gCHCZU=

static.exonym.io ecdsa-sha2-nistp256 AAAAE2VjZHNhLXN...Wnd2GCm0GFjnBbcx8tVeWIBt+rhHk8ZecxhKSMCLIuTI=

static.exonym.io ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIMZcM3xdTq24Pp47J8xJZ8iXuxUP/MvyBBNBLzwfm5l6

```
In your rulebook node envfile; insert these values into variables; `export KNOWN_HOST0="static.ex..."`, `KNOWN_HOST1`, and `KNOWN_HOST2` being careful not to truncate the data.

# Everything should be set up.

Let's start the fileserver and get a TLS certificate.

   ```
   cd /home/<username>
   git clone https://github.com/exonym/static-fileserver.git
   cd static-fileserver
   pico envfile.env // make changes
   su root
   ```

Enter the password. _(TODO - non-root docker set-up)_
   ```
   docker compose up -d
   cd /var/sftp/uploads
   echo "hello world" > file.html
   ```

Open browser and navigate to your static domain and you should see "hello world" and the TLS connection is secured.

The Rulebook Node will publish to; 

```
https://static.example.com/STATIC_DATA_FOLDER/rulebook.json
https://static.example.com/STATIC_DATA_FOLDER/x-node/
https://static.example.com/STATIC_DATA_FOLDER/x-source/
```
The files are indexed via: 

```
https://static.example.com/x-node/signatures.xml
```

The static data will be accessible to all, even during node maintenance. 

Recall that you can use this fileserver for all your Rulebook Nodes and even publish a static website on the location too. 

Simply use `STATIC_DATA_FOLDER` as a namespace.

If you are a Source you can set up your connections publish and bring down your node, only bringing it up when you need to add and remove Advocates.


# Disclaimer

Please note that while this repository is designed for robust and reliable operation, data replication involves inherent risks. Always make sure to have backup systems in place and regularly verify the integrity of your data. This repository is provided as-is, and the maintainers cannot be held responsible for any data loss.

The instructions are intended for educational or testing purposes only and should not be implemented in a production or live environment without proper consideration and understanding of the potential risks involved. Applying these instructions to a production system without adequate knowledge and expertise in security practices may lead to unintended consequences, system vulnerabilities, or service disruptions. 

We strongly advise against applying these instructions to any critical or sensitive systems. Always consult with qualified security professionals and thoroughly assess the impact and suitability of any security measures before implementing them in a production environment. It is important to conduct proper testing, evaluation, and risk analysis specific to your environment before making any changes that affect system security. 

By using these instructions, you acknowledge that you assume all risks and responsibilities associated with their implementation. We disclaim any liability for damages, losses, or security breaches that may arise as a result of following these instructions. Use these instructions at your own discretion and ensure you have proper authorization and consent to perform any actions on the systems involved.
