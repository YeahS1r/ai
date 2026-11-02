# 我的 AI 学习作品集

一个不需要安装依赖的静态个人主页，整理 AI 课程里的数据分析、流程设计和创意练习。

## 本地预览

直接用浏览器打开 `index.html`。也可以把整个文件夹放到静态网页托管服务上；主页和作品链接都使用相对路径。

## 作品内容

- **学生成绩分析仪表盘**：位于 `projects/sc_edu_dashboard_html/`。交互式页面、图表、报告和配套文件都已保留，请保持这个目录内部结构。
- **PPT 生产流程与反思模式**：位于 `assets/` 的三张流程图。
- **科比 × 奶龙短视频**：`assets/kobe-24-wink-laugh.mp4`。

## 发布到 GitHub

1. 在 GitHub 新建一个仓库。
2. 将本文件夹中的全部文件上传到仓库根目录，或在本文件夹运行：

   ```bash
   git init
   git add .
   git commit -m "Add AI learning portfolio"
   git branch -M main
   git remote add origin <你的 GitHub 仓库地址>
   git push -u origin main
   ```

3. 如需网页链接，在仓库的 **Settings → Pages** 中选择从 `main` 分支的根目录发布。

## 上传前请确认数据权限

仪表盘配套文件包含约 1,000 条学生成绩记录，并使用族群、午餐类型和家长教育程度等分组字段。当前材料没有注明原始数据的公开许可或来源。公开仓库前，请先确认这些数据可以公开；如果不能公开，请移除或替换 `projects/sc_edu_dashboard_html/data/` 下的记录文件，并检查仪表盘里相应的数据下载入口。

视频和图表沿用你提供的素材。上传前请确认自己有权在 GitHub 上分享这些素材。此仓库没有附加开源许可证，因为目前没有提供素材和数据的授权信息。
