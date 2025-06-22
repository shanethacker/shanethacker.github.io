---
categories:
- development
date: "2019-10-21T00:00:00Z"
tags:
- git
title: Case matters in weird ways
---

Writing this down for my own edification, even though everyone else probably knows it:

When you checkout a branch in a Windows git repository that matches a branch name on the GitHub remote, but the case in the branch name is different, Windows doesn't care and will happily give you the correct branch.

However, the git client doesn't automatically match that branch with the one on the remote, so you'll likely get messages about needing to set up the remote.

When you do, check the case between your local and GitHub. There doesn't appear to be any issues with doing another checkout with the correct case in Windows, and magically you will be linked to the GitHub branch again.
