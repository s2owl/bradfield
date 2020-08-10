+++
authors = [
    "Tobias Brecker"]
title = "Creating a website with automation and Firebase"
date = "2020-08-07"
description = "How I managed to create a simple Hugo website, build a CI/CD pipeline and deploy on Firebase"
tags = [
    "GCP","Google","GitHub","Cloud Build","Source Repositories","Firebase","Slack"
]
images = [
    "markdown-syntax.jpg",
]
+++

---
First of all, I've wanted to 'do' something for a while and I took the opportunity when I saw what someone had done using Hugo and AWS.  Given I'm on my GCP journey I figured I should take the challenge to myself and have a go.  

So what is this post about?  I've decided to almost make this post my learning journal if you like on how I achieved it.  I'm going to break this in to a few sections to make life easy and try to assume as little as possible but you're going to get signposted to various alternative sites.  What I'm going to cover is:
  1. Prep Work
  2. Using Hugo and setting it up locally
  3. Hosting the site in GitHub
  4. Synchronising GitHub with Google Cloud Repository
  5. Making Cloud Build do something
  6. Firebase setup
  7. Getting Slack to tell you the Google Build status

Some background / objectives:  
There are, however, various operational complexities. You may want to version control your postings, host your web site on a content delivery network (“CDN”) and provision an SSL certificate.
You can address many of these process matters by using a Continuous Integration / Continuous Deployment pipeline. These pipelines enable developers to rapidly innovate by automating the entire deployment process. A website publishing platform would enable a content author to submit new content which would in turn launch the pipeline that in turn performs all the publishing tasks.
In this article, I will try and show you how to leverage Google Cloud to create a pipeline for deploying websites based on Hugo, a static site builder. You create your web content, run it through Hugo which in turn produces the web pages. We will use the following Google Cloud services:
Cloud Source Repositories will hold the content. The service provides private Git-compatible repositories that can trigger Cloud Build pipelines when commits are made to branches. You can also use GitHub if you prefer.
Firebase will host the website. Firebase is an application framework that provides a content delivery network (CDN) and SSL certificates.
Cloud Build will be used to create the pipeline. The pipeline consists of steps each of which is run in a container to produce the build artifacts.

#### Prep Work

- If you want a custom domain - buy it now on https://domains.google/intl/en-GB/ its very simple and very straight forward.  Having used AWS' Route 53 I find them both very intuitive and straight forward.
- If you are following this you need to have signed up to Google Cloud Console
    - You must have a project allocated for this and know how to set one up (inc billing aspects)
    - You have the appropriate permissions on Google Cloud to undertake this activity
- Have a GitHub account (we will create the repro in the steps) and have git commands available

##### Firebase Prep Work
Let’s start by creating a Firebase project. This will in turn set up an underlying Google Cloud project and configure it to use Firebase. We then need to set up billing and enable some APIs.  
- Open the Firebase console in a new browser tab.  
- Click “Add Project” or “Create Project.”  
- Enter “[PROJECT_NAME]” for the project. Edit the suggested project ID if you wish to be the value “[PROJECT_ID]”. Click “Continue”.  
- On the “Google Analytics” page, disable Google Analytics since this is a demonstration. Click “Create Project.”  

##### Google Cloud Prep Work
Open the Google Cloud console in a new browser tab.
Link the Firebase project to a billing account. If you do not have the privileges to do so, work with your organization’s administrators to do this for you. You will not be able to proceed to the next section without a billing account.
In the APIs & Services menu, enable the following APIs:
- Cloud Build
- Cloud Source Repositories

#### Using Hugo and setting it up locally

If you've never used Hugo before (I hadn't up until this moment) its pretty straight forward to setup.  There are two really good guides available on the gohugo.io website and you should follow them **both** before proceeding:
  1. https://gohugo.io/getting-started/installing/
  2. https://gohugo.io/getting-started/quick-start/

The quick start will introduce you to the themes.  You will perhaps spend more time looking at the theme options than anywhere else!

