# Code and Stuff (blog source code)

IMPORTANT: The original version of this doc is in the `source` branch. Edit that
one instead and changes will be copied over using the `publish.sh` command.

## How to use this repo

1. After cloning, pull the theme (`git submodule update --init --recursive`)
1. [Install Hugo](https://github.com/gohugoio/hugo/releases)
1. Checkout `source` branch (`git checkout source`)
1. Mount `master` on `/public` (`git worktree add -B master public
   origin/master`)
1. Create a post (`hugo new content/post/new-post.md`)
1. When ready, publish the post (`./publish.sh`)
1. Don't forget to publish the source too! (`git commit` and `git push`)
