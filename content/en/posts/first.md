+++
date = '2025-12-28T22:20:36+03:00'
draft = false
title = 'From Idea to Publication: How I Built This Blog'
keywords = [
    "hugo",
    "static site generator",
    "GitHub Pages",
    "GitHub Actions",
    "continuous deployment",
    "CI/CD",
    "Git",
    "Cloudflare",
    "custom domain",
    "CNAME",
    "hugo-blog-awesome",
    "i18n",
    "multilingual blog",
    "markdown",
    "CSS",
    "statically generated blog",
    "dev blog"
]
modified = '2025-12-31T15:41:36+03:00'
description = 'How to set up a static site with Hugo, deployed it on GitHub Pages, and configured automatic deployment using GitHub Actions'
images = ['author.png']
+++

Hi! I'm happy to introduce the first post in my blog - and it's about the blog itself. Here, I'm going to talk
about how I set up a static site with Hugo, deployed it on GitHub Pages, and configured automatic deployment using
GitHub Actions.

This post serves both as a guide for those who want to build something similar as well as documentation for myself, so I
won't get lost in my own configurations later.

## Why Hugo + GitHub Pages?

Before diving into the technical details, let's take a look at the tools.

**Hugo** is a static site generator written in Go. It's fast, simple, and doesn't require a database or an application
server. Content is stored in plain Markdown files, which is especially convenient for a blog.

**GitHub Pages** is free hosting for static websites built directly into GitHub. Content and configuration are contained in the
same repository - convenient, reliable, and with no subscription required.

**GitHub Actions** is GitHub's built-in automation system that builds the site and deploys it to GitHub Pages on every
push. No manual actions are required.

The combination of these three tools gives me:

- Simple version control via Git;
- No recurring payments (except for a custom domain);
- Full automation of the publishing process;
- Versioning;
- The ability to write posts in any text editor.

## Step 1: Preparation

## What You'll Need

1. **Git installed** - for working with repositories;
2. **Hugo installed** - the site generator;
3. **GitHub account** - for hosting and automation;
4. **Text editor or IDE** - I use GoLand;
5. **A bit of free time** - 30–60 minutes for the whole process.

## Installing Hugo

