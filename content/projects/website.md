---
title: "yodaimperium.de"
date: 2023-05-12T22:37:17+02:00
draft: false
type: 'post'
description: "A static website made with Hugo and GitHub Actions deployment"
tags: ['projects', 'doc']
showTableOfContents: true
---


# Hosting

The entire website is hosted on a RaspberryPi 3.

## Software

nginx

## Security

In order ensure the website can be accessed securely all traffic is SSL encrypted using certbot for SSL-certificate generation.

Because the SSH port also has to be opened to the internet in order for my GitHub Actions script to work properly, I had to secure the Pi against unauthorized acces via SSH too.

Therefore, I made sure the GitHub Actions script only accesses the server as a deploy user. The deploy users home directory also is the nginx hosting directory.
Additionally, only passwordless SSH login using SSH-keys is allowed on the server. Login as root is also not allowed. I edited the sshd configuration file like this:

    PasswordAuthentication no
    PermitRootLogin no
    #PermitRootLogin prohibit-password

# Deployment

## GitHub Actions

For easy deployment I use a GitHub Actions script that executes as soon as I push to the repository. The script is located in *.github/workflows/ci.yml*.

Because the Action has to run the Hugo build command and push the generated static files to the RaspberryPi, I let it run in a ubuntu environment called deploy which I created for the GitHub Repo. The Environment also contains the SSH credentials als secrets, but more on that later.

    name: CI
    run-name: Hugo blog deployment
    on:
        push:

    jobs:
    build:
        runs-on: ubuntu-latest
        environment: deploy

The script then goes through a few steps.  
Step 1: checking the current branch

    steps:
        - name: Checkout the current branch
          uses: actions/checkout@v3

Step 2: Initialising the ssh-agent in order to then later connect to the RaspberryPi.

    - name: Initialize the ssh-agent
      uses: webfactory/ssh-agent@v0.4.1
      with:
      ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}

Step 3: Installing my static website generator of choice - Hugo

    - name: Install Hugo
      run: sudo snap install hugo --edge

Step 4: Building the website using the Hugo command

    - name: Build the website
      run: hugo

Step 5: Because the annoying message you get when connecting to an unknown SSH server, which is the case everytime you run this script, and this would break the script, I scan for the host key first.

    - name: Scan the host key
      run: mkdir -p ~/.ssh/ && ssh-keyscan -H $DEPLOY_SERVER >> ~/.ssh/known_hosts
      env:
        DEPLOY_SERVER: ${{ secrets.DEPLOY_SERVER }}

Step 6: The static website files the Hugo command put into the *public/* folder are then uploaded to the nginx host directory on the Pi.

    - name: Deploy the website
      run: >-
        rsync -avx --delete --exclude '.ssh' public/ $DEPLOY_USERNAME@$DEPLOY_SERVER:./
      env:
        DEPLOY_SERVER: ${{ secrets.DEPLOY_SERVER }}
        DEPLOY_USERNAME: ${{ secrets.DEPLOY_USERNAME }}

## Environment secrets

You may have already noticed that the GitHub Actions script logs into the Pi as a deploy user via SSH. As I obviously do not want the Actions script to just expose my SSH credentials, I set them up as Environment secrets.

The script contains the information like this:

    ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}
    DEPLOY_SERVER: ${{ secrets.DEPLOY_SERVER }}
    DEPLOY_USERNAME: ${{ secrets.DEPLOY_USERNAME }}

These variables are linked to the Environment secrets I set in the deploy environment that I added to my GitHub repository.

![Screenshot of my Environment Secrets section](/images/screenshots/2023-05-13-112843.png)

This way none of the credentials are visible in logs, commits or other repositories.