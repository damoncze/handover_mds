# WAN saturace dep a návrh traffic shapingu (analýza 10. 9. 2026)

Zdroje: Zabbix (šablona WAN CIR, historie `wan.util.in[*]` 7.–10. 9.), FortiAnalyzer
(FortiView top-applications/top-sources, traffic log 9. 9. 12:00–12:30), FortiManager
(device DB PAC-CZ-RUD-FW, balíček Spoke_CZ-Rudna, SD-WAN konfigurace).

## 1. Co se děje

### 1.1 Kdo jel strop (trigger „at contracted ceiling inbound >95 % 15 min", 4.–9. 9.)

| Lokalita | Linka / CIR | Události | Kdy (CEST) | Nejdelší |
|---|---|---|---|---|
| PAC-CZ-RUD-FW (Rudná) | inet1 200M | 4× | 7. 9. 12:35, 8. 9. 11:53 + 12:29, 9. 9. 12:27 | 46 min (8. 9.) |
| PAC-CZ-ROUDNA-FW | inet1 100M | 2× | 8. 9. 14:42, 9. 9. 14:38 | 6 min |
| PAC-CZ-JIH-FW (Jihlava) | wan1 100M | 2× | 4. 9. 14:51, 8. 9. 14:51 | 7 min |
| PAC-CZ-UNL-FW (Ústí n. L.) | inet1 100M | 2× | 4. 9. 13:03, 7. 9. 13:51 | 1 min |
| PAC-CZ-NEH-FW (Nehvizdy) | port1 100M | 1× | 8. 9. 14:21 | 4 min |
| PAC-SK-TRN-FW (Trnava) | wan1 100M | 1× | 7. 9. 20:36 | 4 min |

Trigger chce 15 min nad 95 %, takže reálná saturace je vždy o 15 min delší než
uvedená délka a spousta špiček (99 % po 5–10 min) trigger vůbec nespustí.

### 1.2 Denní profil je stejný na všech CZ depech

Hodinová maxima / průměry inbound utilizace (Zabbix, 1min vzorky):

| Lokalita | 7. 9. | 8. 9. | 9. 9. |
|---|---|---|---|
| Rudná inet1 | 11–12 h: max 99, avg 54 → **12–13 h: avg 85** | 10–13 h: max 99, **12–13 h avg 97,5** | 9–13 h špičky 99, **12–13 h avg 89** |
| Jihlava | 13 h: max 99 | 14 h: max 99, avg 42 | 14 h: max 99 |
| Nehvizdy | 13 h: max 99 | 14 h: max 100, avg 57 | 13 h: max 99 |
| Roudná | 13 h: max 98, avg 51 | 14 h: max 99, avg 54 | 14 h: max 99, avg 54 |
| Ústí n. L. | 13 h: max 99 | 13–14 h: max 99 | 14 h: max 100 |
| Trnava | jiný vzor: 20 h, 23 h, 0 h | 23 h | 20 h |

Mimo okno 12–15 h jsou CZ depa na 10–50 %. **Není to postupný růst provozu, je to
denní vlna mezi polednem a 15. hodinou**, na Rudné o hodinu dřív než jinde. Trnava má
večerní/noční vzor a je potřeba ji rozebrat zvlášť.

### 1.3 Co tu vlnu tvoří: Works full download z Azure Blob

FortiView top-applications za 24 h (inbound):

| Lokalita | Microsoft.Azure.Blob.Storage | Microsoft.Azure | „users" (klientů) |
|---|---|---|---|
| Rudná | **321 GB** | 87 GB | 504 |
| Ústí n. L. | 43 GB | 12 GB | 89 |
| Roudná | 42 GB | 9 GB | 88 |
| Jihlava | 34 GB | 11 GB | 65 |
| Nehvizdy (7 dní) | 330 GB | 95 GB | 128 |

Na všech lokalitách je Blob Storage první a Microsoft.Azure druhá položka. Vše ostatní
(Google, Windows Update, Teams, SSL) je řádově níž.

Detail Rudná, špička 9. 9. 12:00–12:30 (traffic log, sessions > 5 MB):

- **campus_link_sub → Microsoft.Azure.Blob.Storage: 35 GB, cíl `packetaworksblob.blob.core.windows.net` (20.60.27.196), policy 157, odchod inet1.** To je 155 Mb/s průměr za půl hodiny = celá linka.
- Zdroje: ~140 různých IP z 172.20.8.0–172.20.11.x (adresní objekt `vl700_wlt_zebra_172.20.0.0/15`), tj. Zebra skenery na campus Wi-Fi, které přicházejí přes VLAN 101 `campus_link_sub`. Původní VAP `wlt_zebra` na FW je vypnutý.
- Jeden skener (172.20.11.38): v 12:18–12:25 **jedna session ~400 MB** z Blobu, pak celý zbytek dne 30–60 MB/h inkrementů. Tj. jednorázový **full download lokální DB Works**, ne trvalý tok. 140 skenerů najednou × 400 MB = ~56 GB, to na 200M lince trvá ~40 min přesně tak, jak vypadá průběh v Zabbixu.
- Vedlejší nálezy z téhož okna (WAN neplní, ale jsou vidět): `wlt_packetavd` 172.19.33.57 → vlastní veřejná IP FW tcp/8003 → port12 (kamery, policy 96), 32 GB za 30 min čistě lokální hairpin; `packeta_mgmt` 172.16.36.180/.181 (MAC 50:7c:6f) drží trvalé HTTPS session na 35.201.83.241 s uploadem ~4,7 GB za 33 h, malé.
- Jihlava: `tcp/8000` 83 GB je také hairpin lan → veřejná IP FW → port12 (kamery, policy 110), WAN nezatěžuje. Skutečný WAN žrout je opět Blob.