The official [documentation](https://gohugo.io/installation/) describes multiple installation methods for popular
operating systems. I'm using Homebrew package manager on macOS, so it's simple for me:

{{< terminal "ztsv@mac" "~/src/github.com/zt-sv" >}}
$ brew install hugo
{{< /terminal >}}

Let's verify that Hugo is installed successfully:

{{< terminal "ztsv@mac" "~/src/github.com/zt-sv" >}}
$ hugo version
{{< /terminal >}}

The command outputs the installed Hugo version. In my case:
`hugo v0.153.2+extended+withdeploy darwin/amd64 BuildDate=2025-12-22T16:53:01Z VendorInfo=Homebrew`.

## Step 2: Creating a Hugo Project

Let's create a new project:

{{< terminal "ztsv@mac" "~/src/github.com/zt-sv" >}}
$ hugo new site zt.sv
{{< /terminal >}}

After running the command, Hugo automatically creates a basic directory structure:

{{< terminal "ztsv@mac" "~/src/github.com/zt-sv" >}}
$ tree zt.sv
zt.sv
├── archetypes # Templates for new posts
│   └── default.md
├── assets     # Static assets (images, styles)
├── content    # Content (posts, pages)
├── data       # YAML/JSON data
├── hugo.toml  # Hugo configuration
├── i18n       # Localization
├── layouts    # Custom HTML templates
├── static     # Static files
└── themes     # Themes
{{< /terminal >}}

## Step 3: Configuration

By default, `hugo new site` creates a single `hugo.toml` configuration file in the project root, which becomes
inconvenient for larger projects. However, Hugo supports
a [configuration directory](https://gohugo.io/configuration/introduction/#configuration-directory)
that allows splitting the configuration into multiple files. First, I convert `TOML` to `YAML`. Other
configuration files will be added to this directory later:

```bash
zt.sv
├── config
│   └── _default
│       └── hugo.yaml
```

Contents of `config/_default/hugo.yaml`:

{{< highlight yaml "linenos=inline" >}}
---
baseURL: https://zt.sv/
languageCode: en-us
defaultContentLanguage: en
title: Aleksandr Zaytsev | ztsv's blog
contentDir: content/en
defaultContentLanguageInSubdir: false
{{< /highlight >}}

I chose the `content/en` path for `contentDir` because I'm going to run the blog in multiple languages and use separate
directories for each one. This directory needs to be created manually:

{{< terminal "ztsv@mac" "~/src/github.com/zt-sv/zt.sv" >}}
$ mkdir -p content/en
{{< /terminal >}}

## Step 4: Installing a Theme

There are many [themes](https://themes.gohugo.io/) available for Hugo, but I chose
[hugo-blog-awesome](https://github.com/hugo-sid/hugo-blog-awesome) by Hugo Sid.

We'll install the theme as a module, so first we initialize the project as a Hugo module:

{{< terminal "ztsv@mac" "~/src/github.com/zt-sv/zt.sv" >}}
$ hugo mod init github.com/zt-sv/zt.sv
{{< /terminal >}}

Next, add a configuration file specifying the theme in `config/_default/module.yaml`:

{{< highlight yaml "linenos=inline" >}}
---
imports:
  - path: github.com/hugo-sid/hugo-blog-awesome
{{< /highlight >}}

Install the theme as a dependency:

{{< terminal "ztsv@mac" "~/src/github.com/zt-sv/zt.sv" >}}
$ hugo mod vendor
{{< /terminal >}}

After that, you can start Hugo's built-in local server and make sure everything works fine by opening
`http://localhost:1313/`:

{{< terminal "ztsv@mac" "~/src/github.com/zt-sv/zt.sv" >}}
$ hugo server
{{< /terminal >}}

![First run](posts/first/01.png)

## Step 5: Theme Configuration

The `hugo-blog-awesome` theme supports various additional settings, such as social media icons and the default color
scheme. Unfortunately, not all available parameters are well documented, but you can find an example site and its
[configuration](https://github.com/hugo-sid/hugo-blog-awesome/blob/main/exampleSite/hugo.toml)
in the theme repository.

Let's add a configuration file with parameters in `config/_default/params.yaml`:

{{< highlight yaml "linenos=inline" >}}
---
dateFormat: "January 2, 2006"

socialIcons:
  - name: github
    url: https://github.com/zt-sv
  - name: RSS
    url: /index.xml

sitename: Aleksandr Zaytsev | ztsv
defaultColor: dark
mainSections:
  - posts

author:
  avatar: author.png
  name: Aleksandr Zaytsev
  intro: Aleksandr Zaytsev | ztsv
  description: K8s, split keyboards, IaC, and other things

webmanifest:
  name: zt.sv
  short_name: zt.sv
  theme_color: "#434648"
  background_color: "#ffffff"
  display: standalone
  start_url: /
{{< /highlight >}}

Add `author.png` to the `assets` directory. Additionally, I've generated a set of favicons for the site
using [realfavicongenerator.net](https://realfavicongenerator.net/) and placed them in the `assets/icons` directory:

```bash
assets/
├── author.png
└── icons
    ├── apple-touch-icon.png
    ├── favicon-96x96.png
    ├── favicon.ico
    ├── favicon.svg
    ├── web-app-manifest-192x192.png
    └── web-app-manifest-512x512.png
```

## Step 6: Multi-language Support

As I mentioned earlier, I'm going to run this blog in two languages simultaneously: English and Russian. Therefore, I
have to add a language selection feature.

The `hugo-blog-awesome` theme supports multilingual mode out of the box, so it's enough to define locales in
`config/_default/languages.yaml`. You can also override individual parameters for each language:

{{< highlight yaml "linenos=inline" >}}
---
en:
  disabled: false
  languageCode: en-us
  languageDirection: ltr
  languageName: 🇺🇸
  weight: 1
ru-ru:
  disabled: false
  languageCode: ru-ru
  languageDirection: ltr
  languageName: 🇷🇺
  weight: 2
  params:
    dateFormat: "2 January 2006"
    author:
      name: Aleksandr Zaytsev
      intro: Aleksandr Zaytsev | ztsv
      description: K8s, split keyboards, IaC, and all sorts of things
  contentDir: content/ru
{{< /highlight >}}

After that, a language selector appears on the site:

![Language select](posts/first/02.png)

## Step 7: Customization

Any theme can be fine-tuned with custom styles or overridden templates. For example, the language selector displays
`languageCode`, which looked a bit weird to me, so I replaced it with `languageName`. To do this, it was enough to
override the
[original header template](https://github.com/hugo-sid/hugo-blog-awesome/blob/main/layouts/partials/header.html)
by placing a modified version in `layouts/partials/header.html`.

To use custom styles, I add `layouts/partials/custom-head.html` with the following content:

```html
<link href="{{ (resources.Get "css/custom.css").RelPermalink }}" rel="stylesheet">
```

And add the corresponding file `assets/css/custom.css`:

{{< highlight css "linenos=inline" >}}
html.dark .author .author-avatar {
    border-color: white;
}

.author .author-avatar {
    width: 200px;
    height: 200px;
    border-radius: 25%;
    border: 5px #0d122b solid;
}

.lang-list {
    background: transparent;
    border: none;
    color: #0d122b;
    line-height: 2.25;
    text-decoration: none;
    padding: .3rem .5rem;
    opacity: .7;
    letter-spacing: .015rem;
    font-size: 16px;
}

html.dark .lang-list {
    color: #eaeaea;
}
{{< /highlight >}}

After these changes, the site looks like this:

![Styled](posts/first/03.png)

## Step 8: Menu and the "About Me" Page

To add the main menu to the site, you need to create the file `config/_default/menu.yaml`. For now, I'll only add the
"About" item to the main menu:

{{< highlight yaml "linenos=inline" >}}
---
main:
  - identifier: "menu.about"
    url: /about/
    pageRef: about
{{< /highlight >}}

The `identifier` field allows specifying translations for different languages. Therefore, we add
`i18n/en.yaml` and `i18n/ru-ru.yaml`, assigning a translation to each identifier:

{{< highlight yaml "linenos=inline" >}}
- id: "menu.about"
  translation: "About"
{{< /highlight >}}

And create the page itself:

{{< terminal "ztsv@mac" "~/src/github.com/zt-sv/zt.sv" >}}
$ hugo new pages/about.md
{{< /terminal >}}

This command creates the file `pages/about.md` from the `archetypes/default.md` template in the default content
directory (the `contentDir` parameter in `hugo.yaml`).

## Step 9: Creating the Repository

Initialize a local Git repository:

{{< terminal "ztsv@mac" "~/src/github.com/zt-sv/zt.sv" >}}
$ git init
{{< /terminal >}}

And configure the `.gitignore` file:

{{< highlight gitignore "linenos=inline" >}}
_vendor/
public/
resources/
.hugo_build.lock
{{< /highlight >}}

After that, create a repository on [GitHub](https://github.com). When creating the repository, I skip the automatic
addition of `README.md`, `.gitignore`, and the license file.

![Create repository on GitHub](posts/first/04.png)

Make the first commit:

{{< terminal "ztsv@mac" "~/src/github.com/zt-sv/zt.sv" >}}
$ git add .
$ git branch -M main
$ git commit -m "Initial commit"
{{< /terminal >}}

Specify the remote repository and push changes to GitHub:

{{< terminal "ztsv@mac" "~/src/github.com/zt-sv/zt.sv" >}}
$ git remote add origin git@github.com:zt-sv/zt.sv.git
$ git push -u origin main
{{< /terminal >}}

## Step 10: Custom Domain

Since I plan to use a custom domain for this site, I need to add a simple text file named `CNAME`:

```text
zt.sv
```

And commit it:

{{< terminal "ztsv@mac" "~/src/github.com/zt-sv/zt.sv" >}}
$ git add CNAME
$ git commit -m "Create CNAME"
$ git push
{{< /terminal >}}

I use Cloudflare as my DNS provider, so I've added the IP addresses
from [the GitHub documentation page](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site#configuring-an-apex-domain)
to the Cloudflare dashboard. I've added `A` and `AAAA` records for root domain and `CNAME` record for `www` subdomain.

![Cloudflare DNS management](posts/first/06.png)

After that, in the **Pages** section in the repository settings on GitHub, I enter my custom domain and select
the `gh-pages` branch as the source.

![GitHub custom domain setup](posts/first/07.png)

## Step 11: GitHub Actions and Automatic Publishing

For automatic build and deployment, I use GitHub Actions, so I create the file
`.github/workflows/deploy.yml`:

{{< highlight yaml "linenos=inline" >}}
name: Deploy Hugo site

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: '0.153.2'
          extended: true

      - name: Install Hugo modules
        run: hugo mod vendor

      - name: Build
        run: hugo --minify

      - name: Add CNAME file
        run: cp CNAME public/CNAME

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
{{< /highlight >}}

This workflow:

1. Runs on every push to the `main` branch;
2. Checks out the repository source code;
3. Installs Hugo;
4. Installs dependencies;
5. Builds the site (`hugo --minify`);
6. Adds the CNAME file to the built site;
7. Deploys the result to GitHub Pages (to the special `gh-pages` branch).

Before committing the changes, you need to go to the repository settings again and, in the `Actions → General` section,
allow read and write permissions for workflows.

![Allow workflows to read and write](posts/first/05.png)

Push the changes to GitHub:

{{< terminal "ztsv@mac" "~/src/github.com/zt-sv/zt.sv" >}}
$ git add .github/workflows/deploy.yml
$ git commit -m "Build and deploy Hugo site"
$ git push
{{< /terminal >}}

## What's Next?

Now you have:

* ✅ A working static blog powered by Hugo;
* ✅ Free hosting on GitHub Pages;
* ✅ Automatic deployment on every push;
* ✅ Versioned content.

On this foundation, you can:

* Add new posts (just new `.md` files);
* Customize styles and layout;
* Add new features (search, comments, analytics).

**P.S.** The source code of this blog is
[available on GitHub](https://github.com/zt-sv/zt.sv). If you want to see the exact configuration or copy something from
there, you're welcome!
