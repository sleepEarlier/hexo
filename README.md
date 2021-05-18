# HEXO

hexo操作
```
# 创建文章
hexo new xxxx
# 本地预览
hexo g
hexo s
# 部署
hexo d
```

# 主题
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