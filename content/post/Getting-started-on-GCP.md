+++
authors = [
    "Tobias Brecker"]
title = "Creating a website with automation on Firebase"
date = "2020-08-07"
description = "How I managed to create a simple Hugo website, build a CI/CD pipeline and deploy on Firebase"
tags = [
    "GCP","Google","GitHub","Cloud Build","Source Repositories","Firebase"
]
images = [
    "markdown-syntax.jpg",
]
+++

---
Static web platforms like Hugo have become popular because of their ability to produce websites that do not require web servers. With static web platforms there are no operating systems to patch and no web server software to maintain. There are, however, various operational complexities. You may want to version control your postings, host your web site on a content delivery network (“CDN”) and provision an SSL certificate.
You can address many of these process matters by using a Continuous Integration / Continuous Deployment pipeline. These pipelines enable developers to rapidly innovate by automating the entire deployment process. A website publishing platform would enable a content author to submit new content which would in turn launch the pipeline that in turn performs all the publishing tasks.
In this article, I will show you how to leverage Google Cloud to create a pipeline for deploying websites based on Hugo, a static site builder. You create your web content, run it through Hugo which in turn produces the web pages. We will use the following Google Cloud services:
Cloud Source Repositories will hold the content. The service provides private Git-compatible repositories that can trigger Cloud Build pipelines when commits are made to branches. You can also use GitHub if you prefer.
Firebase will host the website. Firebase is an application framework that provides a content delivery network (CDN) and SSL certificates.
Cloud Build will be used to create the pipeline. The pipeline consists of steps each of which is run in a container to produce the build artifacts.
Assumptions and prerequisites
Here are my assumptions going forward.
You have access to a Google Cloud account with the appropriate permissions.
You are familiar with core Google Cloud concepts such as projects and billing accounts.
You are comfortable with the use of the cloud console and the Cloud Shell.
You understand Git-related concepts such as cloning and committing code.
Let’s start with a picture
I’d like to start off with a picture of what we are going to build.
Image for post
The goal is to be able to commit code and have it trigger the pipeline which will in turn deploy the website. Our journey will be divided into two parts. First, we will build the site locally and deploy it to Firebase manually so we can gain an understanding of the entire process. Second, we will automate the process by building a pipeline with Cloud Build.
Manual Deployment
We will build the site manually to see what happens with the end-to-end process. We will also perform some of the one-time tasks that are needed to get Firebase up and running and also see what the site looks like.
Define Parameters
You will need to select some values that will be used later in this article. When you see the value names in the article, substitute your choices.
Image for post
Create a Firebase project
Let’s start by creating a Firebase project. This will in turn set up an underlying Google Cloud project and configure it to use Firebase. We then need to set up billing and enable some APIs.
Open the Firebase console in a new browser tab.
Click “Add Project” or “Create Project.”
Enter “[PROJECT_NAME]” for the project. Edit the suggested project ID if you wish to be the value “[PROJECT_ID]”. Click “Continue”.
On the “Google Analytics” page, disable Google Analytics since this is a demonstration. Click “Create Project.”
Open the Google Cloud console in a new browser tab.
Link the Firebase project to a billing account. If you do not have the privileges to do so, work with your organization’s administrators to do this for you. You will not be able to proceed to the next section without a billing account.
In the APIs & Services menu, enable the following APIs:
Cloud Build
Cloud Source Repositories
Create a Repository
Now that we have a project, we need to create a repository and clone it to the Cloud Shell to prepare for future commits.
Open the Cloud Shell and ensure the default project ID is set to the ID of the Firebase project. Enter the below.

    cd ~
    gcloud source repos create [REPOSITORY_NAME]
    gcloud source repos clone [REPOSITORY_NAME]

We have now cloned the repository. You should now see a directory with that same name off of the home directory.
Install Hugo locally
We will now install Hugo within the Cloud Shell environment so we can build the website locally.
Open the Cloud Shell and ensure the default project ID is set to the ID of the Firebase project.
Install the Hugo executable from version 0.69.2 of the Linux archive. Place the Hugo executable in a directory off of your Cloud Shell home directory such as ~/bin and add ~/bin to your PATH.
Enter the commands below.

    cd ~
    hugo new site [REPOSITORY_NAME] --force

The hugo command will generate the site scaffolding into the directory associated with the cloned repository.
This will build out the skeleton of the site in the repository. Normally the hugo command creates the directory. The --force option will create the site in the repository directory which already exists.
Install a Hugo theme
Hugo does not come with a default theme so we will now install the Ananke theme. Here are the instructions for doing this.
Download the file https://github.com/budparr/gohugo-theme-ananke/archive/master.zip in a temporary directory.
Extract the directory “gohugo-theme-ananke-master” and rename it to “ananke”.
Move the “ananke” directory to be under the ~/[REPOSITORY_NAME]/themes directory.
Add theme to the Hugo configuration file by using this command:

    echo "theme = “ananke"' >> ~/[REPOSITORY_NAME]/config.toml

