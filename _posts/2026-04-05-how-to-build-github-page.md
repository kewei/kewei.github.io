---
title: How To Build Github Page With Obsidian + Jekyll + Giscus
description: A powerful markdown editor Obsidian and a nice site generator Jekyll that Github supports natively, using Github discussions for comments via Giscus.
layout: post
author: Kewei Zhang
date: 2026-04-05 16:36:02 +0000
categories:
  - Web
  - Github
tags:
  - Web
  - Github
  - Obsidian
  - Jekyll
  - Giscus
pin: true
math: true
mermaid: true
comments: true
image:
  path: /assets/img/2026-04-05/obsidian_jekyll.png
  alt: Obsidian and Jekyll
---

# How To Build Github Page With Obsidian + Jekyll + Giscus
{: .mt-4 .mb-0 }

It is nice to have a free site to store your blogs, which is supposed to be free, simple and elegant. Github page happens to provide a free hosting plan for that. And it supports a nice web generator tool [Jekyll](https://jekyllrb.com/). Obsidian is an excellent choice as markdown tool, which also has nice integration with Github.

### Github Setup
1. Create Github Repo
   Create a new repository on Github with name: `yourname.github.io`.
2. Go to Repository -> Setting, and navigate to `Pages` on the left side panel, then choose Source to `Gihub Actions` on `Build and deployment` section. So that you will build the website via a script and deploy the website, then `yourname.github.io` will be alive.
### Jekyll Setup
1. Find a Theme
Jekyll has many templates/themes that you can use to start your site, choose one [Jekyll theme](https://jekyllthemes.io/free), my theme is [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy).

2. Clone it to local
```bash
git clone https://github.com/yourusername/yourusername.github.io.git
cd yourusername.github.io
```
3. Add Jekyll Files
 You can copy all the files from your theme to the local repo, make sure you have `_posts` folder and `_config.yml` file in the root. I copied the files from a [starter package](https://github.com/cotes2020/chirpy-starter) that has basic skeleton of the site, instead of the master package that has a lot of extra contents.

### Configure Obsidian
1. Open as Vault: Open Obsidian and select "Open folder as vault." Choose your `yourusername.github.io` folder.
    
2. Configure Folder Structure:
    
    - In Obsidian settings, go to `Files & Links`.
        
    - Set "Default location for new attachments" to a specific folder (e.g., `assets/img` or `images`). This ensures images you paste go where Jekyll can find them.   
    
3. Github Integration
   - In Obsidian, go to Community Plugins > Browse > search for "Obsidian Git".
   - Configure the plugin: Vault backup interval: Set this to 0 if you want to push manually, or 60 if you want it to auto-save to the web every hour.	Pull updates on startup: Enable this so if you edit a file directly on GitHub, Obsidian stays synced.
   - Usage: Now, when you finish an article, you can just press Ctrl + P (or Cmd + P) and type "Git: Push". Your blog is now live!
	
 4. Install the "Frontmatter" Plugin (Optional but helpful):
       - Jekyll requires a "YAML block" at the top of every post. It looks like this:
        
        YAML
        
        ```
        ---
        layout: post
        title: "How to use LaTeX in Jekyll"
        date: 2026-04-03
        categories: [Tech, Tutorial]
        ---
        ```
        
    - Using the "Obsidian Templates" core plugin, you can create a "New Post" template so you don't have to type this every time.
        - Enable the Plugin: Go to Settings > Core plugins and toggle on Templates.
		- Create a Folder: Create a new folder in your vault (e.g., "Templates") to store your template files.
		- Configure Settings: Go to Settings > Templates and select that folder under "Template folder location".
		- Create a Template: Create a new note within your template folder and write the markdown content you want to reuse.
		- Insert a Template: In any note, click the "Insert template" icon in the left ribbon or use the command palette (`Ctrl/Cmd + P`) and search for "Insert template," then select your desired template. 

#### Setup Giscus for Post Comments
[**Giscus**](https://giscus.app/) uses GitHub Discussions as the database for your comments. It’s free, has no ads, and they can log in with their GitHub accounts to comment.
#### Github Repo Side
- Go to your `yourname.github.io` repository on GitHub.
- Click the `Settings` tab.
- On the left sidebar, click `General`.
- Scroll down to the "Features" section and check the box for `Discussions`.

#### Giscus Side
- Go to the [giscus.app](https://giscus.app/) website.
- In the `Repository` field, type `yourusername/yourusername.github.io`.
- In the `Page ↔️ Discussions Mapping` section, select "Discussion title contains page `pathname`".
- In the `Discussion Category`, select "General" (or create a "Comments" category in your GitHub Discussions first).
- Scroll down to `Enable giscus`, and you will see a block of JSON/Script code. **Keep this page open; you need those values.**

#### Jekyll Side
###### Update `_config.yml`
Open your `_config.yml` from your local repo with your favorite editor, and look for `comments` section. Update it like this based on the JSON code in giscus page that you configured.

```yaml
comments:
  provider: giscus # Set this to giscus
  # Other settings...
  giscus:
    repo: 'yourusername/yourusername.github.io'
    repo_id: 'R_kgD...' # Copy this from the giscus.app page
    category: 'General'
    category_id: 'DIC_...' # Copy this from the giscus.app page
    mapping: 'pathname'
    strict: '0'
    reactions_enabled: '1'
    emit_metadata: '0'
    input_position: 'bottom'
    theme: 'preferred_color_scheme'
    lang: 'en'
    loading: 'lazy'
```
##### Enable comments in posts
Change the layout to `post`, and make ´comments´ to ´true´:
YAML

```yaml
---
title: My Great Tech Post
date: 2026-04-04
categories: [Tutorial]
layout: post
comments: true
---
```

Now you can create a post and leave a comment to see whether the comment is saved in `Discussions` tab of your Github repository.


> If you can not see the comments after leaving comments, try to change to `Discussion Category` in Giscus from  `General` to `Announcements`, and then update the `Discussion Format` of `Announcements` in `Discussion` tab in Github Repo to `Open-ended discussion`. 