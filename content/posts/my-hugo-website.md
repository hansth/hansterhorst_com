---
title: "My Hugo Website"
date: 2025-09-06
draft: false
tags: ["Hugo", "Cloudflare"]
---

As a software developer with a background in **Java** and **Angular**, I’ve been comfortable building applications that rely on frameworks, APIs, and dynamic backends. But when I decided to create my **own personal website** — a site to share my DevOps learning journey from the **TWN DevOps Bootcamp** — I wanted something simpler, faster, and easier to maintain.

My goal wasn’t to build a complex application. Instead, I needed an easy way to transform my Markdown articles into a clean, professional-looking website that I could deploy quickly. So I can focus on writing and publishing my insights and hands-on learning — without the overhead of managing servers, databases, or backend code.

## Why I Chose Hugo

Hugo is a **free, open-source static site generator** that converts Markdown files into fast, reliable web pages. It gave me exactly what I was looking for:

- A smooth workflow to turn my notes into structured articles.
- Flexibility to design and customize the site without starting from scratch.
- Free and seamless deployment through platforms like **GitHub Pages**, **Cloudflare Pages**, or **Netlify**.

This project isn’t just about building a website — it’s about documenting my journey, sharing what I’ve learned, and creating a resource that I can expand over time as I continue to explore the world of DevOps.

## Install Hugo

To create a new website, first I have to install the Hugo Command Line Interface. This CLI tool is for creating, managing, and generating pages with simple commands. Depending on your operating system, there are a few options to install Hugo:

- **macOS**: Use Homebrew or MacPorts package manager.
- **Linux**: Use package managers for different distros.
- **Windows**: Use Chocolatey or Scoop.

