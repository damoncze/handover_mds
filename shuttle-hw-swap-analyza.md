# Výměna FortiGate na existující lokalitě (FG100F → FG120G) — analýza pro scénář shuttle

Stav k 31. 8. 2026, večer. Read-only sběr: kód shuttle (branch main + WAN-CIR), NetBox API,
FMG JSON-RPC (stejné credentials jako shuttle, jen `get`), Zabbix API, `verify_site.yml` na
sk-depot-1 a cz-depot-28. Nic se nikde nezapsalo. Pilot: **Strečno = `sk-depot-1`,
`PAC-SK-STR-FW`**. Bridged reference: **Bezdečín = `cz-depot-28`, `PAC-CZ-BEZDECIN-FW`**.

Zadání (rozhodnutí 31. 8.): nová 120G se **kompletně připraví v HQ** za běhu staré 100F,
dostane **nové FMG jméno**, na depu se **jen prohodí krabice a přepojí kabely**. Při tom se
AP převedou na **bridged** wifi. Switche a AP mají být v konfiguraci nové FG **předem**.
Ve flotile je 36× FG100F a 15× FG120G, takže scénář se bude opakovat.

---

## 0. Souhrn

1. **Nová lokalita se nezakládá.** Identita FMG zařízení je Site CF `fmg_device_name`; všechno
   per-device ve FMG (variables, fsp/vlan mapy, normalized interfaces, DHCP rezervace, Zabbix
   host) je klíčované tímhle jménem. Site, prefixy, VLANy, proxy, AP, switche, UPS zůstávají.
2. **Každý spoke má vlastní policy package** (`Spoke_SK_STRECNO` → scope PAC-SK-STR-FW). Nový
   device potřebuje **nový package** (clone) se scope na nové jméno. Dnes to nikde v docs ani
   v kódu není, vzniká ručně v GUI.
3. **Šablony jedou přes device groupy**: CLI template group `Spoke_CLI_group` na `Spoke-dual` /
   `Spoke-single`, SD-WAN template `Spoke-dual` na `SDWAN_dual_prio_inet1`. Nový device stačí
   zařadit do skupin (dnes ruční krok 7 v SITE_NEW_GUIDE) a dostane IPsec k hubům, BGP, cam
   port12, DNS forwarder, loopback, syslog, SNMP preset.
4. **Strečno nemá žádné Category M proměnné** (static_route_*, powerbox, WTP_Data jsou prázdné).
   Site-specifikum je jinde: **IPsec tunel `Strecno` k dochádzkovému systému** + normalized
   interface `dochazka` + address mapping + 2 policy. To musí scénář přenést explicitně, a
   tunel přitom mění interface `wan1 → port1`.
5. **Bridged na Strečně = 2 SSID (Zebra, Packeta) mají hotové bridged VAP** (`wlt_zebra_B`
   vlan 700, `w_packeta_zvo_b` vlan 702), Levoča/Michalovce už tak jedou na FAP231F. **Guest
   `packeta-guest` bridged variantu nemá** — rozhodnutí. Pro platformu **FAP-U231F (6 z 12 AP)
   neexistuje žádný bridged WTP profil**, musí vzniknout.
6. **NetBox Strečna je na starém modelu**: chybí `lan_subnet` (FG přitom LAN 172.19.16.0/24
   VLAN 16 má), chybí IP na port10 a kabel k proxy, `ha_position` prázdné, zebra prefix
   172.20.52.0/23 má legacy description a žádný VLAN tag. Před swapem se musí NB dorovnat,
   jinak shuttle LAN přealokuje a změní adresy.
7. **Fyzika 100F → 120G**: WAN `wan1/wan2 → port1/port2`, fortilink `port19+20 → port15+16`,
   `fortilink-split-interface` 1 → 0 (Bezdečín). Zabbix makra `{$WAN.CIR:"wan1"}` →
   `"port1"`. NB interface názvy jsou v obou šablonách `port N` s mezerou, kabel na `port 10`
   funguje i na 120G.

---

## 0a. Upřesnění operátora (31. 8. večer) — platí nad vším níže

