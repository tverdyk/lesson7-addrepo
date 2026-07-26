# Домашнее задание

1) Создать свой RPM пакет (можно взять свое приложение, либо собрать, например,
Apache с определенными опциями).
2) Создать свой репозиторий и разместить там ранее собранный RPM.

# Создать свой RPM пакет
*  Подготовка
*        [root@almalinux9 ~]# yum install -y wget rpmdevtools rpm-build
*        createrepo yum-utils cmake gcc git nano -y
*  Возьмем пакет Nginx и соберем его с дополнительным модулем ngx_broli.
   Загрузим SRPM пакет Nginx для дальнейшей работы над ним.
 
         root@almalinux9 ~]# mkdir rpm && cd rpm
         [root@almalinux9 rpm]# yumdownloader --source nginx
         nginx-1.20.1-28.el9_8.4.alma.1.src.rpm  

*  При установке такого пакета в домашней директории создается дерево каталогов для сборки, далее поставим все зависимости для сборки пакета Nginx:
  
        [root@almalinux9 rpm]# rpm -Uvh nginx*.src.rpm
        Updating / installing...
        1:nginx-2:1.20.1-28.el9_8.4.alma.1 warning: user mockbuild does not exist - using root
        warning: group mock does not exist - using root
        ################################# [100%]
   
        [root@almalinux9 rpm]# yum-builddep nginx # подтянит пачку зависимостей.

*  Нужно скачать исходный код модуля ngx_brotli — он потребуется при сборке:

        [root@almalinux9 ~]# git clone --recurse-submodules -j8 https://github.com/google/ngx_brotli
        Cloning into 'ngx_brotli'...
        [root@almalinux9 ~]# cd ngx_brotli/deps/brotli
        [root@almalinux9 brotli]# mkdir out && cd out

*  Собираем модуль ngx_brotli:

  
       [root@almalinux9 out]# cmake -DCMAKE_BUILD_TYPE=Release -DBUILD_SHARED_LIBS=OFF -DCMAKE_C_FLAGS="-Ofast -m64 -march=native -mtune=native -flto -funroll-loops -ffunction-sections -fdata-sections -Wl,--gc-sections" -DCMAKE_CXX_FLAGS="-Ofast -m64 -march=native -mtune=native -flto -funroll-loops -ffunction-sections -fdata-sections -Wl,--gc-sections" -DCMAKE_INSTALL_PREFIX=./installed ..
       -- The C compiler identification is GNU 11.5.0
       -- Detecting C compiler ABI info
       -- Detecting C compiler ABI info - done
       -- Check for working C compiler: /bin/cc - skipped
       -- Detecting C compile features
       -- Detecting C compile features - done
       -- Build type is 'Release'
       -- Performing Test BROTLI_EMSCRIPTEN
       -- Performing Test BROTLI_EMSCRIPTEN - Failed
       -- Compiler is not EMSCRIPTEN
       -- Looking for log2
       -- Looking for log2 - not found
       -- Looking for log2
       -- Looking for log2 - found
       -- Configuring done (0.4s)
       -- Generating done (0.1s)
       CMake Warning:
        Manually-specified variables were not used by the project:
        CMAKE_CXX_FLAGS
       -- Build files have been written to: /root/ngx_brotli/deps/brotli/out
