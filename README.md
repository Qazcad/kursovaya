## Курсовая работа по дисциплине "Информационные системы аэрокосмических комплексов".  
### Конык Д.А. M3о-312б-18.  
  
# Постановка задачи:  
1. Скачать и установить веб-сервер.  
2. Настроить его на работу с localhost  
3. Реализовать форму с загрузкой файла  
3.1 Захостить python приложение из предыдущего семестра, при загрузке снимка рисовать в веб карту NDVI.  
  
# Ход выполнения работы:  
Выполнять работу будем на Cent OS.  
Предварительно создадим пользователя "konyk" и авторизуемся.  
Создадим директорию для проекта: "mkdir ~/project/"  
  
## Шаг 1. Установка необходимых компонентов.  
Установим репозиторий EPEL, который содержит дополнительные пакеты.  

    [konyk@localhost project]$ sudo yum install epel-release

Установим pip диспетчер пакетов Python, а также файлы разработки Python, необходимые для сборки uWSGI, а также установим Nginx.  

    [konyk@localhost project]$ sudo yum install python-pip python-devel gcc nginx  
  
## Шаг 2. Создание виртуальной среды Python.  
Для того чтобы изолировать приложение настроим виртуальную среду.  
Установим пакет virtualenv:  

    [konyk@localhost project]$ sudo pip install virtualenv  
Создадим видтуальную среду:  

    [konyk@localhost project]$ virtualenv projectvenv  
Активируем её:  

    [konyk@localhost project]$ source projectvenv/bin/activate  
Убедимся что мы начали работу в виртуальной среде. Ввод терминала выглядит следующим образом:  

    (projectvenv) [konyk@localhost project]$  
  
## Шаг 3. Создание и настройка приложения Flask.  
Установим uwsgi и flask:  

    (projectvenv) [konyk@localhost project]$ pip install uwsgi flask  
Создадим приложение Flask:  

    (projectvenv) [konyk@localhost project]$ vi ~/project/app.py  
(Исходный код приложения продемонмтрирован в данном репозитории)  
Сохраним и закроем файл нажав ESC, а затем нажав Ctrl+Z,Z.  
Протестируем созданное приложение. Для этого запустим его в фоновом режиме:  

    (projectvenv) [konyk@localhost project]$ python ~/project/app.py &  
> *Serving Flask app 'app' (lazy loading)  
> *Environment: prodction  
> *Debug mode: on  
> *Running on all addresses.  
> *Running on http://10.0.15:5000 (Press CTRL+C) to quit)  

Далее обратимся к содержимому с помощью curl. Выведем первые 8 строчек html страницы: 

    (projectenv) [konyk@localhost project]$ curl -L http://10.0.2.15:5000 | head -n 8  
    
> \<!DOCTYPE html>  
>  \<html lang="ru">  
>  \<head>  
>   \<meta charset="UTF-8">  
>   \<title>Image processing</title>  
>  \</head>  
> \<body>  
>   \<p class="fadein1">Загрузите изображения для красного и инфракрасного спектров в формате tif:</p>  
      
После этого остановим Flask приложение с помощью fg:  

    (projectvenv) [konyk@localhost project]$ fg  
    python app.py  
    ^C  
  
## Шаг 3. Создание точки входа WSGI.  
Создадим файл wsgi.py:  

    (projectvenv) [konyk@localhost project]$ vi ~/project/wsgi.py  
Внутри напишем:  

    from project import app  
    if __name__ == "__main__":  
      app.run()  
  
## Шаг 4. Настройка конфигурации uWSGI.  
Для начала протестируем что uWSGI может обслуживать наше приложение:  

    (projectvenv) [konyk@localhost project]$ uwsgi --socket 0.0.0.0:8000 --protocol=http -w wsgi &  
Убедимся что по указанному ранее адресу, но с портом 8000 находится содержимое html страницы.  

    (projectvenv) [konyk@localhost project]$ curl -L http://10.0.15:8000 | head -n 5  
    
> \<!DOCTYPE html>  
> \<html lang="ru">  
> \<head>  
> \<meta charset="UTF-8">  
> \<title>Image processing</title>  

