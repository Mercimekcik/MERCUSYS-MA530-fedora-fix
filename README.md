MERCUSYS MA530 BLUETOOTH 5.3 ADAPTÖRÜ - FEDORA ÇÖZÜM REHBERİ
==============================================================

SORUN
-----
Mercusys MA530 (USB ID: 2c4e:0115), Realtek RTL8761BUV chipseti kullanıyor.
Fedora'nın (ve diğer birçok dağıtımın) kernel'inde bu cihaz ID'si
btusb sürücüsünün "quirks_table" listesinde YOK. Bu yüzden:
- Cihaz lsusb'da görünür, hci0 oluşur, power on çalışır
- AMA firmware (rtl8761bu_fw.bin) hiç yüklenmez
- Sonuç: Bluetooth "çalışıyormuş gibi" görünür ama hiçbir cihazı
  tarayamaz / bulamaz (TX çalışır, RX çalışmaz gibi davranır)

Bu, kernel.org'da bilinen bir eksiklik. Farklı kişiler (Michal Piernik
- Şubat 2025, elespink - Ağustos 2025, Hrvoje Nuic - Nisan 2026) bu
ID'yi eklemek için ayrı ayrı patch gönderdi. Hrvoje Nuic'in patch'i
resmen kabul edildi (upstream commit: ce21a5cf3d1fd92b84ea9ad2b7c7240aff2162d2)
ve Ağustos 2026'da eski stabil kernel dallarına (6.18.y, 6.12.y, 6.1.y)
da geriye taşındı (AUTOSEL backport). Ancak Fedora'nın kendi kernel
paketine bu ID'nin gelip gelmediği garanti değil - kontrol etmek gerekiyor.

ADIM 1: SORUNU DOĞRULA
-----------------------
Cihazın gerçekten bu sorunu yaşayıp yaşamadığını kontrol et:

    sudo dmesg | grep -i RTL

Eğer çıktıda şu satırlar YOKSA (sadece ethernet r8169 satırı varsa),
firmware yüklenmiyor demektir, aşağıdaki adımlara devam et:
    RTL: examining hci_ver=...
    RTL: loading rtl_bt/rtl8761bu_fw.bin

Kernel modülünün bu ID'yi tanıyıp tanımadığını da kontrol edebilirsin
(bilgi amaçlı, kesin kanıt değil - quirks_table MODULE_DEVICE_TABLE'da
görünmediği için "alias" listesinde çıkmaz, normal):
    modinfo btusb | grep -i 2c4e

Firmware dosyalarının sistemde olup olmadığını kontrol et (genelde
zaten var, linux-firmware paketiyle gelir):
    ls -la /usr/lib/firmware/rtl_bt/ | grep 8761bu

rfkill kontrolü (bloklu değilse "no" yazmalı):
    rfkill list

ADIM 2: ÖNCE KERNEL GÜNCELLEMESİNİ DENE (EN KOLAY YOL)
--------------------------------------------------------
Belki Fedora bu arada patch'i eklemiştir, önce bunu dene:

    sudo dnf update kernel
    sudo reboot

Güncelleme sonrası tekrar Adım 1'deki dmesg komutunu çalıştır.
RTL satırları çıkıyorsa SORUN ÇÖZÜLMÜŞTÜR, aşağıdaki adımlara gerek yok.

ADIM 3: KERNEL'DE YOKSA - DKMS İLE MANUEL PATCH (ÇALIŞAN ÇÖZÜM)
-------------------------------------------------------------------
Bu yöntem test edildi ve ÇALIŞTI (6.19.10-300.fc44.x86_64 üzerinde).

3.1) Gerekli paketleri kur:

    sudo dnf install -y kernel-devel-$(uname -r) dkms make gcc curl

3.2) Kaynak dosyalarını indir (btusb.c ve kardeş header dosyaları -
     HEPSİ GEREKLİ, sadece btusb.c yetmez):

    mkdir -p ~/btusb-mercusys && cd ~/btusb-mercusys
    curl -o btusb.c "https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/plain/drivers/bluetooth/btusb.c?h=v6.19.10"
    curl -o btintel.h "https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/plain/drivers/bluetooth/btintel.h?h=v6.19.10"
    curl -o btrtl.h "https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/plain/drivers/bluetooth/btrtl.h?h=v6.19.10"
    curl -o btbcm.h "https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/plain/drivers/bluetooth/btbcm.h?h=v6.19.10"
    curl -o btmtk.h "https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/plain/drivers/bluetooth/btmtk.h?h=v6.19.10"

    NOT: "v6.19.10" kısmını o an çalıştırdığın kernel sürümüyle
    (uname -r çıktısındaki ana sürüm) değiştir. Örn. kernel 6.20.x
    ise "v6.20" dene. Doğru indiğini kontrol et:

    head -5 btintel.h

    Çıktı "#ifndef" ile başlayan gerçek C kodu olmalı, HTML/hata
    sayfası DEĞİL. Değilse doğru tag/sürüm adını bulman gerekir.

