---
layout: post
category: dotnet
tags: [github-actions, c#, dotnet]
---

{% include JB/setup %}
Having good documentation is important to the success of any kind of software project. For open source projects libraries it: reduces the amount of questions the maintainers needs to respond to, increases adoption and makes it is easier for others to contribute to the project.

The guide details how you can create a free documentation website hosted on github pages using [Microsoft's open source DocFX](https://dotnet.github.io/docfx/index.html) library for a dotnet project (although it could easily be modified to support typescript/javascript libraries too. This tool will generate a static searchable website based on your markdown files and public exposed classes and members.

## Step 1 - setup your github branches
I prefer to use a single repo to house my library code and its documentation. That way, a single pull request can be created that adds a new feature and its associated documentation.
To achieve this, I use the typical `git flow` setup where the `master` branch houses the latest stable version of my source code and markdown documentation. A special branch `gh-pages` exists as a place for github to serve static html build artifacts based on the source code in `master`.
Once pushed, go to the repository settings and enable github pages pointing the source to your branch `gh-pages`.
 
## Step 2 - create a deploy key
In order to allow automated processes to push to the `gh-pages` branch, we need to create keys with the right level of permissions and secure them. You can find [detailed docuemntation on deploy keys at github's own website](https://developer.github.com/v3/guides/managing-deploy-keys/#deploy-keys). For the purpose of this guide we need to:
- Create a new SSH key
- Add the public key as a deploy key to this repository and give it write access
- Add the private key as a secret named `DEPLOY_KEY` in the repository so that step 3 can access it without it being exposed publicly.

## Step 3 - automate generating static content when master is updated
To keep `gh-pages` updated with the latest markdown articles and API documentation from `master`, we can use a github actions.
Add the following yml file to this location in the repository `/.github/workflows/build-documentation.yml`.
```
name: Build Documentation

on:
  repository_dispatch:
  push:
    branches:
      - master  # We don't want branches in doc to trigger builds

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - name: Setup .NET Core
      uses: actions/setup-dotnet@v1
      with:
        dotnet-version: 3.1.100
      
    - name: Get mono
      run: |
        apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 3FA7E0328081BFF6A14DA29AA6A19B38D3D831EF
        echo "deb https://download.mono-project.com/repo/ubuntu stable-bionic main" | sudo tee /etc/apt/sources.list.d/mono-official-stable.list
        sudo apt-get update
        sudo apt-get install mono-complete --yes
    - name: Get docfx
      run: |
        curl -L https://github.com/dotnet/docfx/releases/latest/download/docfx.zip -o docfx.zip
        unzip -d .docfx docfx.zip
        
    - name: Build docs
      run:  mono .docfx/docfx.exe
        
    - name: Install SSH Client
      uses: webfactory/ssh-agent@v0.2.0
      with:
        ssh-private-key: ${{ secrets.DEPLOY_KEY }}

    - name: Deploy to GitHub Pages
      uses: JamesIves/github-pages-deploy-action@3.2.1
      with:
        BRANCH: gh-pages        
        FOLDER: _site 
        CLEAN: true
        SSH: true
        GIT_CONFIG_NAME: githubactions
        GIT_CONFIG_EMAIL: noreply@example.com

```

The yaml file is fairly self explanatory once you've learned the basics of github actions.
Essentially it is trigger when a commit is pushed to `master`; when trigger it installs dependencies, builds the docs as html, and pushes it the `gh-pages` branch we mentioned in step 1. 
Of note is that the last step uses a less known command [git worktree](https://git-scm.com/docs/git-worktree) which is left as additional reading for keen readers.

## Conclusion
This guide has demonstrated how you can combine `docfx` and `github actions` to create a well documented library project.
