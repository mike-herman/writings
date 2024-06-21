# Creating and deploying a new Genie app

This guide has steps to:

1. Create a new Julia Genie app.
2. Dockerize the app and deploy the app using fly.io
3. Set up a CI/CD workflow for rapid development

## Why all this?

A hobbyist creates things for themselves. A professional creates things for others. Starting a project with a public-facing (or stakeholder-facing) application is the best way to keep prevent your project from just being a toy hobby and ensure it is a viable, useful thing that other people can use. Setting up deployment and CI/CD from the get-go is the easiest way to do this.

## What we're creating and sources

This is specifically for creating a Julia-based, stateless webservice API using the Genie framework. The stateless nature means we don't need a database. Because we'll be using an API, we only need a static information page along with the API endpoints.

The name of the project I'm creating is `GFLOServer`. 

**NOTE: Do not use a project name containing a dash `-`.** The name of your porject will be referenced in the Genie Julia code. Julia will interpret `my-project` as the difference between an object called `my` and an object called `project`. Your life will be much easier if you use something like `MyProject` instead.

## Setup

I'm setting this up on a MacBook with an Apple Silicon chip. If you're using a different system or different architecture, it may or may not make a difference.

If you haven't already, [install julia](https://julialang.org/downloads/). I'll be using Julia 1.10.3.

You'll also need to downolad the Genie package from Julia. I will be using Genie v2.1.0. To install, type `julia` to open a julia command line. Use the following commands:
    using Pkg
    Pkg.add("GenieFramework")

# Step 1: Create a new Julia Genie app

In this step, we create a bare minimum page that we can deploy along with a GitHub repo for the project.

## Quicksteps
1. Use `Genie.Generator` to create the project directory with a standard file structure.
2. Initiate a git repo in the project directory and on GitHub.
3. Create a "hello world" start page.

## 1. Use `Genie.Generator` to create the project directory.