*
*      [root@almalinux9 out]# cmake --build . --config Release -j 2 --target brotlienc
       [root@almalinux9 out]# cmake --build . --config Release -j 2 --target brotlienc
        3%] Building C object CMakeFiles/brotlicommon.dir/c/common/constants.c.o
        6%] Building C object CMakeFiles/brotlicommon.dir/c/common/context.c.o
       13%] Building C object CMakeFiles/brotlicommon.dir/c/common/dictionary.c.o
       13%] Building C object CMakeFiles/brotlicommon.dir/c/common/platform.c.o
       17%] Building C object CMakeFiles/brotlicommon.dir/c/common/shared_dictionary.c.o
       20%] Building C object CMakeFiles/brotlicommon.dir/c/common/transform.c.o
       24%] Linking C static library libbrotlicommon.a
       24%] Built target brotlicommon
       27%] Building C object CMakeFiles/brotlienc.dir/c/enc/backward_references.c.o
       31%] Building C object CMakeFiles/brotlienc.dir/c/enc/backward_references_hq.c.o
       34%] Building C object CMakeFiles/brotlienc.dir/c/enc/bit_cost.c.o
       37%] Building C object CMakeFiles/brotlienc.dir/c/enc/block_splitter.c.o
       41%] Building C object CMakeFiles/brotlienc.dir/c/enc/brotli_bit_stream.c.o
       44%] Building C object CMakeFiles/brotlienc.dir/c/enc/cluster.c.o
       48%] Building C object CMakeFiles/brotlienc.dir/c/enc/command.c.o
       51%] Building C object CMakeFiles/brotlienc.dir/c/enc/compound_dictionary.c.o
       55%] Building C object CMakeFiles/brotlienc.dir/c/enc/compress_fragment.c.o
       58%] Building C object CMakeFiles/brotlienc.dir/c/enc/compress_fragment_two_pass.c.o
       62%] Building C object CMakeFiles/brotlienc.dir/c/enc/dictionary_hash.c.o
       65%] Building C object CMakeFiles/brotlienc.dir/c/enc/encode.c.o
       68%] Building C object CMakeFiles/brotlienc.dir/c/enc/encoder_dict.c.o
       72%] Building C object CMakeFiles/brotlienc.dir/c/enc/entropy_encode.c.o
       75%] Building C object CMakeFiles/brotlienc.dir/c/enc/fast_log.c.o
       79%] Building C object CMakeFiles/brotlienc.dir/c/enc/histogram.c.o
       82%] Building C object CMakeFiles/brotlienc.dir/c/enc/literal_cost.c.o
       86%] Building C object CMakeFiles/brotlienc.dir/c/enc/memory.c.o
       89%] Building C object CMakeFiles/brotlienc.dir/c/enc/metablock.c.o
       93%] Building C object CMakeFiles/brotlienc.dir/c/enc/static_dict.c.o
       96%] Building C object CMakeFiles/brotlienc.dir/c/enc/utf8_util.c.o
      100%] Linking C static library libbrotlienc.a
      100%] Built target brotlienc

*  Поправить сам spec файл, чтобы Nginx собирался с необходимыми нам опциями: находим секцию с параметрами configure (до условий if) и добавляем указание на модуль (не забудьте указать завершающий обратный слэш):

       [root@almalinux9 SPECS]# ls
       nginx.spec
   
       [root@almalinux9 SPECS]# nano nginx.spec
       export DESTDIR=%{buildroot}
       #So the perl module finds its symbols:
       nginx_ldopts="$RPM_LD_FLAGS -Wl,-E"
       if ! ./configure \
       --prefix=%{_datadir}/nginx \
       --sbin-path=%{_sbindir}/nginx \
       --modules-path=%{nginx_moduledir} \
       --conf-path=%{_sysconfdir}/nginx/nginx.conf \
       --error-log-path=%{_localstatedir}/log/nginx/error.log \
       --http-log-path=%{_localstatedir}/log/nginx/access.log \
       --http-client-body-temp-path=%{_localstatedir}/lib/nginx/tmp/client_body \
       --http-proxy-temp-path=%{_localstatedir}/lib/nginx/tmp/proxy \
       --http-fastcgi-temp-path=%{_localstatedir}/lib/nginx/tmp/fastcgi \
       --http-uwsgi-temp-path=%{_localstatedir}/lib/nginx/tmp/uwsgi \
       --http-scgi-temp-path=%{_localstatedir}/lib/nginx/tmp/scgi \
       --pid-path=/run/nginx.pid \
       --lock-path=/run/lock/subsys/nginx \
       --user=%{nginx_user} \
       --group=%{nginx_user} \
       --with-compat \
       --with-debug \
       --add-module=/root/ngx_brotli \   # Добавили строку

*  Запуск сборки

       [root@almalinux9 SPECS]# rpmbuild -ba nginx.spec -D 'debug_package %{nil}'
       setting SOURCE_DATE_EPOCH=1783382400
       Executing(%prep): /bin/sh -e /var/tmp/rpm-tmp.N2ZOWF
       + umask 022
       + cd /root/rpmbuild/BUILD
       + cat /root/rpmbuild/SOURCES/maxim.key /root/rpmbuild/SOURCES/mdounin.key /root/rpmbuild/SOURCES/sb.key
       + /usr/lib/rpm/redhat/gpgverify
       --keyring=/root/rpmbuild/BUILD/nginx.gpg
       --signature=/root/rpmbuild/SOURCES/nginx-1.20.1.tar.gz.asc
       --data=/root/rpmbuild/SOURCES/nginx-1.20.1.tar.gz
       gpgv: Signature made Tue May 25 15:42:56 2021 MSK
       gpgv:                using RSA key 520A9993A1C052F8
       gpgv: Good signature from "Maxim Dounin <mdounin@mdounin.ru>"
       + cd /root/rpmbuild/BUILD
       + rm -rf nginx-1.20.1
       + /usr/bin/gzip -dc /root/rpmbuild/SOURCES/nginx-1.20.1.tar.gz
       + /usr/bin/tar -xof -
       + STATUS=0
       + '[' 0 -ne 0 ']'
       + cd nginx-1.20.1
       + /usr/bin/chmod -Rf a+rX,u+w,g-w,o-w .
       + /usr/bin/cat /root/rpmbuild/SOURCES/0001-remove-Werror-in-upstream-build-scripts.patch
       ...
       Executing(%clean): /bin/sh -e /var/tmp/rpm-tmp.bH4KHE
       + umask 022
       + cd /root/rpmbuild/BUILD
       + cd nginx-1.20.1
       + /usr/bin/rm -rf /root/rpmbuild/BUILDROOT/nginx-1.20.1-28.el9.4.alma.1.x86_64
       + RPM_EC=0
       ++ jobs -p
       + exit 0