1. **Bootstrap VLAN v HQ vidí do internetu i do SD-WAN.** Nová FG by po installu navázala
   tunely k hubům a začala propagovat prefixy Strečna do SD-WAN vedle živé staré. Ve FMG
   samotném dvě zařízení s různými jmény vedle sebe **žít můžou** (device DB, variables,
   package, skupiny); nesmí být zároveň **online s tunely**. Dvě cesty, rozhodnout:
   - **(a) blok z bootstrap VLAN na huby** (193.179.216.17/.19 Balabenka, 108.143.161.177
     Azure) na HQ FG, FGFM k FMG 193.179.246.243 povolit → install v HQ proběhne celý, tunely
     se nenaváží, na depu naskočí. Jednorázová změna v HQ, pokryje všech 36 výměn. Doporučeno.
   - **(b) install až po cutoveru**: v HQ jen stage (device DB + package + skupiny), Install
     Wizard z kanceláře, až nová FG běží na depu a stará je smazaná. Technik pořád jen mění
     box, ale mezi zapnutím a installem má depo jen `Spoke_init`.
   - **(c) staging s placeholder IP hubů — DOPORUČENO (ověřeno v šablonách 31. 8.):** veřejné
     IP hubů vstupují do konfigurace spoke **jen** přes variables `h1_inet1_ip`, `h1_inet2_ip`,
     `h2_inet1_ip`, `h2_inet2_ip` (`spoke_ipsec` → `remote-gw`, `Spoke_static_route` → host
     routy). `spoke_bgp` má neighbory na tunnel IP 172.16.x (jen přes tunely), SD-WAN template
     `Spoke-dual` odkazuje jen jména interfaců a health-checky na loopbacky hubů přes overlay.
     Když scénář po `site_provision` přepíše na **novém** zařízení ty 4 proměnné na
     TEST-NET adresy (192.0.2.1–4; specific-scope push, stará FG netknuta), install proběhne
     celý (LAN, VLANy, DHCP, wifi, DNS forwarder, cam, loopback, SNMP, package), tunely se
     **nikdy nenaváží**, BGP nevznikne, nic se do SD-WAN nepropaguje — ať je box doma, v HQ,
     nebo kdekoli s internetem. FGFM k FMG jede po WAN normálně.
     **Cutover:** box na depu naběhne s plnou lokální konfigurací (LAN/wifi/DHCP fungují,
     internet přes `underlay` funguje, overlay ne) → z kanceláře `site_provision` (vrátí reálné
     hub IP z `hub.yml`, Category A) + Install Wizard → tunely + BGP naskočí během minut.
     Staré zařízení stačí smazat kdykoli potom (jiné jméno, kolize není).
     Vedlejší efekty ve stagingu: SD-WAN health-checky červené, FAZ dostává logy z domova —
     kosmetika. Hlídat: `verify_site` má po cutoveru assertnout, že hub vars = `hub.yml`, aby
     placeholder nezůstal zapomenutý. Site-specifický tunel `Strecno` řešit stejně (placeholder
     remote-gw, nebo přidat až při cutoveru). Blok (a) lze přidat jako pás navíc, nutný není.
2. **Strečno mění i switche a zapojení** → pre-stage `managed-switch`/`wtp` se **nedělá**.
   Na ostatních lokalitách (jen box) chce operátor **report SN** AP a switchů (jméno, serial,
   model, port/VLAN souhrn) do `out/<slug>/`, a do FMG si je dohází ručně po smazání starého
   boxu. Automatický zápis wtp/managed-switch je mimo rozsah.
3. **Fortilink membery zadá operátor jako vstup scénáře** (např. `port15,port16`), nic se
   neodvozuje ani nekopíruje.
4. **Package clone + úprava interfaců udělá operátor ručně** v GUI podle checklistu (§5),
   scénář jen ověří, že package existuje a má scope na nové jméno.
5. **PSK wifi jsou stejné** u tunelového i bridged VAP, mění se jen typ interface → parita se
   neověřuje. PSK tunelu `Strecno` zůstává k dohledání.
6. Smazání starého boxu z FMG = až po cutoveru; teprve pak operátor dohází switche/AP.

---

## 1. Jak shuttle modeluje site a FG (ověřeno v kódu)

| věc | kde | důsledek pro swap |
|---|---|---|
| FMG device name | Site CF `fmg_device_name`; `fortimanager_device_assert` bere CF s přednostní, fallback `-e fmg_device_name` jen když CF prázdné | přepnout CF = shuttle míří na nový device; override přes všechny role by byl víc kódu |
| `inet1_name/inet2_name` | Site CF, odvozeno **jen při založení** ze serialu (`FG..G` → port1/port2) | pro Strečno teď `wan1/wan2`; pro 120G přepsat na `port1/port2` |
| device_type v NB | z FMG `platform_str` (`FortiGate-120G` → `fortigate-120g`) | nový device musí být ve FMG dřív než NB krok |
| `sn1`, WAN CF (GN, inet_ip/mask/gw) | Device `fw-1.<slug>.packeta`, role firewall, `ha_position=1` | `netbox_site_lookup` bere firewally `site+role=firewall`, **bez filtru status** → nutný filtr (staged > active) |
| variables push | `compute_vars.yml` → `push_vars.yml`, specific-scope URL, bezpečné | funguje pro nové jméno beze změny |
| interface config | `fortimanager_interface_config` (fortilink IP, HA gate → port10 nebo VLAN137, port12→sec, apmgmt, lan16, packeta_mgmt, foxpost, wifi katalog) | standalone runner `test_interface_config.yml -e fmg_dev=` |
| wifi výběr | `site_wifi_ssids` (wizard) nebo `wifi_site_ssids` (jen BEZDECIN) | na existujícím situ **neznámý** → odvodit z NB prefixů `SSID:<name>_b` (verify_site to už umí) |
| re-push (menu 3) | `site_provision.yml` = jen variables (+ install) | ne interfacy, ne wifi, ne Zabbix |
| ZTP promote | `site_init.yml`, idempotentní **podle jména** | s novým jménem žádná kolize |
| Zabbix host | `zabbix_host_sync`, jméno = `fmg_device_name`, loopback /32, makra WAN.CIR z CF | rename hosta chybí (jen create/skip) |
| enrich | `site_enrich.yml`: AP rename gap-fill + collect, switche collect, UPS, DHCP rezervace, NB postsync (serial+site), LLDP kabely | funguje na nový device; AP jména by se gap-fillem přečíslovala |
| retire/delete | `site_delete.yml` | maže IPAM → **nepoužít** |
| verify | `verify_site.yml` | model-aware; **bug: čeká proxy os_version ubuntu_22.04, standard je 24.04** (Bezdečín padá 1/68 jen na tom) |
| FMG device lifecycle | jen `get` + `promote` | delete/rename/package clone shuttle nemá |

