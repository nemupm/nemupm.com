# Dependency

* hugo
    * `brew install hugo`

# Development

```.sh
$ git clone <repo>
$ git submodule -f update --init --recursive
$ hugo server -D
```

# Deploy

```.sh
$ hugo -t minimo
$ aws s3 sync --delete ./public/ s3://nemupm.com
```
