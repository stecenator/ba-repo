# Rola `ba_srv`

Rola serwera repozytorium pakietów IBM spectrum Protect Backp Archive Client.
Przetestowana na Fedorze, ale powinna działać na każdym w miarę nowoczesnym Red Hacie.
Playbook jest przystosowany do systemów z SELinuxem w trybie *enforcing*.

Żeby móc użyć tego playbooka musisz:

- wpisać swoje hosty do pliku `hosts` i mieścić je w grupie `ba_srvs`. No i oczywiście zapewnić komunijację po `ssh`. Ja łączyłem się z rootem, kluczem  bez hasła, ale to jest Ansible, więc można to zmienić :-) 
- Ustawić zmienne `lun` i `latest_ba_tar` na odpowiednie dla środoiska.

## Opis zmiennych

Wszyskie zmienne są w pliku `roles/ba_srv/vars/main.yaml`.

 - **LUN** - Ustawić na coś użytecznego. Testowałem to na `vdb`  bo target był na KVMie. W tum miejscu można podać np LUNach z macierzy w postaci `/dev/mapper/freindly_name`.

     `lun: "/dev/vdb"` 
- **latest_ba_tar** - Ta zmienna powinna wskazywać na paczkę tar umieszczoną w katalogu `files/` i ściągniętą z [IBM](http://ftp.software.ibm.com/storage/tivoli-storage-management/maintenance/client/).

     `latest_ba_tar: "8.1.17.0-TIV-TSMBAC-LinuxX86.tar"`

- **VG** - Nazwa grupy wolumenów (LVM) do założenia na LUNie.

     `vg: "repovg"`

- **LV** - Nazwa wolumenu logicznego pod filesystem dla repozytorium.

     `lv: "repolv"`

- **www_mnt** - Punkt montowania filesystemu `/dev/mapper/{{ vg }}/{{ lv }}` i jednocześnie *Server Root* serwera Lighttpd.

     `www_mnt: "/var/www"`

- **doc_root** - *Document root* serwera Lighttpd. Domyślnie podkatalog `{{ www_mnt }}/lighttpd`.

`doc_root: "{{ www_mnt }}/lighttpd"`

- **ba_root** - podkatalog `{{ doc_root }}`, do którego zostaną skopiowane pakiety RPM i gdzie zostianie utorzone repo.

     `ba_root: "{{ doc_root }}/ba"`

- **srv_pkgs** - lista pakietów jakie trzeba zainstalować w ramach przygotowania serwera WWW.

     ```
     srv_pkgs:
       - lighttpd
       - createrepo_c
       - lvm2
       - vim-enhanced
       - tar
       - unzip
       - gzip
       - python3-libselinux
       - python3-libsemanage
       - policycoreutils-python-utils
     ```

- **se_lnx** - kontroluje czy SELinux jest w trybie *enforcing*. Serwer w trybie *permissive* albo *disabled* jest zepsuty, więc nie ruszać!

     `se_lnx: True`

- **fw_services** - dziurki do otwarcia w firewallu. NAzwy usług zgodne z `firewalld`.

     ```
     fw_services:
       - http
       - https
     ```

## Zadania w playbooku `ba_srv_inst.yaml`

### Pakiety serwera repozytorium

Instaluje paiety podane w na zmiennej `{{ srv_pkgs }}`. Wrzucałem tam "co wyszło w praniu" względem instalacji "Fedora minimal".

### VG dla repozytorium

Zakłada volume grupę `{{ vg }}` na PV: `{{ lun }}`. W razie potrzeby założy także PV.

### LV dla repozytorium

Zakłada LV `{{ lv }}` w grupie `{{ vg }}` z założeniem `-l 100%FEE`.

### Punkt mntowania repozytorium

Zakłada katalog `{{ www_mnt }}`. To będzie punkt montowania tworzonego repozytorium i jednocześnie `var.server_root` dla Lighttpd. Domyślnie: `"/var/www"`.

### Formatowanie filesystemu XFS

Formatuje `{{ lv }}` na XFSowo.

### Montowanie filesystemu

Montuje filesystem `{{ www_mnt }}` i dodaje do `/etc/fstab`.

### Podkatalog repozytorium

Na zmontowanym filesytemie zakłada `{{ ba_root }}` czyli katalog, do którego zostaną rozpakowane RPMy z paczkami i zostanie w nim utworzone repozytorium.

### Otwieranie portów firewalla

Dodaje do `firewalld` porty z listy `{{ fw_services }}`: teraz http i https.

### Ustawianie bulionów dla Lighttpd

Lighttpd przy włączonym *SELinux* potrzebuje ustawiania *sebool* `httpd_setrlimit = on`.

### Wrzucanie pliku index.html

Durnostojka `index.html` generowana z szablonu `templates/index.html.j2`. Podstawia tylko nazę hosta, link do katalogu z paczkami i link do pliku repo.

### Odpakowanie TARa z klientem BA do {{ ba_root }}

Rozpakowuje paczkę `{{ lates_ba_tar }}` do `{{ ba_root }}`. 

### "Zezwolenie na directory browsing w {{ ba_root }}"

Nadpisuje plik `/etc/lighttpd/conf.d/dirlisting.conf` serwera Lighttpd wersją z dodanym kawałkiem konfiuracji zezwalającym na *directory browsing* na `{{ ba_root }}`:

```
# Zezwolenie na przeszukiwanie katalogu {{ ba_root }} na serwerze lighttpd
$HTTP["url"] =~ "^/ba($|/)" {
     dir-listing.activate = "enable" 
   }
```

**Uwaga:** ścieżka zahardkodowana w regexpa. Poprawić na template!

### Dodawanie polityk SELinux dla katalogu {{ doc_root}}

Dla serwera z uruchomionym *SELinuxem*: Lighttpd nie dodaje polityki dla swojego standardowego katalogu na html'e!! Bez tego selinux nie pozwala serwerowi serwować czegokolwiek. Kontekst:

```
     target: '{{ doc_root }}(/.*)?'
     seuser: "system_u"
     setype: "httpd_sys_rw_content_t"
```

### Tworzenie repozytorium RPM w {{ ba_root }}

Wywołanie `createrepo {{ ba_root }}`. Tworzy repozytorium yum/dnf. 

### Generowanie pliku repozytorium {{ ba_root }}/ba.repo

Dla leniwych. Tworzy plik z definicją repozytorium. Można go sobie siągnąć i wrzucić do `/etc/yum.repos.d` i skorzystać z tego serwera do instalacji paczek klienta Spectrum Protect.

### Ustawianie kontekstu SELinux na {{ doc_root }}

Po skończonym dodawaniu plików do `{{ doc_root }}` trzeba ustawić kontekst przypisany wcześniej, bo inaczej nic nie będzie widać. 

To zadanie nie jest idempotentne, ale to nie szkodzi. `restorecon` jest zawsze zdrowy ;-)