*  Проверка созданя пакетов;

       [root@almalinux9 ~]# ls rpmbuild/RPMS/x86_64/
       nginx-1.20.1-28.el9.4.alma.1.x86_64.rpm                        nginx-mod-http-perl-1.20.1-28.el9.4.alma.1.x86_64.rpm
       nginx-core-1.20.1-28.el9.4.alma.1.x86_64.rpm                   nginx-mod-http-xslt-filter-1.20.1-28.el9.4.alma.1.x86_64.rpm
       nginx-mod-devel-1.20.1-28.el9.4.alma.1.x86_64.rpm              nginx-mod-mail-1.20.1-28.el9.4.alma.1.x86_64.rpm
       nginx-mod-http-image-filter-1.20.1-28.el9.4.alma.1.x86_64.rpm  nginx-mod-stream-1.20.1-28.el9.4.alma.1.x86_64.rpm
      
*  Копируем пакеты в общий каталог:

	   [root@almalinux9 ~]# cp ~/rpmbuild/RPMS/noarch/* ~/rpmbuild/RPMS/x86_64/
       [root@almalinux9 ~]# cd ~/rpmbuild/RPMS/x86_64
       [root@almalinux9 x86_64]# ls
       nginx-1.20.1-28.el9.4.alma.1.x86_64.rpm              nginx-mod-http-image-filter-1.20.1-28.el9.4.alma.1.x86_64.rpm
       nginx-all-modules-1.20.1-28.el9.4.alma.1.noarch.rpm  nginx-mod-http-perl-1.20.1-28.el9.4.alma.1.x86_64.rpm
       nginx-core-1.20.1-28.el9.4.alma.1.x86_64.rpm         nginx-mod-http-xslt-filter-1.20.1-28.el9.4.alma.1.x86_64.rpm
       nginx-filesystem-1.20.1-28.el9.4.alma.1.noarch.rpm   nginx-mod-mail-1.20.1-28.el9.4.alma.1.x86_64.rpm
       nginx-mod-devel-1.20.1-28.el9.4.alma.1.x86_64.rpm    nginx-mod-stream-1.20.1-28.el9.4.alma.1.x86_64.rpm
      
