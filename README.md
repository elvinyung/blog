# blog
### by [Elvin Yung](https://github.com/elvinyung)
A blog.


### Instructions
Mostly a note to self.

This blog is built with [Hugo](http://gohugo.io/), an excellent static site generator by [Steve Francia](http://spf13.com/). Install Hugo with the following command after installing Go and setting up your `GOPATH`:
```
go get -v github.com/spf13/hugo
```

Syntax highlighting of code blocks is powered by the Python package [Pygments](http://pygments.org/). To install Pygments:

```
pip install pygments
```

To create a new post:
```
hugo new post/<name>.md
```

To run a test server that watches for changes:
```
hugo server --watch
```

To compile into a static site (into the `public` folder):
```
hugo
```
