# HEXO

### hexo 安装
```
# 全局安装
npm install -g hexo-cli

# 初始化博客目录
hexo init blog

# 更新hexo
npm update hexo -g
```

### hexo操作
```
# 创建文章
hexo new xxxx
# 本地预览
hexo g
hexo s
# 部署
hexo clean
hexo g
hexo d
```

### 其他命令
```
hexo server -s #静态模式
hexo server -p 5000 #更改端口
hexo server -i 192.168.1.1 #自定义 IP
```

如遇到部署失败，可以在 `package.json`中删除 `hexo-deployer-git`，然后重新安装:
```
npm install hexo-deployer-git --save
```


### 主题
主题使用 `git subtree` 管理，其中landscape为默认主题，`BlueLake` 为当前主题。
`subtree1` 相关指令:
```
# 添加为remote，后续指令可以使用BlueLake代理git地址
git remote add -f BlueLake https://github.com/chaooo/hexo-theme-BlueLake.git
# 添加subtree，prefix中可以指定路径，squash不拉取完成历史记录，而是生成一条commit记录
git subtree add --prefix=themes/BlueLake BlueLake master --squash
# 更新
git subtree pull --prefix=themes/BlueLake BlueLake master --squash
# push
git subtree push --prefix=themes/BlueLake BlueLake master
```

参考:
[GitHub+Hexo 搭建个人网站详细教程](https://zhuanlan.zhihu.com/p/26625249)