*   Теперь можно установить наш пакет и убедиться, что nginx работает:

        [root@almalinux9 x86_64]# yum localinstall *.rpm
        Last metadata expiration check: 0:29:38 ago on Sun Jul 26 10:21:30 2026.
        Dependencies resolved.
        =============================================================================================================================================
         Package                                      Architecture            Version                                     Repository                           Size
        =============================================================================================================================================
        Installing:
        nginx                                        x86_64                  2:1.20.1-28.el9.4.alma.1                    @commandline                         37 k
        nginx-all-modules                            noarch                  2:1.20.1-28.el9.4.alma.1                    @commandline                  9.      2 k
        nginx-core                                   x86_64                  2:1.20.1-28.el9.4.alma.1                    @commandline                  1.      0 M
        nginx-filesystem                             noarch                  2:1.20.1-28.el9.4.alma.1                    @commandline                         11 k
        nginx-mod-devel                              x86_64                  2:1.20.1-28.el9.4.alma.1                    @commandline                        745 k
        nginx-mod-http-image-filter                  x86_64                  2:1.20.1-28.el9.4.alma.1                    @commandline                         21 k
        nginx-mod-http-perl                          x86_64                  2:1.20.1-28.el9.4.alma.1                    @commandline                         32 k
        nginx-mod-http-xslt-filter                   x86_64                  2:1.20.1-28.el9.4.alma.1                    @commandline                         20 k
        nginx-mod-mail                               x86_64                  2:1.20.1-28.el9.4.alma.1                    @commandline                         54 k
        nginx-mod-stream                             x86_64                  2:1.20.1-28.el9.4.alma.1                    @commandline                         80 k
        Installing dependencies:
        almalinux-logos-httpd                        noarch                  90.7-1.el9                                  appstream                            18 k
        Transaction Summary

        =============================================================================================================================================
        Install  11 Packages
        Total size: 2.0 M
        Total download size: 18 k
        Installed size: 9.5 M
        Is this ok [y/N]: y
    
      
        [root@almalinux9 x86_64]# systemctl start nginx
        [root@almalinux9 x86_64]# systemctl status nginx
        ● nginx.service - The nginx HTTP and reverse proxy server
           Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; preset: disabled)
           Active: active (running) since Sun 2026-07-26 10:53:56 MSK; 7s ago
          Process: 48936 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
          Process: 48937 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
          Process: 48938 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
         Main PID: 48939 (nginx)
            Tasks: 3 (limit: 7816)
           Memory: 5.2M (peak: 5.8M)
              CPU: 38ms
           CGroup: /system.slice/nginx.service
                   ├─48939 "nginx: master process /usr/sbin/nginx"
                   ├─48940 "nginx: worker process"
                   └─48941 "nginx: worker process"
      
        Jul 26 10:53:56 almalinux9 systemd[1]: Starting The nginx HTTP and reverse proxy server...
        Jul 26 10:53:56 almalinux9 nginx[48937]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
        Jul 26 10:53:56 almalinux9 nginx[48937]: nginx: configuration file /etc/nginx/nginx.conf test is successful
        Jul 26 10:53:56 almalinux9 systemd[1]: Started The nginx HTTP and reverse proxy server.

## Далее мы будем использовать его для доступа к своему репозиторию. ##

# Создать свой репозиторий и разместить там ранее собранный RPM
# Приступим к созданию своего репозитория. Директория для статики у Nginx по умолчанию /usr/share/nginx/html. 

*  Создадим там каталог repo:

       [root@almalinux9 x86_64]# mkdir /usr/share/nginx/html/repo

*  Копируем в каталог наши собранные RPM-пакеты:

       [root@almalinux9 x86_64]# cp ~/rpmbuild/RPMS/x86_64/*.rpm /usr/share/nginx/html/repo/

*  Инициализируем репозиторий командой:

       [root@almalinux9 x86_64]# createrepo /usr/share/nginx/html/repo/
       Directory walk started
       Directory walk done - 10 packages
       Temporary output repo path: /usr/share/nginx/html/repo/.repodata/
       Preparing sqlite DBs
       Pool started (with 5 workers)
       Pool finished

