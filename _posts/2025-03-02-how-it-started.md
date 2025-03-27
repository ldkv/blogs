---
layout: post
title: How it started
date: 2025-03-02 20:27 +0100
---

## Template choice

For some time I have been searching for a solution to create my own page, where I could share my development experiences (and my thoughts!) occasionally.

As a simple and NOT front-end oriented man, I was looking for an open source template to use. It should save me the headache of designing the page and allow me to focus solely on the content.

The solution I was looking for should satisfy the following points:

- The template should be clean and minimal, without unnecessary elements that would distract the reader.
- It should be mobile-friendly, with dark mode support.
- The pages should be static generated and responsive.
- The posts should be written under **Markdown** format, which I am comfortable with and easy to maintain with version control. The plus point is that I can write in VSCode with Copilot support, which should greatly speed up the writing process.
- It should be easy to deploy with automation, preferably with **GitHub Pages** or **Cloudflare Pages**.
- A comments system would be nice to have.

After a long search and testing of various solutions, from [Jekyll themes](http://jekyllthemes.org/), [Hugo themes](https://themes.gohugo.io/) to [Pelican themes](https://pelicanthemes.com/) then back to Jekyll themes, I finally found the perfect template: [jekyll-theme-chirpy](https://github.com/cotes2020/jekyll-theme-chirpy).

The template layout meet all my expectations above, it is well documented with many features and customization options. It even has a search function, which is a big plus for me.

## Setup and Customization

To get started, I found a [Medium post](https://medium.com/@svenvanginkel/build-a-blog-with-jekyll-chirpy-on-cloudflare-pages-f204bc538af9) with step-by-step guide to deploy the template on Cloudflare Pages, which is exactly what I was looking for.

After the first successful deployment using the guide, some more customizations are necessary to make the page more personal and to my liking.

Below are the steps I took to customize the template, I won't go into details here since most of them are well-documented, but I will provide the links to the official documentation for each step.

### Disable Github actions

The template comes with a default Github Actions workflow to deploy the page to Github Pages. Since I am using Cloudflare Pages, I disabled the workflow by commenting the `push` event.

### Customize with my personal information and avatar

Most of the customizations are done in the `_config.yml` file, following the [official documentation](https://chirpy.cotes.page/posts/getting-started/).

I put my avatar to `assets/img/avatar/` folder, then point to it in the `_config.yml` file.

The favicons [can also be customized](https://chirpy.cotes.page/posts/customize-the-favicon/) by replacing the default ones in the `assets/img/favicons/` folder.

### Enable post comments with giscus

The template supports comments system with [giscus](https://giscus.app/), which is a lightweight comments system based on GitHub discussions.

To enable giscus, first follow the [official guide](https://giscus.app) to enable the extension in your Github repository.

Then, make appropriate changes in the `_config.yml` file, it should look like [this commit](https://github.com/ldkv/blogs/commit/9835088807f7129856ec58b88ccfea509723b6e9).

Push the changes to your repository, and the comments section should appear by default in your posts.

To disable comments for a specific post, add `comments: false` in the front matter of the post.

### Copy theme files to the repository

It is recommended to copy the theme files to the repository, so that the site can be deployed without the need of the theme gem.

It also allows for easier customization of the theme and its layout.

To locate the theme files, use the command:

```shell
bundle info --path jekyll-theme-chirpy
```

Then copy all 5 folders \_data, \_layouts, \_includes, \_sass, assets to your repository, overwriting the existing files.

### Production and preview builds

To make sure the build process works correctly, the `JEKYLL_ENV` environment variable should be set to `production` in the Cloudflare Pages settings.

Either by setting the build Command to `JEKYLL_ENV=production bundle exec jekyll build`, or by setting the variable in the `Variables and Secrets` section on the same page.

It is also possible to setup a preview build on Cloudflare Pages, by creating a separate branch then pointing the preview environment to that branch.

## Conclusion

After the customizations, the page is now ready to be filled with content.

The deployment process is automated, so I can focus on writing and sharing my thoughts without worrying about the technical details.

I am satisfied with the final results (the page you are reading right now), and I am looking forward to writing more posts in the future.

Stay tuned for more updates!