3.3) Mercusys ID'sini btusb.c içine ekle:

    sed -i '/Additional Realtek 8761BUV Bluetooth devices/a\	{ USB_DEVICE(0x2c4e, 0x0115), .driver_info = BTUSB_REALTEK |\n\t\t\t\t\t\t     BTUSB_WIDEBAND_SPEECH },' btusb.c

    Kontrol et:
    grep -A2 "0x2c4e, 0x0115" btusb.c

3.4) Makefile oluştur:

    cat > Makefile << 'EOF'
obj-m += btusb.o
KDIR := /lib/modules/$(shell uname -r)/build
PWD := $(shell pwd)
default:
	$(MAKE) -C $(KDIR) M=$(PWD) modules
clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
EOF

3.5) dkms.conf oluştur:

    cat > dkms.conf << 'EOF'
PACKAGE_NAME="btusb-mercusys"
PACKAGE_VERSION="1.0"
BUILT_MODULE_NAME[0]="btusb"
DEST_MODULE_LOCATION[0]="/kernel/drivers/bluetooth"
AUTOINSTALL="yes"
EOF

3.6) Secure Boot durumunu kontrol et:

    mokutil --sb-state

    "SecureBoot enabled" ise, dkms install sırasında bir MOK parolası
    belirlemen istenecek, sonra REBOOT edip mavi ekranda "Enroll MOK"
    seçip o parolayı girmen gerekecek. Bu adımı atlarsan modül
    yüklenmez. "SecureBoot disabled" ise bu adımı hiç düşünme.

3.7) DKMS'e ekle ve derle:

    sudo mkdir -p /usr/src/btusb-mercusys-1.0
    sudo cp -r ~/btusb-mercusys/* /usr/src/btusb-mercusys-1.0/
    sudo dkms add -m btusb-mercusys -v 1.0
    sudo dkms build -m btusb-mercusys -v 1.0

    "Building module(s)... done." yazısını görmen lazım.
    Hata alırsan (örn. "fatal error: btintel.h: böyle bir dosya yok"),
    3.2'deki header dosyalarını indirmeyi unutmuşsundur - kontrol et:

    cat /var/lib/dkms/btusb-mercusys/1.0/build/make.log

3.8) Kur:

    sudo dkms install -m btusb-mercusys -v 1.0

3.9) Eski modülü çıkar, yenisini yükle:

    sudo systemctl stop bluetooth
    sudo modprobe -r btusb
    sudo modprobe btusb
    sudo systemctl start bluetooth

3.10) Doğrula - artık RTL satırları görünmeli:

    sudo dmesg | grep -i RTL

    Beklenen çıktı:
    Bluetooth: hci0: RTL: examining hci_ver=0a hci_rev=000b lmp_ver=0a lmp_subver=8761
    Bluetooth: hci0: RTL: rom_version status=0 version=1
    Bluetooth: hci0: RTL: loading rtl_bt/rtl8761bu_fw.bin
    Bluetooth: hci0: RTL: loading rtl_bt/rtl8761bu_config.bin
    Bluetooth: hci0: RTL: fw version 0xdfc6d922

3.11) Bluetooth taramasını test et:

    bluetoothctl
    scan on

    (30 saniye bekle, sonra "devices" yaz, cihazlar listede çıkmalı)

KALICILIK NOTU
--------------
DKMS kaydı KALICIDIR. Reboot'ta hiçbir şey yapmana gerek yok.
Kernel güncellendiğinde (örn. dnf update ile yeni kernel geldiğinde)
DKMS OTOMATİK olarak yeni kernel için modülü yeniden derleyip kurar
(dkms.service arka planda bunu halleder). Format atarsan veya yeni
bir Fedora kurulumu yaparsan, bu adımları (1-3) baştan yapman gerekir
- DKMS kaydı diskle beraber silinir.

Eğer ilerde Fedora resmi kernel'ine bu ID eklenirse, DKMS'in
derlediği versiyon zaten aynı işi yapacağı için çakışma olmaz,
elle bir şey yapmana gerek kalmaz.

FIREWALL / DİĞER NOT (WiVRn ile ilgiliyse ayrı konu)
------------------------------------------------------
Bu dosya sadece Bluetooth/MA530 sorunuyla ilgilidir. WiVRn portları
(5353/udp avahi, 9757/tcp+udp) farklı bir konu, gerekirse ayrı not al.
