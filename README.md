# Heroku buildpack: PHP

It uses Composer for dependency management, supports PHP or HHVM (experimental) as runtimes, and offers a choice of Apache2 or Nginx web servers.

## The Subdirectory Feature

The only difference between the heroku's standard PHP buildpack is that you can *run an application in a Subdirectory*

For now the default subdirectory is application.

*TODO*

* Get the subdirectory from a ENV var


## Usage

You'll need to use at least an empty `composer.json` in your application.

    echo '{}' > composer.json
    git add composer.json
    git commit -m "add composer.json for PHP app detection"

If you also have files from other frameworks or languages that could trigger another buildpack to detect your application as one of its own, e.g. a `package.json` which might cause your code to be detected as a Node.js application even if it is a PHP application, then you need to manually set your application to use this buildpack:

    heroku buildpacks:set https://github.com/jopereyral/heroku-buildpack-php

Please refer to [Dev Center](https://devcenter.heroku.com/categories/php) for further usage instructions.

## Development

The following information only applies if you're forking and hacking on this buildpack for your own purposes.

### Compiling Binaries

#### Preparation

You need an S3 bucket, referenced in `bin/compile`, to host your own binaries if you want custom ones. Your S3 bucket must be set up so that a public listing is allowed (for the root or your intended prefix, see below).

The folder `support/build` contains [Bob](http://github.com/kennethreitz/bob-builder) build scripts for all binaries and dependencies.

To get started, create a Python app (*Bob* is a Python application) on Heroku inside a clone of this repository, and set your S3 config vars:

    $ heroku create --buildpack https://github.com/https://github.com/jopereyral/heroku-buildpack-php/heroku-buildpack-python
    $ heroku config:set WORKSPACE_DIR=/app/support/build
    $ heroku config:set AWS_ACCESS_KEY_ID=<your_aws_key>
    $ heroku config:set AWS_SECRET_ACCESS_KEY=<your_aws_secret>
    $ heroku config:set S3_BUCKET=<your_s3_bucket_name>
    $ heroku config:set S3_PREFIX=<optional_s3_subfolder_to_upload_to_without_leading_or_trailing_slashes>
    $ heroku config:set STACK=cedar-14
    $ git push heroku master
    $ heroku ps:scale web=0

Builds will initially fail, because your bucket is empty and no dependencies can be pulled. You can sync from an official bucket, e.g. `lang-php`, using a helper script - make sure you use the appropriate prefix (in this example, `/dist-cedar-14-master/`):

    $ heroku run "support/build/_util/sync.sh your-bucket /your-prefix/ lang-php /dist-cedar-14-master/"

This only copies over "user-facing" items, but not library dependencies (e.g. `libraries/libmemcached`). You must copy those by hand using e.g. `s3cmd` if you want to use them.

#### Building

    $ heroku run bash
    Running `bash` attached to terminal... up, run.6880
    ~ $ bob build extensions/no-debug-non-zts-20121212/yourextension-1.2.3

    Fetching dependencies... found 1:
      - php-5.5.31
    Building formula extensions/no-debug-non-zts-20121212/yourextension-1.2.3
    ...

If that works, run `bob deploy` instead to have the build upload to your bucket, and see the next section for important info about the manifest for your build.

#### Manifests

After a `bob build` or `bob deploy`, you'll be prompted to upload a manifest (unless your build was for a library or other base dependency). It obviously only makes sense to perform this upload after a `bob deploy`.

You can also run `support/build/_util/deploy.sh php-7.0.2` to have the manifest uploaded automatically for you after the deploy.

The manifest is a `composer.json` specific to your built runtime or extension. All manifests of your bucket together make up a repository (see below).

#### The Repository

Whenever you're happy with the state of your bucket, run `support/build/_util/mkrepo.sh` (you can also run this from a local computer if you give appropriate arguments and/or have the env vars set).

The script downloads all manifests from your bucket, generates a `packages.json` Composer repository, and tells you how to upload it back to S3.

#### Tips:

- To speed things up drastically during compilation, it'll usually be a good idea to `heroku run bash --size Performance-L` instead.
- All manifests generated by Bob formulas, by `support/build/_util/mkrepo.sh` and by `support/build/_util/sync.sh` use an S3 region of "s3" by default, so resulting URLs look like "`https://your-bucket.s3.amazonaws.com/your-prefix/...`". You can `heroku config:set S3_REGION` to change "s3" to another region like "us-west-1".
- If any dependencies are not yet deployed, you need to deploy them first by e.g. running `bob deploy libraries/libmemcached`.
