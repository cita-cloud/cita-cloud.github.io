# 软件依赖

* Node.js & npm
* git

## 安装Hexo

```
npm install -g hexo-cli
```

# 发表新文章

## 克隆本仓库

```
git clone git@github.com:cita-cloud/cita-cloud.github.io.git
cd cita-cloud.github.io
```

## 创建新分支

```
git checkout -b new_post
```

## 创建新文章

```
hexo new new_post_title
```

注意：

* 该命令在`source/_posts/`目录下创建`new_post_title`目录和`new_post_title.md`。
* 在`new_post_title.md`中编辑新文章的内容。
* 如果文章中需要插入图片。可以将图片文件`test.png`放在`new_post_title`目录下，文章中使用`![](test.png)`插入图片。
* 如果要引用站内其他文章，文章的永久地址为`https://cita-cloud.github.io/{post_title}`。


## 安装依赖包

```
npm install
```

## 本地预览

```
hexo clean
hexo g
hexo s
```

然后通过`localhost:4000`预览。

## 发表文章

本地预览没有问题之后，推送分支到仓库，然后创建`PR`合并新文章到`main`分支。

`PR`合并之后，会自动触发`github action`来发表文章。
