#devops-netology
### 08-ansible-03-yandex
# Домашнее задание к занятию "08.03 Использование Yandex Cloud"

## Подготовка к выполнению
1. Создайте свой собственный (или используйте старый) публичный репозиторий на github с произвольным именем.
2. Скачайте [playbook](./playbook/) из репозитория с домашним заданием и перенесите его в свой репозиторий.

## Основная часть
1. Допишите playbook: нужно сделать ещё один play, который устанавливает и настраивает kibana.
2. При создании tasks рекомендую использовать модули: `get_url`, `template`, `yum`, `apt`.
3. Tasks должны: скачать нужной версии дистрибутив, выполнить распаковку в выбранную директорию, сгенерировать конфигурацию с параметрами.  
**Выполнение:**  дописываем плей  
```
- name: Install kibana
  hosts: kibana
  handlers:
    - name: restart kibana
      become: true
      service:
        name: kibana
        state: restarted
  tasks:
    - name: "Download Kibana rpm"
      get_url:
        url: "https://artifacts.elastic.co/downloads/kibana/kibana-{{ kibana_version }}-x86_64.rpm"
        dest: "/tmp/kibana-{{ kibana_version }}-x86_64.rpm"
      register: download_kibana
      until: download_kibana is succeeded
    - name: Install Kibana
      become: true
      yum:
        name: "/tmp/kibana-{{ kibana_version }}-x86_64.rpm"
        state: present
    - name: Configure Kibana
      become: true
      template:
        src: kibana.yml.j2
        dest: /etc/kibana/kibana.yml
      notify: restart Kibana
- name: Install filebeat
  hosts: filebeat
  handlers:
    - name: restart filebeat
      become: true
      service:
        name: filebeat
        state: restarted
  tasks:
    - name: "Download Filebeat rpm"
      get_url:
        url: "https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-{{ filebeat_version }}-x86_64.rpm"
        dest: "/tmp/filebeat-{{ filebeat_version }}-x86_64.rpm"
      register: download_filebeat
      until: download_filebeat is succeeded
    - name: Install filebeat
      become: true
      yum:
        name: "/tmp/filebeat-{{ filebeat_version }}-x86_64.rpm"
        state: present
    - name: Configure Filebeat
      become: true
      template:
        src: filebeat.yml.j2
        dest: /etc/filebeat/filebeat.yml
      notify: restart filebeat
    - name: Set filebeat systemwork
      become: true
      command:
        cmd: filebeat modules enable system
        chdir: /usr/share/filebeat/bin
      register: filebeat_modules
      changed_when: filebeat_modules.stdout != 'Module system is already enabled'
    - name: Load Kibana dashboard
      become: true
      command:
        cmd: filebeat setup
        chdir: /usr/share/filebeat/bin
      register: filebeat_setup
      changed_when: false
      until: filebeat_setup is succeeded
```
4. Приготовьте свой собственный inventory файл `prod.yml`.  
**Содержимое файла хостс:**  
```
---
all:
  hosts:
    el-instance:
      ansible_host: <paste_IP>
    kb-instance:
      ansible_host: <paste_IP>
    ap-instance:
      ansible_host: <paste_IP>
  vars:
    ansible_connection: ssh
    ansible_user: dusk
elasticsearch:
  hosts:
    el-instance:
kibana:
  hosts:
    kb-instance:
filebeat:
  hosts:
    ap-instance:
```
5. Запустите `ansible-lint site.yml` и исправьте ошибки, если они есть.  
**Выполнение:**  
```
dusk@DESKTOP-DQUFL9J:~/devops-netology-ansible-3/playbook$ ansible-lint site.yml
[ANSIBLE0002] Trailing whitespace
site.yml:1
---

[ANSIBLE0002] Trailing whitespace
site.yml:2
- name: Install Elasticsearch

[ANSIBLE0002] Trailing whitespace
site.yml:3
  hosts: elasticsearch

[ANSIBLE0002] Trailing whitespace
site.yml:4
...
```
проблема из-за формата тектового файла:
```
dusk@DESKTOP-DQUFL9J:~/devops-netology-ansible-3/playbook$ file site.yml
site.yml: ASCII text, with CRLF line terminators
```

решаем с помощью dos2unix и повторно запускаем ansible-lint:

```
dusk@DESKTOP-DQUFL9J:~/devops-netology-ansible-3/playbook$ dos2unix site.yml
dos2unix: converting file site.yml to Unix format...
dusk@DESKTOP-DQUFL9J:~/devops-netology-ansible-3/playbook$ ansible-lint site.yml
dusk@DESKTOP-DQUFL9J:~/devops-netology-ansible-3/playbook$ 
```
6. Попробуйте запустить playbook на этом окружении с флагом `--check`.
7. Запустите playbook на `prod.yml` окружении с флагом `--diff`. Убедитесь, что изменения на системе произведены.
8. Повторно запустите playbook с флагом `--diff` и убедитесь, что playbook идемпотентен.
9. Проделайте шаги с 1 до 8 для создания ещё одного play, который устанавливает и настраивает filebeat.  
**Выполнение:**  
При запуске с ключом --check плейбук фейлится на моменте установки из дистрибутива:
```
dusk@DESKTOP-DQUFL9J:~/devops-netology-ansible-3/playbook$ ansible-playbook -i inventory/prod/hosts.yml site.yml --check -v
Using /home/dusk/devops-netology-ansible-3/playbook/ansible.cfg as config file

PLAY [Install Elasticsearch] ********************************************************************

TASK [Gathering Facts] **************************************************************************
ok: [el-instance]

TASK [Download Elasticsearch's rpm] *************************************************************
changed: [el-instance] => {"attempts": 1, "changed": true, "dest": "/tmp/elasticsearch-7.14.0-x86_64.rpm", "msg": "OK (344126921 bytes)", "src": "/tmp/tmpYzX_8_", "state": "absent", "url": "https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.14.0-x86_64.rpm"}

TASK [Install Elasticsearch] ********************************************************************
fatal: [el-instance]: FAILED! => {"changed": false, "msg": "No RPM file matching '/tmp/elasticsearch-7.14.0-x86_64.rpm' found on system", "rc": 127, "results": ["No RPM file matching '/tmp/elasticsearch-7.14.0-x86_64.rpm' found on system"]}
        to retry, use: --limit @/home/dusk/devops-netology-ansible-3/playbook/site.retry

PLAY RECAP **************************************************************************************
el-instance                : ok=2    changed=1    unreachable=0    failed=1
```
При запуске без этого ключа таски отработали, в итоге получил:
```
dusk@DESKTOP-DQUFL9J:~/devops-netology-ansible-3/playbook$ ansible-playbook -i inventory/prod/ site.yml --diff

PLAY [Install Elasticsearch] ********************************************************************

TASK [Gathering Facts] **************************************************************************
ok: [el-instance]

TASK [Download Elasticsearch's rpm] *************************************************************
ok: [el-instance]

TASK [Install Elasticsearch] ********************************************************************
ok: [el-instance]

TASK [Configure Elasticsearch] ******************************************************************
ok: [el-instance]

PLAY [Install kibana] ***************************************************************************

TASK [Gathering Facts] **************************************************************************
ok: [kb-instance]

TASK [Download Kibana rpm] **********************************************************************
ok: [kb-instance]

TASK [Install Kibana] ***************************************************************************
ok: [kb-instance]

TASK [Configure Kibana] *************************************************************************
ok: [kb-instance]

PLAY [Install filebeat] *************************************************************************

TASK [Gathering Facts] **************************************************************************
ok: [ap-instance]

TASK [Download Filebeat rpm] ********************************************************************
ok: [ap-instance]

TASK [Install filebeat] *************************************************************************
ok: [ap-instance]

TASK [Configure Filebeat] ***********************************************************************
ok: [ap-instance]

TASK [Set filebeat systemwork] ******************************************************************
ok: [ap-instance]

TASK [Load Kibana dashboard] ********************************************************************
ok: [ap-instance]

PLAY RECAP **************************************************************************************
ap-instance                : ok=6    changed=0    unreachable=0    failed=0   
el-instance                : ok=4    changed=0    unreachable=0    failed=0   
kb-instance                : ok=4    changed=0    unreachable=0    failed=0 
```
10. Подготовьте README.md файл по своему playbook. В нём должно быть описано: что делает playbook, какие у него есть параметры и теги.  
**Подготовил файл такого содержания:**  
```
#### Переменные group_vars
 * elk_stack_version - версия elk
 * filebeat_version - версия filebeat
 * kibana_version - версия kibana
#### Описание Play 
##### Install Elasticsearch
 * установка handler elasticsearch
 * загрузка пакета
 * Установка с помощью yum
 * конфигурирование по шаблону
##### Install kibana
 * установка handler kibana
 * загрузка пакета
 * Установка с помощью yum
 * конфигурирование по шаблону
##### install filebeat
 * установка handler filebeat
 * загрузка пакета
 * Установка с помощью yum
 * конфигурирование по шаблону
 * включение модулей filebeat
 * загрузка kibana dashboard для filebeat
```
11. Готовый playbook выложите в свой репозиторий, в ответ предоставьте ссылку на него.