`nb_device_upsert` PATCHuje `device_type` i `serial`, ale NetBox při změně typu **nepřegeneruje
interfacy**.

---

## 2. Strečno dnes

### 2.1 NetBox (`sk-depot-1`, id 19)

Site CF: `fmg_device_name=PAC-SK-STR-FW`, `fmg_group_name=Spoke-dual`, `inet1_name=wan1`,
`inet2_name=wan2`, `ap_controller=fortinet`, `site_alias=sk-strecno`, region Slovakia.

Prefixy (VRF SD-WAN):

| prefix | role | VLAN | poznámka |
|---|---|---|---|
| 172.16.53.128/26 | fortilink_subnet | 1 | |
| 172.16.53.192/26 | apmgmt_subnet | 7 | |
| 172.16.32.112/30 | zabbix_subnet | 137 | gw .113 na port10 ve FMG ano, **v NB IP ani kabel nejsou** |
| 172.16.37.192/28 | packeta_mgmt_subnet | 138 | |
| 172.16.35.37/32 | loopback-snmp | — | |
| 172.20.52.0/23 | wlt_zebra_subnet | **žádný** | description „sk-depot-1 - wlt_zebra network (readdress)" — nematchne `SSID:wlt_zebra_b` ani `wlt_zebra` |
| — | **lan_subnet chybí** | 16 | FG má lan 172.19.16.0/24 |

Zařízení: `fw-1` fortigate-100f **FG100FTK21009830**, standalone, `ha_position` prázdné,
primary IP = WAN1 213.81.177.50/30. 12 AP: 6× **fortiap-pu231fth** (ap-1..6), 6× **fortiap-fp231f**
(ap-7..12). 4 switche: sw-1 fortiswitch-148e-poe S148FFTF21003494, sw-2/3/4 fortiswitch-124f-fpoe
(S124EP5920009817, S124FFTF21012655, S124FFTF21012658). Proxy `zabbix-proxy-1-pro` NUC
172.16.32.114/30. UPS ×3 (2× NetMan, 1× APC), teploměr th-1, CPE ×2 (T-Mobile).

Kabely (24): `cpe-1:ETH-1 ↔ fw-1:WAN1`, `cpe-2:ETH-1 ↔ fw-1:WAN2`, `fw-1:port 19 ↔ sw-1:port 47`,
`fw-1:port 20 ↔ sw-1:port 48`, AP↔switch: sw-2 porty 1–5 (ap-3,1,4,5,2), sw-1 port 11 (ap-6),
sw-3 porty 1,3,5 (ap-8,7,9), sw-4 porty 1,3,5 (ap-12,11,10). UPS/th na sw portech. **Chybí kabel
fw-1:port 10 ↔ proxy:eth0.**

NB interfacy fw-1 (100F šablona): DMZ, HA1, HA2, MGMT, WAN1, WAN2, port 1–20, x1, x2.
Pro srovnání 120G (cz-depot-28): HA, MGMT, port 1–24, x1–x4.

verify_site: **43/49**. Padá: VLAN 16 lan (n/a), FW CF ha_position, proxy os_version, port10 IP
v NB, kabel port10↔proxy, normalized `sec`→port12 (STR mapping nemá, package používá `port12`
napřímo).

### 2.2 FMG — device a package

Device: FortiGate-100F, FortiOS **7.4.11**, `ha_mode=0`, oid 24920, conn ok.
Package **`Spoke_SK_STRECNO`** (30 policy), scope jen PAC-SK-STR-FW. Tvar:

- hub/underlay základ: fortilink↔overlay, ntp, lan-to-inet, inet-to-sec / sec-to-inet
  (**`port12` napřímo**, ne normalized `sec`), monitoring-to-*, sdwan-to-packeta_mgmt /
  monitoring / loopback_snmp, any-to-azure-internal-dns, lan-subs-to-azure-NAT
  (src `lan_subnet`, `wlt_packeta_zvo address`, nat=1).
- wifi (tunelové interfacy): `wlt_zebra`→overlay/underlay, `wlt_packeta_zvo`→underlay, lan↔wlt_packeta,
  wlt_packeta→sec, `wlt_packgues_sk`→underlay, monitoring→wlt_packeta/wlt_zebra.
- **site-specifické**: `31 dochadzka-to-lan` (dochazka→lan, dst `Dochadzkovy system`),
  `32 lan-to-dochadzka` (lan→dochazka, src `Dochadzkovy system`).

Address objekty: `Dochadzkovy system` má **per-device mapping pro STR = 172.19.16.251/32**.
`lan_subnet` je sdílený ipmask objekt s **defaultem 172.19.16.0/24** (mapping jen BUK) —
tj. default sdíleného objektu je náhodou Strečno; Bezdečín místo toho používá `<intf> address`
(type 16, odvozené z interface, portable).

### 2.3 FMG — interfacy, DHCP, VPN na zařízení

