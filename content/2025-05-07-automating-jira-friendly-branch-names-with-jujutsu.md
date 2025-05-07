+++
title = "Automating JIRA-Friendly Branch Names with Jujutsu"

[taxonomies]
tags=["jj"]
+++

My current $JOB uses JIRA to manage issues and Github as the forge. I use Jujutsu and create remote branches using `jj git push -c @`. However, the Engineering Standard at work requires that the branch be named in an identifiable way. In practice, every branch is the slug of the commit description. After doing creating a jj bookmark manually a few times, I decided to automate it.

<!-- more -->

My commit message usually looks like this:

```
feat(JIRA-12345): a short description


a longer body with more detail about the change
```

In this case, I want a bookmark named `feat-ENG-12345-a-short-description`.

I created an jj alias to get the commit description:


```toml
# $HOME/.config/jj/config.toml

[aliases]
# print first line of current revision
fl = ["log", "--no-graph", "-r", "@", "--template", "description.first_line()"]
```

To slugify this, I found [https://github.com/Mayeu/slugify](https://github.com/Mayeu/slugify/blob/9ba3fee8063cac2803a8c41f335f3cce6b8d3474/slugify). This is a simple bash script. I looked at Rust options that wrapped the [slug](https://crates.io/crates/slug) crate, but they did not accept `stdin`.

I then added a function to `.zshrc` to create the bookmark:

```
function jbm() {
  jj bookmark create -r @ $(jj fl | ~/bin/slugify | sed 's/eng/ENG/')
}
```

Now my workflow is:

- `jbm` to create the bookmark
- `jj git push --allow-new` to push the branch
