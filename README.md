# BA Roles

Ten playbook składa się z dwóch ról: 

- **ba_srv** - ta rola służy do skonfigurowania serwer Lighttpd na wyznaczonym dysku jako serwera pakietów RPM IBM Spectrum Protect. Wiecej na temat tej roli znajdziesz w [README.md](roles/ba_srv/README.md).

- **ba_client** - hosty w tej roli będą klientami repozytorium skonfigurowanum np rolą `ba_srv`. Bardziej szczegółowy opis tej roli jest w [README.md](roles/ba_client/README.md).