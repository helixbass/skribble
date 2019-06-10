In `gatsby-config.js`:
```
pathPrefix: '/blog'
```

```
gatsby build --prefix-paths
```

From Rails project root:
```
cp -R $PRJ/skribblin/public/* public/blog/
```
commit and push

```
bundle exec cap production deploy
```
