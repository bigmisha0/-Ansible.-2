# Занятие: Ansible — плейбуки, переменные, роли и handlers

**Фамилия Имя:** Махиня Михаил

---

## Структура решения

```
.
├── inventory.ini
├── task1/
│   ├── 01-download-extract.yml
│   ├── 02-tuned.yml
│   └── 03-motd.yml
├── task2/
│   └── motd-personalized.yml
└── task3/
    ├── site.yml
    └── roles/
        └── apache_info/
            ├── defaults/main.yml
            ├── handlers/main.yml
            ├── tasks/main.yml
            └── templates/index.html.j2
```


---

## Задание 1

### 1.1 Скачивание и распаковка архива

Файл: [`task1/01-download-extract.yml`](task1/01-download-extract.yml)

Плейбук:
1. создаёт каталог `/opt/kafka`;
2. скачивает архив Apache Kafka с официального зеркала через модуль `get_url`;
3. распаковывает архив в созданный каталог через модуль `unarchive`.

```yaml
---
- name: Download and extract Kafka archive
  hosts: all
  become: true

  vars:
    kafka_url: "https://downloads.apache.org/kafka/3.7.0/kafka_2.13-3.7.0.tgz"
    kafka_archive_path: "/tmp/kafka_2.13-3.7.0.tgz"
    kafka_extract_dir: "/opt/kafka"

  tasks:
    - name: Create directory for extraction
      ansible.builtin.file:
        path: "{{ kafka_extract_dir }}"
        state: directory
        owner: root
        group: root
        mode: '0755'

    - name: Download Kafka archive
      ansible.builtin.get_url:
        url: "{{ kafka_url }}"
        dest: "{{ kafka_archive_path }}"
        mode: '0644'

    - name: Extract Kafka archive into target directory
      ansible.builtin.unarchive:
        src: "{{ kafka_archive_path }}"
        dest: "{{ kafka_extract_dir }}"
        remote_src: true
        extra_opts: [--strip-components=1]
```

Запуск:

```bash
ansible-playbook -i inventory.ini task1/01-download-extract.yml
```

Вывод выполнения:

```
PLAY [Download and extract Kafka archive] ********************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************************
ok: [Ubuntu]

TASK [Create directory for extraction] ***********************************************************************************************************************************
ok: [Ubuntu]

TASK [Download Kafka archive] ********************************************************************************************************************************************
changed: [Ubuntu]

TASK [Extract Kafka archive into target directory] ***********************************************************************************************************************
ok: [Ubuntu]

PLAY RECAP ***************************************************************************************************************************************************************
Ubuntu                     : ok=4    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0  
```

---

### 1.2 Установка и автозапуск tuned

Файл: [`task1/02-tuned.yml`](task1/02-tuned.yml)

Плейбук устанавливает пакет `tuned` из стандартного репозитория,
запускает соответствующий сервис systemd (конфигурационный файл
появляется автоматически при установке пакета) и добавляет его
в автозагрузку.

```yaml
---
- name: Install and enable tuned service
  hosts: all
  become: true

  tasks:
    - name: Install tuned package
      ansible.builtin.package:
        name: tuned
        state: present

    - name: Start tuned service and enable it on boot
      ansible.builtin.systemd:
        name: tuned
        state: started
        enabled: true
```

Запуск:

```bash
ansible-playbook -i inventory.ini task1/02-tuned.yml
```

Вывод выполнения:

```
PLAY [Install and enable tuned service] **********************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************************
ok: [Ubuntu]

TASK [Install tuned package] *********************************************************************************************************************************************
changed: [Ubuntu]

TASK [Start tuned service and enable it on boot] *************************************************************************************************************************
ok: [Ubuntu]

PLAY RECAP ***************************************************************************************************************************************************************
Ubuntu                     : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0  
```

Проверка на хосте:

```bash
systemctl status tuned
```

```
● tuned.service - Dynamic System Tuning Daemon
     Loaded: loaded (/usr/lib/systemd/system/tuned.service; enabled; preset: enabled)
     Active: active (running) since Tue 2026-06-16 19:24:31 UTC; 21min ago
 Invocation: 560b2591a3484ee49985a6e85d668485
       Docs: man:tuned(8)
             man:tuned.conf(5)
             man:tuned-adm(8)
   Main PID: 5549 (tuned)
      Tasks: 4 (limit: 3943)
     Memory: 16.8M (peak: 20.3M)
        CPU: 534ms
     CGroup: /system.slice/tuned.service
             └─5549 /usr/bin/python3 /usr/sbin/tuned -l -P

```

---

### 1.3 Изменение приветствия системы (motd)

