.. post:: Aug 19, 2009
   :tags: computisses
   :author: Juca Crispim


Instalar  Debian a partir de outro unix-like com debootstrap
============================================================

Este documento mostra como instalar um sistema Debian from scratch a partir de
outro sistema operacional unix-like usando o debootstrap. A primeira coisa a
fazer é baixar o debootstrap.

    Bootstrap a basic Debian system debootstrap is used to create a Debian base
    system from scratch, without requiring the availability of dpkg or apt. It
    does this by downloading .deb files from a mirror site, and carefully
    unpacking them into a directory which can eventually be chrooted into.

Aqui você encontra o debootstrap para baixar.

.. code-block:: sh

   # cd /usr/local/src
   # wget -c http://ftp.debian.org/debian/pool/main/d/debootstrap/debootstrap_1.0.15_all.deb

Depois de baixar é a vez de descompactar e instalar.

.. code-block:: sh

   # ar -x debootstrap_1.0.15_all.deb
   # cd / # zcat /usr/local/data.tar.gz | tar xv

Agora já com o debootstrap instalado podemos começar a instalação do nosso
Debian novo. Para seguir em frente é necessário que já se tenha uma partição
criada e montada para o novo sistema. A partição que usarei neste caso é
identificada como /dev/sda1 e está montada em  /mnt. A sintaxe básica do
debootstrap é: debootstrap --arch <arquitetura> <distribuição> <diretório alvo>
<repositório>. Com isto será instalado um sistema mínimo no diretório alvo
indicado

.. code-block:: sh

   # /usr/sbin/debootstrap --arch amd64 squeeze /mnt http://ftp.br.debian.org/debian

Com o sistema básico instalado é hora de dar um chroot para o novo sistema. Mas
antes disso, vamos montar algumas coisas necessárias.

.. code-block:: sh

   # mount -o bind -t proc /proc /mnt/proc
   # mount -o bind /dev /mnt/dev
   # mount -o bind /dev/pts /mnt/dev/pts
   # mount -o bind /sys /mnt/sys

E agora sim, dar o chroot para o novo sistema:

.. code-block:: sh

   # LANG=C chroot /mnt

Agora que já estamos dentro do novo sistema vamos configurá-lo

A primeira coisa a configurar é o debconf e deixá-lo com prioridade baixa
(maníaco é foda...)

.. code-block::

   # dpkg-reconfigure debconf

Agora ajustando o horário:

.. code-block:: sh

   # dpkg-reconfigure tzdata

Agora vamos configurar o apt.

.. code-block:: sh

   # nano /etc/apt/sources.list

Eu acrescentei os seguintes repositórios:

.. code-block:: sh

   deb-src http://ftp.br.debian.org/debian/ testing main
   deb http://security.debian.org/ testing/updates main
   deb-src http://security.debian.org/ testing/updates main

Depois de adicionar os novos repositórios, recarregar as informações de
pacotes:

.. code-block:: sh

   # aptitude update

Agora vamos configurar teclado e idioma. Para isto instalaremos os pacotes
locales e console-data

.. code-block:: sh

   # aptitude install locales console-data

Ainda nos falta um kernel... Vamos a ele! Procure pelo kernel mais adequado
para você:

.. code-block:: sh

   # aptitude search linux-image

E depois de achar seu kernel, instale-o:

.. code-block:: sh

   # aptitude install linux-image-2.6.30-1-amd64

Vamos configurar também o fstab.

.. code-block:: sh

    #Raiz em /dev/sda1 UUID=f2f94e87-327f-4948-88ef-0338bb23848e /     ext4    relatime,errors=remount-ro 0       1
    # swap em  /dev/sda2 UUID=71c38be9-a1b4-4f45-82b3-c04bdeba533d none            swap    sw              0       0

Depois de configurar  o fstab, vamos montar tudo.

.. code-block:: sh

   # mount -a

Para iniciar o nosso novo sistema precisamos de um bootloader. Você pode usar o
bootloader que você já usa no seu outro sistema operacional ou instalar um novo.
Para instalar um bootloader novo você usa:

.. code-block:: sh

   # aptitude install grub

Mas no caso deste exemplo, vou usar o grub já existente da outra distro. Seguem
as linhas adicionadas ao arquivo de configuração do grub.

.. code-block:: sh

    title        Debian uuid        f2f94e87-327f-4948-88ef-0338bb23848e kernel        /boot/vmlinuz-2.6.30-1-amd64 root=UUID=f2f94e87-327f-4948-88ef-0338bb23848e ro quiet splash initrd        /boot/initrd.img-2.6.30-1-amd64

Agora já estamos quase no fim. Mas antes, vou instalar uns pacotes que eu uso:
pppoeconf para configurar a rede depois do reboot,  emacs que é minha
ferramenta de trabalho e usplash porque eu gosto, oras!

.. code-block:: sh

   # aptitude install emacs23-nox pppoeconf usplash

Os pacotes que preciso já estão instalados, então agora é só configurar os
usuários. Primeiro configurar a senha de root e depois adicionar um usuário
comum pra mim.

.. code-block:: sh

   # passwd
   # adduser juca

Agora tudo pronto! Já temos um Debian novinho em folha. Desmonte tudo o que
foi montado, reinicie a máquina, escolha o Debian no seu bootloader e...
Divirta-se!