*  Для прозрачности настроим в NGINX доступ к листингу каталога. В файле /etc/nginx/nginx.conf в блоке server добавим следующие директивы:


       [root@almalinux9 x86_64]# nano /etc/nginx/nginx.conf
       server {
	        listen       80;
            listen       [::]:80;
            server_name  _;
            root         /usr/share/nginx/html;
    
            # Load configuration files for the default server block.
            include /etc/nginx/default.d/*.conf;
            # add index repository     # Добавили строки
            index index.html index.htm;
            autoindex on;
    
            error_page 404 /404.html;
            location = /404.html {
            }
    
	        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }

*  Проверяем синтаксис и перезапускаем NGINX:
 
       [root@almalinux9 x86_64]# nginx -t
       nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
       nginx: configuration file /etc/nginx/nginx.conf test is successful

       [root@almalinux9 x86_64]# nginx -s reload

*  Теперь ради интереса можно посмотреть в браузере или с помощью curl:

       [root@almalinux9 x86_64]# curl -a http://localhost/repo/
       <html>
       <head><title>Index of /repo/</title></head>
       <body>
       <h1>Index of /repo/</h1><hr><pre><a href="../">../</a>
       <a href="repodata/">repodata/</a>                                          26-Jul-2026 08:01                   -
       <a href="nginx-1.20.1-28.el9.4.alma.1.x86_64.rpm">nginx-1.20.1-28.el9.4.alma.1.x86_64.rpm</a>            26-Jul-2026 08:00               38277
       <a href="nginx-all-modules-1.20.1-28.el9.4.alma.1.noarch.rpm">nginx-all-modules-1.20.1-28.el9.4.alma.1.noarch..&gt;</a> 26-Jul-2026 08:00                9377
       <a href="nginx-core-1.20.1-28.el9.4.alma.1.x86_64.rpm">nginx-core-1.20.1-28.el9.4.alma.1.x86_64.rpm</a>       26-Jul-2026 08:00             1032726
       <a href="nginx-filesystem-1.20.1-28.el9.4.alma.1.noarch.rpm">nginx-filesystem-1.20.1-28.el9.4.alma.1.noarch.rpm</a> 26-Jul-2026 08:00               10977
       <a href="nginx-mod-devel-1.20.1-28.el9.4.alma.1.x86_64.rpm">nginx-mod-devel-1.20.1-28.el9.4.alma.1.x86_64.rpm</a>  26-Jul-2026 08:00              763139
       <a href="nginx-mod-http-image-filter-1.20.1-28.el9.4.alma.1.x86_64.rpm">nginx-mod-http-image-filter-1.20.1-28.el9.4.alm..&gt;</a> 26-Jul-2026 08:00               21361
       <a href="nginx-mod-http-perl-1.20.1-28.el9.4.alma.1.x86_64.rpm">nginx-mod-http-perl-1.20.1-28.el9.4.alma.1.x86_..&gt;</a> 26-Jul-2026 08:00               32888
       <a href="nginx-mod-http-xslt-filter-1.20.1-28.el9.4.alma.1.x86_64.rpm">nginx-mod-http-xslt-filter-1.20.1-28.el9.4.alma..&gt;</a> 26-Jul-2026 08:00               20167
       <a href="nginx-mod-mail-1.20.1-28.el9.4.alma.1.x86_64.rpm">nginx-mod-mail-1.20.1-28.el9.4.alma.1.x86_64.rpm</a>   26-Jul-2026 08:00               55771
       <a href="nginx-mod-stream-1.20.1-28.el9.4.alma.1.x86_64.rpm">nginx-mod-stream-1.20.1-28.el9.4.alma.1.x86_64.rpm</a> 26-Jul-2026 08:00               82317
       </pre><hr></body>
       </html>
    
## *  Все готово для того, чтобы протестировать репозиторий. ##

*  Добавим его в /etc/yum.repos.d:

       [root@almalinux9 x86_64]# cat >> /etc/yum.repos.d/otus.repo << EOF
       [otus]
       name=otus-linux
       baseurl=http://localhost/repo
       gpgcheck=0
       enabled=1
       EOF
    
*  Убедимся, что репозиторий подключился и посмотрим, что в нем есть:

       [root@almalinux9 x86_64]# yum repolist enabled | grep otus
       otus                             otus-linux

*  Добавим пакет в наш репозиторий:

       [root@almalinux9 repo]# wget https://repo.percona.com/yum/percona-release-latest.noarch.rpm
       --2026-07-26 11:19:37--  https://repo.percona.com/yum/percona-release-latest.noarch.rpm
       Resolving repo.percona.com (repo.percona.com)... 49.12.125.205, 2a01:4f8:242:5792::2
       Connecting to repo.percona.com (repo.percona.com)|49.12.125.205|:443... connected.
       HTTP request sent, awaiting response... 200 OK
       Length: 28152 (27K) [application/x-redhat-package-manager]
       Saving to: ‘percona-release-latest.noarch.rpm’
    
       percona-release-latest.noarch.rpm        100%

*  Обновим список пакетов в репозитории:

       [root@almalinux9 repo]# createrepo /usr/share/nginx/html/repo/
       Directory walk started
       Directory walk done - 11 packages
       Temporary output repo path: /usr/share/nginx/html/repo/.repodata/
       Preparing sqlite DBs
       Pool started (with 5 workers)
       Pool finished
    
       [root@almalinux9 repo]#  yum makecache
       AlmaLinux 9 - AppStream  
       AlmaLinux 9 - BaseOS     
       AlmaLinux 9 - Extras     
       otus-linux               
       Metadata cache created.
    
       [root@almalinux9 repo]# yum list | grep otus
       percona-release.noarch 

*  Так как Nginx у нас уже стоит, установим репозиторий percona-release:

       [root@almalinux9 repo]# yum install -y percona-release.noarch
       Last metadata expiration check: 0:02:36 ago on Sun Jul 26 11:21:10 2026.
       Dependencies resolved.
       ...
       Installed:
       percona-release-1.0-33.noarch                                                                                                                                   
       Complete!
    
## Все прошло успешно. 
*  В случае, если вам потребуется обновить репозиторий (а это делается при каждом добавлении файлов) снова, 
   то выполните команду
   
        createrepo /usr/share/nginx/html/repo/

