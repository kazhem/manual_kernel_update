# HW1 С чего начинается Linux

# Установка ПО
Выполняется на macOS Monteray core i5

## Установка Virtualbox
```
brew install --cask virtualbox
```
## Установка Vagrant
```
brew install --cask vagrant
brew install --cask vagrant-manager
```
## Установка Packer
```
brew tap hashicorp/tap
brew install hashicorp/tap/packer
```
# **Kernel update**

### **Клонирование и запуск**

Для запуска рабочего виртуального окружения необходимо зайти через браузер в GitHub под своей учетной записью и выполнить `fork` данного репозитория: https://github.com/dmitry-lyutenko/manual_kernel_update

После этого данный репозиторий необходимо склонировать к себе на рабочую машину. Для этого воспользуемся ранее установленным приложением `git`, при этом в `<user_name>` будет имя уже вашего репозитрия:
```
git clone git@github.com:kazhem/manual_kernel_update.git
```
В текущей директории появится папка с именем репозитория. В данном случае `manual_kernel_update`. Ознакомимся с содержимым:
```
cd manual_kernel_update
ls -1
manual
packer
Vagrantfile
```
Здесь:
- `manual` - директория с данным руководством
- `packer` - директория со скриптами для `packer`'а
- `Vagrantfile` - файл описывающий виртуальную инфраструктуру для `Vagrant`

Запустим виртуальную машину и залогинимся:
```
vagrant up
...
```
**Ошибка**:
```
There was an error while executing `VBoxManage`, a CLI used by Vagrant
for controlling VirtualBox. The command and stderr is shown below.

Command: ["startvm", "35823be1-fd41-4dac-8240-c419a8af810c", "--type", "headless"]

Stderr: VBoxManage: error: The virtual machine 'manual_kernel_update_kernel-update_1644700701530_13951' has terminated unexpectedly during startup with exit code 1 (0x1)
VBoxManage: error: Details: code NS_ERROR_FAILURE (0x80004005), component MachineWrap, interface IMachine
```
Решение:
```
Go to Security and Privacy / General in your System Settings. You will see that software from Oracle has been blocked. Allow it, and the software will work. The 'allow Oracle' option only exists for 30mins after the installer error
Или переустановка virtualbox и повторение выше
```
Еще раз:
```
vagrant up
...
==> kernel-update: Importing base box 'centos/7'...
...
==> kernel-update: Booting VM...
...
==> kernel-update: Setting hostname...

vagrant ssh
[vagrant@kernel-update ~]$ uname -r
3.10.0-1127.el7.x86_64
```
Теперь приступим к обновлению ядра.
### **kernel update**

Подключаем репозиторий, откуда возьмем необходимую версию ядра.
```
sudo yum install -y http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm

...
Running transaction
  Installing : elrepo-release-7.0-3.el7.elrepo.noarch                                                                           1/1
  Verifying  : elrepo-release-7.0-3.el7.elrepo.noarch                                                                           1/1

Installed:
  elrepo-release.noarch 0:7.0-3.el7.elrepo

Complete!
```

В репозитории есть две версии ядер **kernel-ml** и **kernel-lt**. Первая является наиболее свежей стабильной версией, вторая это стабильная версия с длительной поддержкой, но менее свежая, чем первая. В данном случае ядро 5й версии будет в  **kernel-ml**.

Поскольку мы ставим ядро из репозитория, то установка ядра похожа на установку любого другого пакета, но потребует явного включения репозитория при помощи ключа ```--enablerepo```.

Ставим последнее ядро:

```
sudo yum --enablerepo elrepo-kernel install kernel-ml -y

...
Transaction test succeeded
Running transaction
  Installing : kernel-ml-5.16.9-1.el7.elrepo.x86_64                                                                             1/1
  Verifying  : kernel-ml-5.16.9-1.el7.elrepo.x86_64                                                                             1/1

Installed:
  kernel-ml.x86_64 0:5.16.9-1.el7.elrepo

Complete!
```

### **grub update**
После успешной установки нам необходимо сказать системе, что при загрузке нужно использовать новое ядро. В случае обновления ядра на рабочих серверах необходимо перезагрузиться с новым ядром, выбрав его при загрузке. И только при успешно прошедших загрузке нового ядра и тестах сервера переходить к загрузке с новым ядром по-умолчанию. В тестовой среде можно обойти данный этап и сразу назначить новое ядро по-умолчанию.

Обновляем конфигурацию загрузчика:
```
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
...
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-5.16.9-1.el7.elrepo.x86_64
Found initrd image: /boot/initramfs-5.16.9-1.el7.elrepo.x86_64.img
Found linux image: /boot/vmlinuz-3.10.0-1127.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-1127.el7.x86_64.img
done
```
Выбираем загрузку с новым ядром по-умолчанию:
```
sudo grub2-set-default 0
```

Перезагружаем виртуальную машину:
```
sudo reboot
```
После перезагрузки виртуальной машины (3-4 минуты, зависит от мощности хостовой машины) заходим в нее и выполняем:

