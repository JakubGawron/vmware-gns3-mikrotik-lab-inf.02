<h1 align="center">Laboratorium MikroTik Router na patyku (VMware + GNS3 VM)</h1>

<p align="center">
  <img alt="MikroTik CHR" src="https://img.shields.io/badge/MikroTik-CHR_7.24.4-293239">
  <img alt="GNS3" src="https://img.shields.io/badge/GNS3-VM-2F9E44">
  <img alt="VMware" src="https://img.shields.io/badge/VMware-Workstation-607078">
  <img alt="INF.02" src="https://img.shields.io/badge/exam-INF.02-orange">
  <img alt="Topics" src="https://img.shields.io/badge/topics-VLAN_%7C_DHCP_%7C_NAT_%7C_Firewall-blue">
  <a href="https://github.com/JakubGawron/vmware-gns3-mikrotik-lab-inf.02/releases/latest"><img alt="Latest release" src="https://img.shields.io/github/v/release/JakubGawron/vmware-gns3-mikrotik-lab-inf.02"></a>
  <img alt="Platform" src="https://img.shields.io/badge/platform-cross--platform-lightgrey">
  <img alt="Status" src="https://img.shields.io/badge/status-stable-brightgreen">
  <img alt="License" src="https://img.shields.io/github/license/JakubGawron/vmware-gns3-mikrotik-lab-inf.02">
  <img alt="GitHub last commit" src="https://img.shields.io/github/last-commit/JakubGawron/vmware-gns3-mikrotik-lab-inf.02">
  <img alt="GitHub issues" src="https://img.shields.io/github/issues/JakubGawron/vmware-gns3-mikrotik-lab-inf.02">
</p>

<p align="center">
  <a href="./README.md">English documentation</a>
</p>

<p align="center">
  Laboratorium sieciowe oparte na VMware z użyciem GNS3 VM do ćwiczeń z MikroTik i przygotowania do egzaminu INF.02, ale nie tylko: routing, VLAN-y, DHCP, reguły firewalla oraz konfiguracja router na patyku (router-on-a-stick). Oparte na podobnym laboratorium wykorzystanym na wykładach w 2024 roku dla około 200 osób, na których MikroTik był omawiany jako urządzenie warstwy 3 w kontekście egzaminu.
</p>

---

## Przegląd

MikroTik CHR routuje między czterema VLAN-ami przez pojedyncze łącze tagowane do przełącznika GNS3 (router na patyku). Zapewnia również DHCP, przekazywanie DNS, NAT i filtrowanie firewallem. Do testów wykorzystano cztery hosty VPCS.

**Tematy do ćwiczenia:** VLAN-y i 802.1Q, porty access vs. tagowane, routing między VLAN-ami, DHCP, DNS, NAT, reguły filtrowania firewalla, listy interfejsów, testowanie łączności, podstawy administracji RouterOS.

---

## Instalacja

### 1. Pobieranie

