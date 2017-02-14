# VPS大水管

众所周知大陆的网络环境实在糟糕。不科学上网，下载服务器在国外的文件（比如github）不是速度慢就是断流，即使用了ss也不见得有所改善。更加狗血的是断了之后可能要重新开始，实在不能忍受。试了几次用VPS下载效果不错，于是打算在VPS上搭个aria2服务器。

当然把aria2暴露外网后应该加个认证--rpc-secret=\$\$your-token\$\$。按照之前写的[这个](http://tumblr.liuweiqiang.me/post/89850939965/%E4%B8%87%E8%83%BD%E4%B8%8B%E8%BD%BD%E6%9C%BA)和[这个](http://tumblr.liuweiqiang.me/post/108981098575/raspberry-pi-as-home-server)搭建好（自备梯子）。

然后将上面给的aria2rpc里的'params':[opts.URIs]改为'params':['token:\$\$your-token\$\$', opts.URIs]。

文件取回本地，为避免违反搬瓦工的[TOS](http://bandwagonhost.com/knowledgebase.php?action=displayarticle&id=6)（的检测），要加密传输的文件，发现aria2支持sftp（ftp-user、ftp-passwd等aria2参数），只要服务端开了ssh就可以了，使用sftp://host:port/path/to/your/file。另外，homebrew版的aria2不支持sftp，装了官方版的。