- https://themes.gohugo.io/

Follow the installation steps of the themes in to your local copy of the website.  I personally found it useful to copy the Example Site which comes with every theme to be my actual site and see how it works.  It takes time and you really don't need to rush this bit.  By then end of this step you need to be comfortable understanding how to view your site locally.

Static web platforms like Hugo have become popular because of their ability to produce websites that do not require web servers. With static web platforms there are no operating systems to patch and no web server software to maintain.

#### Hosting the site in GitHub

Firstly it might not be everyones approach to use GitHub here but I wanted the ability to invite peer review for my posts in an environment other IT professionals already know about.  My approach here was to transform my website folder as created by Hugo locally to be compatible with GitHub.  Then I wanted to synchronise the local repository with GitHub.  I **understand** people may want to do this first before building the site locally but I *preferred* this way.  Using the CLI browse to the website folder and enter:

    git init
    git add .
    git commit -m "New Repo Content"
    git push origin master

What we have now is a website in GitHub.  If you are familiar with GitHub you can make pull requests of the master branch in GitHub.  Action the following:

Create a .gitignore file to keep extra files from being committed.

    cd ~/[REPO_NAME]
    echo "public" >> .gitignore
    echo "resources" >> .gitignore
    git config --global user.mail "[GIT_NAME]"
    git config --global user.email "[GIT_EMAIL]"
    git add .
    git commit -m "Add app to Cloud Source Repositories"
    git push -u origin master

#### *From this point on we enter Google Cloud Platform*

#### Synchronising GitHub with Google Cloud Repository

I literally followed this document: https://cloud.google.com/source-repositories/docs/mirroring-a-github-repository it was really simple and clear to follow.

![Google Source Repositories](/images/SR.png)

#### Install Hugo in to Cloud Shell
We will now install Hugo within the Cloud Shell environment so we can build the website on Cloud Shell.
Open the Cloud Shell and ensure the default project ID is set to the ID of the Firebase project.
Install the Hugo executable from here: https://github.com/gohugoio/hugo/releases I used version 0.74.3 of the Linux archive (this one https://github.com/gohugoio/hugo/releases/download/v0.74.3/hugo_0.74.3_Linux-64bit.tar.gz). Place the Hugo executable in a directory off of your Cloud Shell home directory such as ~/bin and add ~/bin to your PATH.
Enter the commands below.

    cd ~
    hugo new site [REPO_NAME] --force

The hugo command will generate the site scaffolding into the directory associated with the cloned repository.
This will build out the skeleton of the site in the repository. Normally the hugo command creates the directory. The --force option will create the site in the repository directory which already exists.  Worth noting that I used the same name as the repository in GitHub which was synchronising with Cloud Repository and it will be referred to as [REPO_NAME] for the rest of the guide.

#### Making Cloud Build do something

Now that we have configured our project and understand the end-to-end deployment process, we will automate the process by creating a Cloud Build pipeline.
Configure the build
Cloud Build uses a file named cloudbuild.yaml in the root directory of the repository to perform the build.
Open the Cloud Shell if it is not already open.

    cd ~/[REPO_NAME]

Use your favorite text editor to create the cloudbuild.yaml file with the following text. Note that the line with the Hugo download URL is actually on the same line as the previous dash and that the firebase deploy command appears on two lines below but is actually one line. Also, remember that spacing *does* matter in the YAML syntax.

    steps:
    - name: 'gcr.io/cloud-builders/wget'
      args:
      - '--quiet'
      - '-O'
      - 'firebase'
      - 'https://firebase.tools/bin/linux/latest'

    - name: 'gcr.io/cloud-builders/wget'
      args:
      - '--quiet'
      - '-O'
      - 'hugo.tar.gz'
      - 'https://github.com/gohugoio/hugo/releases/download/v${_HUGO_VERSION}/'hugo_${_HUGO_VERSION}_Linux-64bit.tar.gz
      waitFor: ['-']

    - name: 'ubuntu:18.04'
      args:
      - 'bash'
      - '-c'
      - |
        mv hugo.tar.gz /tmp
        tar -C /tmp -xzf /tmp/hugo.tar.gz
        mv firebase /tmp
        chmod 755 /tmp/firebase
        /tmp/hugo
        /tmp/firebase deploy --project ${_FIREBASE_PROJECT_NAME} --non-interactive --only hosting -m "Build ${BUILD_ID}"

    substitutions:
      _HUGO_VERSION: 0.69.2   