Pobierz najnowsze pliki laboratorium z [**Releases**](https://github.com/JakubGawron/vmware-gns3-mikrotik-lab-inf.02/releases/latest).

### 2. VMware

1. Zainstaluj VMware Workstation.
2. Otwórz **Edit → Virtual Network Editor → Change Settings** i ustaw:
    - `VMnet1`: Host-only, `172.16.0.0/24`, DHCP wyłączony
    - `VMnet8`: NAT, `192.168.100.0`, DHCP wyłączony

### 3. GNS3 VM

1. Zaimportuj pobrany plik `.ova` GNS3 VM przez **File → Open**.
2. Uruchom maszynę wirtualną.
3. Otwórz interfejs webowy GNS3 VM Lab: **http://172.16.0.2**

---

## Topologia

![GNS3 network topology](./images/GNS3/network-topology.png)

### Adresacja routera i VLAN-y

| VLAN | Nazwa            | Host   | Port switcha | Interfejs MikroTik | Sieć               | Brama           | IP hosta        |
| ---- | ---------------- | ------ | ------------ | ------------------ | ------------------ | --------------- | --------------- |
| 10   | ADMIN            | ADMIN  | 1 (access)   | `vlan10-admins`    | `10.10.0.0/24`     | `10.10.0.1`     | `10.10.0.2`     |
| 20   | USER             | USER   | 2 (access)   | `vlan20-users`     | `10.20.0.0/24`     | `10.20.0.1`     | `10.20.0.2`     |
| 30   | GUEST            | GUEST  | 3 (access)   | `vlan30-guests`    | `10.30.0.0/24`     | `10.30.0.1`     | `10.30.0.2`     |
| 40   | SERVER           | SERVER | 4 (access)   | `vlan40-servers`   | `10.40.0.0/24`     | `10.40.0.1`     | `10.40.0.2`     |
| –    | Zarządzanie      | –      | –            | `management`       | `172.16.0.0/24`    | –               | `172.16.0.3`    |
| –    | WAN (VMware NAT) | –      | –            | `ether1`           | `192.168.100.0/24` | `192.168.100.2` | `192.168.100.3` |

---

## Przełącznik GNS3

![GNS3 switch configuration](./images/GNS3/switch-configuration.png)

| Port | VLAN | Typ                         | Podłączony do              |
| ---- | ---- | --------------------------- | -------------------------- |
| 0    | 1    | `dot1q`, EtherType `0x8100` | MikroTik (uplink tagowany) |
| 1    | 10   | access                      | ADMIN                      |
| 2    | 20   | access                      | USER                       |
| 3    | 30   | access                      | GUEST                      |
| 4    | 40   | access                      | SERVER                     |

Porty access wysyłają ramki do hosta bez tagu; uplink tagowany przenosi wszystkie VLAN-y z tagami 802.1Q. Nie zakłada się, że VLAN `1` na porcie `0` jest natywnym VLAN-em.

## Router na patyku

```text
ADMIN (VLAN 10) → port access 1 switcha → uplink tagowany (port 0)
  → MikroTik vlan10-admins → routing + firewall (reguła 16)
  → vlan20-users → uplink tagowany → port access 2 → USER (VLAN 20)
```

Każdy VLAN ma własny interfejs logiczny na pojedynczym porcie fizycznym routera, a adres `.1` tego interfejsu jest bramą dla hostów.

---

## Konfiguracja MikroTik

RouterOS v7 WebFig. `MikroTik CHR 7.24.4`

### Adresy IP

![IP addresses](./images/MikroTik/addresses.png)

### Interfejsy VLAN

![VLAN interfaces](./images/MikroTik/vlan.png)

Interfejsy VLAN są zagnieżdżone pod `ether2` w drzewie interfejsów. Kolumny z ID VLAN-u i interfejsem nadrzędnym nie są tu widoczne.

### Interfejsy

![Interfaces](./images/MikroTik/interface.png)

### Listy interfejsów

![Interface lists](./images/MikroTik/interface-list.png)

### Serwery DHCP

![DHCP servers](./images/MikroTik/dhcp.png)

### Dzierżawy DHCP (leases)

![DHCP leases](./images/MikroTik/leases.png)

### DNS

![DNS](./images/MikroTik/dns.png)

### NAT

![NAT](./images/MikroTik/nat.png)

### Reguły filtrowania firewalla

![Firewall filter rules](./images/MikroTik/filter-rules.png)

| Łańcuch   | Polityka (widoczne reguły)                                                                                                                                                                                                       |
| --------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `input`   | Accept established/related, drop invalid; zezwolenie na DHCP i DNS z `LAN`; ICMP i porty zarządzania (`22,80,443,8291`) tylko z `management` oraz od `ADMINS` na `vlan10-admins`; domyślnie drop.                                |
| `forward` | Accept established/related, drop invalid; drop niezamówionego ruchu z `WAN`; `LAN` → `WAN` dozwolone; między VLAN-ami: `ADMINS` → `USERS`/`GUESTS`/`SERVERS`, `USERS` → `SERVERS`, `SERVERS` → `ADMINS`/`USERS`; domyślnie drop. |

### Pliki kopii zapasowych

![Files](./images/MikroTik/file.png)

| Plik                                 | Zawartość                                                                                                                        | Zastosowanie                                                                                              |
| ------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| `Configuration_MikroTik_Lab.backup`  | W pełni skonfigurowane laboratorium: interfejsy VLAN, adresacja IP, serwery DHCP, DNS, NAT, listy interfejsów i reguły firewalla | **Domyślny.** Użyj go, aby uruchomić gotowe laboratorium i śledzić testy łączności.                       |
| `Configuration_MikroTik_Base.backup` | Tylko wstępnie skonfigurowany interfejs `management`                                                                             | Własne eksperymenty. Zacznij od czystego routera i samodzielnie skonfiguruj VLAN-y, DHCP, NAT i firewall. |

### Kroki przywracania

1. Otwórz MikroTik w WebFig (`172.16.0.3`) i przejdź do **Files**.
2. Zaznacz plik i kliknij **Restore**. Potwierdź po wyświetleniu prośby. W starszych wersjach WebFig opcja znajduje się w **Files → Backup → Restore**.
3. Poczekaj na restart routera, a następnie połącz się ponownie przez `172.16.0.3`.

### Konfiguracja kart sieciowych

![VMware Virtual Network Editor](./images/VMware/virtual-network-editor.png)

| Sieć VMware | Typ       | Podsieć         | DHCP      | Zastosowanie                                                                                                         |
| ----------- | --------- | --------------- | --------- | -------------------------------------------------------------------------------------------------------------------- |
| `VMnet1`    | Host-only | `172.16.0.0/24` | Wyłączony | Zarządzanie (MikroTik `management` = `172.16.0.3`; adresy konsoli GNS3 wskazują na `172.16.0.2`, najpewniej GNS3 VM) |
| `VMnet8`    | NAT       | `192.168.100.0` | Wyłączony | Dostęp zewnętrzny (MikroTik `ether1` = `192.168.100.3`)                                                              |

---

## Testy łączności

Pojedynczy `ping -c 1` do każdego celu. Zielony = odpowiedź, czerwony = timeout.

| ADMIN                             | USER                                |
| --------------------------------- | ----------------------------------- |
| ![ADMIN](./images/VPCS/admin.png) | ![USER](./images/VPCS/user.png)     |
| **GUEST**                         | **SERVER**                          |
| ![GUEST](./images/VPCS/guest.png) | ![SERVER](./images/VPCS/server.png) |

### Macierz wyników

| Źródło ↓ / Cel → | ADMIN | USER | GUEST | SERVER | Router `172.16.0.3` | Internet / DNS |
| ---------------- | :---: | :--: | :---: | :----: | :-----------------: | :------------: |
| **ADMIN**        |   –   |  ✅  |  ✅   |   ✅   |         ✅          |       ✅       |
| **USER**         |  ❌   |  –   |  ❌   |   ✅   |         ❌          |       ✅       |
| **GUEST**        |  ❌   |  ❌  |   –   |   ❌   |         ❌          |       ✅       |
| **SERVER**       |  ✅   |  ✅  |  ❌   |   –    |         ❌          |       ✅       |

Wyniki zgadzają się z regułami firewalla opisanymi powyżej. Osiągalność jest kierunkowa (np. ADMIN → USER działa, USER → ADMIN nie).

---

## Znaczenie egzaminacyjne

Ćwiczenie zagadnień takich jak VLAN-y, adresacja, routing, DHCP, NAT, firewall, rozwiązywanie problemów i RouterOS w zakresie przygotowań do INF.02. Jest to środowisko edukacyjne i niekoniecznie dokładne odwzorowanie aktualnych zadań egzaminacyjnych.

---

## Licencja

Ten projekt jest objęty licencją [GPL-3.0 License](https://choosealicense.com/licenses/gpl-3.0/).

Szczegóły znajdują się w pliku [`LICENSE`](./LICENSE).

---

## Autor / Kontakt

- **Autor:** Jakub Gawron
- **GitHub:** [github.com/JakubGawron](https://github.com/JakubGawron)
- **Repozytorium:** [github.com/JakubGawron/vmware-gns3-mikrotik-lab-inf.02](https://github.com/JakubGawron/vmware-gns3-mikrotik-lab-inf.02)
- **Kontakt:** [contact.jakub.gawron@gmail.com](mailto:contact.jakub.gawron@gmail.com)
