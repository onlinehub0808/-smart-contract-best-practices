# Smart Contract Security Best Practices

Visit the documentation site: https://consensys.github.io/smart-contract-best-practices/

Read the docs in Chinese: https://github.com/ConsenSys/smart-contract-best-practices/blob/master/README-zh.md

## Building the documentation site

```
$ git clone git@github.com:ConsenSys/smart-contract-best-practices.git
$ pip install -r requirements
$ mkdocs build 
```

You can also use the `mkdocs serve` command to view the site on localhost, and live reload whenever you save changes.

## Redeploying the documentation site

```
mkdocs gh-deploy
```

