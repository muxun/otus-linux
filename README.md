# otus-linux
<details>
<summary> Домашнее задание №1 Kernel </summary>
<p>

При выполненни домашнего задания использовались два способа сборки ядра - oldconfig и menuconfig <br>

Cборка ядра menuconfig: <br>
* обновлена система и установлены необходимые пакеты

```
# yum update
# yum install -y ncurses-devel make gcc bc bison flex elfutils-libelf-devel openssl-devel grub2

```

* скачано ядро 5.1

```
wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.1.tar.xz

```
 
* скачана текущая конфигруация ядра в папку новго ядра

```
cp -v /boot/config-3.10.0-693.5.2.el7.x86_64 /usr/src/linux-5*/.config

```  
* с помощью псевдо графического интерфейса выбраны необходимые модули (оставил без практически без изменений, так как не совсем понимаю что нужно а что нет)

```
make menuconfig

```
*  запустил сборку ядра

```
# make bzImage
# make modules
# make
# make install
# make modules_install

```

Сборка ядра с помощью oldconfig

* проделаны первые шаги из предыдущего этапа и использованы команды из пособия к лекции

```
cp /boot/config* .config && make oldconfig && make && make install &&make modules_install

```

в результате в grub можно загрузиться с новым ядром


</p>
</details>