| interface | typ | IP | poznámka |
|---|---|---|---|
| wan1 / wan2 | fyz | 213.81.177.50/30, 213.81.228.54/30 | ISP1/ISP2, GN152988/GN152989 |
| port10 | fyz | 172.16.32.113/30 | alias monitoring, normalized `monitoring` |
| port12 | fyz | 192.168.57.1/24 | alias **cam**, DHCP id 20 .50–.99 — **z CLI template `Spoke_cam_interface`** |
| port19+port20 | aggregate | — | **fortilink** 172.16.53.129/26, `fortilink-split-interface=1`, lacp |
| lan | vlan 16 na fortilink | **172.19.16.1/24**, DHCP .10–.254 | v NB chybí |
| apmgmt | vlan 7 | 172.16.53.193/26 | |
| packeta_mgmt | vlan 138 | 172.16.37.193/16→/28 | |
| wlt_zebra | tunel SSID | 172.20.52.1/23, DHCP .10–.254 | |
| wlt_packeta_zvo | tunel SSID | 172.19.20.1/23 | |
| wlt_packgues_sk | tunel SSID | 172.19.22.1/23, DNS 8.8.8.8/1.1.1.1 | guest |
| loopback_snmp | lo | 172.16.35.37/32 | |
| dmz | fyz | 10.10.10.1/24 | nejspíš default/nepoužité — ověřit |
| mgmt | fyz | 192.168.88.88/24 | |
| **Strecno** | IPsec p1 na **wan1** | remote 195.168.51.242, IKEv2 | dochádzka; p2 172.19.16.251/32 ↔ 172.26.101.176/28 |
| hub1/2_inet1/2 | IPsec | z template `spoke_ipsec` přes `inet1_name` | |

Static routes: seq 1 `172.26.101.176/28 dev Strecno` (site-specifická); 800–1002 ze šablon
(h1-h2 propoj, tunnel subnets, hub adresy, FMG, default, blackhole 10/172.16/192.168).

Normalized `dochazka`: 9 SK sitů (BRA-OFF Vajnorska, TRIB, LEVOCA, NIT, NPO, **STR→Strecno**,
TRENCIN, TRN, ZVO). Normalized `monitoring`→port10 na 40 sitech, `sec`→port12 na 28 (STR ne).

### 2.4 FMG — switche (per-device CMDB) a AP

Managed-switch CMDB: STR-SW-1 (52 portů), STR-SW-2/3/4 (28). Porty: native `lan` většina,
AP porty native `apmgmt` (SW-1: 1 port, SW-2: 5, SW-3: 4, SW-4: 3), UPS/th native
`packeta_mgmt`, uplinky `_default`. **allowed-vlans všude jen `quarantine`** — tunelový model.
`switch-controller global` shodné s Bezdečínem. Fortilink: `fortilink-neighbor-detect=1`,
`auto-auth-extension-device=0`.

WTP: 12 AP, `admin=2`, profily **`FAPU231F_strecno_NEW`** (platform 72, PU231F) a
**`FAP231F_strecno_NEW`** (platform 68, FP231F); obě s tunelovými VAP
`wlt_zebra, wlt_packeta_zvo, wlt_packgues_sk`. Jména AP-1..AP-12 sedí na NB.

### 2.5 FMG — variables (82 mapovaných pro STR)

Podstatné: `hostname=PAC-SK-STR-FW`, `sn1=FG100FTK21009830`, `sn2=""`, `inet1_name=wan1`,
`inet2_name=wan2`, `inet1/2_*` statické WAN, `lan_fortilink_*` (intf `fortilink`),
`lan_mgmt_*` (intf `vlan7`), `lan_monitor_*`, `lan_zabbix_*`, `loopback_snmp_*`,
`zabbix_proxy_ip=172.16.32.114`, `lan_wlt_zebra_network=172.20.52.0/23`,
**`dns_listen_intf=port10,lan,wlt_packeta_zvo,wlt_zebra`** (legacy tvar).
**Prázdné: všech 8× static_route_*, vm_interface_number, foxpost, DC rodiny. Žádný powerbox_ip,
žádné WTP_Data.** → kopie Category M se pro Strečno zúží na nic; kopírovat OLD→NEW všechny
mapované je stejně levné a bezpečné (NB-driven se pak přepíšou).

### 2.6 Šablony (odkud co přichází)

- CLI template group **`Spoke_CLI_group`** → scope `Spoke-single`, `Spoke-dual`, PAC-CZ-NEH-FW.
  Členy: Blackhole_private, DHCP_fortilink, GPS, spoke_ipsec, spoke_bgp, Spoke_tunnel_interface,
  static_route, **Spoke_cam_interface**, admin_account, Spoke_static_route, Spoke_Description,
  local_in_policy, syslog, packeta_snmp-preset, loopback_snmp, config_fortianalyzer,
  **dns_forwarder_packeta**, DNS.
- `Spoke_CLI_group-init` → `Spoke-dual_init`, `Spoke-single_init` (bez DNS/syslog/loopback…).
- SD-WAN template `Spoke-dual` → `SDWAN_dual_prio_inet1` (+ HKR-BREZ, NEH); `Spoke-single`.
- `dns_forwarder_packeta`: dns-database (works.packeta.com, packeta.com → 10.128.0.132 se
  source-ip loopback, privatelink zóny) + `config system dns-server` pro každý interface v
  `$(dns_listen_intf)` (mode recursive, dnsfilter `packeta-default`).
- `spoke_ipsec` / `Spoke_Description` / `DHCP_fortilink` čtou `inet1_name`, `inet1_GN`,
  `lan_fortilink_*` → **jména WAN portů jdou správně z variables**, nic v šabloně není natvrdo.
