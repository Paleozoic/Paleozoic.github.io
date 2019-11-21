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
# 其他
npm install hexo-util --save
# SEO
npm install hexo-generator-sitemap --save
npm install hexo-generator-baidu-sitemap --save
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
- 下载源文件
```bash
 git clone -b source https://github.com/Paleozoic/Paleozoic.github.io.git
 ```
 - 下载完毕将源文件复制和.git文件夹复制至hexo_blog文件夹。

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

# SEO
## 网站验证
### 文件验证
source目录下的
baidu_verify_xxxx.html
googlexxx.html
是用于百度和谷歌进行网站归属验证，从两者官方下载后放到source目录，deploy后需要到git修改。
（因为deoloy后文件会被编译嵌套其他html，所以应该还原）
之后需要添加ignore文件，后续部署忽略掉，防止覆盖。
### CNAME验证
直接添加DNS解析
### 标签验证
修改主题文件的index.ejs，添加meta验证标签。
以上的具体使用参考搜索引擎官网。