### Włączanie Lighttpd

W końcu można używać.

## Zadania w tasku `ba_srv_refresh.yaml`

Ten task ma za zadanie rozpakować archiwum wskazane przez zminną `latest_ba_tar` do już istniejącego repozytorium wskazanego zmienną `ba_root` i przebudować `repodata`.

**To do:** 
1. Być może warto dodać sprawdzanie co jest nadpisywane. Na razie uruchmienie playbooka zawsze nadpisze repo.
1. Dodać kod sprawdzający 

### Odpakowanie TARa z klientem BA do {{ ba_root }}

Rozpakowuje paczkę `{{ lates_ba_tar }}` do `{{ ba_root }}`. 

**Uwaga:** Tutaj usunięto parametr `creates`, co powoduje, że to zdanie się zawsze wykona!

### Dodawanie polityk SELinux dla katalogu {{ doc_root}}

Dla serwera z uruchomionym *SELinuxem*: Lighttpd nie dodaje polityki dla swojego standardowego katalogu na html'e!! Bez tego selinux nie pozwala serwerowi serwować czegokolwiek. Kontekst:

```
     target: '{{ doc_root }}(/.*)?'
     seuser: "system_u"
     setype: "httpd_sys_rw_content_t"
```

### Tworzenie repozytorium RPM w {{ ba_root }}

Wywołanie `createrepo {{ ba_root }}`. Tworzy repozytorium yum/dnf. 
**Uwaga:** Tutaj usunięto parametr `creates`, co powoduje, że to zdanie się zawsze wykona!

### Generowanie pliku repozytorium {{ ba_root }}/ba.repo

Dla leniwych. Tworzy plik z definicją repozytorium. Można go sobie siągnąć i wrzucić do `/etc/yum.repos.d` i skorzystać z tego serwera do instalacji paczek klienta Spectrum Protect.

### Ustawianie kontekstu SELinux na {{ doc_root }}

Po skończonym dodawaniu plików do `{{ doc_root }}` trzeba ustawić kontekst przypisany wcześniej, bo inaczej nic nie będzie widać. 

To zadanie nie jest idempotentne, ale to nie szkodzi. `restorecon` jest zawsze zdrowy ;-)