После этого приостановим uwsgi:  

    (projectvenv) [konyk@localhost project]$ fg  
    uwsgi --socket 0.0.0.0:8000 --protocol=http -w wsgi
    ^C  
На этом работа с виртуальной средой окончена. Выйдем из нее командой deactivate:  

    (projectvenv) [konyk@localhost project]$ deactivate  
Создадим файл конфигурации uWSGI:  

    [konyk@localhost project]$ vi ~/project/project.ini  
Внутри напишем:  

    [uwsgi]  
    module = wsgi  
    master = true  
    processes = 3  
    socket = project.sock  
    chmod-socket = 660  
    vacuum = true  
где "module = wsgi" - исполняемый модуль, созданный ранее файл "wsgi.py";  
"master = true" - означает что uWSGI будет запускаться в главном режиме;  
"processes = 3" - будет иметь 3 рабочих процесса для обслуживания запросов;  
"socket = project.sock" - сокет, который будет использовать uWSGI;  
"chmod-socket = 660" - права на процесс uWSGI;  
"vacuum = true" - сокет будет очищен по завершении работы процесса;  
  
## Шаг 5. Создание файла модуля systemd.  
Создадим файл службы:  

    [konyk@localhost project]$ sudo vi /etc/systemd/system/project.service  
Внутри напишем:

    [Unit]  
    Description=uWSGI for project  
    After=network.target  
    [Service]  
    User=konyk  
    Group=nginx  
    WorkingDirectory=/home/konyk/project  
    Environment="PATH=/home/konyk/project/projectvenv/bin"  
    ExecStart=/home/konyk/project/projectvenv/bin/uwsgi --ini project.ini  
    [Install]  
    WantedBy=multi-user.target  
где "[Unit]", "[Service]", "[Install]" - заголовки разделов;  
"Description" - описание службы;  
"After" - цель, по достижении которой будет производиться запуск;  
"User" - пользователь;  
"Group" - группа;  
"WorkingDirectory" - рабочая директория в которой хранятся исполняемые файлы;  
"Environment" - директория виртуальной среды;  
"ExecStart" - команда для запуска процесса;  
"WantedBy" - когда запускаться службе.  
Запустим созданную службу:  

    [konyk@localhost project]$ sudo systemctl start project  
    [konyk@localhost project]$ sudo systemctl enable project  
  
## Шаг 6. Настройка Nginx.  
Теперь необходимо настроить Nginx для передачи веб-запросов в  сокет с использованием uWSGI протокола.  
Откроем файл конфигурации Nginx:  

    [konyk@localhost project]$ sudo vi /etc/nginx/nginx.conf  
Найдем блок server {} в теле http: 

    ...  
      http {  
      ...  
        include /etc/nginx/conf.d/*.conf;  
        *
        server {  
            listen 80 default_server;  
            }  
      ...  
      }  
    ...  
 
Выше него (*) создадим свой:  

    server {  
      listen 80;  
      server_name 10.0.2.15;  
      
      location / {
        include uwsgi_params;
        uwsgi_pass unix:/home/konyk/project/project.sock;
    }
Закроем и сохраним файл.  
Добавим nginx пользователя в свою группу пользователей с помощью следующей команды:  

    [konyk@localhost project]$ sudo usermod -a -G konyk nginx  
Предоставим группе пользователей права на выполнение в домашнем каталоге:  

    [konyk@localhost project]$ chmod 710 /home/konyk
Запустим Nginx:  

    [konyk@localhost project]$ sudo systemctl start nginx  
    [konyk@localhost project]$ sudo systemctl enable nginx  
  
# Вывод:  
В ходе выполнения курсовой работы было создано приложение Flask в виртуальной средe, которое позволяет загружать файлы формата ".tif" и получать цветное изображение NDVI. 
Была создана и настроена точка входа WSGI, с помощью котрой любой сервер приложений, поддерживающий WSGI, может взаимодействовать с приложением Flask. 
Была создана служба systemd для автоматического запуска uWSGI при загрузке системы. Также был настроен серверный блок Nginx. 

