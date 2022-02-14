### Репозиторий с измененным `playbook` по ДЗ "08.02 Работа с Playbook" - Захаров Сергей Николаевич


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
        src: jdk.sh.j2    # файл шаблона из папки t/emplate
        dest: /etc/profile.d/jdk.sh   # путь на  manage_node, куда надо перенести шаблон и сделать его сценарием .sh
      tags: java    # Тег, позволяющий запускать таску по условию запуска по тегам
      
      
 #  Второй Play по установке Elasticsearch
- name: Install Elasticsearch     # Название Play
  hosts: elasticsearch            # Play будет запускатсья на хостах из группы elasticsearch нашего инвентори
  tasks:                          # Список задач в составе Play
  
  # Задача первая. 
  # Цель задачи - скачать с официального сайта файла с архивом Elasticsearch на manage_node
    - name: Upload tar.gz Elasticsearch from remote URL     #  Название задачи
      get_url:        # Модуль для скачивания файла и переноса его в указанноую директорию
        url: "https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-{{ elastic_version }}-linux-x86_64.tar.gz"   # Адрес расположения архива 
                          # параметризирован при помощи group_vars {{ elastic_version }} на то, какую именно версию искать
                          #  Здесь нет модуля delegate_to, поэтому скачивание будет происходить сразу на manage_node
        
        dest: "/tmp/elasticsearch-{{ elastic_version }}-linux-x86_64.tar.gz"    # Папка, куда будет сохраняться архив
        mode: 0755      # Установление прав доступа к файлу-архиву
        timeout: 60     # Ожидание 60 секунд для get_url
        force: true     # Если архив Elasticsearch уже ранее был скачан и существует, то будет принудительно перезакачивание архива
        validate_certs: false    # get_url не будет реагировать ошибки свзанные с отсутствием сертификата SSL сайта
      register: get_elastic      # результат записываем в переменную get_elastic
      until: get_elastic is succeeded      # цикл until будет запускать задачу скачивания архива до тех пор, пока не будет удачное скачивание
      
      #  Пример работы цикла
      
      # TASK [Upload tar.gz Elasticsearch from remote URL] 
      # *******************************************************************************************************************************************************
      # FAILED - RETRYING: [fedore]: Upload tar.gz Elasticsearch from remote URL (3 retries left).
      # ok: [fedore]
      
      tags: elastic     # Тег, позволяющий запускать таску по условию запуска по тегам
      
      
  # Задача вторая. 
  # Цель задачи - создать директорию для Elasticsearch
    - name: Create directrory for Elasticsearch     #  Название задачи
      become: true        # Модуль повышения привелегий пользователя для выполнения действия.
      file:               # Модуль file для создания state-ом директорий. 
        state: directory    #  Создание модулем state директории без помощи фактов
        path: "{{ elastic_home }}"    #  Путь к домашней директории взят из group_vars
      tags: elastic       # Тег, позволяющий запускать таску по условию запуска по тегам
      
  # Задача третья. 
  # Цель задачи - разархивировать файлы и скопировать их в домашнюю директорию.
  # Действия и модули аналогичные из Play для Java
    - name: Extract Elasticsearch in the installation directory     #  Название задачи
      become: true      
      unarchive:
        copy: false
        src: "/tmp/elasticsearch-{{ elastic_version }}-linux-x86_64.tar.gz"
        dest: "{{ elastic_home }}"
        extra_opts: [--strip-components=1]
        creates: "{{ elastic_home }}/bin/elasticsearch"
      tags:
        - elastic
        
        
   # Задача четвертая. 
  # Цель задачи -  выполнить перенос переменных окружения из шаблонов .j2 в каталог сценариев приложений etc/profile.d/ 
  # Действия и модули аналогичные из Play для Java
    - name: Set environment Elastic
      become: true
      template:
        src: templates/elk.sh.j2
        dest: /etc/profile.d/elk.sh
      tags: elastic
      
      ```
