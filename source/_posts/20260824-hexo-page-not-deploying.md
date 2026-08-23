---
title: [Debugging] Hexo Page not deploying in Github pages?
type: Debugging
---

# Why it happens?

This happens because the branch used for deployment is not correct.

It will point to main branch instead of our gh-pages build branch.

The following must be added on the _config.yml file:

```
deploy:
  type: git
  repo: https://github.com/<username>/<project>
  # example, https://github.com/hexojs/hexojs.github.io
  branch: gh-pages
```

Then redeploy page. Access the deploy page and see if something changed.

# Source

https://hexo.io/docs/github-pages