Файл: [`task1/03-motd.yml`](task1/03-motd.yml)

Текст приветствия задаётся переменной `motd_message` и записывается
в `/etc/motd` модулем `copy`.

```yaml
---
- name: Set custom MOTD message
  hosts: all
  become: true

  vars:
    motd_message: |
      ============================================
       Welcome to the managed server!
       Unauthorized access is strictly prohibited.
      ============================================

  tasks:
    - name: Write custom message to /etc/motd
      ansible.builtin.copy:
        content: "{{ motd_message }}"
        dest: /etc/motd
        owner: root
        group: root
        mode: '0644'
```

Запуск:

```bash
ansible-playbook -i inventory.ini task1/03-motd.yml
```

Вывод выполнения:

```
PLAY [Set custom MOTD message] *******************************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************************
ok: [Ubuntu]

TASK [Write custom message to /etc/motd] *********************************************************************************************************************************
changed: [Ubuntu]

PLAY RECAP ***************************************************************************************************************************************************************
Ubuntu                     : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0 
```

Проверка (новое подключение по SSH к хосту покажет содержимое `/etc/motd`):

```
 Hostname:   Ubuntu
 IP address: 10.0.2.15

 Have a great day, system administrator!

```

---

## Задание 2

Файл: [`task2/motd-personalized.yml`](task2/motd-personalized.yml)

Это модифицированная версия плейбука из п. 1.3. Теперь приветствие
формируется динамически с помощью Ansible facts, которые Ansible
собирает автоматически перед выполнением задач:

- `ansible_hostname` — имя хоста;
- `ansible_default_ipv4.address` — IP-адрес хоста;
- плюс статичный текст с пожеланием хорошего дня администратору.

```yaml
---
- name: Set personalized MOTD with host facts
  hosts: all
  become: true

  vars:
    motd_message: |
      ============================================
       Hostname:   {{ ansible_hostname }}
       IP address: {{ ansible_default_ipv4.address }}

       Have a great day, system administrator!
      ============================================

  tasks:
    - name: Write personalized message to /etc/motd
      ansible.builtin.copy:
        content: "{{ motd_message }}"
        dest: /etc/motd
        owner: root
        group: root
        mode: '0644'
```

Запуск:

```bash
ansible-playbook -i inventory.ini task2/motd-personalized.yml
```

Вывод выполнения:

```
PLAY [Set personalized MOTD with host facts] *****************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************************
ok: [Ubuntu]

TASK [Write personalized message to /etc/motd] ***************************************************************************************************************************
changed: [Ubuntu]

PLAY RECAP ***************************************************************************************************************************************************************
Ubuntu                     : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

Результат (`cat /etc/motd` на управляемом хосте должен показать
реальный hostname и IP этого хоста):

```
 Hostname:   Ubuntu
 IP address: 10.0.2.15

 Have a great day, system administrator!

```

---

## Задание 3

Архив с ролью: [`task3/roles/apache_info`](task3/roles/apache_info)
Главный плейбук: [`task3/site.yml`](task3/site.yml)

В соответствии со статьёй [«Ansible — это вам не bash»](https://habr.com/ru/post/494738/)
все шаги реализованы декларативно через модули `package`, `template`,
`systemd`, `firewalld`/`ufw` и `uri` — без `shell`/`command`.

Главный плейбук подключает одну роль `apache_info`:

```yaml
---
- name: Configure Apache web server with system info page
  hosts: all
  become: true
  roles:
    - apache_info
```

### Структура роли

```
roles/apache_info/
├── defaults/main.yml      # имена пакета/сервиса в зависимости от ОС
├── handlers/main.yml      # перезапуск Apache при изменении конфигурации
├── tasks/main.yml         # основная логика роли
└── templates/index.html.j2  # шаблон главной страницы с фактами
```

### defaults/main.yml

```yaml
---
apache_package: >-
  {{ 'httpd' if ansible_facts['os_family'] == 'RedHat' else 'apache2' }}

apache_service: >-
  {{ 'httpd' if ansible_facts['os_family'] == 'RedHat' else 'apache2' }}

apache_doc_root: /var/www/html
```

### tasks/main.yml

1. **Установка Apache** — через модуль `package` (имя пакета
   определяется автоматически по `ansible_facts['os_family']`:
   `httpd` для RedHat-подобных, `apache2` для Debian-подобных).
2. **Развёртывание index.html** — модуль `template` рендерит
   `index.html.j2` с использованием Ansible facts (CPU, RAM, диск, IP)
   и помещает результат в корень документов Apache. При изменении
   файла срабатывает `notify: Restart Apache`.
3. **Открытие порта 80** — через `firewalld` (RedHat) или `ufw` (Debian),
   в зависимости от семейства ОС.
4. **Запуск и автозагрузка** — модуль `systemd` запускает сервис
   Apache и добавляет его в автозагрузку.
5. **Проверка доступности** — модуль `uri` отправляет запрос на
   `http://localhost/` и убеждается, что код ответа равен `200`.