Preview the site
You are now ready to preview the site. Open the Cloud Shell and enter the commands below.

    cd ~/[REPOSITORY_NAME]
    hugo server -D --port 8080

Now use the Web Preview feature of Cloud Shell to view the site.
Image for post
Deploy the site to Firebase
Now that we have viewed the website locally, we will perform a one time deployment to Firebase to validate the configuration.
Open the Cloud Shell if you previously closed it.

    cd ~/[REPOSITORY_NAME]
    firebase login --no-localhost

You will be asked to visit a Google sign in URL. Do not click on the URL, rather copy and paste the URL into your browser. Take the resulting authorization code and paste it into the prompt.
firebase init
Select Hosting using the arrow and spacebar keys. When asked for a project option, select “Use an existing project” and then select [PROJECT_NAME]. For the public directory, select the default value “public”. For configuring as a single page application, select the default value of “N”.
firebase deploy
After the application has been deployed, you will receive a hosting URL. Click on it and you will see the same website being served from the Firebase CDN (content delivery network). If you receive a generic “welcome” message, wait a few minutes for the CDN to be initialized and refresh the browser window. Save this hosting URL for later use.

#Let’s automate!

Now that we have configured our project and understand the end-to-end deployment process, we will automate the process by creating a Cloud Build pipeline.
Configure the build
Cloud Build uses a file named cloudbuild.yaml in the root directory of the repository to perform the build.
Open the Cloud Shell if it is not already open.

    cd ~/[REPOSITORY_NAME]

Use your favorite text editor to create the cloudbuild.yaml file with the following text. Note that the line with the Hugo download URL is actually on the same line as the previous dash and that the firebase deploy command appears on two lines below but is actually one line. Also, remember that spacing does matter in the YAML syntax.

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
__
#what does this block mean?

There are three named steps in this file each of which is performed by a container image. The first two steps use a Google-supported builder to perform a wget command to download the Hugo and Firebase tools. These two steps run in parallel. Using the wget builder is faster than installing wget manually.
The third step uses a standard Ubuntu container to install Hugo and Firebase after which the site is built and deployed. You could also create your builder container that contains Hugo and Firebase. I chose this approach for two reasons. First, it’s useful to be able to change the version of Hugo. Second, I want to get the latest version of Firebase with the most up to date features.
The file also uses two custom substitution variables (_HUGO_VERSION and _FIREBASE_PROJECT_NAME)_ to allow for this template to be used in different environments.
The Hugo and Firebase binaries are created and installed in a temporary directory so that they do not inadvertently get deployed to the website itself.
Perform the initial commit
Now that we have created all of the files in the site, let’s commit the site back to the Cloud Source Repository. Before we do that, we need to create a .gitignore file to keep extra files from being committed. Open a Cloud Shell and enter the commands below.

    cd ~/[REPOSITORY_NAME]
    echo "public" >> .gitignore
    echo "resources" >> .gitignore
    git config --global user.mail "[GIT_NAME]"
    git config --global user.email "[GIT_EMAIL]"
    git add .
    git commit -m "Add app to Cloud Source Repositories"
    git push -u origin master

#Create the Cloud Build trigger
In order to start Cloud Build when changes are committed to the repository, we will need to define a Cloud Build trigger. Follow these steps to create the trigger making the following changes:
Pick a name for the trigger such as “commit-to-master-branch”.
For the event, select “Push to a branch”, select the Cloud Source Repository we created earlier, and select “master” as the branch.
For Build configuration file type, accept the default location of cloudbuild.yaml.
Add the substitution variable _FIREBASE_PROJECT_NAME __ and set it to be the value of “[PROJECT_NAME]” (without the quotation marks).
Update the Cloud Build service account
The Cloud Build Service account needs to have permissions to use Firebase to deploy the website.
Go to the IAM section of the Cloud Console.
Locate the entry containing “cloudbuild.gserviceaccount.com” and add the role “Firebase Hosting Admin” to it. Note that this role is part of the Firebase Products group.
Test the Pipeline
Now that you have created the pipeline, you can now make a change to the site and commit it and see if the change propagates.
Open the Cloud Shell if it is not already open.
    cd ~/[REPOSITORY_NAME]
Edit the file config.toml and change the title to something different such as “My Cool New Hugo Site” and save the changed file.
    git add .
    git commit -m “updated site tile”
    git push -u origin master
Go to the Cloud Build console and check the build history. You should see a successful deployment if not, consult the build log to identify the problem. Browse to the hosting URL you had received before. If you do not have it, you can go to the Firebase console and examine the project to find the domain name. It may take a few minutes for the CDN to update.
After the build has successfully completed, look at the build details.
The total build time was 25 seconds.

#Add Alerts to Cloud Build

https://cloud.google.com/cloud-build/docs/configuring-notifications/configure-slacks
