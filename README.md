# sturdy-octo-adventure

## hexo
The main command to live serve for iterative development is
```
hexo server
```

## lx theme

### Bundling assets into dist directory
Use gulp. By default you can run `gulp` to build everything, but you can run
```
gulp --tasks
```
to see other tasks, or peek at the gulpfile.

If you have any issues with gulp, check out this [article](https://gist.github.com/noraj/007a943dc781dc8dd3198a29205bae04).

## hexo-math
I use [this module](https://github.com/hexojs/hexo-math) for rendering math symbols. Options specific to katex can be found [here](https://katex.org/docs/options.html).