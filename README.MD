# 安装hexo客户端
- 安装node
- 执行以下命令
```bash
# 创建一个文件夹，hexo_blog，当做博客项目的目录
cd hexo_blog
# 安装hexo
npm install hexo -g
npm install hexo-cli g
hexo init
# 安装hexo依赖
npm install
# 安装git部署插件
npm install hexo-deployer-git --save
# 安装本地搜索插件
npm install hexo-generator-search --save
```
- 删除其他非必要文件（github上已有的文件），只保留node_modules。
- github SSH 授信
```bash
# 生成ssh key
ssh-keygen -t rsa -C "xxx@xmail.com" #一路enter
ssh-agent -s
eval `ssh-agent -s`
ssh-add
# 找到id_rsa.pub，文本打开复制内容至github的ssh key
ssh -T git@github.com # 根据提示验证
```
# hexo使用
- 建议使用命令创建文章：`hexo n "文章名字"`
- hexo新建文章目录：`hexo_blog\source\_posts`
- hexo命令
```bash
# 部署到github
hexo d -g
# 本地启动
hexo server
```
