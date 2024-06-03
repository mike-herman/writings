# Creating and deploying a new Genie app

This guide has steps to:

1. Create a new Julia Genie app.
2. Dockerize the app.
3. Deploy the app using fly.io
4. Set up a CI/CD pipeline for rapid development

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

# Step 2: Dockerize the app

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