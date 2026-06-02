# Домашнее задание 13: Первые шаги с Ansible

## Задания
Подготовить стенд на Vagrant как минимум с одним сервером. На этом сервере, используя Ansible, необходимо развернуть nginx со следующими условиями:<br>

необходимо использовать модуль yum/apt;<br>
конфигурационные файлы должны быть взяты из шаблона jinja2 с переменными;<br>
после установки nginx должен быть в режиме enabled в systemd;<br>
должен быть использован notify для старта nginx после установки;<br>
сайт должен слушать на нестандартном порту — 8080, для этого использовать переменные в Ansible.


## Структура
├── ansible.cfg<br>
├── nginx.yml<br>
├── staging<br>
│   └── hosts<br>
├── templates<br>
│   └── nginx.conf.j2<br>
└── Vagrantfile<br>



## Выполнение

### Установленная версия Ansible
```
dilyam@MacBook-Pro-Dilya-2 Ansible % ansible --version
ansible [core 2.13.13]
  config file = /Users/dilyam/Ansible/ansible.cfg
  configured module search path = ['/Users/dilyam/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /Users/dilyam/.local/pipx/venvs/ansible-core/lib/python3.13/site-packages/ansible
  ansible collection location = /Users/dilyam/.ansible/collections:/usr/share/ansible/collections
  executable location = /Users/dilyam/.local/bin/ansible
  python version = 3.13.3 (main, Apr  8 2025, 13:54:08) [Clang 16.0.0 (clang-1600.0.26.6)]
  jinja version = 3.1.6
  libyaml = True

```

### Подняла виртуальный сервер на Vagrant
```
dilyam@MacBook-Pro-Dilya-2 Ansible % vagrant ssh-config
Host nginx
  HostName 127.0.0.1
  User vagrant
  Port 2222
  UserKnownHostsFile /dev/null
  StrictHostKeyChecking no
  PasswordAuthentication no
  IdentityFile /Users/dilyam/.vagrant.d/insecure_private_keys/vagrant.key.ed25519
  IdentityFile /Users/dilyam/.vagrant.d/insecure_private_keys/vagrant.key.rsa
  IdentitiesOnly yes
  LogLevel FATAL
  PubkeyAcceptedKeyTypes +ssh-rsa
  HostKeyAlgorithms +ssh-rsa

```

### Создала inventory файл и файл ansible.cfg, проверила подключение
```
dilyam@MacBook-Pro-Dilya-2 Ansible % cat ./staging/hosts
[web]
nginx ansible_host=127.0.0.1 ansible_port=2222 ansible_private_key_file=.vagrant/machines/nginx/virtualbox/private_key


dilyam@MacBook-Pro-Dilya-2 Ansible % cat ansible.cfg
[defaults]
inventory = staging/hosts
remote_user = vagrant
host_key_checking = False
retry_files_enabled = False


dilyam@MacBook-Pro-Dilya-2 Ansible % ansible nginx -i staging/hosts -m ping
nginx | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}

```

### Создала сам плейбук, файл шаблона и накатила плейбук
```
dilyam@MacBook-Pro-Dilya-2 Ansible % cat  nginx.yml
- name: NGINX | Install and configure NGINX
  hosts: nginx
  become: true
  vars:
    nginx_listen_port: 8080

  tasks:
    - name: update
      apt:
        update_cache=yes
      tags:
        - update apt

    - name: NGINX | Install NGINX
      apt:
        name: nginx
        state: latest
      notify:
        - restart nginx
      tags:
        - nginx-package

    - name: NGINX | Create NGINX config file from template
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify:
        - reload nginx      
      tags:
        - nginx-configuration

  handlers:
    - name: restart nginx
      systemd:
        name: nginx
        state: restarted
        enabled: yes

    - name: reload nginx
      systemd:
        name: nginx
        state: reloaded 


dilyam@MacBook-Pro-Dilya-2 Ansible % cat templates/nginx.conf.j2
# {{ ansible_managed }}
events {
    worker_connections 1024;
}

http {
    server {
        listen       {{ nginx_listen_port }} default_server;
        server_name  default_server;
        root         /usr/share/nginx/html;

        location / {
        }
    }
}


dilyam@MacBook-Pro-Dilya-2 Ansible % ansible-playbook nginx.yml

PLAY [NGINX | Install and configure NGINX] ************************************************************************************************************************************************

TASK [Gathering Facts] ********************************************************************************************************************************************************************
ok: [nginx]

TASK [update] *****************************************************************************************************************************************************************************
ok: [nginx]

TASK [NGINX | Install NGINX] **************************************************************************************************************************************************************
changed: [nginx]

TASK [NGINX | Create NGINX config file from template] *************************************************************************************************************************************
changed: [nginx]

RUNNING HANDLER [restart nginx] ***********************************************************************************************************************************************************
changed: [nginx]

RUNNING HANDLER [reload nginx] ************************************************************************************************************************************************************
changed: [nginx]

PLAY RECAP ********************************************************************************************************************************************************************************
nginx                      : ok=6    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   



```



### Проверила работу 
```
dilyam@MacBook-Pro-Dilya-2 Ansible % curl http://192.168.11.150:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

```