- **Fortilink membery (port19/20 vs port15/16) šablona nenastavuje** — buď cleanup script
  `cleanup default configuration 120G`, nebo device settings; ověřit, kde to Bezdečín dostal.
  Existují CLI šablony `fortilink_split_interface_disable/enable` a
  `split_hardware_switch_ports_*`.
- Package `Spoke_init` = 1 policy any→any (fáze mezi promote a finálním package).

### 2.7 Zabbix

Host `PAC-SK-STR-FW` (id 13580), SNMP iface loopback 172.16.35.37, proxyid 0, groups Device
Fortinet / Location SK / sk-depot-1, templates FortiGate SNMP + WAN CIR, makra
`{$WAN.CIR:"wan1"}=100`, `{$WAN.CIR:"wan2"}=50`, `{$WAN.IF.MATCHES}=^(wan1|wan2)$`, SITE.LAT/LON.
Bezdečín má tvar `port1/port2`.

---

## 3. Bezdečín jako bridged reference (120G, FortiOS 7.4.9)

- WAN port1/port2; port10 monitoring 172.16.32.189/30; port12 cam (stejná šablona); fortilink
  **port15+port16**, `fortilink-split-interface=0`; lan 172.22.0.1/24 vlan 16; vl700_Zebra
  172.20.100.1/23, vl701_Zasilkovn 172.24.1.1/24, vl702_PacketaVD 172.24.0.1/24 na fortilinku;
  bridged VAP interfacy `wlt_zebra_B`, `wlt_zasilkovn_b`, `wlt_packetavd_b` bez IP; prisonlan vlan 34.
- Package `Spoke_CZ_Bezdecin` (25 policy): wifi pravidla na `vl700/701/702` s `<intf> address`,
  `sec` normalized, žádné site-specifické tunely.
- Switche: AP porty **native apmgmt + allowed `vl702_PacketaVD,vl701_Zasilkovn,vl700_Zebra,quarantine`**,
  ostatní native lan, UPS packeta_mgmt, uplink `_default`.
- WTP: 20× FAP443K, profil `FAP443K-Bezdecin` r1 `wlt_zebra_B`, r2 `wlt_zasilkovn_b,wlt_zebra_B,wlt_packetavd_b`.
- NB: lan_subnet, wifi prefixy s VLAN tagy (700/701/702), kabel port10↔proxy, ha_position=1.
  verify 67/68 (jen os_version).

---

## 4. Rozdíly 100F → 120G, které scénář musí ošetřit

| oblast | FG100F (STR) | FG120G (Bezdečín) | akce |
|---|---|---|---|
| WAN | wan1/wan2 | port1/port2 | Site CF `inet1/2_name`; NB kabely CPE↔`port 1/2`; Zabbix makra |
| fortilink | port19+port20, split=1 | port15+port16, split=0 | zjistit zdroj (cleanup script / device settings); NB kabely sw-1↔`port 15/16` |
| porty v NB | DMZ, HA1/2, WAN1/2, port 1–20, x1–2 | HA, MGMT, port 1–24, x1–4 | nový device object → správná šablona |
| monitoring | port10 | port10 | stejné |
| cam | port12 (template) | port12 (template), normalized `sec` | STR package používá `port12` napřímo → při clone sjednotit na `sec` |
| SD-WAN/IPsec | z variables | z variables | žádná akce, jen správné `inet*_name` |
| FortiOS | 7.4.11 | 7.4.9 | firmware na nové 120G sladit s flotilou 120G |

---

## 5. Mapování Strečna na nový (bridged) model

| dnes (tunel) | SSID | nový (bridged) | VLAN | subnet |
|---|---|---|---|---|
| `wlt_zebra` 172.20.52.0/23 | Zebra | `wlt_zebra_B` | 700 `vl700_Zebra` | **doporučení: ponechat 172.20.52.0/23** — prefix v NB už je (role wlt_zebra_subnet), stačí přepsat description na `SSID:wlt_zebra_b` a přidat VLAN 700; `alloc_wifi` má reuse guard, ale matchuje description, jinak by alokoval nový /23 |
| `wlt_packeta_zvo` 172.19.20.0/23 | Packeta | `w_packeta_zvo_b` | 702 `vl702_PacketaVD` | nový /24 z 172.24.0.0/15 (katalog `wlt_packeta_zvo` → var `lan_wlt_PacketaVD`); adresy klientů se změní |
| `wlt_packgues_sk` 172.19.22.0/23 | packeta-guest | **nemá bridged VAP** | — | rozhodnout: (a) zrušit, (b) nechat tunelový guest i na 120G (jediný tunelový SSID, drží se v profilu), (c) založit `wlt_packgues_sk_b` + VLAN + prefix v katalogu |

- Vysílané SSID a security (WPA2-PSK, sec=16) mají tunel i bridged VAP shodné; **PSK je
  šifrovaný, paritu musí potvrdit operátor** (jinak Zebry nepřipojí).
- `dns_listen_intf` nový = `lan,vl700_Zebra,vl702_PacketaVD` (rozhodnutí 31. 8.: jen lan +
  Packeta wifi; port10 a Foxpost ne). Šablona dns_forwarder na to sedí.
- **WTP profily**: pro FP231F existuje vzor `FAP231F_Levoca_NEW` (r1 `wlt_zebra_B`, r2
  `w_packeta_zvo_b,wlt_zebra_B`) = přesně STR minus guest. Pro **FAP-U231F (platform 72)
  bridged profil neexistuje** → založit `FAPU231F_strecno_B` klonem z `FAPU231F_strecno_NEW`
  se záměnou VAP.
