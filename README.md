### Репозиторий с измененным `playbook` 
#### ДЗ "08.02 Работа с Playbook" - Захаров Сергей Николаевич

Здесь должно быть описано: 
- что делает playbook, 
- какие у него есть: 
  - параметры
  - теги



```yml
---
# Данный ansible-playbook демонстрационный и показывает структуру с описанием задач, модулей, методов, переменных и их значений.
# 

#  Первый Play по установке Java
- name: Install Java    # Название Play
  hosts: all            # Play будет запускатсья на всех хостах нашего инвентори
  tasks:                # Список задач в составе Play
  
  #  Задача первая
  # Цель задачи - создать динамическую переменую, определяющую зависимое от версии дистрибутива имя домашнего каталога для распакованных фалов Java 
    - name: Set facts for Java 11 vars    #  Название задачи. 
      set_fact:         # Создаем новую динамическую переменную чере модуль `set_fact`
        java_home: "/opt/jdk/{{ java_jdk_version }}"    # Ключ перменной, значение которого содержит хардкор /opt/jdk/`  и пеерменная `java_jdk_version` из варсов
      tags: java        # Тег, позволяющий запускать таску по условию запуска по тегам
      
      #  Задача вторая. 
      # Цель задачи - перенос файла с архивом из control_node  на  manage_node
    - name: Upload .tar.gz file containing binaries from local storage     #  Название задачи. 
    
    # Цель задачи: Закачивает на manage_node файл с архивом который находится на 
                 # control_node по пути, указанному в переменной `java_oracle_jdk_package ` 
                 # в файле `group_vars/all/vars.yml`
     
      copy:         # Модуль copy ищет файл на локалхосте и отправляет его на manage_node по пути в dest: "/tmp/jdk-{{ java_jdk_version }}.tar.gz"
        src: "{{ java_oracle_jdk_package }}"    #  Ищет в локальной директории /files
        dest: "/tmp/jdk-{{ java_jdk_version }}.tar.gz"    #  Отправляет на manage_node
      register: download_java_binaries          #  register записывает результат в переменную download_java_binaries
      until: download_java_binaries is succeeded    #  Цикл будет выполнять таску, пока не произойдет условие цикла `download_java_binaries is succeeded`
      tags: java     # Тег, позволяющий запускать таску по условию запуска по тегам
      
      #  Задача третья.
      # Цель задачи - создать поддиректории для переноса в них распакованных файлов из архива
    - name: Ensure installation dir exists    #  Название задачи
      become: true        # Модуль повышения привелегий пользователя для выполнения действия. По умолчению это root. Может запускаться с ключем become-user=root. 
                          # При этом на manage_node должен быть установлен sudo
      file:           # Модуль file для создания state-ом директорий. 
        state: directory    #  Метод state. При помощи переменнх group_vars будут созданы все поддиректории, 
                            # которые в этом пути указаны.
        path: "{{ java_home }}"      #  Путь до домашнего каталога Java на manage_node
      tags: java      # Тег, позволяющий запускать таску по условию запуска по тегам
      
      #  Задача четвертая.
      # Цель задачи - разархивировать файлы и скопировать их в домашнюю директорию.
    - name: Extract java in the installation directory     #  Название задачи
      become: true      # Модуль повышения привелегий пользователя для выполнения действия.
      unarchive:    # с помощью модуля ` unarchive ` мы из `src: "/tmp/jdk-{{ java_jdk_version }}.tar.gz"`  распаковываем архив, 
                    # который распакуется и будет находится на `manage_node` по пути в `dest: "{{ java_home }}"`

        copy: false
        src: "/tmp/jdk-{{ java_jdk_version }}.tar.gz"     # путь до местонахождения архива 
        dest: "{{ java_home }}"                           # путь до директории, куда будет распакован архив
        mode: 0755                                        # установление системных прав доступа к директории
        extra_opts: [--strip-components=1]                # опция для длинных путей
        creates: "{{ java_home }}/bin/java"     # модуль `creates` после того, как распакует архив, проверяет, что по `{{ java_home }}/bin/java` пути
                                                # созданы файлы. Если файлы не найдутся, то таска зафейлится
        
      tags:
        - java          # Тег, позволяющий запускать таску по условию запуска по тегам
        
      #  Задача пятая.  
      # Цель задачи - выполнить перенос переменных окружения из шаблонов .j2 в каталог сценариев приложений etc/profile.d/
    - name: Export environment variables        #  Название задачи
      become: true                              # Модуль повышения привелегий пользователя для выполнения действия.
      template:         # template - модуль для проброса файла шаблона .j2 на manage-node в файл jdk.sh который создается с наполнением, котороее сть в teamplate
                        # Есть папка template. Здесь лежать файлы .j2. И они вызываются по пути имя папки/имя файла. 
                        # Директорию назначеня модуль создавать не может, надо чтобы директория уже была создана.
                        # Системня директория /etc/profile.d/ уже существует.
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
