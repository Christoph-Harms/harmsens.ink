+++
title = "Hugo, Github, Action!"
date = "2022-08-22T14:12:58+02:00"
author = "Christoph"
authorTwitter = "the_harmstar" #do not include @
cover = ""
tags = ["tech", "hugo"]
keywords = ["hugo", "github actions", "sftp", "deploy", "github"]
description = "How to build and deploy a Hugo site via Github Actions via SFTP on any server"
showFullContent = false
readingTime = true
hideComments = false
+++

## Introduction
Recently, I wanted to automate the deployment of a Hugo website. And while there are many how-tos on the web to do exactly that, none of the ones I found fit my use case.

The "classic" way seems to be to initiate a bare git repository on your server and utilize the `post-update` git hook to build and deploy your site. But I did not feel comfortable having the `origin` of my site's repo on my web server. Should it ever go down, I'd lose the repo. A way to prevent this would be to have two remotes to the repo, one on GitHub, the other one on the server. But then I always would have to push to two repositories on every update, and I wanted a "single source of truth".

The other way is to build and deploy your Hugo site via Github Actions, but all of the how-tos I found about this were about deploying the site to GitHub Pages, which is also not what I wanted. I wanted the site to be built and then deployed to _my_ server via SFTP.

It was not particularly complicated to figure this out, but I figured I'd post it as a how-to anyway. It also seems somewhat fitting for my first post on my new site to be about how this site is deployed. :) So here you go.

### SSH access for the GitHub Action
For our Github Action to be able to upload the generated website to our web server via SFTP, it needs SSH access to said server (duh!). There are basically two different ways to achieve this, one being to use a username and password, the other being to connect using an SSH key pair. There are [pros and cons](https://medium.com/head-in-the-clouds/passwords-vs-ssh-keys-whats-better-for-authentication-43b87e9f4862) for both of these methods, but I think that for machine-to-machine communication, using an SSH key pair is far more convenient and secure.

This means: we are going to need a key pair, where the public key is stored on our server, and the private key is accessible by our GitHub action.

#### Generating the key pair
We can generate a pair of keys using the `ssh-keygen` command line tool. This tool has [a ton of command line options](https://www.ssh.com/academy/ssh/keygen), but you should be fine just running `ssh-keygen` without any parameters and following the instructions on the screen:

```
char@Christophs-MBP hugogithubaction % ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/Users/char/.ssh/id_rsa): id_for_github_to_deploy_to_server
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in id_for_github_to_deploy_to_server.
Your public key has been saved in id_for_github_to_deploy_to_server.pub.
The key fingerprint is:
SHA256:QKCtpmjbuUzfPAkvAbfXOuIh2fYzPtkwTTujaZnENGg char@Christophs-MBP.fritz.box
The key's randomart image is:
+---[RSA 3072]----+
|    ...          |
|   o .           |
|  . . o          |
|   o E + .       |
|  o + + S .      |
|.o  o+ * *       |
|o. + +* % o      |
|. = =o=^ .       |
| . =oo==*        |
+----[SHA256]-----+
```

As you see, you can add a passphrase to the key for extra security, but I chose to not do that. This decision is up to you. You should end up with two new files in the directory you placed the key in. As I did not enter a path, but instead just the filename when `ssh-keygen` prompted me to do so, the two files are in the directory where I started the command:

```
char@Christophs-MBP hugogithubaction % ls -lah
total 16
drwxr-xr-x   4 char  staff   128B Aug 22 14:52 .
drwxr-xr-x+ 80 char  staff   2.5K Aug 22 14:51 ..
-rw-------   1 char  staff   2.6K Aug 22 14:52 id_for_github_to_deploy_to_server
-rw-r--r--   1 char  staff   583B Aug 22 14:52 id_for_github_to_deploy_to_server.pub
```
As you might expect, the one ending in `.pub` is the public key, the other one is the private key. 

#### Putting the keys where they are needed
First, we need to upload the public key to our server. Again, [there are multiple ways to do this](https://linuxhandbook.com/add-ssh-public-key-to-server/), but the gist is, the key has to end up in the `~/.ssh/authorized_keys` file of your user on the server. As I am deploying to [Uberspace](https://uberspace.de/), I can conveniently add the key using their web UI.

Next, we need to make the private key accessible for our GitHub action that we are going to create. Conveniently, GitHub offers a way to securely store sensible data like this, called "GitHub secrets". Navigate to your repository's `Settings` --> `Secrets` --> `Actions`:

{{< image src="/img/navigate_to_secrets.png" alt="Screenshot of Github's settings" style="border-radius: 8px;" >}}

As you see, I added three secrets: One for my server's IP, one for my username on my server, and one for the private key. You should do the same (granted, storing the server IP as a secret is not necessary for security reasons, since it is public anyway, but I prefer it like this because it seems more convenient for me to change it, should I ever move the website to another server).

### The Action
Only one thing left to do: create the actual action. In your repository, navigate to `Actions` --> `New workflow`, then click on  "set up a workflow yourself".

{{< image src="/img/navigate_new_workflow.png" alt="Screenshot of Github's settings" style="border-radius: 8px;" >}}
---
{{< image src="/img/navigate_set_up_on_your_own.png" alt="Screenshot of Github's settings" style="border-radius: 8px;" >}}

Delete the workflow boilerplate you are presented with and replace it by this.

```yaml
# This workflow will build the site and deploy it

name: Build and deploy the site

# Controls when the workflow will run
on:
  # Triggers the workflow on push events but only for the "main" branch
  push:
    branches: [ "main" ]

  # Allows to run this workflow manually from the Actions tab
  workflow_dispatch:

jobs:
  build_and_deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Check out repository
        uses: actions/checkout@v3
        with:
          submodules: recursive
          fetch-depth: 0

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: '0.101.0'
          extended: true

      - run: hugo --minify
      
      - name: Deploy
        uses: Creepios/sftp-action@v1.0.3
        with:
          username: ${{ secrets.UBERSPACE_USERNAME }}
          host: ${{ secrets.UBERSPACE_SERVER_IP }}
          privateKey: ${{ secrets.SSH_PRIVATE_KEY }}
          localPath: './public'
          remotePath: '/path/to/html/dir/on/your/server'
```

Replace the `remotePath` with the actual path on your server and that's it! You're done. From now on, whenever you push something to the `main branch of your repository, the Hugo site will automatically be built and deployed.
