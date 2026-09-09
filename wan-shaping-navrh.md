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
