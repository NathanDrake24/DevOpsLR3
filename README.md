# Основы Ansible в DevOps


## 1. Установка Ansible на управляющую машину (Linux/WSL)

### Шаг 1.1: Обновление пакетов системы
```bash
sudo apt update
sudo apt upgrade -y
```

### Шаг 1.2: Установка Python и pip
Ansible требует Python 3.9+:
```bash
sudo apt install -y python3 python3-pip python3-venv
```

Проверка версии Python:
```bash
python3 --version
```

### Шаг 1.3: Установка Ansible
```bash
sudo apt install -y ansible
```

### Шаг 1.4: Проверка установки Ansible
```bash
ansible --version
```

 Вывод:
```
ansible [core 2.14.x]
  config file = None
  configured module search path = ['/home/user/.ansible/plugins/modules']
  ...
```

---

## 2. Подготовка SSH ключей для управляемых машин

### Шаг 2.1: Генерация SSH ключевой пары на управляющей машине
```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/ansible_key -N ""
```

Параметры:
- `-t rsa` - тип ключа (RSA 4096 бит)
- `-b 4096` - размер ключа (безопасный размер)
- `-f ~/.ssh/ansible_key` - путь сохранения приватного ключа
- `-N ""` - пустой пароль для ключа (для автоматизации)

Проверка ключей:
```bash
ls -la ~/.ssh/ansible_key*
```

### Шаг 2.2: Установка прав доступа на приватный ключ
```bash
chmod 600 ~/.ssh/ansible_key
chmod 644 ~/.ssh/ansible_key.pub
```

---

## 3. Запуск управляемого контейнера в Docker

### Шаг 3.1: Создание Dockerfile для управляемого хоста
```yml
FROM ubuntu:22.04

# Установка необходимых пакетов
RUN apt-get update && apt-get install -y \
    openssh-server \
    openssh-client \
    python3 \
    python3-pip \
    sudo \
    curl \
    wget \
    git \
    nano \
    htop \
    && rm -rf /var/lib/apt/lists/*

# Создание пользователя ansible
RUN useradd -m -s /bin/bash ansible && \
    echo "ansible ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

# Настройка SSH
RUN mkdir -p /run/sshd && \
    sed -i 's/#PubkeyAuthentication yes/PubkeyAuthentication yes/' /etc/ssh/sshd_config && \
    sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config

# Создание .ssh директории для пользователя ansible
RUN mkdir -p /home/ansible/.ssh && \
    chown -R ansible:ansible /home/ansible/.ssh && \
    chmod 700 /home/ansible/.ssh

# Запуск SSH сервера в foreground режиме
CMD ["/usr/sbin/sshd", "-D"]
```


### Шаг 3.2: Создание docker-compose.yml
```yml
version: '3.8'

services:
  managed-host:
    container_name: ansible-managed-host
    build: .
    ports:
      - "2222:22"
    environment:
      - TERM=xterm-256color
    networks:
      - ansible-net
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "service", "ssh", "status"]
      interval: 10s
      timeout: 5s
      retries: 3

networks:
  ansible-net:
    driver: bridge
```

### Шаг 3.3: Сборка и запуск контейнера
```bash
# Перейдите в директорию с docker-compose.yml
cd /path/to/project

# Сборка образа
docker-compose build

# Запуск контейнера в фоновом режиме
docker-compose up -d
```

### Шаг 3.4: Проверка запущенного контейнера
```bash
docker-compose ps
```

Ожидаемый вывод:
```
NAME                    COMMAND               STATUS      PORTS
ansible-managed-host    "/usr/sbin/sshd -D"   Up 1 min    0.0.0.0:2222->22/tcp
```

