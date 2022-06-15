# Домашнее задание к занятию "08.02 Работа с Playbook"

## Основная часть

**1. Приготовьте свой собственный inventory файл `prod.yml`.**
```
dorlov@docker:~/Playbook/playbook$ cat inventory/prod.yml
---
clickhouse:
  hosts:
    clickhouse-01:
      ansible_host: 192.168.43.29

vector:
  hosts:
    vector-01:
      ansible_host: 192.168.43.24
```

**2. Допишите playbook: нужно сделать ещё один play, который устанавливает и настраивает [vector](https://vector.dev).**
```
dorlov@docker:~/AnsiBook/playbook$ cat site.yml
---
- name: Install Clickhouse
  hosts: clickhouse
  handlers:
    - name: Start clickhouse service
      become: true
      ansible.builtin.systemd:
        name: clickhouse-server
        enabled: yes
        masked: no
        state: started
  tasks:
    - block:
        - name: Clickhouse - Get distrib
          ansible.builtin.get_url:
            url: "https://packages.clickhouse.com/rpm/stable/{{ item }}-{{ clickhouse_version }}.noarch.rpm"
            dest: "./{{ item }}-{{ clickhouse_version }}.rpm"
          with_items: "{{ clickhouse_packages }}"
      rescue:
        - name: Clickhouse - Get distrib
          ansible.builtin.get_url:
            url: "https://packages.clickhouse.com/rpm/stable/clickhouse-common-static-{{ clickhouse_version }}.x86_64.rpm"
            dest: "./clickhouse-common-static-{{ clickhouse_version }}.rpm"
      tags: clickhouse
    - name: Clickhouse - Install packages
      become: true
      ansible.builtin.yum:
        name:
          - clickhouse-common-static-{{ clickhouse_version }}.rpm
          - clickhouse-client-{{ clickhouse_version }}.rpm
          - clickhouse-server-{{ clickhouse_version }}.rpm
      notify: Start clickhouse service
      tags: clickhouse

    - name: Clickhouse - Flush handlers
      ansible.builtin.meta: flush_handlers
      tags: clickhouse

    - name: Clickhouse - Create database
      ansible.builtin.command: "clickhouse-client -q 'create database logs;'"
      register: create_db
      failed_when: create_db.rc != 0 and create_db.rc !=82
      changed_when: create_db.rc == 0
      tags: clickhouse

- name: Install Vector
  hosts: vector
  handlers:
    - name: Start vector service
      become: true
      ansible.builtin.systemd:
        name: vector
        enabled: yes
        masked: no
        state: started
  tasks:
    - name: Vector - Download repo
      ansible.builtin.get_url:
        url: https://repositories.timber.io/public/vector/cfg/setup/bash.rpm.sh
        dest: /tmp/bash.rpm.sh
        mode: '0755'
      tags: vector

    - name: Vector - Install repo
      become: true
      ansible.builtin.command: "/tmp/bash.rpm.sh"
      changed_when: false
      tags: vector

    - name: Vector - Install package
      become: true
      ansible.builtin.yum:
        name: vector
        state: latest
      notify: Start vector service
      tags: vector
```

**3. При создании tasks рекомендую использовать модули: `get_url`, `template`, `unarchive`, `file`.**
**4. Tasks должны: скачать нужной версии дистрибутив, выполнить распаковку в выбранную директорию, установить vector.**

**3** и **4**.
Использолвал только *get_url*
Нашел инструкцию, где происходит скачивание репы и установка пакета с использованием yum: https://vector.dev/docs/setup/installation/package-managers/yum/

**5. Запустите `ansible-lint site.yml` и исправьте ошибки, если они есть.**
```
dorlov@docker:~/Playbook/playbook$ ansible-lint site.yml
dorlov@docker:~/Playbook/playbook$
```

**6. Попробуйте запустить playbook на этом окружении с флагом `--check`.**
```
dorlov@docker:~/AnsiBook/playbook$ ansible-playbook -i inventory/prod.yml site.yml --check -K
BECOME password:

PLAY [Install Clickhouse] ************************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************
ok: [clickhouse-01]

TASK [Clickhouse - Get distrib] ******************************************************************************************************************************
changed: [clickhouse-01] => (item=clickhouse-client)
changed: [clickhouse-01] => (item=clickhouse-server)
failed: [clickhouse-01] (item=clickhouse-common-static) => {"ansible_loop_var": "item", "changed": false, "dest": "./clickhouse-common-static-22.3.3.44.rpm", "elapsed": 0, "item": "clickhouse-common-static", "msg": "Request failed", "response": "HTTP Error 404: Not Found", "status_code": 404, "url": "https://packages.clickhouse.com/rpm/stable/clickhouse-common-static-22.3.3.44.noarch.rpm"}

TASK [Clickhouse - Get distrib] ******************************************************************************************************************************
changed: [clickhouse-01]

TASK [Clickhouse - Install packages] *************************************************************************************************************************
fatal: [clickhouse-01]: FAILED! => {"changed": false, "msg": "No RPM file matching 'clickhouse-common-static-22.3.3.44.rpm' found on system", "rc": 127, "results": ["No RPM file matching 'clickhouse-common-static-22.3.3.44.rpm' found on system"]}

PLAY RECAP ***************************************************************************************************************************************************
clickhouse-01              : ok=2    changed=1    unreachable=0    failed=1    skipped=0    rescued=1    ignored=0
```

**7. Запустите playbook на `prod.yml` окружении с флагом `--diff`. Убедитесь, что изменения на системе произведены.**
```
dorlov@docker:~/AnsiBook/playbook$ ansible-playbook -i inventory/prod.yml site.yml --diff -K
BECOME password:

PLAY [Install Clickhouse] ************************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************
ok: [clickhouse-01]

TASK [Clickhouse - Get distrib] ******************************************************************************************************************************
changed: [clickhouse-01] => (item=clickhouse-client)
changed: [clickhouse-01] => (item=clickhouse-server)
failed: [clickhouse-01] (item=clickhouse-common-static) => {"ansible_loop_var": "item", "changed": false, "dest": "./clickhouse-common-static-22.3.3.44.rpm", "elapsed": 0, "item": "clickhouse-common-static", "msg": "Request failed", "response": "HTTP Error 404: Not Found", "status_code": 404, "url": "https://packages.clickhouse.com/rpm/stable/clickhouse-common-static-22.3.3.44.noarch.rpm"}

TASK [Clickhouse - Get distrib] ******************************************************************************************************************************
changed: [clickhouse-01]

TASK [Clickhouse - Install packages] *************************************************************************************************************************
changed: [clickhouse-01]

RUNNING HANDLER [Start clickhouse service] *******************************************************************************************************************
ok: [clickhouse-01]

TASK [Clickhouse - Create database] **************************************************************************************************************************
ok: [clickhouse-01]

PLAY [Install Vector] ****************************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************
ok: [vector-01]

TASK [Vector - Download repo] ********************************************************************************************************************************
changed: [vector-01]

TASK [Vector - Install repo] *********************************************************************************************************************************
ok: [vector-01]

TASK [Vector - Install package] ******************************************************************************************************************************
changed: [vector-01]

RUNNING HANDLER [Start vector service] ***********************************************************************************************************************
ok: [vector-01]

PLAY RECAP ***************************************************************************************************************************************************
clickhouse-01              : ok=5    changed=2    unreachable=0    failed=0    skipped=0    rescued=1    ignored=0
vector-01                  : ok=5    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

**8. Повторно запустите playbook с флагом `--diff` и убедитесь, что playbook идемпотентен.**
```
dorlov@docker:~/AnsiBook/playbook$ ansible-playbook -i inventory/prod.yml site.yml --diff -K
BECOME password:

PLAY [Install Clickhouse] ************************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************
ok: [clickhouse-01]

TASK [Clickhouse - Get distrib] ******************************************************************************************************************************
ok: [clickhouse-01] => (item=clickhouse-client)
ok: [clickhouse-01] => (item=clickhouse-server)
failed: [clickhouse-01] (item=clickhouse-common-static) => {"ansible_loop_var": "item", "changed": false, "dest": "./clickhouse-common-static-22.3.3.44.rpm", "elapsed": 0, "gid": 1001, "group": "dorlov", "item": "clickhouse-common-static", "mode": "0664", "msg": "Request failed", "owner": "dorlov", "response": "HTTP Error 404: Not Found", "size": 246310036, "state": "file", "status_code": 404, "uid": 1001, "url": "https://packages.clickhouse.com/rpm/stable/clickhouse-common-static-22.3.3.44.noarch.rpm"}

TASK [Clickhouse - Get distrib] ******************************************************************************************************************************
ok: [clickhouse-01]

TASK [Clickhouse - Install packages] *************************************************************************************************************************
ok: [clickhouse-01]

TASK [Clickhouse - Create database] **************************************************************************************************************************
ok: [clickhouse-01]

PLAY [Install Vector] ****************************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************
ok: [vector-01]

TASK [Vector - Download repo] ********************************************************************************************************************************
ok: [vector-01]

TASK [Vector - Install repo] *********************************************************************************************************************************
ok: [vector-01]

TASK [Vector - Install package] ******************************************************************************************************************************
ok: [vector-01]

PLAY RECAP ***************************************************************************************************************************************************
clickhouse-01              : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=1    ignored=0
vector-01                  : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```
**9. Подготовьте README.md файл по своему playbook. В нём должно быть описано: что делает playbook, какие у него есть параметры и теги.**

[README](https://github.com/Borodatko/AnsiBook/blob/0bfe15d51fa9b5416c5f015b32c6779aa6be7423/playbook/README.md)