Pozn. k číslům z traffic logu: FortiGate loguje dlouhé session průběžně s kumulativními
bajty, takže součty přes řádky dlouhé session nadhodnocují. Poměry a identifikace
zdroje to nemění, FortiView agregace jsou správné.

### 1.4 Proč to bolí celou síť a monitoring

- **Saturace je inbound.** ISP shaper zahazuje na vstupu do depa bez ohledu na to, co to je. Overlay IPsec k hubům (hub1_inet1, hub2_inet1) jede po stejné inet1, takže se zahazuje i ESP: odpovědi Zabbix serveru proxy, SNMP polling FortiGate přes loopback_snmp, DNS, Works API.
- Na FortiGate **není žádný shaping**: v balíčku Spoke_CZ-Rudna má shaper jen policy 173 (per-ip `zasis_secure`), na inet1/inet2 není `inbandwidth`/`outbandwidth`, v traffic logu je `shaperrcvdname` prázdné. FW tedy nemá čím prioritizovat, bottleneck je u operátora.
- **inet2 (100M) je prázdná.** SD-WAN pravidlo `internet` (mode sla, členové inet1+inet2, healthcheck `underlay`) posílá všechno na inet1, dokud je v SLA. Inet2 měla 7.–9. 9. průměr pod 10 %. Rudná má 300M kontrahované kapacity a využívá 200.

## 2. Návrh

Tři vrstvy, v tomto pořadí. První dvě jsou síťové a jdou nasadit přes FMG,
třetí je na Works týmu.

### 2.1 Bottleneck přesunout na FortiGate (interface bandwidth + shaping profile)

Cíl: FortiGate zahazuje/řadí dřív než ISP, a to podle tříd. Na každém WAN členu
nastavit `inbandwidth`/`outbandwidth` na ~95 % CIR a připojit shaping profile.

Třídy (traffic-class):

| class-id | Název | Co tam patří | Garantováno | Max | Priorita |
|---|---|---|---|---|---|
| 2 | `critical` | overlay IPsec (ESP/IKE k hubům), monitoring VLAN, DNS, NTP, Works API `works.packeta.com` (Azure Front Door 150.171.109.0/24) | 20 % | 100 % | top |
| 3 | `business` | ostatní provoz skenerů (`vl700_wlt_zebra`, `wlt_zebra-ts`), M365/Teams, kancelářská LAN | 30 % | 100 % | high |
| 4 | `default` | vše nezařazené | 20 % | 100 % | medium |
| 5 | `bulk` | **Works Blob + CDN** (`packetaworksblob.blob.core.windows.net`, `cdn-packeta-works.azureedge.net`, app Microsoft.Azure.Blob.Storage), Windows Update, Delivery Optimization tcp/7680, Spotify/QUIC video | 10 % | **60 %** | low |

Bulk s maximem 60 % znamená, že full download 140 skenerů poběží místo 40 min zhruba
hodinu, ale zbylých 40 % linky zůstane vždy pro všechno ostatní. Pokud se ukáže, že
hodina je moc, dá se max zvednout, důležitá je garance pro třídy 2 a 3.

Návrh CLI pro Rudnou (pilot), FortiOS 7.x. Jako FMG CLI template s proměnnými
`$(inet1_cir_kbps)` / `$(inet2_cir_kbps)` to jde rozjet na všechny spoky:

```
config firewall address
    edit "works_blob_fqdn"
        set type fqdn
        set fqdn "packetaworksblob.blob.core.windows.net"
    next
    edit "works_cdn_fqdn"
        set type fqdn
        set fqdn "cdn-packeta-works.azureedge.net"
    next
    edit "works_api_fqdn"
        set type fqdn
        set fqdn "works.packeta.com"
    next
    edit "afd_works_150.171.109.0/24"
        set subnet 150.171.109.0 255.255.255.0
    next
end
config firewall addrgrp
    edit "grp_works_bulk"
        set member "works_blob_fqdn" "works_cdn_fqdn"
    next
    edit "grp_works_api"
        set member "works_api_fqdn" "afd_works_150.171.109.0/24"
    next
end

config firewall traffic-class
    edit 2
        set class-name "critical"
    next
    edit 3
        set class-name "business"
    next
    edit 4
        set class-name "default"
    next
    edit 5
        set class-name "bulk"
    next
end

config firewall shaping-profile
    edit "wan_classes"
        set type policing
        set default-class-id 4
        config shaping-entries
            edit 1
                set class-id 2
                set priority top
                set guaranteed-bandwidth-percentage 20
                set maximum-bandwidth-percentage 100
            next
            edit 2
                set class-id 3
                set priority high
                set guaranteed-bandwidth-percentage 30
                set maximum-bandwidth-percentage 100
            next
            edit 3
                set class-id 4
                set priority medium
                set guaranteed-bandwidth-percentage 20
                set maximum-bandwidth-percentage 100
            next
            edit 4
                set class-id 5
                set priority low
                set guaranteed-bandwidth-percentage 10
                set maximum-bandwidth-percentage 60
            next
        end
    next
end

config system interface
    edit "inet1"
        set inbandwidth 190000
        set outbandwidth 190000
        set ingress-shaping-profile "wan_classes"
        set egress-shaping-profile "wan_classes"
    next
    edit "inet2"
        set inbandwidth 95000
        set outbandwidth 95000
        set ingress-shaping-profile "wan_classes"
        set egress-shaping-profile "wan_classes"
    next
end

config firewall shaping-policy
    edit 1
        set name "shape-critical-works-api"
        set srcintf "any"
        set dstintf "inet1" "inet2"
        set srcaddr "all"
        set dstaddr "grp_works_api"
        set service "HTTPS" "HTTP"
        set class-id 2
    next
    edit 2
        set name "shape-critical-infra"
        set srcintf "monitoring" "loopback_snmp" "apmgmt" "srvmgmt"
        set dstintf "inet1" "inet2"
        set srcaddr "all"
        set dstaddr "all"
        set service "ALL"
        set class-id 2
    next
    edit 3
        set name "shape-bulk-works-blob"
        set srcintf "any"
        set dstintf "inet1" "inet2"
        set srcaddr "all"
        set dstaddr "grp_works_bulk"
        set service "ALL"
        set class-id 5
    next
    edit 4
        set name "shape-bulk-updates"
        set srcintf "any"
        set dstintf "inet1" "inet2"
        set srcaddr "all"
        set dstaddr "all"
        set service "ALL"
        set application 16009 40169 17405
        set class-id 5
    next
    edit 5
        set name "shape-business-scanners"
        set srcintf "campus_link_sub" "wlt_zebra-ts" "wlt_zebra"
        set dstintf "inet1" "inet2"
        set srcaddr "all"
        set dstaddr "all"
        set service "ALL"
        set class-id 3
    next
end
```

Poznámky k CLI:

- Pořadí shaping policy je „první shoda vyhrává", proto Works API (třída 2) před
  Blobem (třída 5) a Blob před obecným pravidlem skenerů (třída 3).
- `set application` je app-ctrl ID (16009 Windows Update, 40169 QUIC, 17405 Spotify);
  vyžaduje app-ctrl profil na firewall policy, který tam podle FAZ klasifikace už je.
  Blob by šel i přes ID 57032, ale FQDN je spolehlivější (chytí i první pakety).
- FQDN objekty se opírají o DNS cache FortiGate. Depa už používají FGT dns-database
  (`pl-blob-core-windows-net`, `works-public`), takže resolvery klientů jdou přes FW
  a FQDN match funguje. Ověřit na pilotu: `diagnose firewall fqdn list`.
- ESP overlay k hubům se na ingress inet1 klasifikuje před dešifrováním, takže spadne do
  `default-class-id`. Proto default třída drží garanci 20 % a bulk má tvrdý strop, ne
  jen nižší prioritu. Pokud by to nestačilo, přidat shaping policy `srcaddr = veřejné IP
  hubů, service ESP/IKE, class 2` (hub Balabenka 193.179.216.17, hub Azure
  108.143.161.177 dle FAZ inventáře, ověřit v FMG).
- Hodnoty `inbandwidth`/`outbandwidth` jsou v kb/s a jsou to hodnoty pro pilot Rudná
  (200M/100M). Pro 100M/50M depa 95000/47500.

### 2.2 SD-WAN: bulk poslat na inet2

Rudná má inet2 100M prázdnou. Před pravidlo `internet` (id 2) přidat pravidlo, které
Works bulk přednostně vezme na inet2 a na inet1 spadne až při výpadku:

```
config system sdwan
    config service
        edit 10
            set name "works_bulk_inet2"
            set mode manual
            set dst "grp_works_bulk"
            set priority-members 4 3
        next
    end
end
```

(`4` = inet2, `3` = inet1 podle seq-num v device DB.) V FMG se to dělá v SD-WAN
template pro spoky, pravidlo musí být nad `internet`. Kombinace se shapingem: bulk na
inet2 dostane max 60 % z 95M ≈ 57M, inet1 zůstane celá pro interaktivní provoz.
Alternativa pro depa, kde je druhá linka slabá (50M LTE): pravidlo nepřidávat a nechat
jen shaping.

Pokud by se ukázalo, že full download přes 57M trvá neúnosně dlouho, je lepší cesta
`load-balance` na pravidle `works_bulk` (rozložit na obě linky) než zvedat strop bulku.

### 2.3 Works: řídit full download na straně aplikace

