### Данный мини-гайд описывает порядок установки Apache Kafka в несколько простых команд. В данном примере мы не выходим из среды терминала и не рассматриваем установку и запуск ZooKeeper и Kafka через docker - будет добавлено позже.

***Перед началом установки нужно убедиться, что в нашей системе установлена JAVA (JRE и JDK). Проверить их наличие можно командой:***  
```java -version```  

<Показать вывод>

***Если ПО отсутствует, нужно его установить командами:***  
```sudo apt install default-jre```  
```sudo apt install default-jdk```   

***На [официальном ресурсе](https://kafka.apache.org/downloads) представлены релизы для установки, выбираем нужный дистрибутив или docker образ***  
<br>

#### 1) Установка с помощью дистрибутива (базовый быстрый старт):    
---

***Скачиваем актуальный дистрибутив (актуальная версия может отличаться на момент прочтения)***  
```wget https://downloads.apache.org/kafka/3.9.0/kafka-3.9.0-src.tgz```

***Распаковываем архив командой:***  
```tar xzf kafka-3.9.0-src.tgz```

***Перейдем в распакованную директорию с Kafka***  
```cd kafka-3.9.0-src```


***Выполним следующую команду для Gradle Wrapper.***  
Данная команда собирает наш проект, создавая JAR-файл и использует Scala версии 2.13.14 в процессе сборки(P указывает на то, что это свойство передается в Gradle, а scalaVersion=2.13.14 устанавливает значение этого свойства):  
```./gradlew jar -PscalaVersion=2.13.14```

***Дожидаемся полной установки и примерно такого сообщения:***  
```BUILD SUCCESSFUL in 6m 9s```  
```156 actionable tasks: 156 executed```  

#### Запуск:

***Запускаем ZooKeeper:***  
```bin/zookeeper-server-start.sh config/zookeeper.properties```  

***Открываем <ins>новый терминал</ins> и запускаем Kafka командой:***  
```bin/kafka-server-start.sh config/server.properties```

***Для остановки в отдельном терминале выполянем:***  

  Для остановки Kafka:  
  ```bin/kafka-server-stop.sh```  
  
  Для остановки ZooKeeper:  
  ```bin/zookeeper-server-stop.sh```

  Остановить выполнение Kafka и ZooKeeper в их активных терминалах можно сочетанием клавиш:   
  <kbd>CTRL</kbd> + <kbd>C</kbd>
  
***Запустить также можно через systemctl, включив автозапуск zookeeper и kafka и запускать их одной командой: systemctl start kafka.*** 


***По-дефолту, Kafka стартует на 9092 порту, чтобы убедиться, что kafka работает, можем выполнить команду:***  
```ss -tunlp | grep :9092```

Должны увидеть что-то подобное:

<Добавить скрин>
<br><br>

#### 2) Установка с помощью Docker образа:<dt>
...  
___

#### Тестовый запуск сообщений:  
___
 
Стоит помнить про 3 важные составляющие Kafka, скрипты, которые устанавливаются вместе в директории Bin:  
    - __kafka-topics.sh__ — создание темы, по которой будут отправляться и считываться сообщения  
    - __kafka-console-producer.sh__ — создание обращения, которое направит сообщение  
    - __kafka-console-consumer.sh__ — формирование запроса к брокеру для считывания сообщения   
    
***Итак, первой командой <int>в отдельном терминале</int> мы создаем тему:***  
```bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic hello_world```  
где:  
    - __bootstrap-server localhost:9092__ — адрес хоста kafka, может меняться, если мы запускаем не на том же сервере, гна котором развернута Kafka  
    - __replication-factor__ — количество реплик журнала сообщений.  
    - __partitions__ — количество разделов в теме.  
    - __topic hello_world__— созданная нами тема.  
    
***Проверить созданные топики можем командой:***  
```bin/kafka-topics.sh --bootstrap-server localhost:9092 —list```  

***Теперь отправляем сообщение брокеру:***  
```echo "Hello, World!" | bin/kafka-console-producer.sh --broker-list localhost:9092 --topic hello_world```

***Считаем сообщение командой:***  
```bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic hello_world —from-beginning```  

<Добавить скрин>

