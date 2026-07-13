## Passo a passo da instalação:

Primeiro, instale alguns pacotes importantes importantes. Esses pacotes forncem as ferramentas necessárias para complilar o código, gerar imagens, instalar e configurar o bootloaader e testar a distribuição em máquinas virtuais.

``` bash
sudo apt update && sudo apt install -y build-essential libncurses-dev bison flex libssl-dev libelf-dev bc cpio wget xorriso grub-pc-bin grub-efi-amd64-bin grub-common mtools squashfs-tools qemu-system-x86 tar xz-utils  
```

Em seguida, instale o código fonte do kernel Linux versão 6.1.60. Ele será utilizado na compilação do kernel, gerar o initramfs e construir a imagem do sistema.

``` bash
sudo wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.1.60.tar.xz  
```

Extraia o arquivo instalado:

``` bash
sudo tar -xvf linux-6.1.60.tar.xz
```

Acesse o diretório do arquivo descompactado:

``` bash
cd linux-6.1.60
```

Agora, gere um arquivo de configuração padrão do kernel. Isso evita configurar manualmentente milhares de opções.

``` bash
sudo make defconfig
```

Porém, algumas opções essenciais permanecem desativadas na configuração padrão. Elas são importantes para a comunicação do entre kernel, CPU e disco ser possível. Use `menuconfig`. para habilitar essas opções.

``` bash
make menuconfig
```

O `menuconfig` abre uma interface de configuração que permite alterar as opções do kernel e gerenciar suas dependências.

Procure as opções abaixo utilizando a ferramenta de busca do `menuconfig`.

Caso alguma opção não esteja como Y (YES) na sua configuração, altere manualmente utilizando o `menuconfig` ou o VIM.


#escrever as opções

Agora, gere uma configuração que garanta que os módulos necessários estejam carregados no sistema atual. O `localmodconfig` mantém apenas os drivers em uso e remove opções desnecessárias.

``` bash
sudo make localmodconfig 
# Aperte Yes para tudo
```

Gera imagem + móulos:

``` bash
sudo make -j$(nproc)
```

Volta para a home:

``` bash
cd $HOME
```

Instale o código-fonte do BusyBox:

``` bash
sudo wget https://busybox.net/downloads/busybox-1.37.0.tar.bz2
```

Extraia o arquivo instalado:

``` bash
tar -xvf busybox-1.37.0.tar.bz2
```

Acesse o diretório do BusyBox:

``` bash
cd busybox-1.37.0
```

Crie o arquivo de configuração padrão do BusyBox:

``` bash
sudo make defconfig
```

Verifique se o BusyBox está configurado para modo estático:

``` bash
cat .config | grep "STATIC"
```

A saída deve ser algo como:

``` bash
# CONFIG_STATIC is not set
# CONFIG_FEATURE_LIBBUSYBOX_STATIC is not set
CONFIG_STATIC_LIBGCC=y
```

Caso a opção esteja desativada, habilite CONFIG_STATIC dento do arquivo .config:

``` bash
sed ´s/# CONFIG_STATIC is not set/CONFIG_STATIC=y´ .config
```

Desative também a opção CONFIG_TC no arquivo .config:

``` bash
sed ´s/CONFIG_TC/# CONFIG_TC is not set´ .config
```

Instale a biblioteca musl:

``` bash
sudo apt install -d musl
```

Em seguida a compilação:

``` bash
sudo make -j$(nproc)
```

Volte para home:

``` bash
cd $HOME
```

Crie o diretório do initramfs, responsável por carregar o sistema de arquivos inicial na memória:

``` bash
mkdir initramfs; cd initramfs; mkdir bin proc sys dev mnt
```

Copie o BusyBox para o diretótio bin:

``` bash
cp busybox-1.37.0/busybox initramfs/bin
```

Acesse o diretório bin:

``` bash
cd initramfs/bin
```

Conceda a permissão de execução ao BusyBox:

``` bash
chmod +755 busybox
```

Crie um script para gerar os links simbólicos dos comandos do BusyBox:

``` bash
touch script.sh
```

Dentro do arquivo script.sh, coloque:

``` bash
#!/bin/bash

for programa in $(./busybox --list); do
        ln -s busybox ./$programa
done
```

Torne o script.sh executável:

``` bash
chmod +755 script.sh
```

Execute o script:

``` bash
./script.sh
```

Retorne ao arquivo initramfs:

``` bash
cd ..
```

Crie um arquivo init:

``` bash
touch init
```

Dentro do arquivo init, coloque:

``` bash
#!/bin/sh

mount -t proc none /proc
mount -t sysfs none /sys
mount -t devtmpfs none /dev

echo "Inicializando a disto..."
```

Dê permissão para o arquivo init:

``` bash
chmod +755 init
```

Defina uma variável para o diretório da distribuição:

``` bash
export DISTRO=/home/administrador/distro
```

Acesse o diretório da distribuição:

``` bash
cd distro
```

Crie o diretório rootfs:

``` bash
mkdir rootfs
```

Crie os diretórios, a estrutura básica do sistema de arquivos:

``` bash
mkdir bin sbin dev proc mnt sys etc run root home lib var temp
```

Copie o BusyBox pro diretório bin:

``` bash
cp ~/busybox-1.37.0/busybox bin/
```

Entre no diretório bin:

``` bash
cd bin
```

Crie um arquivo temp:

``` bash
nvim temp
```

Dentro, coloque:

``` bash
#!/bin/bash

for prog in $(./busybox --list); do
        ln -s busybox ./$prog
done
```

Torne o script executável:

``` bash
chmod +755 temp
```

Execute o script:

