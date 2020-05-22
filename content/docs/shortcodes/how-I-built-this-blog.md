# How I built this blog

This blog was built with Hugo, which is one of the most popular open-source static site generators.

Something used in this blog are：
- [Hugo](https://gohugo.io/)
- [Hugo-book](https://github.com/alex-shpak/hugo-book)
- [Github Page](https://pages.github.com/)
- [Github Action](https://github.com/features/actions)

## Hugo

Here's the [homepage of `Hugo`](ttps://gohugo.io/).

## Theme

Here's the [homepage of `hugo-book`](https://github.com/alex-shpak/hugo-book). Just be simple and clean. I lvoe it.


## Branch

The repository is：[`heming6666/heming6666.github.io`](https://github.com/heming6666/heming6666.github.io):
- `source`: Where to store the source code
- `master`: Where to store the static files for `Github Page` to render.

*Note：*

- If your repository is named `username.github.io` , then your `GitHub Pages` site can be only built from the master branch. Here's the [Reference](https://stackoverflow.com/questions/25559292/github-page-shows-master-branch-not-gh-pages)



## Use Github Action for Hugo

`GitHub Actions` is used to automate, customize, and execute your software development workflows right in your repository. It makes it easy to automate all your software workflows with world-class CI/CD.

Here, we used `GitHub Actions` to build and deploy the blog right from GitHub. Fortunately, There have been some open source libraries available on GitHub for us and we don't need to build the workflow from scratch by ourself.

What we used in this blog is [actions-hugo](https://github.com/peaceiris/actions-hugo). You can refer to its [homepage](https://github.com/peaceiris/actions-hugo) for more details and usage.

And here's the [configuration](https://github.com/heming6666/heming6666.github.io/blob/source/.github/workflows/github-page-deploy.yml) for this blog.