- **Switch porty s AP** (13 portů: SW-1 p11; SW-2 p1–5; SW-3 p1,3,5(,19?); SW-4 p1,3,5) →
  allowed-vlans + `vl700_Zebra,vl702_PacketaVD` (+ vl701 kdyby Zasilkovna). NB LLDP kabely
  říkají které porty; FMG CMDB říká, které jsou native apmgmt — obojí sedí (SW-3 má 4 apmgmt
  porty vs 3 AP kabely v NB, SW-1 1 vs 1) → před zápisem porovnat.
- **LAN 16**: ponechat 172.19.16.0/24 → do NB importovat jako `lan_subnet` (VLAN 16) —
  `reconcile_prefixes.yml` (FMG→NB) to umí; pak `lan_vlan.yml` vyrobí stejnou mapu na novém.
  **Nealokovat nový unikátní LAN**, změnil by adresy drátovým klientům a dochádzkovému systému
  (172.19.16.251).
- Package pro nový device: clone. Dvě cesty: (a) clone `Spoke_SK_STRECNO` a přepsat wifi
  interfacy `wlt_zebra→vl700_Zebra`, `wlt_packeta_zvo→vl702_PacketaVD`, `port12→sec`, guest dle
  rozhodnutí; (b) clone bridged vzoru (Bezdečín) a doplnit `dochazka` pravidla 31/32 +
  lan-subs-to-azure-NAT ekvivalent. **(a) zachová site-specifika, (b) zachová standard** —
  doporučuju (a) pro první běh, aby nic nevypadlo, a diff proti (b) jako kontrolu.

## 6. Site-specifický balík Strečna (co se musí přenést ručně / kopií)

1. IPsec `Strecno`: phase1 na **port1** (ne wan1), remote 195.168.51.242, IKEv2, proposal 23,
   net-device 0; phase2 172.19.16.251/32 ↔ 172.26.101.176/28; **PSK** (šifrovaný, zjistit).
2. Static route seq 1 `172.26.101.176/28 dev Strecno`.
3. Normalized `dochazka` → mapping pro nový device = `Strecno`.
4. Address `Dochadzkovy system` → per-device mapping nový device = 172.19.16.251/32.
5. Policy 31/32 v novém package.
6. Guest SSID (viz 5).
7. `dmz` 10.10.10.1/24 — zjistit, jestli se používá.
Nic z toho není v NetBoxu ani v shuttle. Stejný vzor má 8 dalších SK sitů (dochazka), takže
„kopie site-specifik OLD→NEW" má smysl jako obecný krok, ne jednorázovka.

---

## 7. Co shuttle umí dnes vs. co chybí (pro scénář)

| krok | dnes | chybí / úprava |
|---|---|---|
| NB dorovnání starého situ (lan_subnet, port10 IP+kabel, ha_position, zebra prefix retag) | `reconcile_prefixes.yml` (FMG→NB) pro prefixy; zbytek ne | „site pre-swap normalize" task; retag wifi prefixu (description + VLAN) místo alokace |
| nový FG device v NB | `nb_device_upsert` | „replace firewall": rename starého (`fw-1-100f.<slug>`), nový `fw-1` status `staged`, kopie WAN CF, přepis Site CF; cutover statusy + přesun IP/kabelů |
| lookup firewallu | `site+role=firewall` | filtr status: `staged` > `active`, ostatní ignorovat |
| wifi výběr existujícího situ | wizard / statická mapa | odvodit z NB prefixů `SSID:<name>_b` (verify_site to dělá) |
| ZTP promote nového jména | `site_init.yml` | beze změny |
| variables | `site_provision.yml` | beze změny; předtím **kopie všech mapovaných OLD→NEW** (nový malý krok) |
| interface config | `test_interface_config.yml` | beze změny (HA gate, port10, sec, lan16, wifi) |
| package | ruční GUI | **zůstává ruční** (rozhodnutí 0a.4): clone + scope na nové jméno + přepis interfaců podle checklistu §5; scénář jen assertne, že package se scope na nové jméno existuje |
| device groupy + SD-WAN template | ruční krok 7 | `dvmdb/adom/SDWAN/group/<g>/object member` add (nový, nebo ruční) |
| site-specifika (VPN, routes, dochazka, address map, policy) | nic | kopie device-level objektů OLD→NEW s náhradou interface (`wan1→port1`) — nový krok |
| fortilink membery | nic | **vstup scénáře** (`-e fortilink_members=port15,port16`), zapsat do device DB nové FG (rozhodnutí 0a.3) |
| AP + switche | `aps_fortinet` / `switches_fortilink` čtou CMDB starého | **jen report** `out/<slug>/swap_devices.md`: jméno, serial, model, profil, AP-port↔switch-port z NB kabelů, per-switch souhrn native/allowed VLAN (rozhodnutí 0a.2). Žádný zápis wtp/managed-switch. Na Strečně se ani report nepoužije (nové switche i zapojení) |
| bridged WTP profily | nic | ruční: `FAP231F_strecno_B` (vzor Levoca), `FAPU231F_strecno_B` (nový pro platform 72); scénář vypíše, které chybí |
| Zabbix | create/skip | rename hosta + přepis WAN maker (po flipu CF stačí re-run sync pro makra, rename je nový) |
| install | `fortimanager_install` | beze změny |
| enrich / verify | beze změny | verify: opravit očekávání os_version 24.04 |
| FMG smazání starého | nic | ruční GUI po cutoveru (nebo `dvm/cmd/del/device`) |

