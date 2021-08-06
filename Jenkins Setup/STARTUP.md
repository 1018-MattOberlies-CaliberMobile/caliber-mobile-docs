# Jenkins Setup

## Setting up the server

First, we want to set-up the server that will be running Jenkins. We'll be using the AWS EC2 service.

- In the inbound rules use these: https://api.github.com/meta, and lso add port 8080 for jenkins.

## Hacking into the server

### Prepping the ssh connection

In the home directory create a `.ssh` folder, and inside that folder create a `config` file and paste the following with the appropriate information (LEAVE THE `user` as is):

```txt
Host some-name
  Hostname endpoint-url
  IdentityFile ~/.ssh/keys/name.pem
  user ec2-user
```

As you can see above you'll need to create a `keys` folder and paste/move the `.pem` file (the key file) you downloaded when setting up the EC2.

To connect to the server run this command (and say yes):

```sh
ssh some-name
```

### Installing Everything

Create a file with these contents (this will install java, git, jenkins, and docker):

```bash
#!/bin/bash

# updating the server
sudo yum update

# requirements for jenkins
sudo yum install java-1.8.0-openjdk-devel -y
sudo yum install git -y

# jenkins:
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
sudo yum install jenkins -y

# start jenkins:
sudo systemctl start jenkins

# docker:
sudo yum install docker -y
sudo systemctl start docker
sudo usermod -aG docker ec2-user
sudo usermod -aG docker jenkins

# restart jenkins due to providing docker accesss
sudo systemctl restart jenkins

echo "Please reconnect"
```

Call it whatever you want (e.g., `jenkins-setup.sh`), give it write permissions and run it:

```sh
chmod +x jenkins-setup.sh && ./jenkins-setup.sh
```

### Adding more RAM to the server

If your server has less than `4G` of memory, you can use this script (call it e.g., `swap-it.sh`):

```bash
#!/bin/bash
MEM_ALLOCATE=${l:-"4G"} # default to 1 gig of memory
FILE='/swapfile'
echo 'Free memory before swap (MB):'
free -m
echo
echo 'Allocating disk space to swap file...'
sudo fallocate -l  $MEM_ALLOCATE $FILE
sudo chmod 600 $FILE
sudo mkswap $FILE
sudo swapon $FILE
echo
# we have to run the next command this way because fstab is owned by root (need to write with sudo)
# The tee command uses streams to write to the fstab file
# By writing to it, we can configure AWS to automatically mount an attached EBS volume after every reboot
echo "$FILE swap swap defaults 0 0" | sudo tee -a /etc/fstab # make the change permanent
echo "Memory allocated. Current free memory (MB):"
free -m
echo
echo "Run 'sudo swapon --show' to view the swapfile"
# To deactivate and remove the swap file:
# sudo swapoff $FILE
# remove entry in /etc/fstab
# sudo rm $FILE
```

If you need more RAM you can modify the second line and add more. Give the `swap-it.sh` file execution permissions and run it:

```sh
chmod +x swap-it.sh && ./swap-it.sh
```

Validate that swapfile worked with:

```
sudo swapon --show
```

### Setting up the jenkins password

At this point jenkins should be up and running. Go to your browser, and enter your server (EC2) endpoint with the port (e.g., `ec2.compute.aws.com:8080`).

Install all the suggested plugins (when prompted to).

Once the page loads it'll ask you for the password, it'll provide you with a path to a file which contains the default password:

```sh
cat /var/path/to/jenkins/password/file
```

This command will display the content of the file, therefore displaying the jenkins default password.

Enter that password and then setup your new password, and setup a user (and/or admin).

## Setting up the projects

### Install More Plugins

`Dashboard > Manage Jenkins > Manage Plugins`

Go to the Available tab and search for these:

1. Docker API Plugin
2. EnvInject API Plugin
3. NodeJS

### Setting up the project

Add a new item

Add a name and select freestyle project, and click OK.

[](./images/0.png)
[](./images/1.png)
[](./images/2.png)
[](./images/3.png)
[](./images/4.png)
[](./images/5.png)
[](./images/6.png)
[](./images/7.png)
[](./images/8.png)
[](./images/9.png)
[](./images/10.png)
[](./images/11.png)

We'll be using this docker file:

Go to Dashboard > Manage Configuration

In the Global properties setting set the environment variables (this will be used to deploy the backend through serverless):

- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
