# Deploying a node project with codeship
## Prerequisites
* You must have [git](https://git-scm.com/downloads) and [xcode](https://developer.apple.com/xcode/) installed
* You must have a [Ubuntu 16.04.02](https://github.com/spencerlee200/spinnynotes/blob/master/setup.md) server set up

## 1. Setting up your local repo
Lets get the easy part out of the way. Navigate to the folder where you want to keep your local files and clone down your git repository. If you don't already have a git repository go create one making sure to initialise it with a readme.md file and clone it down with into your desired folder. Go ahead and cd into that project folder 


```shell
cd <your project directory>
```

## 2. Adding CodeShip to your project
Next we are going to add CodeShip to our project. Thankfully you only have to go through this process the one time. To get started head over to codeship.com and make a free account. Once you are logged in hit the new project button.

You will be presented with three options, github, bitbucket, and gitlabs. Click on the github icon on the far left.
![Connecting scm](http://i.imgur.com/IwdsBtW.png)

Next you are going to be prompted to enter the cloning URL for your github repository. Go copy that from your repo and paste it in the box.
![Connect your repo](http://i.imgur.com/kJMvbmt.png)

Next you will be prompted to pick a package. Unless you plan on going way outside of this projects scope you shouldnt need anymore than the basic package.

![packages](http://i.imgur.com/qP1rDBC.png)

After that you will be asked to verify some test scripts. For now we are not going to be worrying about those but if you wanted to add tests into your pipeline that would be a great place to do it. Hit save and go to dashboard to continue on to the next step.

![continue](http://i.imgur.com/mHt7IH9.png)

Now you need to find the settings menu in the top right. Click on it and navigate to the deployment page, as seen in the image below.

![menu](http://i.imgur.com/zaCwN0G.png)

Once you are on that page you will see a form that looks like this. Whenever codeship sees an action has occured on the branch that you name it is going to run our deployment script. To fix in our pipeline that is going to be on release for staging and master for production. So you will need to repeat this step twice to make a pipeline for each. All you have to change is the server IP in the custom script.

![form](http://i.imgur.com/bFA6GaC.png)

The next screen you get will give you a bunch of options for scripts. Scroll to the bottom and select the ever faithful custom script module.

![custom script](http://i.imgur.com/GBbFPeU.png)

Now just add in the line below replacing the IP with the IP of whatever server you are deploying to and the path with the path to your projects working directory you configured earlier on the server be it production or staging (`var/www/html/<your project name>`).. I've also typed it into a code block for your convenience. 
![the script](http://i.imgur.com/d6aygP2.png)

```shell
rsync -avz ~/clone/ root@<your server IP>:/path/on/server/
```
The next and final steps are simple. Navigate back to that same settings menu from earlier and go to the settings page. Here you will find an SSH key box like the one below.

![key](http://i.imgur.com/I7RuzWk.png)

Copy the entire contents of that box and then follow the commands below to get into the SSH key access file on your server.

```shell
ssh root@<your server IP>
cd .ssh 
nano authorized_keys
```

Now just paste in that SSH key on a new line, save the file, and exit. Thats all there is to it! Now as long as you follow the deployment process below you can push deployments in as little as one line from the development branch.

## 3. Deploying
First lets set up our branch structure. From the root of your directory we are going to first branch off the master branch to create a branch for our releases and then branch off of that to create a branch for our development to take place on. Under dev you should be creating feature branches that you merge back into dev. For this example though we are just going to work directly on dev.Lets go ahead and create our release and dev branch. 

```shell
git checkout -b release
git push origin release
git checkout -b dev
```

Next drop in any files you want to upload into the folder and add your files to the git stage.

```shell
bash deploy.sh
```

Its going to ask you to enter a commit message and once you’ve done that everything will be pushed to github. Now you just have to create a pull request to merge dev into release and when that goes through codeship will automatically deploy your project.


## 4. Enabling your site
Because this a node application there is actually one more step that is neccesary for our deployment process. First we need to SSH into our server.

```shell
ssh <name>@<your server IP>
```

Next we need to navigate into our working directory.

```shell
cd /var/www/html
```

Now we are just going to go ahead and start our new app. In your next deployment you should use the next step and skip this one.

```shell
sudo npm install -g pm2
pm2 start <your apps starting point>.js
```

Remember to do this everytime you deploy to your server so that PM2 actually knows it needs to make changes on your app.

```shell
pm2 restart all
```

Finally just restart nguni and everything should be up and running!

```shell
sudo systemctl reload nginx
```
