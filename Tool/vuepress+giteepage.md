## vuepress加gitee page部署博客网站

准备：新建gitee、github仓库，命名my-blog，同时生成rsa，实验是否能够push成功。

利用vuepress构建博客

```cmd
# 安装vuepress
npm install -g vuepress

# 按照主题并初始化
npm install @vuepress-reco/theme-cli -g
theme-cli init my-blog

# install
cd my-blog
npm install

# run 在本地预览效果
npm run dev

# build 生成public文件夹，push该文件夹到gitee上，配置gitee page即可
npm run build
```

## 使用GitHub Action完成自动更新gitee page

从本地push到github，触发github action，然后自动同步到gitee，同时更新gitee page

参考链接

- https://github.com/yanglbme/gitee-pages-action

## 一键部署

在你博客主文件夹中新建一个脚本文件，deploy.sh

```bash
set e
sudo npm run build
sudo cp -r public/ git/

cd git/public
git add .
git commit -m "deploy"
git push
```

使用如下命令执行

```cmd
bash ./deploy.sh
```