### Шаг 3.5: Копирование публичного SSH ключа в контейнер
```bash
# Создаёте директорию .ssh в контейнере и копируете публичный ключ
docker exec ansible-managed-host mkdir -p /home/ansible/.ssh

docker cp ~/.ssh/ansible_key.pub ansible-managed-host:/home/ansible/.ssh/authorized_keys

# Установка правильных прав доступа
docker exec ansible-managed-host chown -R ansible:ansible /home/ansible/.ssh
docker exec ansible-managed-host chmod 700 /home/ansible/.ssh
docker exec ansible-managed-host chmod 600 /home/ansible/.ssh/authorized_keys
```

---

## 4. Проверка SSH подключения к контейнеру

### Шаг 4.1: Проверка SSH подключения
```bash
ssh -i ~/.ssh/ansible_key -p 2222 ansible@localhost
```

 Результат: Попадаем в bash контейнера без ввода пароля.

Выход из контейнера:
```bash
exit
```

---

## 5. Создание инвентарного файла Ansible (inventory)

### Шаг 5.1: Создание файла `inventory.ini`

Создайте файл `inventory.ini` в рабочей директории:
```ini
[managed_hosts]
managed1 ansible_host=localhost ansible_port=2222 ansible_user=ansible ansible_ssh_private_key_file=~/.ssh/ansible_key ansible_python_interpreter=/usr/bin/python3

[all:vars]
ansible_ssh_common_args=-o StrictHostKeyChecking=no
```

**Объяснение параметров:**
- `[managed_hosts]` - группа хостов (можно иметь несколько групп)
- `managed1` - имя хоста в инвентаре (локальное имя, не обязательно реальное)
- `ansible_host=localhost` - IP адрес или FQDN реального хоста
- `ansible_port=2222` - порт SSH (из docker-compose)
- `ansible_user=ansible` - пользователь для подключения
- `ansible_ssh_private_key_file` - путь к приватному SSH ключу
- `ansible_python_interpreter` - путь к интерпретатору Python на управляемом хосте
- `ansible_ssh_common_args` - отключает проверку ключа хоста (для первого подключения)

### Шаг 5.2: Проверка инвентаря
```bash
ansible-inventory -i inventory.ini --list
```

Ожидаемый вывод (JSON формат):
```json
{
    "_meta": {
        "hostvars": {
            "managed1": {
                "ansible_host": "localhost",
                "ansible_port": "2222",
                ...
            }
        }
    },
    "all": {...},
    "managed_hosts": {...},
    "ungrouped": {}
}
```

---

## 6. Проверка подключения Ansible к управляемому хосту

### Шаг 6.1: Тест ping
```bash
ansible -i inventory.ini managed_hosts -m ping
```

Ожидаемый вывод:
```
managed1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

### Шаг 6.2: Сбор информации о системе (facts)
```bash
ansible -i inventory.ini managed1 -m setup
```

Выведет всю информацию о системе управляемого хоста.

### Шаг 6.3: Выполнение простой команды
```bash
ansible -i inventory.ini managed1 -m command -a "uname -a"
```

Ожидаемый вывод:
```
managed1 | CHANGED | rc=0 >>
Linux f3a4c8b0c4a2 5.15.0-92-generic #102-Ubuntu SMP Thu Jan 9 10:54:01 UTC 2025 x86_64 GNU/Linux
```

---

## 7. Создание и запуск Ansible Playbook

### Шаг 7.1: Структура проекта
```
project/
├── Dockerfile
├── docker-compose.yml
├── inventory.ini
├── playbook.yml
└── README.md
```

### Шаг 7.2: Создание playbook.yml
```yml
---
- name: Демонстрационный playbook для Ansible
  hosts: managed_hosts
  gather_facts: yes
  
  tasks:
    - name: Обновление списка пакетов
      apt:
        update_cache: yes
        cache_valid_time: 3600
      become: yes

    - name: Установка требуемых пакетов
      apt:
        name:
          - curl
          - wget
          - git
          - nano
        state: present
      become: yes

    - name: Создание тестовой директории
      file:
        path: /tmp/ansible_test
        state: directory
        mode: '0755'

    - name: Создание тестового файла с содержимым
      copy:
        content: |
          Hello from Ansible!
          This is a test file created by Ansible playbook.
          Hostname: {{ ansible_hostname }}
          OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
          Date: {{ ansible_date_time.iso8601 }}
        dest: /tmp/ansible_test/test_file.txt
        mode: '0644'

    - name: Вывод содержимого файла
      command: cat /tmp/ansible_test/test_file.txt
      register: file_content

    - name: Отображение содержимого файла
      debug:
        msg: "{{ file_content.stdout }}"

    - name: Сбор информации о системе
      debug:
        msg: |
          System: {{ ansible_system }}
          Hostname: {{ ansible_hostname }}
          Uptime: {{ ansible_uptime_seconds }} seconds
          CPU cores: {{ ansible_processor_cores }}
          Total memory: {{ ansible_memtotal_mb }} MB
