# Домашнее задание
Работа с LVM

Цель:
создавать и работать с логическими томами;


Описание/Пошаговая инструкция выполнения домашнего задания:
Для выполнения домашнего задания используйте методичку

Работа с LVM

Что нужно сделать?
на имеющемся образе (centos/7 1804.2)
https://gitlab.com/otus_linux/stands-03-lvm

/dev/mapper/VolGroup00-LogVol00 38G 738M 37G 2% /

уменьшить том под / до 8G
выделить том под /home
выделить том под /var (/var - сделать в mirror)
для /home - сделать том для снэпшотов
прописать монтирование в fstab (попробовать с разными опциями и разными файловыми системами на выбор)
Работа со снапшотами:
сгенерировать файлы в /home/
снять снэпшот
удалить часть файлов
восстановиться со снэпшота

(залоггировать работу можно утилитой script, скриншотами и т.п.)

Задание со звездочкой*
на нашей куче дисков попробовать поставить btrfs/zfs:
с кешем и снэпшотами
разметить здесь каталог /opt

Если возникнут вопросы, обращайтесь к студентам, преподавателям и наставникам в канал группы в Telegram.

Удачи при выполнении!

Критерии оценки:
Статус "Принято" ставится при выполнении основной части.
Задание со звездочкой выполняется по желанию.