Síť to umí zkrotit, ale příčina je aplikační. Ptát se Works týmu:

1. Proč full download (~400 MB/zařízení) probíhá **denně** mezi 12. a 15. hodinou
   na všech depech? Full download by měl proběhnout jen při prvním přihlášení nebo po
   ztrátě lokální DB, jinak inkrementy. Souvisí s otevřeným nálezem
   `has_data = false` z 31. 8. (Ołtarzew, cizí `depot:20` v `blob_urls` shodí sync
   a full download se opakuje). Pokud se full download restartuje i v CZ, je to
   násobitel objemu.
2. Rozložit start full downloadu v čase (jitter desítky minut) nebo omezit paralelní
   stahování na zařízení.
3. Zmenšit blob (delta/komprese) nebo stahovat jen data vlastního depa.

Rudná: 504 klientů × ~640 MB/den Blob = 321 GB/den. To je 10× víc než jakékoli jiné
depo a i po shapingu bude linka mezi 12. a 13. hodinou plná, jen řízeně.

### 2.4 Monitoring po nasazení

- Trigger „at contracted ceiling" sleduje 95 % CIR. Po nastavení `inbandwidth` 95 %
  bude linka u stropu FortiGate, nikoli ISP, a trigger přestane pálit. Aby se
  nestal slepým, doplnit do šablony WAN CIR item na drop counter shaperu
  (REST `api/v2/monitor/firewall/shaper` nebo SNMP `fgIntfEntry` per class) a
  informační trigger „bulk class na stropu", bez PagerDuty.
- Akce 3 stále pageuje každý nový trigger bez filtru severity (viz paměť
  `zabbix-new-trigger-pages-pagerduty`), nový trigger nejdřív otestovat s tagem, který
  akce 3 ignoruje.

## 3. Postup nasazení

1. **Pilot Rudná** (největší objem, obě linky): CLI template z 2.1 + SD-WAN pravidlo z
   2.2, nasadit v odpoledním klidu (po 15 h), sledovat 10.–12. 9. v Zabbixu
   `wan.util.in[inet1]` a v FAZ `shaperdroprcvdbyte` / `shaperrcvdname`.
2. Po 2 dnech rozšířit na ROUDNA, JIH, UNL, NEH (100M/50M). U nich zvážit, jestli
   druhou linku (50M) zatěžovat bulkem, nebo jen shapovat.
3. Trnava zvlášť: vzor je večerní (20 h, 23 h), pravděpodobně jiná příčina, prověřit
   FAZ top-applications v okně 19–24 h.
4. Paralelně otevřít téma s Works týmem (2.3).

## 4. Co se v analýze nepodařilo ověřit

- FortiView `top-destinations` a `top-sources` na ostatních depech vytimeoutovaly
  (FAZ zvládne 1–2 paralelní dotazy), Roudná/JIH/UNL mám jen top-applications.
- Existující shaper `zasis_secure` (policy 173) jsem nečetl, FMG MCP nemá nástroj na
  `firewall shaper`; z FAZ logu je ale jasné, že na WAN provoz se žádný shaper
  neaplikuje.
- Trnava (PAC-SK-TRN-FW) není rozebraná.

---

## 5. Pokračování 10. 9. (druhý stroj): ověření přes FAZ/FMG a úpravy návrhu

Zdroje: FAZ event log `subtype=sdwan` 3.–10. 9., FAZ traffic log a FortiView
(Rudná 7. 9. 00–01 h, Trnava 8. 9. 19 h – 9. 9. 01 h), FMG device DB PAC-CZ-RUD-FW
(`get_device_sdwan`, `get_device_interface_config`, `get_device_sdwan_monitor`),
FMG šablony (`list_sdwan_templates`, `list_cli_template_groups`), policy 157/173.

### 5.1 Ověřeno: „flapuje vše okolo" = SD-WAN pravidlo `internet` přeskakuje inet1↔inet2

Konfigurace health-checku `underlay` na Rudné (device DB, stejná šablona pro všechny spoky):

| Parametr | Hodnota | Důsledek |
|---|---|---|
| server | 193.179.246.243, 213.29.0.1 (ICMP) | probe replies přicházejí inbound po saturované lince |
| interval / failtime / recoverytime | 500 ms / 5 / 5 | out-of-SLA i návrat za **2,5 s** |
| SLA 1 | latency 250 ms, jitter 50 ms, **loss 5 %** | ISP shaper při saturaci zahazuje >5 % → out-of-SLA |
| update-static-route | enable | |
| pravidlo `internet` (id 2) | mode **sla**, členové 3 (inet1), 4 (inet2), **hold-down-time 0** | při out-of-SLA se celý internet přepne na inet2, za 2,5 s po uklidnění zpět |

Overlay health-checky `Hub1_Loopback` / `Hub2_Loopback` (172.16.32.1/.2 přes tunel) mají
latency 300 / loss 20 %, proto padají méně, ale také padají (Jihlava 58×, Nehvizdy 89× za 7 dní).

Počet událostí `Member status changed. Member out-of-sla` za 3.–10. 9. (FAZ event log,
`query_logs`, řádky spočítány lokálně):