```
uname -r
...
5.16.9-1.el7.elrepo.x86_64
```
# **Packer**
Теперь необходимо создать свой образ системы, с уже установленым ядром 5й версии. Для это воспользуемся ранее установленной утилитой `packer`. В директории `packer` есть все необходимые настройки и скрипты для создания необходимого образа системы.

### **packer provision config**
Файл `centos.json` содержит описание того, как произвольный образ. Полное описание можно найти в документации к `packer`. Обратим внимание на основные секции или ключи.

Создаем переменные (`variables`) с версией и названием нашего проекта (artifact):
```
    "artifact_description": "CentOS 7.9 with kernel 5.x",
    "artifact_version": "7.9.2009",
```

В секции `builders` задаем исходный образ, для создания своего в виде ссылки и контрольной суммы. Параметры подключения к создаваемой виртуальной машине.

```
    "iso_url": "http://mirror.corbina.net/pub/Linux/centos/7.9.2009/isos/x86_64/CentOS-7-x86_64-Minimal-2009.iso",
    "iso_checksum": "07b94e6b1a0b0260b94c83d6bb76b26bf7a310dc78d7a9c7432809fb9bc6194a",
    "iso_checksum_type": "sha256", # deprecated
```
В секции `post-processors` указываем имя файла, куда будет сохранен образ, в случае успешной сборки

```
    "output": "centos-{{user `artifact_version`}}-kernel-5-x86_64-Minimal.box",
```

В секции `provisioners` указываем каким образом и какие действия необходимо произвести для настройки виртуальой машины. Именно в этой секции мы и обновим ядро системы, чтобы можно было получить образ с 5й версией ядра. Настройка системы выполняется несколькими скриптами, заданными в секции `scripts`.

```
    "scripts" :
      [
        "scripts/stage-1-kernel-update.sh",
        "scripts/stage-2-clean.sh"
      ]
```
Скрипты будут выполнены в порядке указания. Первый скрипт включает себя набор команд, которые мы ранее выполняли вручную, чтобы обновить ядро. Второй скрипт занимается подготовкой системы к упаковке в образ. Она заключается в очистке директорий с логами, временными файлами, кешами. Это позволяет уменьшить результирующий образ. Более подробно можно ознакомиться с ними в директории `packer/scripts`

Секция `post-processors` описывает постобработку виртуальной машины при ее выгрузке. Мы указыаем имя файла, в который будет сохранен результат (artifact). Обратите внимание, что имя задается на основе ранее созданной пользовательской переменной `artifact_version` значение которой мы задали ранее:

```
    "output": "centos-{{user `artifact_version`}}-kernel-5-x86_64-Minimal.box",
```

### **packer build**
Для создания образа системы достаточно перейти в директорию `packer` и в ней выполнить команду:

```
packer build centos.json
...
==> Builds finished. The artifacts of successful builds are:
--> centos-7.9: 'virtualbox' provider box: centos-7.9.2009-kernel-5-x86_64-Minimal.box
```

Если все в порядке, то, согласно файла `config.json` будет скачан исходный iso-образ CentOS, установлен на виртуальную машину в автоматическом режиме, обновлено ядро и осуществлен экспорт в указанный нами файл. Если не вносилось изменений в предложенные файлы, то в текущей директории мы увидим файл `centos-7.9.2009-kernel-5-x86_64-Minimal.box`. Он и является результатом работы `packer`.


### **vagrant init (тестирование)**
Проведем тестирование созданного образа. Выполним его импорт в `vagrant`:

```
vagrant box add --name centos-7-5 centos-7.9.2009-kernel-5-x86_64-Minimal.box
...
==> box: Successfully added box 'centos-7-5' (v0) for 'virtualbox'!
```

Проверим его в списке имеющихся образов (ваш вывод может отличаться):

```
# vagrant box list
centos-7-5 (virtualbox, 0)
centos/7   (virtualbox, 2004.01)
```

Он будет называться `centos-7-5`, данное имя мы задали при помощи параметра `name` при импорте.

Теперь необходимо провести тестирование полученного образа. Для этого создадим новый Vagrantfile или воспользуемся имеющимся. Для нового создадим директорию `test` и в ней выполним:

```
vagrant init centos-7-5
```

Для имеющегося произведем замену значения `box_name` на имя импортированного образа. Соотвествующая строка примет вид:

```
:box_name => "centos-7-5",
```

Теперь запустим виртуальную машину, подключимся к ней и проверим, что у нас в ней новое ядро:

```
vagrant up
```
ОШИБКА:
```
==> default: Mounting shared folders...
    default: /vagrant => /Users/kazhem/develop/otus/linux/manual_kernel_update/test
Vagrant was unable to mount VirtualBox shared folders. This is usually
because the filesystem "vboxsf" is not available. This filesystem is
made available via the VirtualBox Guest Additions and kernel module.
Please verify that these guest additions are properly installed in the
guest. This is not a bug in Vagrant and is usually caused by a faulty
Vagrant box. For context, the command attempted was:

mount -t vboxsf -o uid=1000,gid=1000,_netdev vagrant /vagrant

The error output from the command was:
```

```
vagrant ssh
```

и внутри виртуальной машины:

