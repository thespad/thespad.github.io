---
title: "Making Playwright Work on Alpine Out of Spite"
date: 2024-03-13T13:23:00.000Z
lastmod: 2024-03-15T17:28:00.000Z
tags: ["howto","docker","playwright","microsoft"]
author: "Adam"
---

### Doing Who On What Now?

[Playwright](https://github.com/microsoft/playwright) is a Web Testing and Automation framework developed by Microsoft, it's similar to Selenium or Puppeteer. The core project is written in nodejs and there are sub-projects offering the same framework in Python, .NET, and Java. It's the Python project that I was specifically interested in due to its use in the [changedetection.io](https://changedetection.io/) container that I maintain for [Linuxserver.io](https://www.linuxserver.io/). The problem is that the container uses an Alpine base image whereas Microsoft only publish wheels for glibc, and they don't publish the source to Pypi for pip to build, which means you can't just do `pip install playwright` because it won't be able to find a muslc wheel to install from.

### A Source of Concern

This isn't a problem though, right, because the source is right there on Github, we can just clone the repo and do a local build with pip? Well...that makes some assumptions about the completely mad things Microsoft have decided to do for their build process.

If you run the build, it seems to work, everything reports as successful, but trying to actually use Playwright will fail horribly. Digging into the setup.py we find [this strange-looking section](https://github.com/microsoft/playwright-python/blob/main/setup.py#L178-L197) where the build process downloads a "driver" zip file from `https://playwright.azureedge.net/builds/driver/`. Bit weird. Grab the zip file yourself and dig into it and you find that it's a nodejs package, with a bundled static `node` executable, which is obviously all built against glibc and doesn't work.

So I started digging into the source of the main project to figure out where they were building this zip and whether it could be done locally at image build time instead. This led me to [build-playwright-driver.sh](https://github.com/microsoft/playwright/blob/main/utils/build/build-playwright-driver.sh), which does some very special things, including downloading the latest LTS node binary from `https://nodejs.org` and bundling it into the driver zip. It also generates packages for Windows and Mac, which we can ignore for the purposes of this project but need removing so we don't spend ages messing around with packages we're never going to use.

### Do It Yourself

There were a couple of approaches that could be taken here; either prebuild the zip file(s) myself, push them up to a host somewhere, and then modify the setup.py to grab it from that address, or build them as part of the image and just overwrite the driver that's already been downloaded by the setup.py. I went for the latter option because it's more flexible on the scale I'm operating and is marginally less effort.

Here we go then:

* Step one: Clone and build playwright-python, with its weird static nodejs driver package.
* Step two: Install the LTS `nodejs` and `npm` from the Alpine repos, then clone and build playwright.
* Step three: sedcrimes.

Step three is where it gets messy, first thing we have to do is get rid of the platforms we don't want to build:

```shell
sed -i '/-darwin-x64/d' ./utils/build/build-playwright-driver.sh
sed -i '/-darwin-arm64/d' ./utils/build/build-playwright-driver.sh
sed -i '/-linux-arm64/d' ./utils/build/build-playwright-driver.sh
sed -i '/-win-x64/d' ./utils/build/build-playwright-driver.sh
```

Then we have to stop it from downloading the static node binary that we don't want and won't use

```shell
sed -i '/curl ${NODE_URL}/d' ./utils/build/build-playwright-driver.sh
```

Stop it from extracting the tarball we didn't download

```shell
sed -i '/elif \[\[ "${ARCHIVE}" == "tar.gz" \]\]; then/,/else/d'  ./utils/build/build-playwright-driver.sh
```

Stop it trying to copy a `LICENSE` file that doesn't exist (because of the above)

```shell
sed -i '/cp .\/output\/${NODE_DIR}\/LICENSE .\/output\/playwright-${SUFFIX}\//d' ./utils/build/build-playwright-driver.sh && \
```

And finally point it to the OS install of npm rather than the one we don't have

```shell
sed -i 's/"..\/..\/${NODE_DIR}\/${NPM_PATH}"/\/usr\/lib\/node_modules\/npm\/bin\/npm-cli.js/' ./utils/build/build-playwright-driver.sh && \
```

* Step four: Run the build script.
* Step five: Delete the existing driver files from the Python `site-packages/playwright/driver/` directory.
* Step six: Copy our new and improved driver into the directory

Now, there's still a problem, because it's expecting there to be a `node` binary in the driver directory, and there isn't, and it's hardcoded to use that, even if node is installed in the OS `PATH` somewhere so, finally:

* Step seven: Symlink the OS `node` binary from `/usr/bin` to the driver directory.

### OK, But Still, Why Do This?

Because I could, because I wasn't supposed to, and because I'm stubborn and wanted to see if I could make it work. The resulting image is about 25% smaller than a comparable Ubuntu-based one, which is ultimately only about 100Mb, so it's not a huge saving in most cases. There's no *real* reason Microsoft couldn't just support Alpine with Playwright, all they'd have to do is have the user install nodejs rather than downloading and bundling a static node executable. A muslc wheel would be nice, but not necessary if they published the source package to Pypi so people could build it themselves via `pip`. Then again, Microsoft don't seem to provide Alpine packages for any of their OSS projects, so this is hardly an outlier.

Anyway, I don't really recommend anyone actually does this themselves, but it's not like I can stop you.

### Addendum

A few days after writing this I had a [fridge logic](https://tvtropes.org/pmwiki/pmwiki.php/Main/FridgeLogic) moment where I suddenly realised that actually there's probably no real reason to have to build the whole driver as part of the build because all that *really* needs replacing is the `node` binary. i.e. Drop steps three, four, five, and six, and simply delete the `node` binary and symlink the OS package in there. And it does work. My only concern is that doing so risks breakage if at some point Playwright adds dependencies on native modules that are built against glibc.