| Depo | out-of-SLA celkem | z toho `underlay` | špičkové hodiny |
|---|---|---|---|
| PAC-CZ-RUD-FW | 176 | 173 | **00 h: 47**, 12 h: 36, 10 h: 14 |
| PAC-CZ-JIH-FW | 188 | 130 | 14 h: 38, 05 h: 27, 03 h: 24 |
| PAC-CZ-NEH-FW-1 (název ve FAZ) | 202 | 113 | **00 h: 60**, 13 h: 39, 14 h: 34 |
| PAC-CZ-UNL-FW | 93 | 89 | 12 h: 20, 01 h: 12 |
| PAC-CZ-ROUDNA-FW | 56 | 56 | 14 h: 9, 13 h: 8, 01 h: 8 |
| PAC-SK-TRN-FW | 84 | 78 | 06 h: 11, 01 h: 9 |

Každé přepnutí = změna SNAT IP pro všechny internetové session (Works API, M365, Zabbix
proxy → server přes overlay ne, ten jde jinou cestou, ale probe a ESP na inet1 trpí stejně).
Rudná 7. 9. 00–01 h měla 35 z 195 CDN session na inet2 a 156 na inet1, tj. přepínalo to
uprostřed stahování.

**Závěr:** shaping sám nestačí, když pravidlo `internet` reaguje na 2,5 s výpadek SLA
s nulovým hold-downem. Obojí je potřeba: FGT drží linku pod stropem ISP (žádný loss →
SLA drží) a SD-WAN přestane reagovat na sekundové špičky.

### 5.2 Nový nález: druhá vlna o půlnoci = Works CDN (update aplikace)

Rudná 7. 9. 00:00–01:00: **Microsoft.Azure 40 GB / hod**, 100 % `cdn-packeta-works.azureedge.net`
(Azure Front Door 150.171.109.99/.104/.105/.193/.194), policy 157 `campus_5g_sub-to-inet`,
**98 zdrojů × shodných 178 MB** (172.20.8–11.x campus Zebra + 192.168.100.x `wlt_zasilzam`).
To je noční distribuce balíčku aplikace, ne data. Vysvětluje půlnoční out-of-SLA špičky
(Rudná 47, Nehvizdy 60) a `Microsoft.Azure` na 2. místě v FortiView.

**Oprava návrhu z 2.1:** AFD prefix 150.171.109.0/24 NESMÍ být ve třídě `critical`.
Works API (`works.packeta.com`) i CDN (`cdn-packeta-works.azureedge.net`) jedou přes tentýž
AFD rozsah, takže klasifikace podle IP by pustila 40 GB/h updatů do nejvyšší priority.
Klasifikovat jen podle FQDN; `afd_works_150.171.109.0/24` z `grp_works_api` vyhodit.

### 5.3 Trnava rozebraná (8. 9. 19 h – 9. 9. 01 h, session > 50 MB, 600 řádků)

| Provoz | Objem (kumulativně logované session, nadhodnoceno) | Zdroje | Hodiny |
|---|---|---|---|
| **tcp/8443 wan1 → port12 (kamery), dst 195.146.137.122, policy 104** | 165 GB | 7 externích IP: 213.81.225.94, 34.250.127.182 (AWS), 185.110.144.194, **213.81.177.50 (= PAC-SK-STR-FW Strečno)**, **213.81.132.242 (= PAC-SK-BRA-TRIB-FW)**, 185.122.55.131, 178.143.191.182 | 23 h, 00 h |
| Microsoft.Azure.Blob.Storage (20.60.27.196 + `packetaworksblob`) z `wlt_zebra` | 56 GB | 24 + 14 skenerů | 23 h, 00 h |
| Works CDN z `wlt_zebra` | 3,5 GB | 18 | 00 h |
| SSL 184.104.206.14 z `wlt_packeta_zvo` | 5 GB | 1 | 23–00 h |

Trnava má tedy dvě příčiny: (a) Zebra full download běží v SK **v noci** (23–01 h), ne
v poledne; (b) **export kamerových záznamů přes veřejnou IP** ven na 7 adres, z toho dvě
jsou vlastní FortiGaty (Strečno, Bratislava Tribečská) – vnitrofiremní přenos jde přes
internet a NAT místo overlay, a jeden cíl je AWS Irsko (34.250.127.182, pravděpodobně
cloud NVR/VMS). To je upload a saturuje `wan1` odchozím směrem; Zabbix trigger hlídá
jen inbound, takže to vidíme pouze nepřímo.

Akce Trnava: (1) zjistit, kdo/co stahuje archiv kamer (policy 104, dst 195.146.137.122:8443/8001),
zda jde o plánovaný export a zda musí jet v 23–00 h; (2) Strečno a BRA-TRIB
mají jít přes overlay (BGP prefix kamerové sítě + policy), ne přes veřejnou IP;
(3) kamerový export do třídy `bulk` na egress `wan1`.

### 5.4 Co dalšího z FMG mění návrh