```
[vagrant@kernel-update ~]$ uname -r
5.16.9-1.el7.elrepo.x86_64
```

Если все в порядке, то машина будет запущена и загрузится с новым ядром. В данном примере это `5.16.9`.

Удалим тестовый образ из локального хранилища:
```
vagrant box remove centos-7-5
```
---
# **Vagrant cloud**

Поделимся полученным образом с сообществом. Для этого зальем его в Vagrant Cloud. Можно залить через web-интерфейс, но так же `vagrant` позволяет это проделать через CLI.
Логинимся в `vagrant cloud`, указывая e-mail, пароль и описание выданого токена (можно оставить по-умолчанию)
```
vagrant cloud auth login
Vagrant Cloud username or email: kazhem
Password (will be hidden):
Token description (Defaults to "Vagrant login from DS-WS"):
...
You are now logged in.
```
Теперь публикуем полученный бокс:
```
vagrant cloud publish --release kazhem/centos-7-5 1.0 virtualbox \
        ../packer/centos-7.9.2009-kernel-5-x86_64-Minimal.box
```
Здесь:
 - `cloud publish` - загрузить образ в облако;
 - `release` - указывает на необходимость публикации образа после загрузки;
 - `kazhem/centos-7-5` - `username`, указаный при публикации и имя образа;
 - `1.0` - версия образа;
 - `virtualbox` - провайдер;
 - `../packer/centos-7.9.2009-kernel-5-x86_64-Minimal.box` - имя файла загружаемого образа;

После успешной загрузки вы получите сообщение:

```
Complete! Published <username>/centos-7-5
tag:             <username>/centos-7-5-cli
username:        <username>
name:            centos-7-5
private:         false
...
providers:       virtualbox
```

В результате создан и загружен в `vagrant cloud` образ виртуальной машины. Данный подход позволяет создать базовый образ виртульной машины с необходимыми обновлениями или набором предустановленного ПО. К примеру при создании MySQL-кластера можно создать образ с предустановленным MySQL, а при развертывании нужно будет добавить или изменить только настройки (то есть отличающуюся часть). Таким образом существенно экономя затрачиваемое время.

# **Заключение**

Результат выполнения ранее описанных действий по клонированию базового репозитория, созданию своего, создание кастомного образа с обновленным ядром и его публикация является необходимым для получения зачета по базовому домашнему заданию. Для проверки вам будет необходимо прислать ссылку на ваш репозиторий в чат с преподавателем в Личном кабинете. Репозиторий, соотвественно, должен быть публичным. На все возникшие вопросы можно получить ответ в переписке с преподавателем в чате с преподавателем или, что более рекомендуется, в слаке вашей группы.


# ДЗ со *  Сборка ядра из исходников

## "Ручной способо"
### Загрузка и установка
#### Загрузим последнее ядро с https://www.kernel.org/
```
sudo yum install wget
```
```
wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.16.9.tar.xz
100%[============================================================================================>] 118 148 040  606KB/s   за 4m 57s

2022-02-13 17:15:40 (389 KB/s) - «linux--5.16.9.tar.xz» сохранён [118148040/118148040]
```

#### Распаковка:
```
tar xvf linux-5.16.9.tar.xz
...
linux-5.16.9/virt/kvm/vfio.h
linux-5.16.9/virt/lib/
linux-5.16.9/virt/lib/Kconfig
linux-5.16.9/virt/lib/Makefile
linux-5.16.9/virt/lib/irqbypass.c
```


#### Установка пакетов для сборки

```
sudo yum groupinstall "Development Tools"
sudo yum install ncurses-devel bison flex elfutils-libelf-devel openssl-devel centos-release-scl devtoolset-7-gcc*
```

### Конфигурирование ядра
Перейдем в каталог с исходниками ядра
```
cd linux-5.16.9
```
Скопируем существующий файл конфигурации
```
sudo cp -v /boot/config-$(uname -r) .config
...
«/boot/config-5.16.9-1.el7.elrepo.x86_64» -> «.config»
```
Чтобы внести изменения в файл конфигурации, выполните команду make:
```
make menuconfig
```
### Сборка ядра
Для начала сборки выполнить:
```
make
...
  CC [M]  drivers/net/ethernet/aquantia/atlantic/hw_atl2/hw_atl2_utils.o
  CC [M]  drivers/net/ethernet/aquantia/atlantic/hw_atl2/hw_atl2_utils_fw.o
  CC [M]  drivers/net/ethernet/aquantia/atlantic/hw_atl2/hw_atl2_llh.o
  CC [M]  drivers/net/ethernet/aquantia/atlantic/macsec/macsec_api.o
  CC [M]  drivers/net/ethernet/aquantia/atlantic/aq_macsec.o
  CC [M]  drivers/net/ethernet/aquantia/atlantic/aq_ptp.o
  LD [M]  drivers/net/ethernet/aquantia/atlantic/atlantic.o
...
```
После длительной сборки устновить модули
```
sudo make modules_install
```
И установить ядро
```
make install
```
После установки обновить загрузчик и перезагрузиться
```
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
sudo grub2-set-default 0
sudo reboot
```
После перезагрузки:
```
uname -r
...
5.16.9
```
