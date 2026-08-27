# Setting up a Docusaurus Docs Site with Automated Deployment 

Good documentation infrastructure is invisible when it works. Nobody notices a docs site that builds cleanly and deploys automatically every time a PR merges. What they do notice is a broken build blocking a release, or docs that are three versions behind the product because publishing depended on someone remembering to run a script by hand.

This walkthrough sets up a Docusaurus site with automated deployment from scratch, so publishing becomes something that just happens, not something anyone has to remember to do.

## What you will end up with

- A working Docusaurus site running locally
- Versioned docs, so you can maintain documentation for multiple product releases at once
- A CI pipeline that builds and deploys automatically on every merge to main

## Step 1: Scaffold the site

```bash
npx create-docusaurus@latest my-docs-site classic
cd my-docs-site
npm start
```

This gives you a working site on `localhost:3000` with the classic Docusaurus theme, sidebar navigation, and a starter set of docs pages already wired up. Everything from here is configuration, not scaffolding from zero.

## Step 2: Organize content the way readers actually navigate

Docusaurus reads folder structure from `docs/` to build your sidebar automatically, but the default sidebar rarely matches how a real audience moves through content. For a platform with multiple audiences, developers integrating an API, business analysts configuring a workflow, and platform teams handling deployment, that structure matters more than it looks like it should.

Edit `sidebars.js` to group content by audience and task rather than by internal team structure:

```js
module.exports = {
  docs: [
    'introduction',
    {
      type: 'category',
      label: 'Getting Started',
      items: ['installation', 'quickstart', 'first-deployment'],
    },
    {
      type: 'category',
      label: 'Configuration',
      items: ['environment-setup', 'authentication', 'scaling'],
    },
  ],
};
```

A sidebar organized around what someone is trying to do, get started, configure something, troubleshoot a problem, holds up better as content grows than one that mirrors how the engineering org happens to be split up.

## Step 3: Add versioning

If your product ships multiple supported versions at once, which most platforms with an enterprise customer base do, versioned docs stop being optional. Docusaurus handles this natively:

```bash
npm run docusaurus docs:version 1.0
```

This snapshots your current docs into a versioned folder and leaves your working `docs/` folder free for the next release. Readers on an older version get docs that actually match what they're running, instead of instructions for features they don't have yet.

## Step 4: Automate the build and deploy

This is the step that turns a docs site into docs infrastructure. Add a GitHub Actions workflow that builds and deploys automatically on every merge:

```yaml
name: Deploy Docs

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 18
      - run: npm ci
      - run: npm run build
      - uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./build
```

Once this is in place, publishing a docs update looks exactly like shipping a code change: open a PR, get it reviewed, merge it, and the site updates on its own. No separate publishing step for anyone to forget.

## Why this is a technical writing problem, not just a DevOps one

It's tempting to treat pipeline setup as someone else's job and docs content as the writer's job, but in practice the two are tightly coupled. A writer who understands the build pipeline can diagnose a broken deploy without waiting on an engineer, structure content in a way that scales cleanly with versioning, and make a real case for infrastructure changes, like adding versioning before it becomes urgent, because they understand what the system can and can't do.

None of the steps above require deep software engineering experience. They require the same instinct good technical writing already runs on: understand how a system actually works before you try to explain it, or in this case, before you build the thing that publishes the explanation.
