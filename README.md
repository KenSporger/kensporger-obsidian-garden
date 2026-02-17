# Digital Obsidian Garden

这是一个Obsidian数字花园的模板，基于Github Theme，并集成了[jrtxio](https://github.com/jrtxio/jrtxio-obsidian-blogs/tree/main)开发的外观样式。

---


## 官方文档

参考[官方文档](https://dg-docs.ole.dev/)完成搭建与配置。

---



## 本地构建

为了方便样式调整，可以先进行本地构建，构建过程如下：

1. 仓库里至少包含一篇已发布的主页笔记（`dg-publish=true`、`dg-home=true`）。

2. 环境准备

   ```sh
   node -v # 本次使用版本为v20.20.0
   ```

3. 进入仓库根目录
   ```sh
   cd kensporger-obsidian-garden # 换成自己的仓库名
   ```

4. 安装依赖

   ```sh
   npm install
   ```

5. 构建（修改样式文件等要重新构建）

   ```sh
   npm run build
   ```

6. 本地预览

   ```sh
   npm run dev
   ```