---

## 8. Ověřit před stavbou (nejde vyčíst z kódu)

1. **Bootstrap VLAN 999 vidí do internetu i SD-WAN — potvrzeno operátorem.** Nová FG by po
   installu v HQ navázala IPsec+BGP se stejnými tunnel IP, loopbackem a LAN prefixy jako živé
   Strečno. **Rozhodnout mezi (a) blokem hub IP z bootstrap VLAN a (b) installem až po cutoveru**
   (viz 0a.1). Bez toho se scénář nesmí pustit.
2. **Autorizace FortiSwitchů na nové FG** po ručním doházení podle SN (`fortilink-neighbor-detect=1`,
   `auto-auth-extension-device=0`) — ruční authorize v GUI, nebo pre-auth záznam. Pro Strečno
   nerelevantní (nové switche), pro ostatní lokality ano.
3. **`fortilink-split-interface` na 120G** (Bezdečín 0, Strečno 1): membery zadá operátor, ale
   split musí scénář nastavit vědomě (CLI šablona `fortilink_split_interface_disable` existuje).
4. **PSK tunelu `Strecno`** (dochádzka) — dohledat před přenosem VPN na port1. PSK wifi řešit
   netřeba (0a.5).
5. Guest `packeta-guest` — rozhodnutí (zrušit / tunel / bridged).
6. `dmz` 10.10.10.1/24 na STR — používá se?
7. Nic na hubu neodkazuje na spoke **jménem** (BGP neighbor po tunnel IP; ověřit hub package/
   CLI šablony `Hub_bgp_*`, `Hub_prefix-list_*`).
8. Který FortiOS na novou 120G (flotila 120G je na 7.4.9, Strečno 100F na 7.4.11).

---

## 9. Kostra scénáře (k rozpracování)

**Jména**: nové FMG jméno musí projít `^PAC-[A-Z]{2}-[A-Z0-9-]+-FW$`; návrh generační token,
např. `PAC-SK-STR-2-FW` (nebo `-G-`). NB device: starý → `fw-1-100f.sk-depot-1.packeta`
(status `active` do cutoveru, pak `decommissioning`), nový → `fw-1.sk-depot-1.packeta`
(`staged` → `active`). Site CF se přepnou hned na začátku přípravy (cílový stav).

**A. Příprava v HQ** (stará 100F běží, netknuta)
1. Pre-check: `verify_site` baseline, dump STR (variables, mapy, wtp, switche, VPN, routes,
   address mapy, package policy) do `out/backups/` — read-only snapshot pro diff i rollback.
2. NB normalize: import lan_subnet 172.19.16.0/24 vlan 16 (reconcile), retag zebra prefixu
   (`SSID:wlt_zebra_b`, VLAN 700), alokovat `wlt_packeta_zvo` /24 z 172.24.0.0/15, port10 IP +
   kabel doplnit, ha_position=1, proxy os_version.
3. NB replace firewall: rename starého, nový `fw-1` staged (120G, nový SN, WAN CF kopie), Site
   CF `fmg_device_name=<NEW>`, `inet1/2_name=port1/port2`.
4. ZTP promote `<NEW>` z bootstrap VLAN (ověřeno bod 8.1), cleanup script 120G (ruční).
5. FMG kopie variables OLD→NEW (všechny mapované), pak `site_provision` (přepíše hostname,
   sn1, inet*, dns_listen_intf).
6. `test_interface_config -e dry_run=false` s wifi výběrem z NB → fortilink IP, port10/monitoring,
   port12→sec, apmgmt, lan16 (stejná /24), packeta_mgmt, vl700/vl702 s DHCP.
7. Site-specifika: IPsec Strecno (port1), static route, dochazka mapping, address mapping.
8. Package **ručně**: clone `Spoke_SK_STRECNO` → `Spoke_SK_STRECNO_2` (nebo konvence), scope
   `<NEW>`, přepis wifi/sec interfaců podle §5; device groupy `Spoke-dual` +
   `SDWAN_dual_prio_inet1`. Scénář: assert package se scope na `<NEW>` existuje, assert členství
   ve skupinách, vypsat chybějící bridged WTP profily.
9. Fortilink membery z vstupu (`-e fortilink_members=`) + split podle rozhodnutí 8.3 → device DB.
   Report `out/<slug>/swap_devices.md` (AP + switche: jméno, SN, model, profil, porty) — na
   Strečně informativní, na dalších lokalitách podklad pro ruční doházení do FMG.
10. Install device settings + package **jen pokud platí 0a.1(a)** (blok hub IP z bootstrap VLAN);
    jinak stage only a install až v kroku 12. Vypnout, zabalit.

**B. Depo**: vypnout starou, vyměnit, přepojit (CPE→port1/port2, uplinky switchů→fortilink
membery, proxy→port10, cam→port12; na Strečně i nové switche a zapojení), zapnout. Konec pro
technika.

**C. Kancelář po cutoveru**
11. FMG: smazat starý device (uvolní SN switchů/AP), pak ručně dohází switche a AP podle
    reportu z kroku 9 (mimo Strečno), authorize.
