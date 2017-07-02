# Arch Linux with EFI boot installation guide in Polish

I am writing this guide mostly for my own usage.
If you are looking for english version please contact me.
I will do my best to provide it.

## 1 Partycjonowanie dysku

* przy użyciu linux live CD (np. Puppy Live CD) i gparted. Polecam tą wersje mniej zaawansowanym użytkownikom.
* wersja tekstowa np przy użyciu programu cfdisk.
* Musimy stworzyć minimum dwie partcje. W moim odczuciu najlepiej 3 lub 4 w przypadku gdy mamy mało pamięci RAM.
* Tablica pratycji: gpt
* Partcja boot (sda1): +-512MB
* Partycja swap (sda2, opcjonalna): ROZMIAR ZALEŻNY OD ILOŚĆI RAMU W MASZYNIE, JEJ ZASTOSOWAŃ ITD.
* Partycja root (sda3): rozmiar według uznania, 50GB minimum, jeżeli jest to system do pracy.
* Partycja home (sda4, opcjonalna): rozmiar wedlug uznania.

## 2 Tworzymy systemy pilków, uruchamiany swap
Partycja boot (flagi Boot, esp, mkfs.msdos może być niedostępny, znajduje się w pakiecie dosfstools):

    mkfs.msdos -F 32 /dev/sda1

Partycja root:

    mkfs.ext4 /dev/sda3

Partycja swap (jeżeli taką utworzyliśmy):

    mkswap /dev/sda2

Partycja home (jeżeli taką utworzyliśmy):

    mkfs.ext4 /dev/sda4

## 3 Uruchamiany swap i montujemy partycje
UWAGA: tutaj bardzo ważna jest kolejność operacji,
operacja muszą być wykonywane dokładnie w tej samej kolejności co podano niżej.

jeżeli zrobiliśmy partycje swap to ją uruchamiamy:

    swapon /dev/sda2

Montowanie partycji instalacji:

    mount /dev/sda3 /mnt

Montowanie partycji boot:

    mkdir /mnt/boot
    mount /dev/sda1 /mnt/boot

Jeżeli przygotowaliśmy partycje home:

    mkdir /mnt/home
    mount /dev/sda4 /mnt/home

## 4 Automatyczna konfiguracja sieci

    dhcpcd

## 5 Instalacja podstawowego systemu

    pacstrap /mnt base base-devel

## 6 Generujemy fstab

    genfstab /mnt >> /mnt/etc/fstab

## 7 Przechodzimy do zainstalowanego systemu

    arch-chroot /mnt

## 8 Nadajemy nazwe naszemu komputerowi

    echo nazwakomputera > /etc/hostname

## 9 Konfigurujemy strefe czasową

    ln -s /usr/share/zoneinfo/Europe/Warsaw /etc/localtime

## 10 Dodajemy locale

Otwieramy plik w edytorze:

    nano /etc/locale.gen

Szukamy wpisów:

    en_US.UTF-8 UTF-8
    pl_PL.UTF-8 UTF-8

Usuwamy znak '#' przed nimi. Zapisujemy plik. Aktualizujemy locale poprzez wydanie polecenia:

    locale-gen

powinno wyświetlić się:

    Generating locales...
    en_US.UTF-8... done
    pl_PL.UTF-8... done
    Generation complete.

Jeżeli chcemy spolszyczć system (opcjonalne) to edytujemy plik:

    nano /etc/locale.conf

Wpisujemy do niego:

    LANG=pl_PL.UTF-8

Zapisujemy.

## 11 Tworzomy nowego użytkownika i hasła

Nadajemy hasło użytkownikowi root:

    passwd root

Tworzymy nowego użytkownika:

    useradd -m -g users -G audio,disk,network,optical,power,storage,video,rfkill,wheel -s /bin/bash nazwausera

Tworzymy hasło dla nowego uzytkownika:

    passwd nazwausera

## 12 Nadajemy naszemu użytkownikowi prawo do sudo

Edytujemy plik:

    nano /etc/sudoers

