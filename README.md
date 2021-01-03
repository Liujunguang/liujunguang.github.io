# liujunguang.github.io

## Description
技术博客

## Development

### 1. 初始化

```shell script
hexo init <folder>
cd <folder>
npm install
```
将`README.md`、`_config.yml`拷贝进来

### 2. 编辑
1. 新建
```shell script
hexo new [layout] <tittle>
// -p,--path: 自定义新文章的路径
// -r,--replace: 如果存在同名文章，将其替换
// -s,--slug: 文章的Slug，作为新文章的文件名和发布后的URL
```
<font color="Maroon" size="1">如果没有设置layout，默认使用`__config.yml`中的default_layout参数代替。
如果标题包含空格，要使用“”引起来。</font>

### 3. 调试
启动服务，默认 http://localhost:4000
```shell script
hexo server
// 或
hexo s
```

### 4. 发布
- 配置`__config.yml`
- 部署:
```shell script
hexo clean && hexo d
```
