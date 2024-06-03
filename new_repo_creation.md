# A quick guide for creating new github repos via CLI

I don't create repos that often. So it's helpful to have a quick reference for how to do so quickly.

This assumes you know how to create a local git repository via `git init` and understand basic git. It also assumes you already have a GitHub account and are familiar with GitHub.

We'll assume we're creating a project called `simple-repo`.

# Step 1: Create a local repo

Create the directory where you want the repo to live then navigate to it in the terminal. You can do that in the UI or use `mkdir /path/to/repo/simple-repo && cd /path/to/repo/simple-repo`.

Create a `.gitignore` file so you have something to commit. Here's a one-line to create the file and ignore ".DS_Store": `echo ".DS_Store" > .gitignore`.

Now initiate the repo.

`git init`

# Step 2: Create the Github `origin` repo.

Make sure you [download the GitHub CLI tool](https://cli.github.com/). If you're on a mac, this is as easy as `brew install gh`. You'll need to follow the prerequisites [GitHub CLI quickstart](https://docs.github.com/en/github-cli/github-cli/quickstart) your first time.

We'll use the `gh repo create` command. I'm assuming this is a `public` repo. We also need to specify that we're using the current directory as a source.

For a full list of options use `gh repo create --help`.

Here's the command to create the repo: `gh repo create --public --source=.`

It will create the repo and give the URL where you can access it. Check out the URL to make sure it's up-and-running.

# Step 3: Push your local repo to the new GitHub remote.

Now we just add commits to our local repo and push as usual.

`git add .`
`git commit -m "Initial commit"`
`git push origin main`

# Additional Steps
You may want to add a linense. I'm not sure how to do this via the GitHub CLI. [Here's how to add a license to a repository](https://docs.github.com/en/communities/setting-up-your-project-for-healthy-contributions/adding-a-license-to-a-repository) and [here's the licenses you can add](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/licensing-a-repository). A license is importand because, among other things, it limits your liability.

# Troubleshooting
- Is your main branch called `main`? That's been the default for years now, but older versions of git used the name `master`.
- In your GitHub settings, are you blocking commits that expose your email? There's a setting in GitHub where it will deny any commits to repos that include your email address. It sounds nice from a security standpoint, but it can block you from pushing commits to a repo.