See the [Hugo installation docs](https://gohugo.io/installation/) for more information.

**I’m working with a **Mac**, so I’ll use [Homebrew](https://brew.sh)**:
```sh
brew install hugo
```

**Verify the installation**:
```sh
hugo version
```

## Create a Hugo Site

With Hugo installed, the next step is to create a website. In the terminal navigate to the projects' directory.

**Create a new website**:
```sh
cd /Projects
hugo new site hansterhorst_com --format yaml
```
- `hugo new site`: Creates a new Hugo site.
- `-f yaml`: Sets the configuration file format to **YAML** instead of the default TOML. I choose for YAML for its readable syntax.

This command sets up the foundation for the website. Hugo generates a directory with a complete project file structure inside it.

## Add a Theme

To give the website a specific look, one of the best things about Hugo is its wide range of free themes. Instead of designing everything from scratch, you can choose a theme you like, apply it to your site, and customize it later to fit your style.

### Install a Theme

On the [Hugo Themes](https://themes.gohugo.io/) page, I chose a minimal-looking theme called **[Shibui (渋い)](https://themes.gohugo.io/themes/shibui/)**. It looks clean and simple, with a paper-like color scheme and a monospace font.

This theme is hosted on GitHub, and all I need to do is clone the theme’s repository into the project’s `themes` directory. By following the instructions in the [README](https://github.com/ntk148v/shibui), it downloads all the files for the design, layout, and styling of the website.

**Create a Git repository**:
```sh
cd /Projects/hansterhorst_com
git init
```

**Install the theme**:
```sh
git submodule add https://github.com/ntk148v/shibui.git themes/shibui
git submodule update --init --recursive
```
- `git submodule add`: Git command to add a submodule.
- `git submodule update`: Updates submodules to match the commit specified in the parent repository
- `--init`: Initializes any submodules that haven't been initialized yet.
- `--recursive`: If the submodule has submodules, this makes sure they’re all pulled in as well.

These commands make sure the theme’s files are actually downloaded into the project in the `themes` directory.


### Configure the Theme

Inside the project directory, there is a `hugo.yaml` file. This file controls the website’s settings like the website title, language, base URL, and of course, which theme is used.

**Add the theme settings to `hugo.yaml`**:
```yaml
baseURL: https://example.org/
languageCode: en-us
title: My New Hugo Site
theme: shibui    
params:  
  author: Your Name  
  email: your.email@example.com
```
Modify the settings with your credentials.

## Run Hugo server

With the theme installed and configured, it’s time to preview the website. Hugo comes with a built-in development server that lets you see it in real time while working on it. This means every change you make will automatically be compiled and refreshed in the browser.

**Start the server**:
```sh
hugo server
```

This starts a local development server at [http://localhost:1313](http://localhost:1313/). Open the URL in your browser, and you’ll see the website with the **Shibui (渋い)** theme applied.

## Create a Blog page

When working with Hugo, one of the most important things to understand is the file structure inside the project directory. When generating a new Hugo site, the working directory comes with a handful of directories. Each of these plays a specific role:

```
hansterhorst_com/
	├── archetypes/
	├── assets/
	├── content/
	├── data/
	├── i18n/
	├── layouts/  
	├── static/
	├── themes/
	└── hugo.yaml
```

**The key directories**:
- `content`: This is the heart of your site — where all your actual content lives. Blog posts, articles, pages, and any subfolders you create to organize them will be stored here.
- `layouts`: This directory is used for **theme overrides**. Every theme comes with its own `layouts` directory that contains the HTML templates used to render the site. If you want to customize those layouts, you can copy them into this directory and edit them without touching the original theme files.
- `static`: This directory stores all the static assets: images, CSS, JavaScript, and any other files that should be served as-is.
- `themes`: This is where all your installed themes live. Each theme has its own subdirectory containing layouts, static files, and configuration options.

**Other directories**:
- `archetypes`: Contains the template file for new content.
- `assets`: Stores files that need processing by Hugo's asset pipeline.
- `data`: Contains data files like JSON, YAML, TOML, or CSV that you want to use across your site.
- `i18n`: Holds internationalization files for multi-language sites.
- `hugo.yaml`: The **main** configuration file for your Hugo site.

### Add new Content

To publish blog posts, articles, or static pages, Hugo gives you multiple options to create and organize your content. You can use the built-in `hugo new` command to generate posts with template front matter, or you can manually create Markdown files for full control.

#### Options 1, Using the Terminal

Creating content in Hugo is straightforward and flexible. The most common way is by using the terminal command:
```sh
hugo new posts/my-hugo-website.md
```

After running this command, Hugo will generate a new file inside the `content/posts/` directory with a default header called **front matter**. It’s a set of metadata that helps Hugo manage the content:
```md
---
date: '2025-09-06T12:54:19+02:00'
draft: false
title: 'My Hugo Website'
---
```
- `date`: The creation date.
- `draft`: If true, the article won’t be published on your site.
- `title`: The title of the post.

#### Option 2, Creating Content Manually

You can also create Markdown files manually by adding them directly inside the `content` directory. Don't forget to include the **front matter** block at the top; otherwise Hugo won’t know how to handle the file.
```md
---
date: '2025-09-10'
draft: true
title: 'Setup a Ubuntu VPS'
---
```

Below the metadata, you can start writing your articles in Markdown. It’s a lightweight markup language that’s easy to learn — use this [Markdown Guide](https://www.markdownguide.org/cheat-sheet/) cheat sheet to get started.

## Add Menus and Custom Layouts in Hugo

When working with Hugo, one of the most common tasks is customizing how the articles and pages look. Whether it’s adding an image, creating navigation menus, or overriding a theme’s layout, Hugo makes it all fairly straightforward.

### Adding Navigation Menus

Hugo makes this easy with its built-in **menu system**, which you configure in the `hugo.yaml` file. By defining a `menu` section, you can create links like **Posts**, **About**, or **Tags** that appear in the header or sidebar, depending on the theme.

**Create the menu links**:
```yaml
baseURL: https://www.hansterhorst.com
#...
menu:  
  main:  
    - name: Posts  
      url: /posts/  
      weight: 1  
    - name: About  
      url: /about/  
      weight: 2  
    - name: Tags  
      url: /tags/  
      weight: 3
```
- `name`: Name of the link.
- `url`: Location of the page.
- `weight`: The order of the menu links.

### Make Custom Changes

To customize a specific theme HTML section of the website, you copy the HTML into your local `layouts/default` directory and edit it there. Hugo will always override the local version over the one inside the theme directory.

For this, I copied the entire `themes/layout` directory and modified the `footer` section by adding icon links for **GitHub** and **LinkedIn**, and changing the CSS styling.

**Copy the theme layout directory**:
```sh
cp themes/shibui/layouts layouts/
```

**Add the icon links to the footer**:
```html
<p>&copy; Copyright {{ now.Year }} &middot;</p>
<p>
    <a href="https://github.com/hansth">
        <img src="/github.svg" alt="GitHub" class="footer-social-icon">
    </a>
</p>
<p> &middot;</p>
<p>
    <a href="https://www.linkedin.com/in/hansth/">
        <img src="/linkedin.svg" alt="LinkedIn" class="footer-social-icon">
    </a>
</p>
```

This is how you can adjust HTML elements, tweak CSS styling classes, or insert custom components without touching the theme’s core files.

## Deploy the Website on Cloudflare

After tweaking the theme and creating an article about this project, it’s time to deploy the website on a hosting platform. To do this, there are multiple ways to host your website for free on **GitHub Pages**, **Cloudflare Pages**, or **Netlify**.

For deploying and hosting, I chose **Cloudflare** because I already have a registered domain. Cloudflare is widely known for its global Content Delivery Network (CDN) and security features, but with **Pages**, they’ve expanded into seamless static site hosting with automatic builds and deployments (CI/CD).

### Push it to GitHub or GitLab

Before deploying to Cloudflare Pages, ensure that your Hugo project is ready in version control. If it is not, initialize the project with `git init`. On GitHub or GitLab, create a repository and link your local repository to the remote Git repository. Cloudflare Pages will use this remote repository to build and deploy the website.
This will allow Cloudflare Pages to automatically build and deploy your site every time you make changes.

**Add the local repository to the remote repository**:
```sh
git remote add origin git@github.com:hansth/hansterhorst_com.git
git branch -M main
git push -u origin main
```

### Set Up Cloudflare Pages

After storing the Hugo website on GitHub or GitLab, the next step is to link it to Cloudflare Pages. Cloudflare will automatically pull your project directly from the repository and manage the entire deployment process automatically.

**Configure Cloudflare Pages**:
- Create/Login to **[Cloudflare](https://dash.cloudflare.com/login)**
- Go to **Compute (Workers)** --> **Workers & Pages**.
- Click **Import a repository**.
- Connect your **GitHub** or **GitLab** account.
- Select your Hugo site repository.

**Cloudflare will ask for some details**:
- **Project name**: `hansterhorst`
- **Build command:** `hugo`.

### Deploy the Website

To enable deployment, you’ll need to add a `wrangler.toml` file. This configuration file tells Cloudflare how to build and deploy your site correctly.

**Create the `wrangler.toml` worker file**:
```toml
name = 'hansterhorst'  
  
[[routes]]  
pattern = "hansterhorst.com/*"  
custom_domain = true  
  
[[routes]]  
pattern = "www.hansterhorst.com/*"  
custom_domain = true  
  
[build]  
command = "hugo"  
  
[assets]  
directory = "./public"  
not_found_handling = "404-page"
```
- `name = 'hansterhorst'`: Sets the project name in Cloudflare
- `hansterhorst.com/*` and `www.hansterhorst.com/*` are configured as custom domains.
- `custom_domain = true`: Tells Cloudflare these are your own domains.
- `command = "hugo"`: Tells Cloudflare to run this build command.
- `directory = "./public"`: Specifies the built directory of files.
- `not_found_handling = "404-page"`: Tells Cloudflare to serve a custom 404 page when someone visits a non-existent URL

With this configuration file, you can deploy the website. Cloudflare will handle the build using its internal tools for deploying the website to Cloudflare Pages.

- Click **Save** to deploy the website.

Cloudflare will:
- Clone the repo.
- Run the Hugo build command.
- Publish the contents of the `public/` build directory.
- Set up your domain and configure the DNS.

This process will take a few minutes before the domain name works. When it’s done, go to:
```
https://hansterhorst.com
```

## Conclusion

By combining Hugo with Cloudflare Pages, I now have a fast, reliable, and fully customizable static website. This setup not only simplifies the process of publishing my DevOps articles but also gives me the flexibility to scale the site as my needs grow.

What I like most about this approach is the simplicity. I can focus on writing content in Markdown while Hugo handles the structure, and Cloudflare ensures smooth, global delivery, and automatic builds (CI/CD). Over time, I can enhance the site further by adding custom layouts or integrations with other tools — all without sacrificing performance.

In short, Hugo has given me a developer-friendly workflow, and Cloudflare has provided a robust platform to share my learning journey with the world.

>**_GitHub_**: [Repository](https://github.com/hansth/hansterhorst_com)