``` bash
./temp
```

Remova o script temporário:

``` bash
rm temp
```

Acesse o diretório etc:

``` bash
cd ../etc
```

Crie o diretório init.d:

``` bash
mkdir init.d
```

Acesse o diretório var:

``` bash
cd ../var
```

Crie os diretórios:

``` bash
mkdir log run lib
```

Volte para o diretório etc:

``` bash
cd ../etc
```

Crie um arquivo chamado group:

``` bash
nvim group
```

Nele, coloque:

``` bash
#!/bin/sh

root:x:0:0:root:/root:/bin/sh
```

Dê a permissão:

``` bash
chmod +755 group
```

Crie outro arquivo:

``` bash
nvim inittab
```

Nele, coloque:

``` bash
#!/bin/sh

::sysinit:/etc/init.d/rcS

ttyS0::respawn:/bin/sh
tty1::respawn:/bin/sh

::ctrlaltdel:/bin/reboot
::shutdown:/bin/unmount -a -r
```

Dê a permissão:

``` bash
chmod +755 inittab
```

Entre no diretório init.d:

``` bash
cd init.d
```

Crie o arquivo rcS (run command SINGLE):

``` bash
nvim rcS
```

Dentro, coloque:

``` bash
#!/bin/sh

mount -t proc none /proc
mount -t sysfs none /sys
mount -t devtmpfs none /dev

mkdir /dev/pts
mount -t devpts none /dev/pts
```

Crie a sua variável da pasta da distro:

``` bash
export DISTRO=/home/administrador/distro
```

Vá para a sua pasta distro:

``` bash
cd distro
```

Crie o diretório rootfs:

``` bash
mkdir rootfs
```

Crie os diretórios:

``` bash
mkdir bin sbin dev proc mnt sys etc run root home lib var temp
```

Copie o busybox pro diretório bin:

``` bash
cp ~/busybox-1.37.0/busybox bin/
```

Entre no diretório bin:

``` bash
cd bin
```

Crie um arquivo temp:

``` bash
nvim temp
```

Dentro, coloque:

``` bash
#!/bin/bash

for prog in $(./busybox --list); do
        ln -s busybox ./$prog
done
```

Dê permissão:

``` bash
chmod +755 temp
```

Execute o script:

``` bash
./temp
```

Remova o script temporário:

``` bash
rm temp
```

Acesse a pasta etc:

``` bash
cd ../etc
```

Crie o diretório:

``` bash
mkdir init.d
```

Acesse o diretório var:

``` bash
cd ../var
```

Crie os diretórios:

``` bash
mkdir log run lib
```

Retorne para o diretório etc:

``` bash
cd ../etc
```

Crie um arquivo chamado group:

``` bash
nvim group
```

Nele, coloque:

``` bash
root:x:0:0:root:/root:/bin/sh
```

Dê a permissão:

``` bash
chmod +755 group
```

Crie outro arquivo:

``` bash
nvim inittab
```

Nele, coloque:

``` bash
::sysinit:/etc/init.d/rcS

ttyS0::respawn:/bin/sh
tty1::respawn:/bin/sh

::ctrlaltdel:/bin/reboot
::shutdown:/bin/unmount -a -r
```

Dê a permissão:

``` bash
chmod +755 inittab
```

Entre no diretório init.d:

``` bash
cd init.d
```

Crie o arquivo rcS (run command SINGLE):

``` bash
nvim rcS
```

Dentro, coloque:

``` bash
!/bin/sh

# pseudofilesystems essenciais
mount -t proc none /proc
mount -t sysfs none /sys
mount -t devtmpfs none /dev

# terminais para os pseudo-terminal
mkdir /dev/pts
mount -t devpts none /dev/pts

# gerenciador de dispositivos
echo /bin/mdev > /proc/sys/kernel/hotplug
mdev -s
```

Dentro do etc, crie o arquivo passwd:

``` bash
touch passwd
```

Dentro, coloque:

``` bash
root:x:0:0:root:/root:/bin/sh
bin:x:1:1:bin:/dev/null:/bin/false
daemon:x:6:6:Daemon User:/dev/null:/bin/false
messagebus:x:18:18:D-Bus Message Daemon User:/run/dbus:/bin/false
nobody:x:65534:65534:Unprivileged User:/dev/null:/bin/false
```

Crie o arquivo group:

``` bash
touch group
```

Nele, coloque:

``` bash
root:x:0:
bin:x:1:daemon
sys:x:2:
kmem:x:3:
tape:x:4:
tty:x:5:
daemon:x:6:
floppy:x:7:
disk:x:8:
lp:x:9:
dialout:x:10:
audio:x:11:
video:x:12:
utmp:x:13:
usb:x:14:
cdrom:x:15:
adm:x:16:
messagebus:x:18:
input:x:24:
mail:x:34:
kvm:x:61:
wheel:x:97:
nogroup:x:65533:
nobody:x:65534:
```

Crie o arquivo hostname:

``` bash
touch hostname
```

No diretório etc, digite o comando:

``` bash
echo "my-distro" > /etc/hostname
```

Para testar, crie o arquivo profile:

``` bash
touch profile
```

Nele, coloque:

``` bash
export FEATURE_SH_STANDALONE=1
export PATH=/bin

echo ""
echo "Hello There"
echo ""
```

Retorne para a pasta rootfs:

``` bash
cd ..
```

Execute o comando:

``` bash
sudo chroot /home/administrador/distro/rootfs /bin/sh -l
```

Caso compilar o kernel tenha dado errado, dentro do busybox digite:

``` bash
make CONFIG_PREFIX=<caminho_para_distro_rootfs> install
```
