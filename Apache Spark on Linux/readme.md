### Данный мини-гайд описывает порядок установки Apache Spark в несколько простых команд с использованием пакетного менеджера APT, что актуально для ряда дистрибутивов Linux, таких как Ubuntu и Debian. Стоит отметить, что установку рассматриваем в виртуальное окружение вместе с установкой JupyterLab

***Сперва установим pip, если ранее он не был установлен:***  
```sudo apt install python3-pip python3-dev```

***Устанавливаем пакет для создания изолированных виртуальных окружений для Python:***  
```sudo apt install virtualenv```

***Cоздаем каталог проектов для Jupyter:***  
```mkdir ~/jupyter_projects```

***Переходим в созданную директорию:***  
```cd jupyter_projects```

***Создаем и активируем виртуальное окружение:***  
```virtualenv jupyter_venv```  
```source jupyter_venv/bin/activate```

***Устанавливаем JupyterLab:***  
```pip install jupyterLab```

***В Виртуальном окружении Jupyter ставим Pyspark:***  
```pip install pyspark```

***Может понадобиться посмотреть где лежит java:***  
```sudo update-alternatives --config java```

***C помощью vim или nano открываем .bashrc***  
```vi ~/.bashrc```

***И добавляем в конец 2 строки с системными переменными окружения:***  
```export SPARK_HOME=/home/sparkuser/spark```  
```export PATH=$PATH:$SPARK_HOME/bin```

***Также добавляем в конец .bashrc экспорт Java, к примеру, такой (как узнать свой путь см. выше):***  
```export JAVA_HOME=/usr/lib/jvm/bin/java/java-11-openjdk-amd64```

***После выполнения всех действий можем запускать JupyterLab:***  
```jupyter lab```

***И в JupyterLab поднимаем spark-сессию:***  
```from pyspark.sql import SparkSession```  
```spark = SparkSession.builder.getOrCreate()```

***Также можем запустить ядро spark в терминале нашего виртуального окружения, к примеру, такой командой:***  
```spark-shell -v```

***Деактивировать виртуальное окружение можем командой:***  
```deactivate```
