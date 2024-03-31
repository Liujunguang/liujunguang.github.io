# liujunguang.github.io

My programing technology Blog.

## New

```shell
hugo new content posts/my-first-post.md
```

## Preview

```shell
hugo server
```

## Deploy

1. change `draft = false` to true
2. git push to github repository

## Images

将图片放在 `/static/images` 下，hugo 命令执行的时候，会把 `static` 目录下的子目录或文件复制到 `public` 目录下。
使用时直接以 `/public/[图片路径]` 取图片：

```markdown
![](/images/pic.png)
```
