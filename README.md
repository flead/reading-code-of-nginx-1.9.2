### Mac下编译源码和安装Nginx

```shell
1. 资源下载
    nginx: http://nginx.org/download/nginx-1.12.2.tar.gz   [或者使用git中的1.9.2版本]
    zlib: http://zlib.net/zlib-1.2.11.tar.gz
    pcre: https://ftp.pcre.org/pub/pcre/pcre-8.38.tar.gz
    openssl: https://www.openssl.org/source/openssl-1.1.0g.tar.gz
这些都是 Nginx 编译需要的依赖，下载后分别解压， 注意解压的文件要在同一个目录下面

2. 解压
tar -zxvf nginx-1.12.2.tar.gz
tar -zxvf zlib-1.2.11.tar.gz
tar -zxvf pcre-8.38.tar.gz
tar -zxvf openssl-1.1.0g.tar.gz

3. 进入nginx目录后执行3个命令
./configure --with-debug --with-cc-opt='-g -O0' --prefix=/usr/local/nginx --with-zlib=../zlib-1.2.11 --with-pcre=../pcre-8.38 --with-openssl=../openssl-1.1.0

make

sudo make install

以上执行完毕后， nginx 就被安装到 /usr/local/nginx 目录下
Tips： 如果编译失败,检查 obj/Makeifle文件中的CFLAGS标记，是否有-Werror
CFLAGS =  -pipe  -O -W -Wall -Wpointer-arith -Wno-unused-parameter -g -O0
```

### Nginx的启停

```shell
sudo /usr/local/nginx/sbin/nginx
sudo /usr/local/nginx/sbin/nginx -s stop
sudo /usr/local/nginx/sbin/nginx -s reload
```

### 开启代码调试

```shell
./configure --with-debug --with-cc-opt='-g -O0' --prefix=/usr/local/nginx --with-zlib=../zlib-1.2.11 --with-pcre=../pcre-8.38 --with-openssl=../openssl-1.1.0g

/Users/lifei/work/nginx_source/nginx/objs/run_log

./configure --with-debug --with-cc-opt='-g -O0' --prefix=/usr/local/nginx --with-zlib=../zlib-1.2.11 --with-pcre=../pcre-8.38 --with-openssl=../openssl-1.1.0g
```