12. Pokud 0a.1(b): Install Wizard device settings + package na `<NEW>`.
13. NB: statusy (starý `decommissioning`, nový `active`), přesun IP port10 + kabelů (WAN,
    fortilink, proxy) na nový device; na Strečně nové switche přes `site_enrich`.
14. Zabbix: rename `PAC-SK-STR-FW → <NEW>`, makra WAN.CIR/IF.MATCHES na port1/port2.
15. `site_enrich` (AP jména gap-fill nebo z NB), `verify_site`; smazat starý package.

Přepínače přípravy: wifi bridged (ano), guest (rozhodnutí), LAN16 (ponechat adresy),
loopback (ano, existuje), fortilink membery (vstup), install v HQ (jen s blokem hubů).

---

## 10. Rizika

- Dvě FG se stejnými tunnel IP / loopback / prefixy online zároveň během installu v HQ —
  potvrzeno, že bootstrap VLAN to umožňuje (0a.1); bez bloku hubů nebo odloženého installu se
  scénář nepouští.
- Na lokalitách „jen box": mezi smazáním starého a ručním doházením switchů/AP podle SN je okno,
  kdy nová FG switche neautorizuje → depo bez LAN/wifi. Report SN musí být hotový před cutoverem
  a doházení první věc po něm.
- AP přečíslování gap-fillem při `site_enrich` na nové FG (wtp se nepředstageují) — buď jména
  z NB, nebo přijmout nová čísla a nechat NB postsync přepsat.
- Zebra klienti: PSK parita; nový subnet u Packeta SSID; DHCP rezervace/whitelisty na starých
  wifi subnetech 172.19.20–23.
- Dochádzka: tunel na novém WAN portu + nový PSK entry u protistrany? (protistrana vidí stejnou
  public IP, měnit nic nemusí, pokud PSK přeneseme).
- `lan_subnet` sdílený address objekt s defaultem = Strečno: pokud clone package nahradí
  `lan_subnet` za `lan address`, sjednotí se to s Bezdečínem.
- verify_site false-negative na os_version.

## 11. Pomůcky z této analýzy

- `/tmp/shuttle-wip-backup/fmg_dump.py` (WSL, dočasné) — read-only JSON-RPC dumper; dumpy
  v podadresářích A–K (STR/BEZ wtp, managed-switch, vap, wtp-profile, fsp/vlan, dynamic
  interface, variables, packages, policy, VPN, routes, šablony).
- `verify_sk-depot-1.log`, `verify_cz-depot-28.log` tamtéž.
- Zabbix host ids: STR 13580, BEZ 13587.

---

## 12. Stav 10. 9. 2026 — kód napsaný, živě neběžel

Branch **`feat/site-swap`** v shuttle (GitHub, commit `c9b957d`, PR čeká). Vychází z §0a
(varianta **c**: nové FMG jméno + TEST-NET placeholder v `h1/h2_inet1/2_ip`).

- `playbooks/site_swap.yml` — `-e phase=precheck|prepare|cutover`, dry-run default,
  `-e swap_apply=true` zapisuje. `bin/run_site_swap.sh precheck|prepare|cutover <slug>`
  řídí vstupy, ZTP promote, cleanup pauzu a dry-run → confirm → apply. Menu **S**.
- `roles/site_swap/`: `snapshot` (JSON do `out/backups/`), `fmg_checks` (package se scope
  na nové jméno, device groupy, bridged WTP profily — hlídá, nezapisuje), `nb_normalize`
  (ha_position, retag wifi prefixů, port10 IP + kabel; chybějící `lan_subnet` **zastaví**),
  `nb_replace_fw` (starý fw-1 → `fw-1-<model>`, nový staged, Site CF flip), `copy_vars`
  (old→new, Category M přežije), `hub_placeholders` (placeholder/restore), `site_specific`
  (ne-hubové IPsec s náhradou `wan1→port1`, routy < seq 800, normalized intf a address
  mapy), `report` (`out/<slug>/swap_devices.md` pro ruční doházení AP/switchů), `nb_cutover`
  (statusy, přesun IP/kabelů podle mapy, FMG `port1` → NB `port 1`).
- Mimo swap: `netbox_site_lookup` staged > active + `nb_wifi_ssids`; `zabbix_host_sync
  tasks_from=rename`; `verify_site` os_version 24.04 + hub-placeholder guard.
- Ověření: syntax-check v kontejneru, YAML, bash -n, menu 70 sloupců, **harness 38 Jinja
  kontrol** proti fixturám ze Strečna (`~/tmp-claude/test_swap_jinja.py` ve WSL) — odhalil
  5 chyb (phase1name je list, mapování device u rout, substring kolize wlt_packeta/
  wlt_packetavd, `vlan` na switch portech je list, robustnost při chybějícím `data`).
- Runbook `docs/SITE_SWAP.md`; otevřené body pro pilot v `CLAUDE.md` §8.0 (a)–(g).

**Než se pustí na Strečno:** LAN 172.19.16.0/24 není v NB → `prepare` zastaví; řešení je
`-e swap_import_lan=true` (přečte fsp/vlan `lan` mapu starého boxu a založí prefix + VLAN 16 +
gateway v NB; `prefix_reconcile` lan_subnet záměrně neumí — 11. 9. opraveno v kódu i docs, původní
odkaz na reconcile byl špatný). Dál PSK tunelu `Strecno`, rozhodnutí o guest SSID, bridged profil
pro FAP-U231F, a první `precheck sk-depot-1` (read-only).
