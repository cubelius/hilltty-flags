## Почему не [флаги Айкара](https://aikar.co/2018/07/02/tuning-the-jvm-g1gc-garbage-collector-flags-for-minecraft/)?
Все очень просто, сбор мусора у него основан, как он говорит, на невероятно стабильном, но крайне медленном по текущим меркам алгоритме D1GC. При этом всем он максимально устарел, все что он реализовал было инновационным во времена JDK 8, сейчас - нет. Действительно, зачем менять то, что работает? А стоило бы.

На замену я предлагаю поставить ShenandoahGC - это сборщик мусора с крайне малым временем паузы, что так раз подходит для нашей любимой игры, мы же все не любим фризы. На стабильность это никак не повлияло, за все время безперебойного тестирования не было выявлено ни одной проблемы.
## Отказ от ответственности
Я не призываю всех тут же менять свои свойста запуска сервера, я лишь даю понять, что ничего идеального не бывает. Так же я не отвечаю за стабильность работы моих параметров в вашем конкретном случае, все системы разные, а результаты абсолютно индивидуальны.
## Флаги
**Внимание!** Только JDK 16, работоспособность на старых версиях не гарантирована.

**Поддерживается:**
- [x] Vanilla
- [x] Bukkit, Spigot, Paper...
- [x] Fabric
- [x] Forge

**Готовые настройки:**
```yml
java -jar -server -Xms6G -Xmx6G -XX:+UseLargePages -XX:LargePageSizeInBytes=2M -XX:+UnlockExperimentalVMOptions -XX:+UseShenandoahGC -XX:ShenandoahGCMode=iu -XX:+UseNUMA -XX:+AlwaysPreTouch -XX:-UseBiasedLocking -XX:+DisableExplicitGC -Dfile.encoding=UTF-8 launcher-airplane.jar --nogui
```
**А теперь внимательно разберем, что за что отвечает:**

*-Xms6G* и *-Xmx6G*: устанавливает границы использования памяти вашим сервером Minecraft, рекомендую не использовать более 12 Гб для вашего сервера и всегда оставлять 1 - 2 Гб свободной памяти для системы.

*-XX:+UseLargePages* и *-XX:LargePageSizeInBytes=2M*: **только для опытных пользователей**, позволяет использовать зарегистрированую память большыми страницами, ускоряет скорость запуска и отзывчевость сервера. Заставим Linux регистрировать страницы для нас. Добавляем эту строку в `/etc/sysctl.conf`:
```yml
vm.nr_hugepages = 3372
```
Как мы получили это число? Допустим я хочу зарегистрировать 6 Гб большых страниц, для этого делю 6 Гб на 2.
```yml
6 * 1024 / 2 = 3072
```
Далее я рекомендую оставить немного свободного места, и добавить 300 к нашему числу.
```yml
3072 + 300 = 3372
```
После перезагружаем систему для применения изменений. Убедиться в том, что память успешно зарегистрована можно командой `grep /proc/meminfo`.

---
*-XX:+UnlockExperimentalVMOptions*: включает возможность использования эксперементальных возможностей.

*-XX:+UseShenandoahGC*: использование в роли алгоритма сборки мусора проект Шенандоа (именно так читает это название переводчик).

*-XX:ShenandoahGCMode=iu*: включение экспераментального режима работы нашего сборщика, он являеться зеркалом режима SATB, что сделает разметку менее консервативной, особенно в отношении доступа к слабым ссылкам.

---
*-XX:+UseNUMA*: включает чередование NUMA на хостах с несколькими сокетами, в сочетании с AlwaysPreTouch он обеспечивает лучшую производительность, чем стандартная готовая конфигурация. Более подробно о данной архитектуре можно узнать [отсюда](https://en.wikipedia.org/wiki/Non-uniform_memory_access).

*-XX:+AlwaysPreTouch*: предрегистрация сразу всей выделенной памяти, уменьшает заддержки ввода.

*-XX:-UseBiasedLocking*: существует компромисс между пропускной способностью неограниченной (предвзятой) блокировки и безопасными точками, которые JVM делает, чтобы включать и выключать их по мере необходимости. Для рабочих нагрузок, в том числе сервера Minecraft, ориентированных на задержку, имеет смысл отключить предвзятую блокировку.

*-XX:+DisableExplicitGC*: вызов System.gc () из пользовательского кода заставляет ShenandoahGC выполнить дополнительный цикл сборки мусора, отключение защищает от кода злоупотребляющего этим.
## Серверное обеспечение (ядро)
В качестве максимально стабильного и производительного варианта, я бы порекомендовал [Airplane](https://github.com/TECHNOVE/Airplane).

Любите эксперементировать? Попробуйте [Yatopia](https://github.com/YatopiaMC/Yatopia), но сначала ознакомьтесь с [этой статьей](https://github.com/KennyTV/Yaptapia), и оцените все возможные риски.
## Дополнительная конфигурация
### bukkit.yml
```yml
chunk-gc:
 period-in-ticks: 600
```
**Рекомендованое значение `chunk-gc.period-in-ticks`:**  
Не выделяйте больше чем 12 Гб памяти, это не даст никакого эффекта в большинстве случаев.
| Память / Кол-во игроков | до 30 | 30 - 60 | 60 - 100 | более 100 |
| :--- | :---: | :---: | :---: | :---: |
| 4 Гб | 400 | - | - | - |
| 8 Гб | 600 | 400 | 300 | - |
| 12 Гб | 1200 | 800 | 600 | 400 |
### spigot.yml
```yml
world-settings:
 default:
  max-tick-time:
   tile: 10
   entity: 20
```
## На этом пока-что все
Данная страница еще будет дорабатываться, так что следите за обновлениями :)
