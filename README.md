# Shanty Blog

This is the source code for my personal blog found at https://www.huderlem.com/blog

It's built with [Hugo](https://gohugo.io/), a static site generator.

## Building the Website

To view changes locally when authoring a post
```
hugo server
```

Note, paths to images and other content in the `static/` directory should not be prefixed with `/blog`. I couldn't figure out how to get that working seamlessly both locally and live at https://www.huderlem.com/blog. For local testing, just strip the `/blog` prefix. For example:
```
// Bad
{{<figure src="/blog/posts/myimage.png" >}}
// Good
{{<figure src="/posts/myimage.png" >}}
```

To build the website for a release:
```
hugo -b https://www.huderlem.com/blog
```

This will output the full website contents into the `public/` directory. Its contents can simply be copied to the webserver.
