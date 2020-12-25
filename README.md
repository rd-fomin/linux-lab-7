# linux-lab-7
## Практика по Software distribution
##### Установим все необходимое
```
root@roman-VirtualBox:~# yum-builddep rpmbuild/SPECS/nginx.spec
root@roman-VirtualBox:~# wget https://nginx.org/packages/centos/7/SRPMS/nginx-1.18.0-2.el7.ngx.src.rpm
root@roman-VirtualBox:~# rpm -i nginx-1.18.0-2.el7.ngx.src.rpm
root@roman-VirtualBox:~# wget https://www.openssl.org/source/latest.tar.gz
root@roman-VirtualBox:~# tar -xvf latest.tar.gz
root@roman-VirtualBox:~# yum-builddep rpmbuild/SPECS/nginx.spec
root@roman-VirtualBox:~# ls
anaconda-ks.cfg  latest.tar.gz  openssl-1.1.1i  original-ks.cfg  rpmbuild
root@roman-VirtualBox:~# pwd
/root
```
##### Правим файл nginx.spec
```
root@roman-VirtualBox:~# nano rpmbuild/SPECS/nginx.spec
...
%build
./configure %{BASE_CONFIGURE_ARGS} \
    --with-cc-opt="%{WITH_CC_OPT}" \
    --with-ld-opt="%{WITH_LD_OPT}" \
    --with-openssl=/root/openssl-1.1.1i \
    --with-debug
```
##### Устанавливаем gcc для сборки, запускаем саму сборку и смотрим, что пакеты создались после сборки
```
root@roman-VirtualBox:~# yum install gcc
root@roman-VirtualBox:~# rpmbuild -bb rpmbuild/SPECS/nginx.spec
root@roman-VirtualBox:~# ls -la rpmbuild/RPMS/x86_64/
total 4584
drwxr-xr-x. 2 root root      98 Dec 14 16:18 .
drwxr-xr-x. 3 root root      20 Dec 14 16:18 ..
-rw-r--r--. 1 root root 2222324 Dec 14 16:18 nginx-1.18.0-2.el8.ngx.x86_64.rpm
-rw-r--r--. 1 root root 2467084 Dec 14 16:18 nginx-debuginfo-1.18.0-2.el8.ngx.x86_64.rpm
```
##### Устанавливаем наш пакет и проверяем, что nginx работает
```
root@roman-VirtualBox:~# yum localinstall -y rpmbuild/RPMS/x86_64/nginx-1.18.0-2.el8.ngx.x86_64.rpm
root@roman-VirtualBox:~# systemctl start nginx
root@roman-VirtualBox:~# systemctl status nginx
● nginx.service - nginx - high performance web server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Mon 2020-12-14 16:25:12 UTC; 1s ago
     Docs: http://nginx.org/en/docs/
  Process: 21750 ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx.conf (code=exited, status=0/SUCCESS)
 Main PID: 21751 (nginx)
    Tasks: 2 (limit: 2881)
   Memory: 2.0M
   CGroup: /system.slice/nginx.service
           ├─21751 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx.conf
           └─21752 nginx: worker process

Dec 14 16:25:12 localhost.localdomain systemd[1]: Starting nginx - high performance web server...
Dec 14 16:25:12 localhost.localdomain systemd[1]: nginx.service: Can't open PID file /var/run/nginx.pid (yet?) after start: No such file or>
Dec 14 16:25:12 localhost.localdomain systemd[1]: Started nginx - high performance web server.
```
##### Создадим каталог repo в директории nginx и скопируем туда наш собранный RPM и RPM для установки репозитория
```
root@roman-VirtualBox:~# mkdir /usr/share/nginx/html/repo
root@roman-VirtualBox:~# cp rpmbuild/RPMS/x86_64/nginx-1.18.0-2.el8.ngx.x86_64.rpm /usr/share/nginx/html/repo/
root@roman-VirtualBox:~# wget https://repo.percona.com/centos/7Server/RPMS/noarch/percona-release-1.0-9.noarch.rpm -O /usr/share/nginx/html/repo/percona-release-1.0-9.noarch.rpm
```
##### Инициализируем репозиторий
```
root@roman-VirtualBox:~# createrepo /usr/share/nginx/html/repo/
Directory walk started
Directory walk done - 2 packages
Temporary output repo path: /usr/share/nginx/html/repo/.repodata/
Preparing sqlite DBs
Pool started (with 5 workers)
Pool finished
```
##### Добавлеям автоиндексирование в файле конфигурации и перезапустим nginx
```
root@roman-VirtualBox:~# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
root@roman-VirtualBox:~# nginx -s reload
```
##### Проверим, что репозиторий создался
```
root@roman-VirtualBox:~# curl -a http://localhost/repo/
<html>
<head><title>Index of /repo/</title></head>
<body>
<h1>Index of /repo/</h1><hr><pre><a href="../">../</a>
<a href="repodata/">repodata/</a>                                          14-Dec-2020 16:39                   -
<a href="nginx-1.18.0-2.el8.ngx.x86_64.rpm">nginx-1.18.0-2.el8.ngx.x86_64.rpm</a>                  14-Dec-2020 16:38             2222324
<a href="percona-release-1.0-9.noarch.rpm">percona-release-1.0-9.noarch.rpm</a>                   12-Mar-2019 13:35               16664
</pre><hr></body>
</html>
```
##### Добавляем репозиторий
```
root@roman-VirtualBox:~# cat >> /etc/yum.repos.d/mai.repo << EOF
> [mai]
> name=mai-linux
> baseurl=http://localhost/repo
> gpgcheck=0
> enabled=1
> EOF
```
##### Убедимся, что репозиторий есть
```
root@roman-VirtualBox:~# yum repolist enabled | grep mai
Failed to set locale, defaulting to C.UTF-8
mai                                mai-linux
```
##### Переустановим nginx
```
root@roman-VirtualBox:~# yum reinstall nginx
Failed to set locale, defaulting to C.UTF-8
Last metadata expiration check: 0:01:46 ago on Mon Dec 14 16:44:50 2020.
Dependencies resolved.
============================================================================================================================================
 Package                Architecture            Version                                                    Repository                  Size
============================================================================================================================================
Reinstalling:
 nginx                  x86_64                  1:1.14.1-9.module_el8.0.0+184+e34fea82                     AppStream                  570 k

Transaction Summary
============================================================================================================================================

Total download size: 570 k
Installed size: 1.7 M
Is this ok [y/N]: y
Downloading Packages:
nginx-1.14.1-9.module_el8.0.0+184+e34fea82.x86_64.rpm                                                       1.4 MB/s | 570 kB     00:00    
--------------------------------------------------------------------------------------------------------------------------------------------
Total                                                                                                       989 kB/s | 570 kB     00:00     
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                                                                    1/1 
  Running scriptlet: nginx-1:1.14.1-9.module_el8.0.0+184+e34fea82.x86_64                                                                1/1 
  Reinstalling     : nginx-1:1.14.1-9.module_el8.0.0+184+e34fea82.x86_64                                                                1/2 
  Running scriptlet: nginx-1:1.14.1-9.module_el8.0.0+184+e34fea82.x86_64                                                                1/2 
  Running scriptlet: nginx-1:1.14.1-9.module_el8.0.0+184+e34fea82.x86_64                                                                2/2 
  Cleanup          : nginx-1:1.14.1-9.module_el8.0.0+184+e34fea82.x86_64                                                                2/2 
  Running scriptlet: nginx-1:1.14.1-9.module_el8.0.0+184+e34fea82.x86_64                                                                2/2 
  Verifying        : nginx-1:1.14.1-9.module_el8.0.0+184+e34fea82.x86_64                                                                1/2 
  Verifying        : nginx-1:1.14.1-9.module_el8.0.0+184+e34fea82.x86_64                                                                2/2 

Reinstalled:
  nginx-1:1.14.1-9.module_el8.0.0+184+e34fea82.x86_64                                                                                       

Complete!
```
##### Установим percona-release из нашего репозитория
```
root@roman-VirtualBox:~# yum install percona-release -y
...
root@roman-VirtualBox:~# yum list | grep mai
Failed to set locale, defaulting to C.UTF-8
mailx.x86_64                                         12.5-29.el8                                      @BaseOS               
nginx-mod-mail.x86_64                                1:1.14.1-9.module_el8.0.0+184+e34fea82           @AppStream            
percona-release.noarch                               1.0-9                                            @mai                  
fetchmail.x86_64                                     6.3.26-19.el8                                    AppStream             
git-email.noarch                                     2.27.0-1.el8                                     AppStream             
glibc-langpack-mai.x86_64                            2.28-127.el8                                     BaseOS                
google-noto-sans-imperial-aramaic-fonts.noarch       20161022-7.el8.1                                 AppStream             
hunspell-mai.noarch                                  1.0.1-15.el8                                     AppStream             
langpacks-mai.noarch                                 1.0-12.el8                                       AppStream             
libreoffice-emailmerge.x86_64                        1:6.3.6.2-3.el8                                  AppStream             
libreoffice-langpack-mai.x86_64                      1:6.3.6.2-3.el8                                  AppStream             
libreport-plugin-mailx.x86_64                        2.9.5-15.el8                                     AppStream             
mailcap.noarch                                       2.1.48-3.el8                                     BaseOS                
mailman.x86_64                                       3:2.1.29-10.module_el8.3.0+548+3169411d          AppStream             
pcp-pmda-mailq.x86_64                                5.1.1-3.el8                                      AppStream             
pcp-pmda-sendmail.x86_64                             5.1.1-3.el8                                      AppStream             
procmail.x86_64                                      3.22-47.el8                                      AppStream             
sendmail.x86_64                                      8.15.2-32.el8                                    AppStream             
sendmail-cf.noarch                                   8.15.2-32.el8                                    AppStream             
sendmail-doc.noarch                                  8.15.2-32.el8                                    AppStream             
sendmail-milter.i686                                 8.15.2-32.el8                                    AppStream             
sendmail-milter.x86_64                               8.15.2-32.el8                                    AppStream
```
