---
title: Bad Robots
date: 2024-08-01T16:04:00.000Z
tags: ["howto","robots.txt","AI","openai","github","github actions","hugo","automation"]
author: Adam
---

{{< ad-info title=" " >}}
"The whole internet loves ChatGPT, a lovely chatbot that gives plausible-sounding answers to your questions"

*5 seconds later*

"We regret to inform you that ChatGPT is stealing all your content"
{{< /ad-info >}}

### AI Scraper Bots

Everyone and their mum is on the AI hype train right now, and that means they're all desperate for training data. They don't care whose data, they just need it, and that's why they've all got bots scraping every corner of the internet. The *good* news is that *most* of them are at least pretending to be good internet citizens by obeying [robots.txt](https://www.robotstxt.org/). The TL;DR is that robots.txt gives instructions to bots about which bits of your site they are and aren't allowed to scrape. The problem is, how do you know which bots to exclude, without tanking your search visibility or accidentally blocking the [Internet Archive](https://archive.org/) from accessing your site?

Thankfully there are several projects dedicated to tracking all these awful bots; the one I've been using is [ai.robots.txt](https://github.com/ai-robots-txt/ai.robots.txt/). The problem is they're updating their lists pretty regularly and I'm quite ~~lazy~~ busy so I wanted to find a way to automate updates of the robots.txt on my site.

### Lights, Camera, Github Actions!

As my blog uses [Hugo](https://gohugo.io/) hosted on Github Pages, the simplest way to handle things is a Github Action that checks for updates to the robots.txt and opens a PR against the repo to update it. When the PR is merged, the site will automatically get rebuilt and published.

If you're in a rush, the full action is available [here](https://github.com/thespad/thespad.github.io/blob/main/.github/workflows/robots.yaml), otherwise here it is in pieces.

We're going to run this daily, because that seems reasonable, at a randomly-picked time (because doing stuff at midnight is a bad idea)

```yaml
name: Check for AI robots.txt update

on:
  workflow_dispatch:
  schedule:
    - cron:  '12 5 * * *'
```

Next we need to get the current version of the file *we're* using, and compare it to the latest upstream release. We also need to pass that information onto subsequent steps. I've added a Version comment to the top of the robots.txt that can be read and updated for version tracking purposes.

```yaml
jobs:
  get-robots-version:
    runs-on: ubuntu-latest
    outputs:
      modified: ${{ steps.robots-check.outputs.modified }}
      version: ${{ steps.robots-check.outputs.version }}
    steps:
      -
        name: Checkout
        uses: actions/checkout@v4
      -
        name: Fetch robots version
        id: robots-check
        run: |
          ROBOTS_VERSION=$(curl -sL https://api.github.com/repos/ai-robots-txt/ai.robots.txt/releases/latest | jq -r .tag_name)
          FILE_VERSION=$(grep -Poe '# Version: \K(v\d+.\d+)' ./layouts/robots.txt)
          echo "**** Current version on disk is ${FILE_VERSION}"
          echo "**** Remote AI robots.txt version is ${ROBOTS_VERSION}"
          if [[ ${ROBOTS_VERSION} == ${FILE_VERSION} ]]; then
              echo "modified=false" >> $GITHUB_OUTPUT
          else
              echo "modified=true" >> $GITHUB_OUTPUT
              echo "version=${ROBOTS_VERSION}" >> $GITHUB_OUTPUT
          fi
```

Assuming there is a new version, we can then update it, first the version comment, then the contents of the upstream file, and finally any extra directives you want to include. Doing it this way and creating the file from scratch each time is much simpler than trying to do sedcrimes to update things in-place.

```yaml
  update-robots-version:
    runs-on: ubuntu-latest
    needs: get-robots-version
    if:  ${{ needs.get-robots-version.outputs.modified == 'true' }}
    steps:
      -
        name: Checkout
        uses: actions/checkout@v4
      -
        name: Update robots version
        id: robots-update
        env:
          ROBOTS_VERSION: ${{ needs.get-robots-version.outputs.version }}
        run: |
          echo "# Version: ${ROBOTS_VERSION}" > ./layouts/robots.txt
          curl -Ls https://raw.githubusercontent.com/ai-robots-txt/ai.robots.txt/${ROBOTS_VERSION}/robots.txt >> ./layouts/robots.txt
          cat <<-EOF >>./layouts/robots.txt

          User-agent: *
          {{- if hugo.IsProduction | or (eq site.Params.env "production") }}
          Disallow:
          {{- else }}
          Disallow: /
          {{- end }}
          Sitemap: {{ "sitemap.xml" | absURL }}
          EOF
```

My repo requires signed commits, so I need to import my GPG key

```yaml
      -
        name: Import GPG Key
        id: import-gpg
        uses: crazy-max/ghaction-import-gpg@v6
        with:
          gpg_private_key: ${{ secrets.GPG_PRIVATE_KEY }}
          passphrase: ${{ secrets.GPG_PASSPHRASE }}
          git_user_signingkey: true
          git_commit_gpgsign: true
```

And then finally I can create a PR to merge my updated robots.txt. Now, I *could* just commit straight to `main`, but this way if something goes wrong with the process, I don't risk breaking my site and it's rare that I'm not in a position to merge a PR, even if I'm away from home on shonky wi-fi somewhere. Note that the `committer` has to exactly match the information associated with the GPG key or it'll fail to sign it. For extra visibility I'm assigning the PR to myself.

```yaml
      -
        name: Create Pull Request
        id: create-pr
        uses: peter-evans/create-pull-request@v6
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          commit-message: Update robots to ${{ needs.get-robots-version.outputs.version }}
          committer: thespad <git@spad.co.uk>
          branch: ai-robots
          delete-branch: true
          title: "[ai-robots] Update to ${{ needs.get-robots-version.outputs.version }}"
          body: |
            - Auto-generated by [create-pull-request][1]

            [1]: https://github.com/peter-evans/create-pull-request
          assignees: thespad
          draft: false
```

And there we go. There are definitely some improvements I'm planning to make, including proper error handling for the fetch of the robots.txt and pulling the release notes into the PR, but my immediate concern was getting this up and running so I didn't have to worry about it as much.
