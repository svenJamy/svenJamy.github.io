# svenJamy.github.io


## operation

`hexo s` :本地查看效果

```
//等于一次性执行了，清空、刷新、部署三个命令
 hexo clean && hexo g && hexo d
```

## hexo 部署常见问题
1:
```
ERROR Script load failed: themes\next\scripts\tags\exturl.js
Error: Cannot find module 'hexo-util'
```
执行：`npm install -- save-dev hexo-util`重新安装下`hexo-util`


2：在执行 hexo deploy 后,出现 `error deployer not found:git `

执行：`npm install hexo-deployer-git --save`
