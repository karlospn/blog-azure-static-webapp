---
title: "Create and host a blog with Hugo and GitHub Pages in less than 30 minutes"
date: 2020-06-21T18:28:38+02:00
tags: ["hugo", "github", "actions", "go"]
draft: false
---


Do you want to start a blog and don't want to lose a lot of time setting everything up?  
That was exactly my thought! I wanted to build a blog to write about my ramblings, but I didn't want to spend long hours settings things up.  
After doing a little of research about what options are available nowadays I found that Hugo could be a good fit.

# What's Hugo?

Hugo is a static HTML and CSS website generator, it is written in Golang and it relies on markdown files and I like how few steps you need to do to have a site ready to rock. 

You just need to:  

+ Create a new static site using the Hugo CLI
+ Choose a theme from his website: https://themes.gohugo.io/
+ Do some minor tinkering on how the theme is going to work in your website 
+ Choose a hosting provider
+ Publish the content

And that's it! You are ready to go!

There are a lot of options for hosting a static website, but I chose GitHub Pages because it is free, easy to work with and I already have an existing account. 


# Steps

After reading a little bit about how to set everything up I begin building my site.

## Install Hugo

I work mainly with Windows so I use *Chocolatey*.

> Chocolatey it's just a package manager for Windows if you are interested in learning more about check it out its website:  https://chocolatey.org/  

You can install Hugo with the following one-liner:

```bash
choco install hugo -confirm
```

If you use another OS just check out the official docs about how to install it:   
https://gohugo.io/getting-started/installing/

After you install Hugo, you can use it via the command line, just type _hugo -help_ to list all the options available.

## Create a new site

We are going to create a new Hugo site in a folder named **dotnetramblings**.

```bash
hugo new site dotnetramblings
```

## Add a theme

Next step is to style our site.   
We choose an existing theme from the website: https://themes.gohugo.io/    
I pick the **hello-friend** theme and add it into the project as a submodule. You have to place the submodule in the themes folder.

```bash
cd dotnetramblings
git submodule add https://github.com/panr/hugo-theme-hello-friend.git themes/hello-friend
```

You can see how your website currently looks by typing:

```bash
hugo server -D
```

## Create a GitHub repository and modify the config.toml file

Go to GitHub and create a new repository.   
In my case I'm going to create the repository **dotnetramblings**.   

Once the repository is created you **HAVE** to modify the config.toml file found on the root directory of your site. You need to modify the baseUrl property to point to your GitHub Page. 

```toml
baseurl = "https://myuser.github.io/dotnetramblings/"
```

## Build your site

If you want you can tweak the theme options until you're satisfied with it, after that the next step is to build the site.  
We execute: 

```bash
hugo
```

The output of the build will be put in the **./public/** directory. The public directory is what we are going to publish into GitHub.   


## Push your code into GitHub

It's time to upload the contents of the **./public** directory into the GitHub repository.

```bash
cd dotnetramblings/public
git init
git remote add origin https://github.com/myuser/dotnetramblings.git
git add .
git commit -m "Initial commit"
git push --set-upstream origin master
```

After pushing the code into GitHub you still need to modify the Repository settings.   
Change the setting found in: **Settings > Options > GitHub pages > Source**  to **"master branch"**

## Test your webpage

Just wait a couple of minutes for the page to refresh and browse to: https://myuser.github.io/dotnetramblings and you will see your website up and running.


## Automate everything!

The site is up and running and everything works great but be aware that right now the source code is only on your local machine.  

So our final steps are going to be:    

- Create a second github repository: **dotnetramblings_source**
  -  The second repository is going to host the source code so every time we want to add a new post we will push the changes into that repository.     
- After we push the changes a github action will grab the content from **dotnetramblings_source**, build it and publish the output into the **dotnetramblings** repository.

First of all I'm creating a .gitignore file because I don't want to push the **./public** folder into GitHub. Remember that the **public** folder contains the output of the Hugo build.   
After that I push everything into the **dotnetrambligs_source** repository.

```bash
cd dotnetramblings
echo "public/" >> .gitignore
git init
git remote add origin https://github.com/myuser/dotnetramblings_source.git
git add .
git commit -m "Add site files"
git push --set-upstream origin master

```
The last step is to build a GitHub Action.   
The GitHub Action is going to grab the content from the **dotnetramblings_source** repository, build it using Hugo and push the output into the **dotnetramblings** repository.

```yaml

name: hugo CI

on:
  push:
    branches: [ master ]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v2
        with:
          submodules: true 
          fetch-depth: 1   

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest'

      - name: Build
        run: hugo

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          personal_token: ${{ secrets.PERSONAL_TOKEN }}
          external_repository: myuser/dotnetramblings
          publish_branch: master
          publish_dir: ./public

```

And we are good to go! With less than 30 minutes we have achieved:  
- A running static website hosted with GitHub Pages on our main repository.
- A secondary repository where we put all the site source code and a pipeline to compile and deploy the site source code into the main repository in an automatic way.
