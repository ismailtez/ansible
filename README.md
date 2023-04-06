 Задача: попрактиковаться в написании плейбука, со следующими условиями: -Подготовить стенд на vagrant; -настроить ansible для доуступа у стенду -написать плейбук, который устанавливает nginx в конфигурации по умолчанию, с применением модуля yum -добавить в плейбук копирование новой конфигурации сервиса на стенд -должен быть использован мехенизм notify для рестарта nginx после установки или изменения конфигурации
                                                Установка ansible
Сначала нам нужно проверить версию питона и версию ansible, команада: python -V и ansible --version
Подготовка стенда

Создать каталог ansible нам нужно поднять присланный нам Vagrantfile, с помощью команды vagrant up
Для подключения к хосту nginx нам необходимо передать множество параметров. Узнаем эти параметры с помощью vagrant ssh-config.
Inventory

Чтобы создать свой перый inventory файл, нам нужно прописать команду nano inventory, далее, внутри этого файла нам надо написать некие параметры:

[webservers] nginx ansible_host=127.0.0.1 ansible_port=2222 ansible_user=vagrant ansible_private_key_file=/home/alex/os_labs2/lab1/.vagrant/machines/nginx/virtualbox/private_key

У каждого пользователя параметры индивидуальны, мы указываем свой порт, свой путь к приватному ключу ssh и тд. Все это мы узнали после команды vagrant ssh-config.

Далее, с помощью команды cat inventory выводим записанный нами в файле скрипт.

И наконец, убеждаемся в том, что Ansible может управлять нашим хостом. Делаем это с помощью команды: ansible nginx -i inventory -m ping, результат должен быть такой:

nginx | SUCCESS => { "ansible_facts": { "discovered_interpreter_python": "/usr/bin/python" }, "changed": false, "ping": "pong" } Если результат не SUCCESS, значит ошибка в inventory.
ansible.cfg

Чтобы нам каждый раз указывать наш инвентори файл, создадим файл конфигурации ansible.cfg. Все так же, как и в прошлом пункте, создаем, записываем туда скрипт:

[defaults] inventory = inventory remote_user= vagrant host_key_checking = False transport=smart

Потом, снова убеждаемся, что управляемый хост доступен, только теперь без явного указания инвентори файла:

ansible -m ping nginx, получаем ответ:

nginx | SUCCESS => { "ansible_facts": { "discovered_interpreter_python": "/usr/bin/python" }, "changed": false, "ping":"pong" }
Playbook epel

Напишем простой Playbook, который будет выполнять установку пакета epel-release. Создаем файл epel.yml со следующим содержимым:

    name: Install EPEL Repo hosts: webservers become: true tasks:
    name: Install EPEL Repo package from standard repo yum: name: epel-release state: present

После чего запустим выполнение Playbook, с помощью команды ansible-playbook epel.yml, должны получить:

`PLAY [Install EPEL Repo]

TASK [Gathering Facts]

ok: [nginx] TASK [Install EPEL Repo package from standard repo]

ok: [nginx] PLAY RECAP

nginx : ok=2 changed=0 unreachable=0 failed=0 skipped= 0 rescued=0 ignored=0`
Playbook nginx

   Повторяем как в пунктах выше
                                                  Шаблон

cat nginx.conf.j2

events { worker_connections 1024; } http { server { listen {{ nginx_listen_port }} default_server; server_name default_server; root /usr/share/nginx/html; location / { } } }
Handlers
Плейбук сейчас уже почти работоспособен, осталось только добавить секции handler и notify для рестарта nginx не при любом старте плейбука, а только при изменения в конфигурации. Для рестарта сервисов применяется модуль systemd, документация к модулю https://docs.ansible.com/ansible/latest/collections/ansible/builtin/systemd_module.html [user@fedora ansible]$ cat nginx.yml
name: Install nginx package from epel repo hosts: webservers become: true vars: nginx_listen_port: 8080 tasks:
    name: Install EPEL Repo package from standard repo yum: name: epel-release state: present
    name: install nginx from repo yum: name: nginx state: latest notify:
    restart nginx tags:
    nginx-package
    packages
    name: Create config file from template template: src: nginx.conf.j2 dest: /etc/nginx/nginx.conf notify:
    reload nginx tags:
    nginx-configuration handlers:
    name: restart nginx systemd: name: nginx state: reconfigured enabled: yes

Проверка

Теперь проверяем на работу Nginx c помощью команды:

vagrant ssh -c "ip addr show"

после выводв этой команды, нам нужно найти там публичный ip-адресс

И наконец:

Мы проверяем отвечает ip или нет, с помощью команды:

Если вы все сделали верно, то должно вывести веб-страничку
