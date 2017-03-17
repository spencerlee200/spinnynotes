# Setting up a server for SpinnyNotes

##Table of Contents
* [Initial Server Setup](#1)
* [Installing Nginx](#2)
* [Installing Node](#3)
* [Installing MongoDB](#4)
* [Installing ALL the packages](#5)
* [Creating a remote repo](#6)
* [Deployment](#7)


<a name="1"></a>
## Initial Server Setup
### Setting up a virtual server with Digital Ocean
Once you've logged in or created a new account with [Digital Ocean]('https://www.digitalocean.com/') spinning up a virtual server (otherwise known as a droplet) is super easy. Navigate to the droplets page in the top navigation and hit the create droplet button. For this set up we are going to be using the **Ubuntu 16.04.2 x64** image. Select it from the dropdown menu under Ubuntu.

![Digital Ocean Images](http://i.imgur.com/vha5P8m.png)

Now you are going to have to select a payment plan. For this project the 5 dollar a month option should be plenty. You can always go in and change this as your application scales up.

![Digital Ocean Plans](http://i.imgur.com/lXhXhlp.png)

Next you are going to be prompted to select a datacenter. You should go with the one that is closest to you and your target audience. You may also notice that each datacenter has a number associated with it. The higher the number the newer the datacenter is.

![Digital Ocean Datacenters](http://i.imgur.com/l4QD4ks.png)

That's really all there is to it. Digital Ocean provides you with a couple more options such as private networking and adding SSH keys but we will handle all of that stuff as we come to it. Just give it a hostname, hit that big green create button and move on to the next step.

![Digital Ocean Options](http://i.imgur.com/ADtkDBx.png)

### Configuring the new server
Now its time for the fun part. Start by opening up a new terminal window. First things first we need to SSH into our server. To do this copy the command below and replace `<your server IP address>` with the IP address from your droplet on Digital Ocean.

```shell
ssh root@<your server IP address>
```

Since this is your first time connecting you will probably get a warning about the authenticity of the host. Just type in yes and hit enter one more time. Here you should be prompted for a password. Digital Ocean should of provided you with this password in an email. Copy in your password and hit enter once again to finally connect to your server. You should see a screen similiar to this one: 

![Root ssh](http://i.imgur.com/OITGzLK.png)

By default Ubuntu requires you to immediately change the password for your root user for security reasons. To change it simply enter your old password again and enter your new password following the onscreen prompts. Once you've done that you should see something line `root@wordpress:~#` on a new line. Now we can start editing things.

#### Adding a new user
Because of the root users extremely broad privileges, working in root opens up the potential for a lot of destructive changes to occur. To mend this we are going to go ahead and set up a new user. Copy this line into terminal replacing `<name>` with the desired name for your new user and hit enter. Then follow the onscreen prompts to set that users password and fill in the prompts with any additional information you need your user to have.

```shell
adduser <name>
```

Next, we need to give your new user regular account privileges. Without this step your new user wouldn't be very useful. To do this simply enter the line below into terminal, once again replacing `<name>` with your new user.This will give your user the power to use sudo commands by adding it to the sudo group.

```shell 
usermod -aG sudo <name>
```

Finally to switch to your newly created user enter this line into your terminal subbing in your own user name.

```shell 
su - <name>
```

#### Disabling root access (optional)
Now that we are on our new users account we can go ahead and create an extra security feature by configuring our SSH daemon to not allow remote SSH access to our server. Luckily this is pretty easy to do and can be done in just a few commands. First access the config file:

```shell
sudo nano /etc/ssh/sshd_config
``` 

Next look for a line that says `PermitRootLogin yes` and modify it to say `PermitRootLogin no` then exit and save the file (`CTRL+X`, then `Y`, then `ENTER`). Finally reload SSH with the command below to finalize these changes.

```shell
sudo service ssh restart
```

#### Adding public key authentication (optional)
This step is another optional security step but itll make it quicker and easier for you to connect to your sever. If you were to disable passwords then only users with your key could get in. For now though we are just going to make it easier for our current user to connect. Open a new terminal window and enter this line: 

```shell
ssh-keygen
```

You can set a passphrase to secure the key with or just leave it blank. Hit enter through the prompts until the keys are generated. Remember that if you do set a passphrase you will need both the passphrase and password to SSH into your server. Next you need to install the public key to your user. To do this enter this line in the same terminal window you generated the keys in replacing `<name>` with your users name and `<your server IP address>` with your servers IP just like you would when connecting. You will have to enter your users password again at the end of the prompt.

```shell
ssh-copy-id <name>@<your server IP address> 
```

<a name="2"></a>
## Installing Nginx
Just in case we have any packages in need of update we should run an update before installing nginx. Make sure you are logged into your server as your user and enter this line into terminal. You'll have to confirm that you want to allocate disk space for the install. 

```shell
sudo apt-get update && sudo apt-get install nginx
```

One this command finishes running you can check if it worked by navigating to your servers public IP in the browser. You should see the basic nginx install screen.

![nginx install screen](http://i.imgur.com/gniYuwZ.png)

<a name="3"></a>
## Installing Node

Serving node applications with Digital Ocean is also a surprisingly easy process. To get started, lets move to our home directory on our server and download the script we need to install node.

```shell
cd ~
curl -sL https://deb.nodesource.com/setup_6.x -o nodesource_setup.sh
```

Next we need to actually run that script in order to get the actually get Nodes functionality available to us.

```shell
sudo bash nodesource_setup.sh
```

There are a few packages we also should go ahead and grab that we will need to actually run the code inside of our application. Navigate into your host directory, give your user write permissions, and then install the neccesary packages with the lines below:

```shell
cd /var/www/html
sudo chown -R <name>:www-data /var/www/html/

sudo apt-get install nodejs && sudo apt-get install build-essential && sudo npm install express
```

Now we need to update our nginx configs server block that we will need to access our node application. This server block takes advantage of reverse proxies so that rather than having to enter the port in your URL bar the server will automatically point you to the information you are going to be altering.


```shell
sudo nano /etc/nginx/sites-enabled/default

server {
        listen 80;
        listen [::]:80;

        root /var/www/html/;

        server_name <your server IP>;

        location / {
                proxy_pass http://<your server IP>:3000;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection 'upgrade';
                proxy_set_header Host $host;
                proxy_cache_bypass $http_upgrade;
        }
}
```

<a name="4"></a>
## Installing Mongoose 
Next we need to install mongoos. First lets get back to our root directory and get the repo that we need.

```shell
cd ~

sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv EA312927
```

Then we need to create a list file for MongoDB with the command below

```shell
echo "deb http://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.2 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-3.2.list
```

After adding the repository details we need to update the packages list:

```shell
sudo apt-get update
```

Now we can install the MongoDB package itself.

```shell
sudo apt-get install -y mongodb-org
```

Next we need to create a unit file to manage the MongoDB service. Create the file with the command the nano command below and then paste in the code below it.

```shell
sudo nano /etc/systemd/system/mongodb.service

#Copy into mongodb.service

[Unit]
Description=High-performance, schema-free document-oriented database
After=network.target

[Service]
User=mongodb
ExecStart=/usr/bin/mongod --quiet --config /etc/mongod.conf

[Install]
WantedBy=multi-user.target
```

Finally we just need to start it MongoDB and everything is done!

```shell
sudo systemctl start mongodb
```

<a name="5"></a>
## Install ALL the packages
![meme](http://i.imgur.com/qOHfy68.jpg) 

First we need to get into our working directory

```shell
cd /var/www/html
```

This next step is admittedly pretty tedious and annoying. Because I created a dependency heavy project there are quite a lot of packages that need to install. To simplify this, I've just chained a massive list of commands. 

```shell
sudo npm install body-parser connect-ensure-login dotenv ejs express express-session glob mongoose passport passport-github2 passport-google-oauth20 passport-local pug request request-promise
```

<a name="6"></a>
## Create remote repo
This remote repositories are important to our deployment pipeline because it will allow us to push code live to the site from our local machines.To start lets go ahead and create the directory where we will be storing our project and give our user ownership of it.

```shell
sudo mkdir /var/repos
sudo chown <user> /var/repos
cd /var/repos
```

Finally lets create our repo for our node app. I decided to call mine `node.git`. 

```shell
cd /var/repos
mkdir node.git
```

Next we need to setup our new repo as a bare git respository. 

```shell
cd node.git
git init --bare
```

Then we are going to create a new git hook that will allow us to automate pushing our code live when we deploy it to the site. First, lets nano into a new git hook file:

```shell
nano hooks/post-receive
```

Inside of this file you need to create a bash script that will copy your files to the live server directory (`/var/www/html/static`):

```shell
#!/bin/sh
git --work-tree=/var/www/html/ --git-dir=/var/repos/node.git checkout -f
```

Now just exit and save the file (CTRL + X, then y, then ENTER). The last thing that you have to do it give this file executable permissions with:

```shell
chmod +x hooks/post-receive
```

<a name="7"></a>
## Deployment
Thats it for the set up. Next you have to actually deploy the project from your local machine. You can see how I did that [here](https://github.com/spencerlee200/spinnynotes/tree/master).