The main source for this the is [Developing Genie Web Services](https://genieframework.github.io/Genie.jl/dev/tutorials/4--Developing_Web_Services.html) tutorial.

In command line, navigate to the directory that will contain your project. I keep all my projects in a directory called `Repos`, so it would be `cd ~/Repos`.

Run the Genie generator for a new web service. We can run this directly from the command line as follows:

`julia -e 'using GenieFramework; Genie.Generator.newapp_webservice("GFLOServer")'`

**NOTE:** The use of single quote `'` around the julia command string is important here, since we need the explicit double quote `"` inside the Julia command. Alternately, you can open `julia` and run the commands individually.

This creates a directory called `GFLOServer` and then creates the standard application file structure.

## 2. Initiate a git repo locally and in GitHub

Navigate to the new `GFLOServer` directory. Typing `git status` should give you an error because a repository hasn't been initiated. We'll need to create a repository, add all the existing files, then commit that as our initial commit.

    git init
    git add .
    git commit -m "Initial commit"

Now we need to get this into GitHub. Follow the `new_repo_creation.md` guide for more detail there. If you've done it before, here's the code for creating a **public** repo based on your current project directory. (Don't copy-paste the code below if you need this to be a private repo.) We also need to push the commit we just made.

    gh repo create GFLOServer --public --source=.
    git push origin main

Now we're ready to throw together a simple "hello world" starter page.

## Create a "hello world" page

When we ran `Genie.Generator.newapp_webservice` a default page was created. So we should already be able to run this locally. We'll want to open a julia prompt _with this project_ by typing `julia --project=@.`. Then type the following.

    using Pkg
    Pkg.instantiate()
    using GenieFramework
    Genie.loadapp()
    up()

The `Pkg.instantiate()` is necessary to make sure all the correct packages are accounted for and loaded. Otherwise you may get an error.

You can now see the webpage at https://localhost:8000.

Let's alter the default page to something more specific to our application. That way, when we deploy it, we can be sure it's _our_ thing. We'll do this by just editing the HTML file for the default page.

Open up the file at `GFLOServer/public/welcome.html`. And make the following changes:
- On line 8, change the `<title>` field to `<title>GFLO SERVER: This is my project</title>`
- On line 22, change the `<h1>` field to `<h1 id="main-heading">GFLO Server</h1>`
- On line 23, change the `<h3>` field to `<h3>It's alive!</h3>`.

No need to restart the server. You should be able to refresh the browser and see the change.

Obviously the project will be more than this. You'll probably need to replace this with a better starter page. But...and this is important...**it's better to start building your app once it's running!** So we're going to focus on getting this up-and-running before making any more changes.

# Step 2: Dockerize the app and deploy the app using fly.io

If you're not familiar with Docker, it is a way to encapsulate your app in a "mini-OS" that reliably runs on any machine. There's a lot to it, but we won't be going into the nitty-gritty detail here. You _will_ need to make sure you [download Docker](https://www.docker.com/products/docker-desktop/) if you haven't already.

Most of the rest of this guide borrows heavily from the [Deploying Genie apps](https://learn.genieframework.com/docs/guides/deploying-genie-apps) guide, which is very straightforward.

> Containerization is a lightweight form of virtualization that encapsulates an application and its dependencies into a standalone, executable package. This makes apps portable and scalable, and solves the universal "it works on my machine" problem.
> 
> Docker makes container creation straightforward via its Dockerfile blueprint. Moreover, advanced workflows are possible such as including multiple containers running services such as databases or other backend services.

I'll be repeating the steps in that guide here. But feel free to follow those instructions instead (using the _lowercase_ `gfloserver` instead of `imagetag` where appropriate). If you do, follow the steps until you run the app container (`docker run -p 8000:8000 imagetag -d`) then skip to the next section of this guide.

To start, we'll need to change the environment to `prod`. The webservice template already creates the file we need to change. So open `config/env/global.jl` and uncomment (or create) the line that says `ENV["GENIE_ENV"] = "prod"`. 

The "prod" environment isn't as good for development, since it doesn't pick up dynamic changes to your app. Remember when we changed the HTML file above? We didn't need to re-launch the server to see the changes, only refresh the browser. When `ENV["GENIE_ENV"] = "prod"` that doesn't work.

Now we need to make the Dockerfile using the CLI command `touch Dockerfile`. Open that file and add the Docker file details.

    FROM julia:1.10
    RUN useradd --create-home --shell /bin/bash genie
    RUN mkdir /home/genie/app
    COPY . /home/genie/app
    WORKDIR /home/genie/app
    RUN chown -R genie:genie /home/
    USER genie
    RUN julia -e "using Pkg; Pkg.activate(\".\"); Pkg.instantiate(); Pkg.precompile();"
    EXPOSE 8000
    EXPOSE 80
    ENV JULIA_DEPOT_PATH "/home/genie/.julia"
    ENV JULIA_REVISE = "off"
    ENV GENIE_ENV "prod"
    ENV GENIE_HOST "0.0.0.0"
    ENV PORT "8000"
    ENV WSPORT "8000"
    ENV EARLYBIND "true"
    ENTRYPOINT ["julia", "--project", "-e", "using GenieFramework; Genie.loadapp(); up(async=false);"]

It's worth noting that I've changed the first line to explicitly use `julia 1.10`.

It will take a minute or so to load up. Once it does, you should be able to go to [https://localhost:8000](https://localhost:8000) and see the page we edited.

Now follow the remaining instructions to launch via [fly.io](https://fly.io). You'll have to create an account and upload a credit card. **THIS WILL COST YOU MONEY.** But as of this writing, a "Hobby" account is $5/month. The costs scale with usage, memory, storage, etc though. So keep an eye on it and be careful.

Once you've deployed your app you should be able to access it at [https://gfloserver.fly.dev](https://gfloserver.fly.dev). It's live! It's on the internet!

We're getting close. The last thing to do is implement a CI/CD workflow.

## Debugging & Troubleshooting

### Troubleshooting Docker
I recommend downloading the Docker VS Code extension. I find it more intuitive to use than the Docker launcher.

If you run your app in Docker and it doesn't seem to be working, something may be erroring out. Luckily, there are logs. Click on the VS Code Docker extension and under the "CONTAINERS" section locate the name of your container. Run it once. If it stops right away, something might have errored out. You can right-click the container and "View logs". This will show the logs in the terminal.

This will show you both errors thrown by Julia if something is wrong and the generic Genie screen if something isn't.

I only had one bug: the `GenieFramework` package wasn't automatically added to `Project.toml`. I had to `julia` in then `using Pkg; Pkg.add("GenieFramework")` and that fixed it.

### Troubleshooting fly.io

**How to look at the logs**. This can be pretty helpful. You can view them through the dashboard at fly.io or in the terminal by typing `fly logs -a gfloserver`. When viewed from the command line, it will stream the logs live. This can be especially helpful when deploying or restarting your app. When the Genie server successfully starts up, the log will give you the big ole "GENIE" logo, which lets you know it's up-and-running.

**Re-deploying instead of re-launching**.If you already launched an app but it isn't working **don't follow the `fly launch` instructions again!** This will launch a _new_ app instead of updating the existing one. Instead, use `fly deploy` to re-deploy your app with any changes.

**Out-of-memory**. I ran into a memory issue. I was able to deploy the Genie app locally via Docker. But when I loaded it to fly.io it wouldn't work. While I was inspecting the logs I got an email from fly.io letting me know the machine had crashed because it ran out of memory! You can run a command to increase the memory on a machine, but this won't be a permanent fix. To up the memory permanently, open the file `fly.toml` in your project. Under the `[[vm]]` section, the `memory` value defaults to `'1gb'`. I had to change it to `'2gb'`.

**Be patient when deploying**
The deployment can take a while. If it doesn't work right away, go get a cup of coffee, come back in five minutes, and then check.

# Step 3: Set up a CI/CD workflow

CI/CD stands for "continuous integration/continuous development" or "continuous integration/continuous deployment", depending who you ask. Either way, the general principal is that deploying your app is easy and automated.

Fly.io has [a great write-up](https://fly.io/docs/app-guides/continuous-deployment-with-github-actions/) on how to integrate GitHub actions, which we'll be using.

This specific workflow is simple. When the main branch is updated, it automatically deploys the change to your fly.io app.

We'll skip forward to step 5, where we get the FLy API deploy token by running `fly tokens create deploy -a gfloserver -x 999999h`. **Copy what prints out to the terminal and keep it somewhere secret and safe.**

Now we add the key to GitHub.

> Go to your newly-created repository on GitHub and select Settings.
> Under Secrets and variables, select Actions, and then create a new repository secret called FLY_API_TOKEN with the value of the token created above. 

Now we add the yaml file to the project. Create the directory `.github/workflows` and create a file called `.github/workflows/fly.yml`. Here's the code you'll need to paste in:

    name: Fly Deploy
    on:
    push:
        branches:
        - main
    jobs:
    deploy:
        name: Deploy app
        runs-on: ubuntu-latest
        concurrency: deploy-group    # optional: ensure only one action runs at a time
        steps:
        - uses: actions/checkout@v4
        - uses: superfly/flyctl-actions/setup-flyctl@master    # THIS SHOULD BE "master", regardless of your base branch name.
        - run: flyctl deploy --remote-only
            env:
            FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}

Next you will commit the changes and push them to the GitHub repo. **But first!** It's fun to see if things are deploying in real time! You'll be able to see this in two places: the GitHub Actions page for your repo and the logs from fly.io. So let's get ready.
- Open your GitHub repo and click on the Actions tab. There won't be anything here yet. We'll refresh the page after we push changes.
- In the terminal run `fly logs -a gfloserver` to start the server logs.

_Now_ you can commit your changes and push them to the GitHub repo. Pushing them to the repo will both _create_ the rule and _activate_ it, meaning the Fly.io app should automatically re-delpoy. If you refresh the Actions tab in the project GitHub, you'll see it running. And if you keep an eye on the fly.io logs, you'll eventually see the big "GENIE" logo, which means it worked!.

Let's test it by making a small update and checking that it worked.

In your project open the file `public/welcome.html` again. Change the `<h3>` field where it says "It's alive!" to say "It's alive! (Now with GitHub Actions)". Commit and push the changes. Wait for the deployment to go through. Then navigate to the web page and take a look.

# Development workflow

We can now focus on developing our app. The general workflow steps are...
- Comment out the `ENV["GENIE_ENV"] = "prod"` in the file `config/env/global.jl`. This will let your app update quickly during development. **Remember to undo this change before committing to github.**
- Make the changes you need to make.
- Run Genie locally on your machine.
- If that works, build and run the Docker image on your machine.
- If that works, uncomment the prod env, commit, and load your changes.

A good enhancement to this flow would be adding environment variables that handle the genie environment for us and adding a testing build environment to GitHub Actions. But now that we have _something_ live on the internet, it's time to start developing.