---
sidebar_position: 15
---

# Deploy your site
GitHub Pages picks up deploy-ready files (output from docusaurus build) from 
    * master/main branch or
    * gh-pags branch


## Command deployment    
This command helps you deploy your site from the source branch to the deployment branch in one command: clone, build, and commit
```
docusaurus deploy
```

## Configure Github Setting
1. From the github repository, select Settings, default branch, leave it as `main`
2. Click `Environments` from the left menu, click New `Environment` give it a name eg: __github-pages__.
3. Under __Deployment branches and tags__ section:
    * 
