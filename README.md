# Часть 1
## Восстановление забытого пароля от IPMI различных вендоров
> ## Dell iDRAC
1. Перейдем на сайт поддрежки Dell https://www.dell.com/support
2. В поиске введем модель сервера, от которого забыли пароль и выбераем тип ОС установленной локально (bare metal) на сервере
3. Нам нужен пакет Dell EMC iDRAC Tools for OS_NAME. В этом пакете нас интересует утилита **racadm**
    * Также можно найти утилиту **racadm** просто введя запрос Dell iDRAC Tools for OS_NAME в поиске на сайте поддрежки
4. Разберем установку утилиты **racadm** на базе Rocky Linux 9

* **Все команды выполняются от имени root**

     * Подключемся к ОС по ssh и скачаем архив с утилитами любым способом
     * Распакуем архив
        ```bash
            tar -xf Dell-iDRACTools-Web-LX-*.tar.gz
        ```
     * Перейдем в директорию с файлами **racadm**
        ```bash
            cd iDRACTools/racadm/
        ```
     * В директории мы можем запустить скрипт установки или вручную установить три RPM пакета из директории OS_NAME/x86_64/
        ```bash
            # В моем случае это директория RHEL9/x86_64
            rpm -i RHEL9/x86_64/srvadmin-*
        ```
        ```bash
            # В скрипте установки есть проверка ОС, Rocky Linux 9 в ней нет, из-за этого скрипт завершается ошибкой
            # Нужно добавить в строку проверки ОС еще одно условие "или" "$ID" == "rocky"
            # Рабочий вариант строки выглядит следующим образом:
            elif [[ "$ID" == "rhel" || "$ID" == "centos" || "$ID" == "rocky" ]] && [  "$VER" == "9" ]; then
            # Теперь можно запустить скрипт и он отработает корректно
            ./install_racadm.sh
            # Получим такой вывод
            Verifying...                          ################################# [100%]
            Preparing...                          ################################# [100%]
            Updating / installing...
                1:srvadmin-hapi-11.0.0.0-5139.el9  ################################# [ 33%]
                2:srvadmin-argtable2-11.0.0.0-5139.################################# [ 67%]
                3:srvadmin-idracadm7-11.0.0.0-5139.################################# [100%]
                **********************************************************
                After the install process completes, you may need
                to logout and then login again to reset the PATH
                variable to access the RACADM CLI utilities

                **********************************************************
            # После этого можем перелогиниться для загрузки новых переменных окружения
        ```
5. После установки утилиты можем сбросить пароль пользователя по умолчанию (2) root командой ниже, выполнив ее из под root пользователя локальной ОС
    ```bash
        racadm set iDRAC.Users.2.Password Новый_пароль_рута
    ```
#### Используемые материалы
* https://www.dell.com/support/home/en-lt/drivers/driversdetails?driverid=dfhk6
* https://www.dell.com/support/manuals/en-lt/idrac9-lifecycle-controller-v3.0-series/idrac_3.00.00.00_ug/changing-the-default-login-password-using-racadm?guid=guid-ff412eb0-fd83-432c-92f8-79472c5a2077&lang=en-us
* https://www.dell.com/community/PowerEdge-Hardware-General/Unluckly-I-forgot-my-dell-server-R720-IDRAC-password-now-I-can-t/td-p/4458788
* https://www.techbeatly.com/how-to-install-racadm-for-idrac-on-rhel-linux/
* https://gist.github.com/inscite/e5c6f95fbf25379c400e9ea76f2360ec
* https://www.ispcolohost.com/2020/04/06/resetting-a-dell-idrac-password-from-inside-vsphere-esxi/
> ## HPE ILO
1. В случае HP для сброса пароля используется утилита **hponcfg** установленная в ОС развернутой локально (bare metal) на сервере 
2. Перейдем на сайт подержи HP https://support.hpe.com/ и наберем поиске ***HPE Lights-Out Online Configuration Utility for Windows/Linux***
    * Для ESXi необходимо скачать бандл ***HPE ESXi Utilities Offline Bundle for VMware HYPERVISOR_VERSION***, в котором находиться утилита **hponcfg**
3. Разберем процесс установки утилиты **hponcfg** на примере Rocky Linux 9