#### What does this block mean?

There are three named steps in this file each of which is performed by a container image. The first two steps use a Google-supported builder to perform a wget command to download the Hugo and Firebase tools. These two steps run in parallel. Using the wget builder is faster than installing wget manually.
The third step uses a standard Ubuntu container to install Hugo and Firebase after which the site is built and deployed. You could also create your builder container that contains Hugo and Firebase. I chose this approach for two reasons. First, it’s useful to be able to change the version of Hugo. Second, I want to get the latest version of Firebase with the most up to date features.
The file also uses two custom substitution variables (_HUGO_VERSION and _FIREBASE_PROJECT_NAME)_ to allow for this template to be used in different environments.
The Hugo and Firebase binaries are created and installed in a temporary directory so that they do not inadvertently get deployed to the website itself.

#### Deploy the site to Firebase
Now that we have viewed the website locally, we will perform a one time deployment to Firebase to validate the configuration.
Open the Cloud Shell if you previously closed it.

    cd ~/[REPO_NAME]
    firebase login --no-localhost

You will be asked to visit a Google sign in URL. Do **not** click on the URL, rather copy and paste the URL into your browser. Take the resulting authorization code and paste it into the prompt.

    firebase init

Select Hosting using the arrow and spacebar keys. When asked for a project option, select “Use an existing project” and then select [PROJECT_NAME]. For the public directory, select the default value “public”. For configuring as a single page application, select the default value of “N”.

    firebase deploy

After the application has been deployed, you will receive a hosting URL. Click on it and you will see the same website being served from the Firebase CDN (content delivery network). If you receive a generic “welcome” message, wait a few minutes for the CDN to be initialized and refresh the browser window. Save this hosting URL for later use.

#### Create the Cloud Build trigger
In order to start Cloud Build when changes are committed to the repository, we will need to define a Cloud Build trigger. Follow these steps to create the trigger making the following changes:
- Pick a name for the trigger such as “commit-to-master-branch”.
- For the event, select “Push to a branch”, select the Cloud Source Repository we created earlier, and select “master” as the branch.
- For Build configuration file type, accept the default location of cloudbuild.yaml.
- Add the substitution variable _FIREBASE_PROJECT_NAME __ and set it to be the value of “[PROJECT_NAME]” (without the quotation marks).

##### Update the Cloud Build service account
The Cloud Build Service account needs to have permissions to use Firebase to deploy the website.
- Go to the IAM section of the Cloud Console.
- Locate the entry containing “cloudbuild.gserviceaccount.com” and add the role “Firebase Hosting Admin” to it. Note that this role is part of the Firebase Products group.

##### Test the Pipeline
Now that you have created the pipeline, you can now make a change to the site and commit it and see if the change propagates.
Open the Cloud Shell if it is not already open.

    cd ~/[REPO_NAME]

Edit the file config.toml and change the title to something different such as “My Cool New Hugo Site” and save the changed file.

    git add .
    git commit -m “updated site tile”
    git push -u origin master

Go to the Cloud Build console and check the build history. You should see a successful deployment if not, consult the build log to identify the problem. Browse to the hosting URL you had received before. If you do not have it, you can go to the Firebase console and examine the project to find the domain name. It may take a few minutes for the CDN to update.
After the build has successfully completed, look at the build details.

The total build time was on average 31 seconds.

![Google Cloud Build](/images/CB.png)

#### Add Alerts to Cloud Build

This is a work in progress >> https://cloud.google.com/cloud-build/docs/configuring-notifications/configure-slacks
