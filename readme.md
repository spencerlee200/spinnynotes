#Deploying static projects utilizing Git hooks

##Prerequisites
* You must have [git](https://git-scm.com/downloads) and [xcode](https://developer.apple.com/xcode/) installed
* You must have a [Ubuntu 16.04.02]() server set up

##1. Setting up your local repo
Lets get the easy part out of the way. Navigate to the folder where you want to keep your local files and clone down your git repository. If you don't already have a git repository go create one making sure to initialise it with a readme.md file and clone it down with into your desired folder. Go ahead and cd into that project folder 

```shell
cd <your project directory>
```

We also need to go ahead and add in our remote that we created on `/var/repos` in the setup for this static build. To add this just copy the line below and change `<user>` to your ssh users name, `<your project name>` with your projects name, and `<server ip>` to your servers IP. Git will handle the rest.

```shell
git remote add Production ssh://<user>@<your production ip>:/var/repos/<your project name>.git
```

##2. Creating your git hooks

Now its time to start creating the githooks that will allow for the automation of certain processes in your deployment pipeline. To get started navigate inside of the hooks folder in your `.git` directory

```shell
cd .git/hooks
```

###post-merge

There isnt an example file for a post merge created in the git init process so we are going to have to create this file ourselves. First lets create the file:

```shell
nano post-merge
``` 

Next you need to copy or write out the bash script below into the file. 

```shell
#!/bin/bash
branch=$(git symbolic-ref --short HEAD)

if [[ ${branch} == "release" ]];
        then
        echo "Pushing to remote Production server"
        git push Production release:master
        exit
else
        echo "Branches merged. Remote not updated"
        exit
fi
```

To explain what this is doing in greater detail, the variable branch is being set to the name of the current git branch and then it is being compared to the string "release". In my git workflow I only want to push code live if a merge occurs on my release branch. You could change this to any branch that you wanted or just leave it and use the git workflow that I outline later in this tutorial. From the results of that conditional the hooks either pushes the code to the remote repository or it exits after informing you if it succeeded or not.

We also need to give this hook the ability to be executed with the line below:

```shell
chmod +x post-merge
```

Then return to your root directory by running `cd ..` twice in terminal.

##4. Deploying
Its finally time to tie all of these steps together and see if all of your hardwork has payed off. First lets set up our branch structure. From the root of your directory we are going to first branch off the master branch to create a branch for our releases and then branch off of that to create a branch for our development to take place on. Usually you would have feature branches under that development branch but for this small change we dont need to worry about it. 

```shell
git checkout -b release
git checkout -b dev
```

Next drop in any files you want to upload into the folder and add your files to the git stage. Remember that you dont need to include your node_modules folder since it already exits in the repo. The easiest way to do this would be to add it to a .gitignore file like 

```shell
echo 'node_modules' > .gitignore
git add .
```

Next we have to commit it, just like we normally would in a git workflow.

```shell
git commit -m "Your descriptive commit message here"
```

Now we need to switch back to our release branch and merge in our dev branch so that the two are even. Once you've done this your humans.text file should be sent to your virtual environment.

```shell
git checkout release
git merge dev
```

If this all works then you should see a message like the one below confirming your files have been sent to the remote repository. Now if you ssh into your virtual server all your added files should be there!

![Deployment success](http://i.imgur.com/SHWYjOH.png)

## 5. Enabling your site
Because this a node application there is actually one more step that is neccesary for our deployment process. First we need to SSH into our server.

```shell
ssh <name>@<your server IP>
```

Next we need to navigate into our working directory.

```shell
cd /var/www/html
``` 

If this is your first time deploying and you followed along with my setup process you already have an app running from this directory with the name `hello.js`. We are just going to go ahead and remove that app from PM2 and start our new app. You can skip this step in all future deployments.

```shell
pm2 delete all
sudo rm hello.js
pm2 start <your apps starting point>.js
```

All thats left to do is restart PM2! Remember to do this everytime you deploy to your server so that PM2 actually knows it needs to make changes on your app.

```shell
pm2 restart all
```