# Dependency

* hugo
    * `brew install hugo`

# Preview

```.sh
$ hugo server -D
```

# Deploy

```.sh
$ hugo -t minimo
$ aws s3 sync --delete ./public/ s3://nemupm.com
```
