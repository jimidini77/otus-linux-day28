# otus-linux-day28
## *DHCP, PXE*

# **Prerequisite**
- Host OS: Debian 12.0.0
- Guest OS: CentOS 8.4
- VirtualBox: 6.1.40
- Vagrant: 2.3.2
- Ansible: 2.13.4

# **Содержание ДЗ**

- Следуя шагам из документа https://docs.centos.org/en-US/8-docs/advanced-install/assembly_preparing-for-a-network-install установить и настроить загрузку по сети для дистрибутива CentOS8.
- В качестве шаблона воспользуйтесь репозиторием https://github.com/nixuser/virtlab/tree/main/centos_pxe.
- Поменять установку из репозитория NFS на установку из репозитория HTTP.
- Настроить автоматическую установку для созданного kickstart файла (*) Файл загружается по HTTP.

# **Выполнение**

Создан `Vagrantfile` для развёртывания двух машин `pxeserver` , `pxeclient`

Задания ansible-playbook для настройки `pxeserver` :

- Конфигурирование репозиториев:
```yml
    - name: Setup repository
      replace:
        path: "{{ item }}"
        regexp: 'mirrorlist'
        replace: '#mirrorlist'
      with_items:
        - /etc/yum.repos.d/CentOS-Linux-AppStream.repo
        - /etc/yum.repos.d/CentOS-Linux-BaseOS.repo

    - name: Setup repository
      replace:
        path: "{{ item }}"
        regexp: '#baseurl=http://mirror.centos.org'
        replace: 'baseurl=http://vault.centos.org'
      with_items:
        - /etc/yum.repos.d/CentOS-Linux-AppStream.repo
        - /etc/yum.repos.d/CentOS-Linux-BaseOS.repo
```
- Установка пакетов:
```yml
    - name: Install packages
      yum:
        name:
        - wget
        - epel-release
        - httpd
        - tftp-server
        - dhcp-server
        state: present
        update_cache: true
```
- Скачивание дистрибутива ОС (длительный процесс) и копирование файлов в каталог, из которого будет выполняться установка:
```yml
    - name: Download ISO image CentOS 8.4.2105
      get_url:
        url: https://mirror.sale-dedic.com/centos/8.4.2105/isos/x86_64/CentOS-8.4.2105-x86_64-dvd1.iso
        dest: ~/CentOS-8.4.2105-x86_64-dvd1.iso
        mode: '0755'
        validate_certs: no

    - name: Create ISO directory
      file:
        path: /iso
        state: directory
        mode: '0755'

    - name: Mount ISO image
      mount:
        path: /mnt
        src: /root/CentOS-8.4.2105-x86_64-dvd1.iso
        fstype: iso9660
        opts: ro,loop
        state: mounted

    - name: copy ALL files from /mnt to /iso
      copy:
        src: /mnt/
        dest: /iso
        remote_src: yes
        directory_mode: yes
```

- Настройка веб-сервера:
```yml
    - name: set up httpd config
      template:
        src: pxeboot.conf
        dest: /etc/httpd/conf.d/pxeboot.conf
        owner: root
        group: root
        mode: 0640

    - name: restart httpd
      service:
        name: httpd
        state: restarted
        enabled: true
```

- Настройка tftp-сервера, копирование файлов загружаемых pxe-клиентом:
```yml
    - name: Create TFTP directory
      file:
        path: /var/lib/tftpboot/pxelinux.cfg
        state: directory
        mode: '0755'

    - name: Setup pxelinux
      template:
        src: default
        dest: /var/lib/tftpboot/pxelinux.cfg/default
        owner: root
        group: root
        mode: 0644

    - name: extract packages syslinux
      shell: rpm2cpio /iso/BaseOS/Packages/syslinux-tftpboot-6.04-5.el8.noarch.rpm | cpio -dimv

    - name: copy files to TFTP share
      copy:
        src: /home/vagrant/tftpboot/{{ item }}
        dest: /var/lib/tftpboot/{{ item }}
        mode: '0644'
        remote_src: true
      with_items:
        - pxelinux.0
        - ldlinux.c32
        - libmenu.c32
        - libutil.c32
        - menu.c32
        - vesamenu.c32

    - name: copy initrd and vmlinuz files to TFTP share
      copy:
        src: "/iso/images/pxeboot/{{ item }}"
        dest: "/var/lib/tftpboot/{{ item }}"
        mode: '0755'
        remote_src: true
      with_items:
        - initrd.img
        - vmlinuz

    - name: restart tftp-server
      service:
        name: tftp.service
        state: restarted
        enabled: true        
```
- Конфигурирование DHCP-сервера:
```yml
    - name: set up dhcp-server
      template:
        src: dhcpd.conf
        dest: /etc/dhcp/dhcpd.conf
        mode: '0644'

    - name: restart dhcp-server
      service:
        name: dhcpd
        state: restarted
        enabled: true
```
В каталоге `/ansible/templates/` содержатся предварительно созданные конфигурационные файлы:
- `pxeboot.conf` для веб-сервера
- `dhcpd.conf` для dhcp-сервера
- `default` меню загрузки PXE
- `ks.cfg` Kickstart-файл для автоматической установки, дефолтное значение меню установки настроено на него

После развёртывания и конфигурирование `pxeserver` при запуск `pxeclient` выполняется автоматическая установка ОС на машину. Запуск `pxeclient` из Vagrant завершается по таймауту:

```
...
==> pxeclient: Running 'pre-boot' VM customizations...
==> pxeclient: Booting VM...
==> pxeclient: Waiting for machine to boot. This may take a few minutes...
    pxeclient: SSH address: 127.0.0.1:22
    pxeclient: SSH username: vagrant
    pxeclient: SSH auth method: private key
    pxeclient: Warning: Authentication failure. Retrying...
    pxeclient: Warning: Authentication failure. Retrying...
...
    pxeclient: Warning: Authentication failure. Retrying...
    pxeclient: Warning: Authentication failure. Retrying...
Timed out while waiting for the machine to boot. This means that
Vagrant was unable to communicate with the guest machine within
the configured ("config.vm.boot_timeout" value) time period.

If you look above, you should be able to see the error(s) that
Vagrant had when attempting to connect to the machine. These errors
are usually good hints as to what may be wrong.

If you're using a custom box, make sure that networking is properly
working and you're able to connect to the machine. It is a common
problem that networking isn't setup properly in these boxes.
Verify that authentication configurations are also setup properly,
as well.

If the box appears to be booting properly, you may want to increase
the timeout ("config.vm.boot_timeout") value.
root@nas:/study/day28# client_loop: send disconnect: Connection reset
```

![Screenshot](https://github.com/jimidini77/otus-linux-day28/blob/main/Screenshot01.png?raw=true)

![Screenshot](https://github.com/jimidini77/otus-linux-day28/blob/main/Screenshot02.png?raw=true)

![Screenshot](https://github.com/jimidini77/otus-linux-day28/blob/main/Screenshot03.png?raw=true)

# **Результаты**

Полученный в ходе работы `Vagrantfile` и плейбук Ansible помещены в публичный репозиторий:

- **GitHub** - https://github.com/jimidini77/otus-linux-day28
