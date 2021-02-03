# coldboots.github.io

A Jekyll-based blog.

## Initial setup 

0. Ensure to have a proper Ruby version setup. Use `rvm` like sane people.
1. Clone the repo.
2. First time: run script/bootstrap

# Authoring a new blog post

1. Checkout a feature branch for the blog post
2. Create a draft; `jekyll draft my-nifty-draft`
3. Fire up `script/server` in a terminal window.
4. Be sure to have the LiveReload extension for your favorite browser.
5. Edit the draft post in your editor of choice (VS Code)
   1. Tip: Be sure to use these VS Code extensions:
      * `yzhang.markdown-all-in-one`
      * `goessner.mdmath`
6. When you're done drafting up your post, publish it with `jekyll post _drafts/my-nifty-draft.md`
7. Stage, commit and push the changes.
8. Create a Pull Request, either by using the GitHub CLI (`gh`) or in the GitHub web.
9. Tell someone to take a look at it.
10. Rebase onto master and delete feature branch.

