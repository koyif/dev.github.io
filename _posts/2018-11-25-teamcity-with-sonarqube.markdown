---
layout: post
title:  "Как подружить TeamCity и SonarQube"
date:   2018-11-25 05:35:34 +0500
categories: dev
---

* contents
{:toc}

Как нам всем известно, качественный код лучше, чем некачественный. Для большинства разработчиков звучит как банальность, но почему-то, когда дело доходит до целенаправленного отслеживания этого показателя, всплывает масса проблем: по каким критериям оценивать это самое качество, до какого уровня необходимо его подтянуть, кто этим должен заниматься, да и лень, в конце концов, тоже никто не отменял. 

Тут к нам на помощь и приходят всякие вспомогательные средства:
1.	IDE (в нашей компании стандарт де-факто **Intellij IDEA**) подсвечивает базовые ошибки в коде и даже предлагает способы их исправления в некоторых случаях;
2.	Плагин **SonarLint** для IDEA проводит более глубокий анализ кода и также дает подсказки;
3.	И, наконец, самый полный анализ может сделать **SonarQube** – в отличие от плагина из пункта 2, это уже целая платформа для непрерывного анализа и измерения качества кода.

Как раз про **SonarQube** мы сегодня и поговорим, а именно - про внедрение его в систему **continuous integration** (непрерывной интеграции) **TeamCity** как этапа сборки проекта.

### Установка программного обеспечения
Установка и настройка TeamCity и SonarQube выходит за рамки данной статьи, так что примем за факт, что они уже установлены и настроены.

### Настройка TeamCity
Для удобства пользователей в TeamCity уже есть специальный плагин: **«SonarQube Scanner»**. Посмотреть его версию или установить его, если это еще не сделано, можно в настройках администрирования: **Administration -> Tools (Integration)**.

<div class="img_container">
![Так выглядит плагин в списке плагинов TeamCity](/assets/img/2018-11-25-teamcity-with-sonarqube/image_1.png)
*Рис. 1: Так выглядит плагин в списке плагинов TeamCity*
</div>
<br />

После установки сканера, в настройках проекта появится дополнительный пункт **«SonarQube Servers»**, в котором можно добавлять подключения к серверам Sonar’а. Как вы можете увидеть на скрине ниже, серверов может быть несколько:

<div class="img_container">
![Список серверов SonarQube](/assets/img/2018-11-25-teamcity-with-sonarqube/image_2.png)
*Рис. 2: Список серверов SonarQube*
</div>
<br />

Добавить сервер крайне просто: нажать на кнопку **«Add new server»** и заполнить несколько полей в форме.

<div class="img_container">
![Форма добавления нового сервера](/assets/img/2018-11-25-teamcity-with-sonarqube/image_3.png)
*Рис. 3: Форма добавления нового сервера*
</div>
<br />

После добавления сервера, можно перейти к созданию шага сборки (Build step) в вашей конфигурации сборки (Build Configuration). Для этого на странице **Build Steps** нажимаем на кнопку **«Add build step»**. На первом этапе надо выбрать тип runner’а, а именно **«SonarQube Runner»**:

<div class="img_container">
![Выбор runner’а](/assets/img/2018-11-25-teamcity-with-sonarqube/image_4.png)
*Рис. 4: Выбор runner’а*
</div>
<br />

Далее нужно выбрать сервер и версию сканера. 

<div class="img_container">
![Выбор сервера и версии сканера](/assets/img/2018-11-25-teamcity-with-sonarqube/image_5.png)
*Рис. 5: Выбор сервера и версии сканера*
</div>
<br />

Некоторые данные TeamCity заполнит за нас своими переменными окружения: `%system.teamcity.projectName%`, `%teamcity.project.id%`, `%build.number%` - и нас это полностью устраивает. Если у вас есть такая необходимость, эти значения можно заменить своими.

<div class="img_container">
![Настройки проекта в SonarQube](/assets/img/2018-11-25-teamcity-with-sonarqube/image_6.png)
*Рис. 6: Настройки проекта в SonarQube*
</div>
<br />

Sources location, Tests location и Binaries location для java-проектов обычно являются: `src/main/java`, `src/test/java` и `target/classes`. Причем если у вас многомодульный проект, то не надо указывать путь до исходников конкретного модуля, можно указать все модули через запятую в поле **«Modules»**, а сканер сам подставит названия модулей перед всеми путями.

<div class="img_container">
![Настройки структуры проекта](/assets/img/2018-11-25-teamcity-with-sonarqube/image_7.png)
*Рис. 7: Настройки структуры проекта*
</div>
<br />

В **«Additional parameters»** можно передать дополнительные параметры. В нашем случае это даже нужно, т.к. в проекте используется библиотека **Lombok**, и с анализом такого проекта у Сонара могут быть трудности. Мы добавим параметр, который указывает на каталог с зависимостями проекта: `-Dsonar.java.libraries="target/dependency/*.jar"`. Естественно, до запуска анализа надо скопировать зависимости в этот каталог (это делается командой: `mvn dependency:copy-dependencies`).

Помимо этого, для правильного анализа классов с аннотациями Lombok необходимо установить плагин для SonarQube: **“SonarJava”**.

<div class="img_container">
![Дополнительные параметры](/assets/img/2018-11-25-teamcity-with-sonarqube/image_8.png)
*Рис. 8: Дополнительные параметры*
</div>
<br />

### Что в итоге?
Перейдя по ссылке `http://localhost:9000/dashboard?id=ключ_проекта`, мы увидим подробные результаты анализа нашего кода.

<div class="img_container">
![Дашборд с результатами последнего анализа](/assets/img/2018-11-25-teamcity-with-sonarqube/image_9.png)
*Рис. 9: Дашборд с результатами последнего анализа*
</div>
<br />

<div class="img_container">
![Отслеживание динамики](/assets/img/2018-11-25-teamcity-with-sonarqube/image_10.png)
*Рис. 10: Отслеживание динамики*
</div>
<br />

<div class="img_container">
![Рекомендации по улучшению качества кода](/assets/img/2018-11-25-teamcity-with-sonarqube/image_11.png)
*Рис. 11: Рекомендации по улучшению качества кода*
</div>
<br />

В результате за каких-то 5-10 минут мы получили удобную систему контроля качества кода, которая будет автоматически запускаться после каждого коммита и вести историю для отслеживания динамики.