* **Все команды выполняются от имени root**

    * Подключимся к ОС по ssh и скачаем пакет с утилитами любым способом
      ```bash
          wget https://downloads.hpe.com/pub/softlib2/software1/pubsw-linux/p215998034/v206544/hponcfg-6.0.0-0.x86_64.rpm
      ```
    * Установим утилиту по инструкции с ее странице на сайте HPE
        ```bash
            rpm -ivh hponcfg-6.0.0-0.x86_64.rpm
        ```
    * Проверим корректность установки
        ```bash
            hponcfg --version
        ```
4. Теперь согласно [инструкции](https://support.hpe.com/hpesc/public/docDisplay?docId=emr_na-c03487111), необходимо создать файл xml с нужными параметрами *mod_user user_login* и *password value*
    ```bash
        vi passwd_reset_ilo.xml
    ```
    ```xml
        <ribcl version="2.0">  
        <login user_login="Administrator" password="boguspassword">  
        <user_info mode="write">   
        <mod_user user_login="Administrator">    
        <password value="newpass"/>    
        </mod_user>  
        </user_info>  
        </login>       
        
        </RIBCL>
    ```
5. После этого можем передать созданный файл утилите для сброса пароля и записи результата выполнения в файл log.txt
    ```bash
        hponcfg -f passwd_reset_ilo.xml -l log.txt
    ```
#### Используемые материалы
* https://support.hpe.com/hpesc/public/docDisplay?docId=emr_na-c03487111
* https://support.hpe.com/connect/s/softwaredetails?language=en_US&softwareId=MTX_094cdfb6840f48389ba985bf92&tab=Installation+Instructions
* https://support.hpe.com/hpesc/public/docDisplay?docId=emr_na-a00007610en_us
* https://www.it-react.com/index.php/2022/04/23/reset-or-configure-hp-ilo-directly-from-esxi-host/
* https://www.vcloudnine.de/reset-the-hp-ilo-administrator-password-with-hponcfg-on-esxi/
* https://servermall.ru/blog/ipmi-ot-hpe-nachalnaya-nastroyka-ilo-5/
> ## Supermicro BMC/IPMI
1. В системах Supermicro для сброса пароля используется утилита **ipmicfg**
2. Скачать утилиту можно на [официальном сайте](https://www.supermicro.com/en/support/resources/downloadcenter/smsdownload) или с [официального ftp](https://www.supermicro.com/wdl/utility/IPMICFG/)
3. Разберем процесс установки утилиты **ipmicfg** на примере Rocky Linux 9

* **Все команды выполняются от имени root**

    * Подключимся к ОС по ssh и скачаем пакет с утилитой любым способом, например wget
      ```bash
          wget https://www.supermicro.com/wdl/utility/IPMICFG/IPMICFG_1.34.1_build.221026.zip
      ```
    * Распакуем архив
      ```bash
          unzip IPMICFG_1.34.1_build.221026.zip
      ```
    * Для удобства использования добавим файл IPMICFG_1.34.1_build.221026/Linux/64bit/IPMICFG-Linux.x86_64 в перменную PATH
      ```bash
          # Создадим директорию с подходяшим именем в /opt
          mkdir /opt/supermicro/ipmicfg
          # Скопируем в новую директорию нужный нам файл с именем ipmicfg
          cp IPMICFG_1.34.1_build.221026/Linux/64bit/IPMICFG-Linux.x86_64 /opt/supermicro/ipmicfg
          # Экспортируем директорию в $PATH для текущего пользователя, отредактировав .bashrc
          vi .bashrc
          # Заменим строку export PATH на export PATH=$PATH:/opt/supermicro/ipmicfg
      ```
4. Теперь мы можем приступить к сбросу пароля. Для начала определим ID пользователя
    ```bash
        ipmicfg -user list
    ```
5. После этого можно приступить к сбросу пароля интересующего нас пользователся
    ```bash
        ipmicfg –user setpwd User_ID newpassword
    ```
#### Используемые материалы
* https://www.supermicro.com/wdl/utility/IPMICFG/IPMICFG_UserGuide.pdf
* https://www.servethehome.com/reset-supermicro-ipmi-password-default-lost-login/
* https://winitpro.ru/index.php/2019/08/30/ipmi-nastrojka-i-udalennoe-upravlenie-serverom/

# Часть 2
## Ansible роль для конфигурации DHCP сервера
> ## Функции
* Что умеет эта роль?
  * Устанавливать пакет DHCP сервера вне зависимости от версии RedHat like ОС
  * Добавлять 67 порт в разрешения firewall вне зависимости от того что используется (firewalld или iptables)
  * Проверять наличие нужных директорий и создавать их
  * Проверять конфиг DHCP сервера перед тем как добавить его на сервер
* Что не умеет эта роль?
  * Устанавливать пакет DHCP сервера на Debian like ОС
  * Проверять сетевые интерфейсы на наличие соответсвия подсетей указанных в конфиге DHCP сервера
  * Определять, был ли до этого настроен DHCP сервер и дополнять уже существующий конфиг
  * Определять, есть ли в firewall запрещающие правила перед тем как добавлять разрешающие
#### Из всего перечисленного выше, можно сделать вывод что данную роль стоит применять с осторожностью. В первый раз лучше запустить с --diff --check и убедиться что ничего не сломается.
#### Также стоит учитывать что DHCP сервер будет слушать только на том сетевом интерфейсе где адрес соответствует объявленной подсети в файле конфигурации DHCP сервера. Если соответсвия адресов нет, то демон dhcpd упадет в ошибку.
#### Например так:
  ```bash
    ansible-playbook playbooks/dhcp.yml --diff --check
  ```
> ## Зависимости
  * community.general для модуля iptables_state
> ## Доступные переменные
#### *Глобальные переменные*
| Переменная                        | Комментарий                                                            |
| :-------------------------------- | :--------------------------------------------------------------------- |
| `dhcp_server_domain_name`         | Доменное имя сервер                                                    |
| `dhcp_server_dns_servers`         | DNS сервера                                                            |
| `dhcp_server_default_lease_time`  | Время аренды адреса по умолчанию                                       |
| `dhcp_server_max_lease_time`      | Максимальное время аренды адреса по уполчанию                          |
| `dhcp_server_authoritative`       | Является ли сервер ответственным DHCP                                  |
| `dhcp_server_log_facility`        | Глубина логирования                                                    |

#### *Переменные подсети*
| Переменная                        | Комментарий                                                            |
| :-------------------------------- | :--------------------------------------------------------------------- |
| `ip`                              | IP адреса сети. (прим `10.1.1.0`)                                      |
| `netmask`                         | Маска подсети (прим `255.255.255.0`)                                   |
| `range_start`                     | Начало пула выдаваемых адресов (прим `10.1.1.2`)                       |
| `range_end`                       | Конец пула выдаваемых адресов  (прим `10.1.1.250`)                     |
| `domain_name`                     | Имя домена подсети                                                     |
| `dns_servers`                     | Адреса DNS серверов подсети                                            |
| `routers`                         | Адрес маршрутизатора подсети                                           |
| `broadcast_address`               | Broadcast адрес (прим `10.1.1.255`)                                    |
| `default_lease_time`              | Время аренды адреса по умолчанию                                       |
| `max_lease_time`                  | Максимальное время аренды адреса подсети                               |

### Также есть возможность настроить интерфейсы сервера для Multihomed DHCP

#### *Переменные интерфейсов*
| Переменная                        | Комментарий                                                            |
| :-------------------------------- | :--------------------------------------------------------------------- |
| `name`                            | Уникальное имя                                                         |
| `mac`                             | MAC адрес сетевого интерфейса                                          |
| `fixip`                           | Filename to request for boot                                           |

> ## Протестированно на:
* CentOS 7
* AlmaLinux 8
* RockyLinux 9 

#### Используемые материалы
* https://github.com/bertvv/ansible-role-dhcp
* https://gitlab.com/paddyoneill/ansible-role-isc-dhcp-server/
* https://jinja.palletsprojects.com/en/3.1.x/templates/#if-expression
* https://access.redhat.com/documentation/ru-ru/red_hat_enterprise_linux/7/html/networking_guide/sec-dhcp-configuring-server
* https://access.redhat.com/documentation/ru-ru/red_hat_enterprise_linux/7/html/networking_guide/sec-configuring_a_multihomed_dhcp_server
* https://wiki.merionet.ru/servernye-resheniya/15/nastrojka-dhcp-servera-na-centos-ili-ubuntu/