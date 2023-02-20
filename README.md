### Getting Started

1. You will need [Ruby](https://www.ruby-lang.org/en/) and [Bundler](https://bundler.io/) to use [Jekyll](https://jekyllrb.com/). Following [Using Jekyll with Bundler](https://jekyllrb.com/tutorials/using-jekyll-with-bundler/) to fullfill the enviromental requirement.

2. Installed dependencies in the `Gemfile`:

```sh
$ bundle install 
```

3. Serve the website (`localhost:4000` by default):

```sh
$ bundle exec jekyll serve  # alternatively, npm start
```

>
> Note: `tags` section can also be written as `tags: [Life, Meta]`.

After [Rake](https://github.com/ruby/rake) is introduced, we can use the command below to simplify the post creation:

```
rake post title="Hello 2015" subtitle="Hello World, Hello Blog"
```

This command will automatially generate a sample post similar as above under the `_posts/` folder.
