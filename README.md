# BA Repo Server

Rola serwera repozytorium pakietów IBM spectrum Protect Backp Archive Client.
Przetestowane na Fedorze, ale powinno działać na każdy w miarę nowoczesnym Red Hacie.

Żeby móc użyć tego playbooka musisz:

- wpisać swoje hosty do pliku `hosts`. I zapewnić komunijację po `ssh`. Ja łączyłem się z rootem, bez hasła, ale to jest Ansible, więc można to zmienić :-) 
- Ustawić zmienne `lun` i `latest_ba_tar` na odpowiednie dla środoiska.

Wszyskie zmienne są w pliku `group_vars/ba_srvs.yaml`.

### LUN

`lun: "/dev/vdb"` 

Na coś użytecznego. Testowałem to na `vdb` i na LUNach z macierzy (tu można podać nap /dev/mapper/freindly_name)

### latest_ba_tar

`latest_ba_tar: "8.1.17.0-TIV-TSMBAC-LinuxX86.tar"`

Ta zmienna powinna wskazywać na paczkę tar umieszczoną w katalogu `files/` i sciągniętą z [IBM](http://ftp.software.ibm.com/storage/tivoli-storage-management/maintenance/client/).


## Zadania w playbooku `ba_srv.yaml`

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