```

### Шаг 7.3: Запуск playbook
```bash
# Запуск playbook
ansible-playbook -i inventory.ini playbook.yml
```

### Шаг 7.4: Вывод playbook
```
PLAY [managed_hosts] ************************************************************

TASK [Gathering Facts] **********************************************************
ok: [managed1]

TASK [Update package list] ******************************************************
changed: [managed1]

TASK [Install required packages] ************************************************
changed: [managed1]

TASK [Create test directory] ****************************************************
changed: [managed1]

TASK [Create test file with content] ********************************************
changed: [managed1]

TASK [Display file content] *****************************************************
ok: [managed1] => {
    "msg": "File content from managed host: Hello from Ansible!\nThis is a test file created by Ansible playbook."
}

TASK [Get system information] ***************************************************
ok: [managed1] => {
    "msg": "System: Linux, Hostname: f3a4c8b0c4a2, Uptime: 12 min"
}

PLAY RECAP ************************************************************
managed1 : ok=7 changed=4 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
```

---

## 8. Задания для выполнения

### Задание 1: Базовое подключение

**Результат:** успешный ответ "pong" от управляемого хоста

---

### Задание 2: Базовые ad-hoc команды
1. Получите информацию о ядрах CPU управляемого хоста:
   ```bash
   ansible -i inventory.ini managed1 -m setup -a "filter=ansible_processor_cores"
   ```

2. Проверьте свободное место на диске:
   ```bash
   ansible -i inventory.ini managed1 -m command -a "df -h"
   ```

3. Получите список всех пользователей:
   ```bash
   ansible -i inventory.ini managed1 -m command -a "cat /etc/passwd"
   ```

4. Измените временную зону хоста на UTC:
   ```bash
   ansible -i inventory.ini managed1 -m command -a "timedatectl set-timezone UTC"
   ```

**Результат:** вывод команд без ошибок

---

### Задание 3: Работа с файлами
1. Создайте новый playbook `task3_files.yml`:
   ```yaml
   ---
   - name: Work with files
     hosts: managed_hosts
     tasks:
       - name: Create multiple directories
         file:
           path: /tmp/{{ item }}
           state: directory
           mode: '0755'
         loop:
           - test_dir1
           - test_dir2
           - test_dir3
   
       - name: Create files in directories
         copy:
           content: "This is {{ item }} file\n"
           dest: /tmp/{{ item }}/content.txt
         loop:
           - test_dir1
           - test_dir2
           - test_dir3
   
       - name: Display files
         command: cat /tmp/{{ item }}/content.txt
         loop:
           - test_dir1
           - test_dir2
           - test_dir3
         register: file_content
   
       - name: Show file contents
         debug:
           msg: "{{ item.stdout }}"
         loop: "{{ file_content.results }}"
   ```

2. Запустите playbook:
   ```bash
   ansible-playbook -i inventory.ini task3_files.yml
   ```

**Результат:** три директории с файлами, созданные на управляемом хосте

---
