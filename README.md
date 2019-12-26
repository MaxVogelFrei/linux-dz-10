# Домашнее задание 10

[playbook](playbooks/nginx.yml)
[inventory](inventories/hosts.yml)
[vars](roles/nginx/defaults/main.yml)
[handlers](roles/nginx/handlers/main.yml)
[tasks](roles/nginx/tasks/main.yml)
[Vagrantfile](Vagrantfile)

## Ansible

Подготовить стенд на Vagrant как минимум с одним сервером. На этом сервере используя Ansible необходимо развернуть nginx со следующими условиями:

* необходимо использовать модуль yum/apt
* конфигурационные файлы должны быть взяты из шаблона jinja2 с перемененными
* после установки nginx должен быть в режиме enabled в systemd
* должен быть использован notify для старта nginx после установки
* сайт должен слушать на нестандартном порту - 8080, для этого использовать переменные в Ansible
* Сделать все это с использованием Ansible роли

## Процесс решения

### начинаю с описания хоста в yaml формате в inventories/hosts.yml  

```bash
[root@centos7 inventories]# cat hosts.yml
```
```yaml
all:
  hosts:
    nginx:
      ansible_host: 127.0.0.1
      ansible_port: 2200
      ansible_private_key_file: .vagrant/machines/nginx/virtualbox/private_key
```

### в ansible.cfg указываю inventory, и путь к роли  
```bash
[root@centos7 linux-dz-10]# cat ansible.cfg
[defaults]
inventory = inventories/hosts.yml
remote_user = vagrant
host_key_checking = False
retry_files_enabled = False
roles_path=./roles
```
### Создание роли nginx

Создаю директории defaults  handlers  tasks  templates в roles/nginx/

#### defaults - создаю переменные для имени репозитория epel, порта nginx, прав на конфиг nginx и index.html, пути к index.html
```bash
[root@centos7 defaults]# cat main.yml
```
```yaml
nginx_listen_port: 8080
epel: epel-release
html: /usr/share/nginx/html/index.html
nginx_conf_mode: '0750'
```

#### handlers
* рестарт nginx + состояние enabled
* перечитывание конфига 
```bash
[root@centos7 handlers]# cat main.yml
```
```yaml
---
- name: restart_nginx
  systemd:
    name: nginx
    state: restarted
    enabled: yes

- name: reload_nginx
  systemd:
    name: nginx
    state: reloaded
```

#### tasks
* установка репозитория
* nginx с notify для его рестарта
* замена конфига из templates с notify для его перечитывания
* замена дефолтной страницы index.html
```bash
[root@centos7 tasks]# cat main.yml
```
```yaml
---
- name: Install EPEL Repo package from standard repo
  yum:
    name: "{{ epel }}"
    state: present
  tags:
    - epel-package
    - packages

- name: Install nginx package from epel repo
  yum:
    name: nginx
    state: present
  notify:
    - restart_nginx
  tags:
    - nginx-package
    - packages

- name: NGINX | Create NGINX config file from template
  template:
    src: templates/nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: "{{ nginx_conf_mode }}"
  notify:
    - reload_nginx
  tags:
    - nginx-configuration

- name: NGINX | Create NGINX html file from template
  template:
    src: templates/index.html.j2
    dest: "{{ html }}"
    owner: nginx
    group: nginx
    mode: "{{ nginx_conf_mode }}"
```


#### templates
* index.html используя переменную ansible_hostname для подстановки имени машины
* nginx.conf с другим портом из переменной nginx_listen_port

```bash
[root@centos7 templates]# cat index.html.j2
# {{ ansible_managed }}
<p>Hello from {{ ansible_hostname }}</p>
```
```bash
[root@centos7 templates]# cat nginx.conf.j2
events {
 worker_connections 1024;
}
http {
 server {
  listen {{ nginx_listen_port }} default_server;
  server_name default_server;
  root /usr/share/nginx/html;
  location / {
  }
 }
}
```