Szukamy linii:

    ALL=(ALL) ALL

Pod nią dodajemy następującą:

    nazwausera ALL=(ALL) ALL

## 13 Inicjalizacja systemu

    mkinitcpio -p linux

## 14 Konfiguracja efi boot

X i Y dobieramy tak aby reprezentowały naszą partycje boot (np: /dev/sda1 -p 1)
Wpis root=/dev/sdb4 zamieniamy na reprezentujący naszą partycje root

    pacman -S efibootmgr
    efibootmgr -d /dev/sdX -p Y -c -L "Arch Linux" -l /vmlinuz-linux -u "root=/dev/sdb4 rw initrd=/initramfs-linux.img"

W tym momencie można zrestartować system i sprawdzić czy bootwanie działa:

    exit
    umount -R /mnt
    reboot

Wyciągamy medium instalacyjne.
Po poprawnym uruchomieniu się systemu logujemy się na naszego użytkownika i ponownie uruchamiamy automatyczna kofigurację sieci.

    dhcpcd

## 15 Instalacja środowiska

Na początku sterownik grafiki:

    sudo pacman -S sterownik_karty

Sterowniki:
* Nvidia GPU → nvidia
* AMD/ATI GPU → xf86-video-amdgpu
* Intel → mesa

czcionki:

    sudo pacman -S freetype2 ttf-dejavu

sieć i Xorg:

    sudo pacman -S wget net-tools networkmanager network-manager-applet wireless_tools wpa_supplicant wpa_actiond dialog xorg-server xf86-input-evdev xf86-input-mouse xf86-input-keyboard xf86-video-vesa dbus xorg-xinit xorg-twm xorg-xclock terminator mesa

Alsa+PulseAudio:

    sudo pacman -S alsa-firmware alsa-lib alsa-plugins alsa-utils pulseaudio pulseaudio-alsa libcanberra libcanberra-pulse

Środowisko graficzne i narzedzia:

    sudo pacman -S <środowisko graficznie> ark zip unzip unrar gconf gtk2 firewalld

Przykładowe środowiska graficzne cinnamon, xfce4

## 16 Manadzer logowania

    pacman -S lightdm lightdm-gtk-greeter

## 17 Wyłączamy i włączamy demony:

    sudo systemctl disable dhcpcd
    sudo systemctl disable iptablets.service
    sudo systemctl enable NetworkManager
    sudo systemctl enable firewalld.service
    sudo systemctl enable lightdm.service

W tym momencie można zrestartować system i sprawdzić czy środowisko graficzne działa:

    reboot

## 18 Instalacja menadżera pakietów AUR:

Instalacja niezbędnych pakietów:

    sudo pacman -S yajl

Pobieranie i instalacja package-query:

    wget https://aur.archlinux.org/cgit/aur.git/snapshot/package-query.tar.gz
    tar xzf package-query.tar.gz
    cd package-query && makepkg
    sudo pacman -U package-query*.pkg.tar.xz

Pobranie i instlacja Yaourt:

    wget https://aur.archlinux.org/cgit/aur.git/snapshot/yaourt.tar.gz
    tar xzf yaourt.tar.gz
    cd yaourt && makepkg
    sudo pacman -U yaourt*.pkg.tar.xz


## EXTRA, pakiety w mojej opini warte uwagi:

* odtwarzacz audio/video: VLC
* bardzo intuicyjny graficzny menadżer plików: nemo
* proste montowanie urzadzen z andoidem (jeżeli uzywamy menadżera plików opartego na gnome np nemo): gvfs-mtp
* montowanie dysków ntfs: ntfs-3g
* kodeki:
a52dec faac faad2 flac jasper lame libdca libdv libmad libmpeg2 libtheora libvorbis libxv wavpack x264 xvidcore gstreamer0.10-plugins
* narzędzie do screenow: spectacle
* W mojej opini najlepszy ciemny styl dla gtk (swietnie wspolpracuje z xfce): numix-gtk-theme