```yaml
---
- name: Install Apache web server
  ansible.builtin.package:
    name: "{{ apache_package }}"
    state: present

- name: Deploy index.html with system information
  ansible.builtin.template:
    src: index.html.j2
    dest: "{{ apache_doc_root }}/index.html"
    owner: root
    group: root
    mode: '0644'
  notify: Restart Apache

- name: Ensure firewalld allows HTTP (RedHat family)
  ansible.posix.firewalld:
    service: http
    permanent: true
    immediate: true
    state: enabled
  when: ansible_facts['os_family'] == "RedHat"

- name: Ensure ufw allows HTTP (Debian family)
  community.general.ufw:
    rule: allow
    port: '80'
    proto: tcp
  when: ansible_facts['os_family'] == "Debian"

- name: Start and enable Apache service
  ansible.builtin.systemd:
    name: "{{ apache_service }}"
    state: started
    enabled: true

- name: Verify that the website responds with HTTP 200
  ansible.builtin.uri:
    url: "http://localhost/"
    status_code: 200
  register: apache_check

- name: Show website check result
  ansible.builtin.debug:
    msg: "Website check status: {{ apache_check.status }}"
```

### handlers/main.yml

```yaml
---
- name: Restart Apache
  ansible.builtin.systemd:
    name: "{{ apache_service }}"
    state: restarted
```

### templates/index.html.j2

Шаблон использует Ansible facts:
`ansible_processor_vcpus`, `ansible_processor`, `ansible_memtotal_mb`,
`ansible_devices` и `ansible_default_ipv4.address`.

```jinja
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>System info - {{ ansible_hostname }}</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; }
        h1 { color: #2c3e50; }
        table { border-collapse: collapse; width: 50%; }
        td, th { border: 1px solid #ccc; padding: 8px 12px; text-align: left; }
        th { background-color: #f4f4f4; }
    </style>
</head>
<body>
    <h1>System information for {{ ansible_hostname }}</h1>
    <table>
        <tr><th>Parameter</th><th>Value</th></tr>
        <tr>
            <td>CPU</td>
            <td>{{ ansible_processor_vcpus }} vCPU(s) ({{ ansible_processor[-1] }})</td>
        </tr>
        <tr>
            <td>RAM</td>
            <td>{{ ansible_memtotal_mb }} MB</td>
        </tr>
        <tr>
            <td>First HDD ({{ ansible_devices.keys() | list | first }})</td>
            <td>{{ ansible_devices[ansible_devices.keys() | list | first].size }}</td>
        </tr>
        <tr>
            <td>IP address</td>
            <td>{{ ansible_default_ipv4.address }}</td>
        </tr>
    </table>
</body>
</html>
```

### Запуск

```bash
ansible-playbook -i inventory.ini task3/site.yml
```

Вывод выполнения:

```
PLAY [Configure Apache web server with system info page] *****************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************************
ok: [Ubuntu]

TASK [apache_info : Install Apache web server] ***************************************************************************************************************************
ok: [Ubuntu]

TASK [apache_info : Deploy index.html with system information] ***********************************************************************************************************
changed: [Ubuntu]

TASK [apache_info : Ensure firewalld allows HTTP (RedHat family)] ********************************************************************************************************
skipping: [Ubuntu]

TASK [apache_info : Ensure ufw allows HTTP (Debian family)] **************************************************************************************************************
changed: [Ubuntu]

TASK [apache_info : Start and enable Apache service] *********************************************************************************************************************
ok: [Ubuntu]

TASK [apache_info : Verify that the website responds with HTTP 200] ******************************************************************************************************
ok: [Ubuntu]

TASK [apache_info : Show website check result] ***************************************************************************************************************************
ok: [Ubuntu] => {
    "msg": "Website check status: 200"
}

RUNNING HANDLER [apache_info : Restart Apache] ***************************************************************************************************************************
changed: [Ubuntu]

PLAY RECAP ***************************************************************************************************************************************************************
Ubuntu                     : ok=8    changed=3    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0   

```

Проверка через браузер/curl:

```bash
curl http://<IP_хоста>
```

```
https://github.com/bigmisha0/-Ansible.-2/blob/ff8bee6866f5322a3d69f7576aa4e3982e8e3df9/image.png
```

---

