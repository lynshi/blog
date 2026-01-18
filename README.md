# My blogging site

Made with [Hugo](https://gohugo.io/)

Theme: [PaperMod](https://github.com/adityatelange/hugo-PaperMod)

Check it out at [lynshi.github.io](https://lynshi.github.io/)!

# Getting started
1. [Install Hugo](https://gohugo.io/installation/).
2. Clone the repo with `git clone --recurse-submodules` to fetch submodules (e.g. required for the theme).
3. `hugo serve --buildDrafts` to run the site with live reloading.

# Creating a new post
`hugo new posts/${POST_NAME}.md`

# Publishing
After merging to main, a GitHub Action builds and pushes the built website to [lynshi.github.io](https://github.com/lynshi/lynshi.github.io), where the blog is hosted.