- **inet1/inet2 jsou VLAN 10/11 na agregátu `fortilink`** (device DB Rudná; stejný vzor
  ze šablon `Spoke_tunnel_interface`/`Spoke_cam_interface`). Interface shaping profile na
  VLAN sub-interface FortiOS 7.4 umí, ale běží v CPU, ne v NP (NP6xlite na 100F/200F,
  SoC na 120G). Při 200–300 Mb/s je to pro 200F/120G v pořádku, na pilotu sledovat
  `get system performance status` a `diagnose netlink intf-class list inet1`.
- `inbandwidth`/`outbandwidth`/`*-shaping-profile` na inet1, inet2 jsou v device DB
  prázdné – potvrzeno, že shaping dnes není nikde.
- Policy 157 `campus_5g_sub-to-inet`: srcintf `campus_link_sub`, dstintf zóna `underlay`,
  utm-status enable, žádný shaper. Policy 173 `any-to-zasis`: per-ip shaper `zasis_secure`
  na FQDN `zasilkovna.cz`, s WAN saturací nesouvisí. Aplikační jména ve FortiView jsou,
  takže app-ctrl profil na policy je – přes MCP ale `application-list` nečitelný, ověřit v GUI.
- Provisioning: SD-WAN šablony `Spoke-single` (skupina Spoke-single) a `Spoke-dual`
  (skupina `SDWAN_dual_prio_inet1` + HKR-BREZ + NEH) + `Spoke-dual-inet2-inet1`
  (`SDWAN_dual_prio_inet2`). CLI template group `Spoke_CLI_group` (18 šablon, proměnné
  `inet1_name`, `inet2_name`, `inet1_GN`…). Shaping = nová CLI šablona `Spoke_wan_shaping`
  přidaná do `Spoke_CLI_group` (+ `_fox+packeta`, `_cam_252`, `_CZ-PHA-PRUM`), SD-WAN
  úpravy do obou Spoke-* SD-WAN šablon. Obsah SD-WAN šablon přes MCP nejde číst
  (vrací jen hlavičku), prahy beru z device DB Rudné.
- Hub IP pro local-in klasifikaci: Balabenka 193.179.216.17, Azure 108.143.161.177
  (FMG `list_devices`). ADVPN shortcuty (hub1_inet1_0…3) mají jako peer další spoky
  (62.84.151.182, 62.77.90.217, 213.81.181.69) – ESP od nich pokrýt službou, ne IP.

### 5.5 Upravený návrh (nahrazuje část 2.1 a 2.2)

**A. Třídy a dva profily místo jednoho.** Primární linka má bulk strop 50 %, sekundární
(u dual dep prázdná) 90 %. Pořadí garancí zůstává.

```
config firewall shaping-profile
    edit "wan_primary"
        set type policing
        set default-class-id 4
        config shaping-entries
            edit 1
                set class-id 2
                set priority top
                set guaranteed-bandwidth-percentage 20
                set maximum-bandwidth-percentage 100
            next
            edit 2
                set class-id 3
                set priority high
                set guaranteed-bandwidth-percentage 30
                set maximum-bandwidth-percentage 100
            next
            edit 3
                set class-id 4
                set priority medium
                set guaranteed-bandwidth-percentage 20
                set maximum-bandwidth-percentage 100
            next
            edit 4
                set class-id 5
                set priority low
                set guaranteed-bandwidth-percentage 10
                set maximum-bandwidth-percentage 50
            next
        end
    next
    edit "wan_secondary"
        set type policing
        set default-class-id 4
        config shaping-entries
            edit 1
                set class-id 2
                set priority top
                set guaranteed-bandwidth-percentage 10
                set maximum-bandwidth-percentage 100
            next
            edit 2
                set class-id 3
                set priority high
                set guaranteed-bandwidth-percentage 10
                set maximum-bandwidth-percentage 100
            next
            edit 3
                set class-id 4
                set priority medium
                set guaranteed-bandwidth-percentage 10
                set maximum-bandwidth-percentage 100
            next
            edit 4
                set class-id 5
                set priority low
                set guaranteed-bandwidth-percentage 20
                set maximum-bandwidth-percentage 90
            next
        end
    next
end
config system interface
    edit "$(inet1_name)"
        set inbandwidth $(inet1_cir_kbps_95)
        set outbandwidth $(inet1_up_kbps_95)
        set ingress-shaping-profile "wan_primary"
        set egress-shaping-profile "wan_primary"
    next
    edit "$(inet2_name)"
        set inbandwidth $(inet2_cir_kbps_95)
        set outbandwidth $(inet2_up_kbps_95)
        set ingress-shaping-profile "wan_secondary"
        set egress-shaping-profile "wan_secondary"
    next
end
```

Ingress profil musí být `type policing` (queuing FortiOS na ingress nepodporuje).
Upload CIR (`*_up_kbps`) je u asymetrických linek jiný než download – doplnit jako
nové proměnné šablony, zdroj Zabbix WAN CIR makra / NetBox.

**B. Objekty: jen FQDN, žádný AFD prefix.**

