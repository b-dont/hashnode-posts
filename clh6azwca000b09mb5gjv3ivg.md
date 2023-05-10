---
title: "Using Git in Your Static Site Workflow"
datePublished: Sun Apr 30 2023 13:23:19 GMT+0000 (Coordinated Universal Time)
cuid: clh6azwca000b09mb5gjv3ivg
slug: using-git-in-your-static-site-workflow
canonical: https://brandont.dev/blog/static-site-workflow-using-git/
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/842ofHC6MaI/upload/8e76ea36e37ce7da53168a4197c8fec5.jpeg
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1683033999044/48ada312-5e9f-4900-8b9c-c9a27b846bac.jpeg
tags: git, devops

---

This website (meaning my website) and its contents are deployed to my VPS via git. The repository is pushed to a production remote and uses hooks to run some basic scripts that organize the incoming content. Git can be a powerful tool to build simple deployment pipelines to remote servers.

### Local and Remote Repo Setup

First, we want to set up the basic development repository with a `git init`. I always have a remote backup to push to on Codeberg, weather I plan on publishing the code publicly or not, for some redundancy. Run a `git remote add origin` [`git@example.com`](mailto:git@example.com)`/me/my-remote`, and of course replace this example repo URL with your real one.

Next, we're going to set up our production remote. Connect to your target server, and navigate to the `/var/www` directory, then run `mkdir my-website-bare`. In this post, I'm going to be using a static website as an example, and it's common for production websites to live in this directory. Once, there run `git init --bare` to initialize a [bare git repository](https://git-scm.com/book/en/v2/Git-on-the-Server-Getting-Git-on-a-Server). You'll notice that the bare repo does not have a `.git` directory, but instead will show all the contents of what *would* be in that directory at the root.

```plaintext
❯ ls -l
total 12
drwxr-xr-x. 1 brandon brandon   0 Apr 30 15:11 branches
-rw-r--r--. 1 brandon brandon  66 Apr 30 15:11 config
-rw-r--r--. 1 brandon brandon  73 Apr 30 15:11 description
-rw-r--r--. 1 brandon brandon  21 Apr 30 15:11 HEAD
drwxr-xr-x. 1 brandon brandon 506 Apr 30 15:11 hooks
drwxr-xr-x. 1 brandon brandon  14 Apr 30 15:11 info
drwxr-xr-x. 1 brandon brandon  16 Apr 30 15:11 objects
drwxr-xr-x. 1 brandon brandon  18 Apr 30 15:11 refs
```

Now, let's add this bare repository as a remote to our local repo by running the following:

```plaintext
git remote add prod me@myserver:/var/www/my-website-bare
```

For the purposes of this little walk through, I won't get into the weeds of each of these files and directories; the only directory we're concerned with here is `hooks`. [Git hooks](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks) are scripts that Git will execute when certain events happen in the repository. They are flexible and very useful when automating workflows that involve Git. I use them in my website's production workflow, and in some of my development workflows.

### The `post-receive` Hook

The workflow we're aiming for here is as follows:

* Push to the production repository from your local repo.
    
* When the prod remote receives the new code it will distribute it where it needs to be.
    
* Once the code has been updated, we can also rebuild the static site or run any commands needed to update the website with the changes.
    

In your bare repo, create a file called `post-receive` in the `hooks` directory, and write the following in it:

```bash
#!/bin/bash

# Unset the git environment variables
local_desc=$(git describe)
foreign_desc=$(unset $(git rev-parse --local-env-vars); git -C /var/www/my-website-prod/.git describe)

# navigate to the production repo and execute 
# a git pull on the prod main branch 
cd /var/www/my-website-prod || exit
git --git-dir=/var/www/my-website-prod/.git pull prod main
```

The first thing you'll notice here, is we have another website repository at `/var/www/my-website-prod`. This is where the website will actually live, and where your web server will be pointing to your static files. Go ahead and navigate there and run the following:

```plaintext
git clone /var/www/my-website-bare /var/www/my-website-prod
```

This will clone your bare repo into a new, un-bare repo located in `my-website-prod`.

So to break down our `post-receive` hook: Lines 4, 5, and 6 will unset the git environment variables from our bare repository, and then set them to the env variables from our production repository at `/var/www/my-website-prod`. Next, we will navigate to the prod repo and execute a `git pull prod main`, to pull the main branch of the production remote to our production repository, deploying the code.

Now, when you run a `git push prod main` in your local repository, it will push to the bare remote, the `post-receive` hook will trigger, and will then pull the code from the bare repo, to the production repo.

### Conclusion

The flexibility of Git hooks are very useful. I throw in other script calls to run tests and rebuild my website on a single push, allowing me to deploy changes to the website without ever having to `ssh` to my production server, creating a small and efficient CI/CD pipeline for my personal website. Easy peasy, right? Have fun!