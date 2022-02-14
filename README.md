### Репозиторий с измененным `playbook` 
#### ДЗ "08.02 Работа с Playbook" - Захаров Сергей Николаевич

Здесь должно быть описано: 
- что делает playbook, 
- какие у него есть: 
  - параметры
  - теги
  - 
```yml
---
- name: Install Java    # Первый Play по установке Java
  hosts: all            # Для всех хостов
  tasks:                #  Задача первая
    - name: Set facts for Java 11 vars
      set_fact:         # Создаем новую динамическую переменную чере модуль `set_fact`
        java_home: "/opt/jdk/{{ java_jdk_version }}"    # Ключ перменной, значение которого содержит хардкор /opt/jdk/`  и пеерменная `java_jdk_version` из варсов
      tags: java        # Тег, позволяющий запускать таску по условию запуска по тегам
    - name: Upload .tar.gz file containing binaries from local storage     #  Задача вторая. Закачивает на manage_node файл с архивом который находится на 
                                                                           # control_node по пути, указанному в переменной `java_oracle_jdk_package ` 
                                                                           # в файле `group_vars/all/vars.yml`
     
      copy:         # Модуль copy ищет файл на локалхосте и отправляет его на manage_node по пути в dest: "/tmp/jdk-{{ java_jdk_version }}.tar.gz"
        src: "{{ java_oracle_jdk_package }}"    #  Ищет в локальной директории /files
        dest: "/tmp/jdk-{{ java_jdk_version }}.tar.gz"    #  Отправляет на manage_node
      register: download_java_binaries          #  register записывает результат в переменную download_java_binaries
      until: download_java_binaries is succeeded    # Цикл будет выполнять таску, пока не произойдет условие цикла `download_java_binaries is succeeded`
      tags: java     # Тег, позволяющий запускать таску по условию запуска по тегам
    - name: Ensure installation dir exists    #  Задача третья.
      become: true        # Модуль повышения привелегий пользователя для выполнения действия. По умолчению это root. Запускается с ключем become-user=root. 
                          # При этом на manage_node должен быть установлен sudo
      file:           # Модуль file для создания state-ом директорий. 
        state: directory    #  Метод state. При помощи переменнх group_vars будут созданы все поддиректории, 
                            # которые в этом пути указаны.
        path: "{{ java_home }}"      #  Путь до домашнего каталога Java на manage_node
      tags: java      # Тег, позволяющий запускать таску по условию запуска по тегам
    - name: Extract java in the installation directory     #  Задача четвертая.
      become: true      # Модуль повышения привелегий пользователя для выполнения действия.
      unarchive:
        copy: false
        src: "/tmp/jdk-{{ java_jdk_version }}.tar.gz"
        dest: "{{ java_home }}"
        mode: 0755
        extra_opts: [--strip-components=1]
        creates: "{{ java_home }}/bin/java"
      tags:
        - java
    - name: Export environment variables
      become: true
      template:
        src: jdk.sh.j2
        dest: /etc/profile.d/jdk.sh
      tags: java
- name: Install Elasticsearch
  hosts: elasticsearch
  tasks:
    - name: Upload tar.gz Elasticsearch from remote URL
      get_url:
        url: "https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-{{ elastic_version }}-linux-x86_64.tar.gz"
        dest: "/tmp/elasticsearch-{{ elastic_version }}-linux-x86_64.tar.gz"
        mode: 0755
        timeout: 60
        force: true
        validate_certs: false
      register: get_elastic
      until: get_elastic is succeeded
      tags: elastic
    - name: Create directrory for Elasticsearch
      become: true
      file:
        state: directory
        path: "{{ elastic_home }}"
      tags: elastic
    - name: Extract Elasticsearch in the installation directory
      become: true
      unarchive:
        copy: false
        src: "/tmp/elasticsearch-{{ elastic_version }}-linux-x86_64.tar.gz"
        dest: "{{ elastic_home }}"
        extra_opts: [--strip-components=1]
        creates: "{{ elastic_home }}/bin/elasticsearch"
      tags:
        - elastic
    - name: Set environment Elastic
      become: true
      template:
        src: templates/elk.sh.j2
        dest: /etc/profile.d/elk.sh
      tags: elastic
      ```