```
config firewall address
    edit "works_blob_fqdn"
        set type fqdn
        set fqdn "packetaworksblob.blob.core.windows.net"
    next
    edit "works_cdn_fqdn"
        set type fqdn
        set fqdn "cdn-packeta-works.azureedge.net"
    next
    edit "works_api_fqdn"
        set type fqdn
        set fqdn "works.packeta.com"
    next
    edit "hub_balabenka_pub"
        set subnet 193.179.216.17 255.255.255.255
    next
    edit "hub_azure_pub"
        set subnet 108.143.161.177 255.255.255.255
    next
    edit "sdwan_hc_underlay_servers"
        set type iprange
        set start-ip 193.179.246.243
        set end-ip 193.179.246.243
    next
    edit "sdwan_hc_underlay_server2"
        set subnet 213.29.0.1 255.255.255.255
    next
end
config firewall addrgrp
    edit "grp_works_bulk"
        set member "works_blob_fqdn" "works_cdn_fqdn"
    next
    edit "grp_sdwan_hubs"
        set member "hub_balabenka_pub" "hub_azure_pub"
    next
    edit "grp_sdwan_hc"
        set member "sdwan_hc_underlay_servers" "sdwan_hc_underlay_server2"
    next
end
```

**C. Shaping policy: local-in pro ESP/IKE a SLA probe, pak forwarding.** FortiOS 7.4
umí `set traffic-type local-in|local-out` (Traffic shaping extensions 7.4.0). Tím se
ESP od hubů a ICMP odpovědi health-checku dostanou do třídy 2 místo default třídy.
Syntaxi ověřit na pilotu (`show firewall shaping-policy`), na 7.4.9 by měla být.

```
config firewall shaping-policy
    edit 1
        set name "li-critical-overlay-esp"
        set traffic-type local-in
        set srcintf "$(inet1_name)" "$(inet2_name)"
        set srcaddr "all"
        set dstaddr "all"
        set service "ESP" "IKE"
        set class-id 2
    next
    edit 2
        set name "li-critical-sdwan-probe"
        set traffic-type local-in
        set srcintf "$(inet1_name)" "$(inet2_name)"
        set srcaddr "grp_sdwan_hc" "grp_sdwan_hubs"
        set dstaddr "all"
        set service "PING" "ALL_ICMP"
        set class-id 2
    next
    edit 3
        set name "lo-critical-overlay-esp"
        set traffic-type local-out
        set dstintf "$(inet1_name)" "$(inet2_name)"
        set srcaddr "all"
        set dstaddr "all"
        set service "ESP" "IKE" "PING"
        set class-id 2
    next
    edit 10
        set name "shape-bulk-works-blob-cdn"
        set srcintf "any"
        set dstintf "$(inet1_name)" "$(inet2_name)"
        set srcaddr "all"
        set dstaddr "grp_works_bulk"
        set service "ALL"
        set class-id 5
    next
    edit 11
        set name "shape-critical-works-api"
        set srcintf "any"
        set dstintf "$(inet1_name)" "$(inet2_name)"
        set srcaddr "all"
        set dstaddr "works_api_fqdn"
        set service "HTTPS"
        set class-id 2
    next
    edit 12
        set name "shape-critical-infra"
        set srcintf "monitoring" "loopback_snmp" "apmgmt" "srvmgmt" "packeta_mgmt"
        set dstintf "$(inet1_name)" "$(inet2_name)"
        set srcaddr "all"
        set dstaddr "all"
        set service "ALL"
        set class-id 2
    next
    edit 13
        set name "shape-bulk-updates"
        set srcintf "any"
        set dstintf "$(inet1_name)" "$(inet2_name)"
        set srcaddr "all"
        set dstaddr "all"
        set service "ALL"
        set application 16009 40169 17405 17136
        set class-id 5
    next
    edit 14
        set name "shape-bulk-cam-export"
        set srcintf "$(inet1_name)" "$(inet2_name)"
        set dstintf "port12"
        set srcaddr "all"
        set dstaddr "all"
        set service "ALL"
        set class-id 5
    next
    edit 15
        set name "shape-business-scanners"
        set srcintf "campus_link_sub" "wlt_zebra-ts" "wlt_zebra"
        set dstintf "$(inet1_name)" "$(inet2_name)"
        set srcaddr "all"
        set dstaddr "all"
        set service "ALL"
        set class-id 3
    next
end
```

Pořadí: bulk Blob/CDN (10) před Works API (11), protože oba resolvují do AFD; FQDN
match to rozliší, IP by ne. Pravidlo 14 řeší Trnavu (kamerový export přes VIP): je to
reverse směr (odpovědi kamer odcházejí egress WAN), class-id se aplikuje na celou session.
17136 = HTTP.Segmented.Download (Trnava). Ostatní app-ID viz 2.1.

**D. SD-WAN: bulk přes obě linky s preferencí inet2 + hold-down na `internet`.**

```
config system sdwan
    config service
        edit 10
            set name "works_bulk"
            set mode sla
            set dst "grp_works_bulk"
            set priority-members 4 3
            set hold-down-time 120
            config sla
                edit "underlay"
                    set id 1
                next
            end
        next
        edit 2
            set hold-down-time 60
        next
    end
end
```

