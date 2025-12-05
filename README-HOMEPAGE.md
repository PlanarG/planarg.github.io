# 主页设置指南

## 已创建的文件

我已经为你创建了一个漂亮的个人主页，包含以下文件：

1. `_layouts/homepage.html` - 自定义主页布局
2. `assets/css/homepage.css` - 主页样式表
3. `index.markdown` - 主页内容

## 添加图片

### 1. 背景图片

将你喜欢的背景图片（建议 1920x1080 或更高分辨率）保存为：
```
assets/images/hero-bg.jpg
```

你可以从以下免费图片网站下载：
- [Unsplash](https://unsplash.com/) - 搜索 "cityscape" 或 "skyline"
- [Pexels](https://www.pexels.com/)
- [Pixabay](https://pixabay.com/)

### 2. 个人头像

将你的个人照片（建议正方形，至少 500x500px）保存为：
```
assets/images/profile.jpg
```

## 自定义内容

编辑 `index.markdown` 文件，修改以下内容：

### 个人信息
```yaml
name: Your Name              # 你的名字
job_title: Web Designer      # 你的职位/头衔
bio: "你的个人简介..."       # 个人简介
profile_image: /assets/images/profile.jpg  # 头像路径
```

### Publications（发表文章）
在主页内容部分，你可以添加或修改发表的论文：

```html
<div class="publication">
  <h3>论文标题</h3>
  <p class="authors"><strong>你的名字</strong>, 合作者</p>
  <p class="venue">会议/期刊名称 2024</p>
  <div class="links">
    <a href="pdf链接">[PDF]</a>
    <a href="代码链接">[Code]</a>
    <a href="项目链接">[Project Page]</a>
  </div>
</div>
```

### 导航链接
修改页脚导航链接：

```yaml
nav_links:
  - name: Home
    icon: 🏠
    url: /
  - name: Projects
    icon: 🚀
    url: /projects
  # 添加更多链接...
```

## 启动网站

在 `my-site` 目录下运行：

```bash
bundle exec jekyll serve
```

然后在浏览器访问：`http://localhost:4000`

## 样式自定义

如需修改颜色主题，编辑 `assets/css/homepage.css`：

- 主色调：`#667eea` 和 `#764ba2`（渐变紫色）
- 背景色：`#2c3e50`（深蓝灰）
- 文字颜色：`#333`（深灰）

## 提示

- 如果背景图片没有显示，CSS 会显示一个漂亮的紫色渐变背景
- 所有样式都是响应式的，在手机和平板上也能完美显示
- 可以添加更多 sections 来展示更多内容
