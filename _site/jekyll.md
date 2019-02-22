# jekyll配置笔记 -- ubuntu16.04

- 为了确保软件源最新，

    sudo apt update

- 确保开发环境正确

    sudo apt install build-essential

- 安装 ruby 相关

    sudo apt install ruby rubygems-integration ruby-bundler ruby-dev

- 安装 nodejs 与 python2.7

    sudo apt install nodejs Python2.7

- 安装 jekyll

    sudo gem install jekyll server

- 下面下载好博客的项目，在项目下构建本地服务

    jekyll server

- 报错一如下：
```bash
xhy@xhy-SVF14316SCB:~/xhy3054.github.io$ jekyll server
Configuration file: /home/xhy/xhy3054.github.io/_config.yml
  Dependency Error: Yikes! It looks like you don't have jekyll-paginate or one of its dependencies installed. In order to use Jekyll as currently configured, you'll need to install this gem. The full error message from Ruby is: 'cannot load such file -- jekyll-paginate' If you run into trouble, you can find helpful resources at https://jekyllrb.com/help/! 
jekyll 3.8.5 | Error:  jekyll-paginate

解决方法： sudo gem install jekyll-paginate
```

- 报错二如下：
```bash
xhy@xhy-SVF14316SCB:~/xhy3054.github.io$ jekyll server
Configuration file: /home/xhy/xhy3054.github.io/_config.yml
  Dependency Error: Yikes! It looks like you don't have jekyll-sitemap or one of its dependencies installed. In order to use Jekyll as currently configured, you'll need to install this gem. The full error message from Ruby is: 'cannot load such file -- jekyll-sitemap' If you run into trouble, you can find helpful resources at https://jekyllrb.com/help/! 
jekyll 3.8.5 | Error:  jekyll-sitemap

解决方法： sudo gem install jekyll-sitemap
```

- 解决完上述两个错误，构建成功
```bash
xhy@xhy-SVF14316SCB:~/xhy3054.github.io$ jekyll server
Configuration file: /home/xhy/xhy3054.github.io/_config.yml
            Source: /home/xhy/xhy3054.github.io
       Destination: /home/xhy/xhy3054.github.io/_site
 Incremental build: disabled. Enable with --incremental
      Generating... 
                    done in 3.118 seconds.
 Auto-regeneration: enabled for '/home/xhy/xhy3054.github.io'
    Server address: http://127.0.0.1:4000
  Server running... press ctrl-c to stop.

```