`works_bulk` musí být nad `internet` (id 2). Varianta `mode load-balance` přes 4 a 3
dá Rudné až ~185 Mb/s pro bulk (90 % ze 100M + 50 % ze 200M) a inet1 přesto nikdy
nepřijde o polovinu kapacity; začít s `sla` + preferencí inet2, load-balance až podle
doby full downloadu na pilotu. `hold-down-time 60` na `internet` zabrání přepnutí zpět
dřív než za minutu po návratu do SLA; první přepnutí (fail) zůstává rychlé, což je správně.
U single-WAN dep (`Spoke-single`) pravidlo 10 nepřidávat, jen hold-down.

Health-check `underlay` neměnit hned; až po shapingu vyhodnotit, jestli loss 5 % / 2,5 s
ještě padá. Pokud ano, zvednout `failtime`/`recoverytime` na 10 (5 s), ne prahy.

**E. Works (aplikační příčina), doplněno o CDN a cache.**

1. Noční update aplikace (178 MB × všechna zařízení depa ve stejné minutě po půlnoci) →
   rozložit start (jitter 0–120 min), nebo řídit přes MDM (SOTI) staged rollout po depech.
2. Denní full download DB (~400 MB) → totéž jako v 2.3; navíc: SK depa ho dělají v noci,
   CZ v poledne. Ptát se, čím je čas daný (time zone? konfigurace per depo?) – pokud jde
   nastavit, dát CZ depa také mimo provozní špičku.
3. **Blob je per depo, ne per zařízení** (viz `depot:20` v `blob_urls` z 31. 8.). 140 skenerů
   stahuje 140× stejný soubor. Lokální cache v depu (reverse proxy s cache, DNS override
   přes FGT dns-database, který depa už používají pro `pl-blob-core-windows-net`) sníží
   WAN objem na 1× za den. Rudná má lokální compute (VLAN `kubernet` 10.210.56.0/21).
   Je to největší páka: shaping problém zmírní, cache ho odstraní.

### 5.6 Monitoring doplněný o SD-WAN

- Zabbix: SNMP `fgVWLHealthCheckLinkTable` (FORTINET-FORTIGATE-MIB, 7.4) → item
  `sdwan.hc.state[underlay,inet1]` + trigger „SD-WAN member out-of-SLA > 3× za hodinu",
  bez PagerDuty (pozor na akci 3, viz 2.4). To je přímý KPI pro efekt shapingu:
  cíl je z 176/týden (Rudná) na jednotky.
- FAZ: event handler na `logid 0113022934`/`0113022923` s `out-of-sla` je alternativa,
  ale Zabbix už drží WAN CIR, tak to patří k němu.
- Po nasazení sledovat i `outbandwidth` drop counter na `wan1` Trnavy (upload).

### 5.7 Postup nasazení (upřesnění k části 3)

1. Pilot Rudná: CLI šablona `Spoke_wan_shaping` s bloky A–C (proměnné
   `inet1_cir_kbps_95=190000`, `inet1_up_kbps_95` dle upload CIR, `inet2_*=95000`),
   SD-WAN blok D do šablony `Spoke-dual`. Nasadit po 15 h, kdy je linka na 10–50 %.
   Ověřit: `diagnose firewall fqdn list` (Blob/CDN resolvují), `show firewall shaping-policy`
   (traffic-type přijat), `diagnose netlink intf-class list inet1` (pakety v třídách 2/5),
   CPU.
2. Metriky pilotu 11.–12. 9.: Zabbix `wan.util.in[inet1]` v 00–01 h a 12–13 h (očekávání:
   strop 95 %, ne 99–100 %), počet `out-of-sla` `underlay` za den (FAZ; před: 25–45/den),
   doba full downloadu jednoho skeneru (Works tým / traffic log session duration).
3. Rollout CZ dual (ROUDNA, JIH, UNL, NEH), pak single-WAN depa jen s A–C bez pravidla 10.
4. Trnava: nejdřív vyjasnit kamerový export (5.3), pak shaping.

### 5.8 Neověřeno / nedotaženo

- Syntaxe `traffic-type local-in` na 7.4.9 a chování ingress profilu na VLAN sub-interface
  (dokumentace Fortinet se z tohoto stroje nedala načíst, stránky jsou JS-only) → pilot.
- Obsah SD-WAN šablon `Spoke-dual`/`Spoke-single` (MCP vrací jen hlavičku). Prahy jsou z device
  DB Rudné; předpoklad, že šablona je shodná pro celou flotilu, ověřit v FMG GUI.
- Tabulka CIR pro celou flotilu (proměnné šablony): Zabbix MCP na tomto stroji není
  nakonfigurovaný, dodat z prvního stroje (`usermacro.get` WAN CIR makra) nebo z NetBoxu.
- Objemy kamerového exportu Trnava jsou z kumulativně logovaných session (nadhodnoceno);
  identita 7 externích IP ověřena jen pro 2 vlastní FGT (FMG `list_devices`).
- Filtr `appid==` v FAZ `query_logs` vrací prázdno (tichý fail) – aplikace identifikovat
  přes `app`/`hostname` v řádcích, ne přes appid.
