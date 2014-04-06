---
layout: post
title: "Production сервер для Ruby on Rails"
date: 2013-01-26 05:31
comments: true
categories: 
---

![Production сервер для Ruby on Rails](/assets/img/ror-production/rubyubuntu.jpg)  

На моем VPS стоит система Ubuntu 12.04.1 32 bit. Ее и будем использовать для создания нашего собственного сервера для развертывания Ruby on Rails приложений =)  

Сразу оговорюсь, целью является именно запуск Rails приложения на своем сервере. Перекраивать конфиги с целью достижения максимальной производительности я пока не стал, так что всё будет работать в дефолтном режиме.

<!-- more -->    

Итак, в начале ничего кроме виртуального сервера с 256 Mb оперативной памяти у нас нет.  

Начнем, пожалуй! =)  


Нужно выбрать веб-сервер, который мы будем использовать. Для целей разработки в Rails есть встроенный Webrick, который запускает написанное приложение на http://localhost:3000  

Но это же несерьезно, использовать его на production сервере =)  

В итоге я выбрал связку Nginx и Passenger. Можно спросить, почему? Просто она дает бОльшую производительность (в настройках по default), чем Apache + Passenger и менее сложна в настройке, чем Nginx + Unicorn. Во всяком случае, мне так кажется =)  

Доступ по SSH. Консоль, откройся!  

{% codeblock%}
apt-get update
apt-get upgrade
{% endcodeblock %}

Только что мы обновили все пакеты в системе. Теперь остановим и удалим Apache (если он есть), раз уж ставим вместо него Nginx.  

{% codeblock%}
/etc/init.d/apache2 stop && apt-get purge apache2*
{% endcodeblock %}

Теперь займемся установкой rvm (контроль версий Ruby)  

{% codeblock%}
apt-get install curl
\curl -L https://get.rvm.io | bash -s stable --rails
{% endcodeblock %}

В конце установки rvm даст правильную команду для своего запуска. У меня это было:  

{% codeblock%}
source /usr/local/rvm/scripts/rvm
{% endcodeblock %}

Теперь поставим зависимости rvm. Команда  

{% codeblock%}
rvm requirements
{% endcodeblock %}

Подскажет нам, что же именно еще нужно установить. Это длинный список разных пакетов.  

{% codeblock%}
/usr/bin/apt-get install build-essential openssl libreadline6 libreadline6-dev curl git-core zlib1g zlib1g-dev libssl-dev libyaml-dev libsqlite3-dev sqlite3 libxml2-dev libxslt-dev autoconf libc6-dev ncurses-dev automake libtool bison subversion pkg-config
{% endcodeblock %}

Запускаем и ждем, покуда все установится =)  

Установим Ruby (версия у Вас может быть и новее) и укажет, какую версию мы будем использовать в системе по умолчанию.  

{% codeblock%}
rvm install 1.9.3
rvm use 1.9.3 --default
{% endcodeblock %}

Далее установим RubyGems  

{% codeblock%}
rvm rubygems current
{% endcodeblock %}

и рельсы  

{% codeblock%}
gem install rails
{% endcodeblock %}

Теперь займемся, собственно, сервером.  
Ставим Passenger  

{% codeblock%}
gem install passenger
{% endcodeblock %}

В качестве модуля для Passenger устанавливаем Nginx.

{% codeblock%}
rvmsudo passenger-install-nginx-module
{% endcodeblock %}

Инсталлер nginx скажет нам, если ему чего-нибудь не хватает. Мне не хватало одного пакета:  

{% codeblock%}
apt-get install libcurl4-openssl-dev
{% endcodeblock %}

Если все зависимости установлены, в меню установки nginx выбираем «1″ для быстрой установки.  
Примечание: init.d (файл для запуска) для nginx при такой установке не создается, так что тырим его с linode.com =)  

{% codeblock%}
wget -O init-deb.sh http://library.linode.com/assets/660-init-deb.sh
sudo mv init-deb.sh /etc/init.d/nginx
sudo chmod +x /etc/init.d/nginx
sudo /usr/sbin/update-rc.d -f nginx defaults
{% endcodeblock %}

Далее стартуем nginx.  

{% codeblock%}
sudo service nginx start
{% endcodeblock %}

Теперь зайдя на адрес нашего vps мы можем увидеть стандартную заглушку nginx. Это значит, что все идет хорошо =)  
Лишним не будет установить node.js  

{% codeblock%}
sudo apt-get install nodejs
{% endcodeblock %}

Теперь настроим nginx. Для этого нужно подредактировать его файл nginx.conf (у меня он был в /opt/nginx/conf/nginx.conf). Его расположение зависит, конечно, от директории, в которую nginx установился.  

{% codeblock%}
nano /opt/nginx/conf/nginx.conf
{% endcodeblock %}

Найдите в файле участок «server { тут настройки по-умолчанию }». Придадим ему следующий вид:  

{% codeblock%}
server {
        listen      80;
        server_name имя_вашего_сервера;
        rails_env production;
        passenger_use_global_queue on;
        root /var/www/название_вашего_приложения/public;
        passenger_enabled on;
        error_page  404              /404.html;
        error_page   500 502 503 504  /50x.html;
    }
{% endcodeblock %}

В строке root должен быть путь к папке public вашего приложения.  
После правки не забываем перезапустить сервер nginx.  

{% codeblock%}
service nginx restart
{% endcodeblock %}

И не забывайте, что теперь мы работаем не с development базой данных, а с production.  
Возможно, еще придется сделать:  

{% codeblock%}
rake assets:precompile
{% endcodeblock %}

Пока я не ставил Capistrano для нормального деплоя приложений…  
Если будут вопросы — задавайте их в комментариях =)  