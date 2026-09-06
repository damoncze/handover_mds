# Wi-Fi sonda: retransmise čteček měřené z proxy/FortiGate — handover

Stav k 6. 9. 2026, ~02:00. Pokračování na druhém PC: pustit recon (sekce 4),
výstup dát Claudovi, z něj se navrhne sonda + Zabbix objekty.

## 1. Zadání

Měřit zdraví Wi-Fi pro čtečky (Zebra) z **retransmisí**, ne jen z channel
utilizace, kterou hlásí AP. Motivace: vadný firmware update na Ruckus AP se
projevil retransmisemi; utilizace nic neukázala.

**Klíčové upřesnění:** NEměřit z AP. Flotila je heterogenní — někde FortiAP,
někde Ruckus, někde privátní campus 5G. Měřit ze **zabbix proxy** v depu,
ideálně tahat z **FortiGate** (všechen provoz čteček jím teče).

## 2. Co už je zjištěno (nezjišťovat znovu)

- Depotní AP (`ap-N.<depot>.packeta`) mají v Zabbixu **jen ICMP Ping**
  (+ prázdnou Generic agent šablonu), žádný SNMP interface.
- Šablona `SNMP_Ruckus_APs` (id 13469) umí jen CPU/paměť/availability a na
  depotní AP není nalinkovaná. Channel utilizace žije jen v Ruckus GUI.
- Žádný Unleashed/ZD/SmartZone controller host v Zabbixu neexistuje.
- SNMP na FortiGate PROKAZATELNĚ funguje — celý works health-check
  (fgVWLHealthCheckLink* itemy) jede přes SNMP LLD na PAC-*-FW.

## 3. Dohodnutá architektura (tři vrstvy)

**Poctivé omezení předem:** TCP retransmise tranzitního provozu FortiGate
neexportuje — nemá je session tabulka, SNMP ani NetFlow. Proto vrstvy:

1. **Jednotné jádro (všechny technologie):** external script na depotní
   proxy si z FGT vezme seznam živých klientů (DHCP leases / ARP / device
   inventory) a aktivně je měří z LAN — **loss a RTT p95** na vzorek čteček
   za minutu. Drátová noha je nulová, takže naměřené = bezdrátová část.
   Vadné AP = loss/jitter ke všem jeho klientům naráz.
   POZOR na power-save: čtečka legitimně odpovídá se zpožděním stovek ms
   (buffering na AP do DTIM). Signál je **sustained loss a p95**, ne průměr;
   prahy se kalibrují až z burn-in dat, ne odhadem.
2. **FortiAP depa — skutečné 802.11 retransmise z FGT:** FGT je tam wireless
   controller → REST `monitor/wifi/client` (per-klient signal/SNR/retry)
   nebo SNMP větev `fgWc` (1.3.6.1.4.1.12356.101.14).
3. **Pilot na pravé TCP retransmise (1 depo):** SPAN z FortiSwitche na volný
   port proxy + tshark `tcp.analysis.retransmission` per klient — validace,
   že vrstva 1 koreluje s realitou. Pak rozhodnout, jestli SPAN plošně.

Zabbix strana stejný vzor jako AFD sonda (osvědčený): external script → JSON
master item → dependent itemy + LLD, per-depo rollup „nejhorší AP / % klientů
s problémem", alerting až po burn-in.

## 4. DALŠÍ KROK: recon proti FortiGate

Spustit z depotní zabbix proxy, ideálně na DVOU depech: jedno s FortiAP,
jedno s Ruckusem. Skript níže uložit jako `wifi-recon-fgt.sh`:

```bash
./wifi-recon-fgt.sh -H <ip-FGT> -c <snmp-community> -t <api-token> -s <subnet-ctecek>
```

- `-c` = stejná community, kterou Zabbix používá na PAC-*-FW
- `-t` = read-only REST token; vyrobí se přes FMG jako u fortinet-mcp
  (`system api-user` + read-only accprofile). Bez něj běží jen SNMP část.
- `-s` = subnet čteček (fping vzorek → power-save chování naostro)

Výstup poslat Claudovi — z něj se navrhnou konkrétní itemy a napíše sonda.

### wifi-recon-fgt.sh

```bash
#!/bin/bash
# wifi-recon-fgt.sh - inventura: co o klientech (cteckach) umi rict FortiGate
# depa, mereno ze zabbix proxy. READ-ONLY.
set -u
HOST=""; COMMUNITY=""; TOKEN=""; SUBNET=""
while getopts "H:c:t:s:" o; do
    case "$o" in
        H) HOST="$OPTARG";; c) COMMUNITY="$OPTARG";;
        t) TOKEN="$OPTARG";; s) SUBNET="$OPTARG";;
        *) exit 2;;
    esac
done
[ -z "$HOST" ] && { echo "usage: $0 -H <fgt-ip> -c <community> [-t token] [-s subnet]" >&2; exit 2; }

section() { echo; echo "===== $* ====="; }

if [ -n "$COMMUNITY" ]; then
    section "A1. SNMP: identifikace"
    snmpget -v2c -c "$COMMUNITY" -t 3 -r 1 "$HOST" 1.3.6.1.2.1.1.1.0 2>&1

    section "A2. SNMP: wireless-controller vetev fgWc (1.3.6.1.4.1.12356.101.14)"
    snmpbulkwalk -v2c -c "$COMMUNITY" -t 5 -r 1 "$HOST" 1.3.6.1.4.1.12356.101.14 2>&1 \
        | awk -F'101.14.' 'NF>1 {split($2,a,"."); k=a[1]"."a[2]"."a[3]; c[k]++}
                           END {for (x in c) printf "  ...101.14.%s : %d radku\n", x, c[x] | "sort"}'
    echo "  (0 radku vsude = depo bez FortiAP, nebo vetev vypnuta)"

    section "A3. SNMP: vzorek fgWc (prvnich 60 radku)"
    snmpbulkwalk -v2c -c "$COMMUNITY" -t 5 -r 1 "$HOST" 1.3.6.1.4.1.12356.101.14 2>&1 | head -60

    section "A4. SNMP: ARP tabulka jako zdroj seznamu klientu (pocet radku)"
    snmpbulkwalk -v2c -c "$COMMUNITY" -t 5 -r 1 "$HOST" 1.3.6.1.2.1.4.35.1.4 2>&1 | wc -l
else
    section "A. SNMP: PRESKOCENO (chybi -c)"
fi

if [ -n "$TOKEN" ]; then
    section "B1. REST: monitor/wifi/client (FortiAP: per-klient vc. retry/signal)"
    curl -sk -m 10 -H "Authorization: Bearer $TOKEN" \
        "https://$HOST/api/v2/monitor/wifi/client?count=3" | head -c 4000; echo
    section "B2. REST: monitor/system/dhcp (leases = zivi klienti, vsechny technologie)"
    curl -sk -m 10 -H "Authorization: Bearer $TOKEN" \
        "https://$HOST/api/v2/monitor/system/dhcp?count=5" | head -c 3000; echo
    section "B3. REST: monitor/user/device/query (device inventory)"
    curl -sk -m 10 -H "Authorization: Bearer $TOKEN" \
        "https://$HOST/api/v2/monitor/user/device/query?count=5" | head -c 3000; echo
else
    section "B. REST API: PRESKOCENO (chybi -t)"
fi

if [ -n "$SUBNET" ]; then
    section "C. Ping vzorek ctecek ($SUBNET) - loss a RTT vc. power-save efektu"
    command -v fping >/dev/null 2>&1 || { echo "  fping neni - sudo apt install fping"; exit 0; }
    fping -g "$SUBNET" -C 5 -q -p 100 2>&1 | awk '
        {alive=0; for(i=3;i<=NF;i++) if($i!="-") alive++
         if (alive>0) {printf "  %s odpovedi %d/5:", $1, alive; for(i=3;i<=NF;i++) printf " %s",$i; print ""} }' | head -40
else
    section "C. Ping vzorek: PRESKOCENO (chybi -s)"
fi

section "HOTOVO - vystup poslat zpet"
```

### Doplněk pro Ruckus depa (vrstva navíc, není jádro): wifi-recon.sh

Per-klient retransmise na Ruckus Unleashed neumí SNMP (jen per-radio) —
umí je admin web API (`_cmdstat.jsp`). Tenhle skript zmapuje obojí.
Spouštět z proxy proti Unleashed masterovi:
`./wifi-recon.sh -H <ip-mastera> -c <community> -u admin -p '<heslo>'`

```bash
#!/bin/bash
# wifi-recon.sh - inventura: co leze z Ruckus AP/mastera. READ-ONLY.
set -u
HOST=""; COMMUNITY="public"; USER=""; PASS=""
while getopts "H:c:u:p:" o; do
    case "$o" in
        H) HOST="$OPTARG";; c) COMMUNITY="$OPTARG";;
        u) USER="$OPTARG";; p) PASS="$OPTARG";;
        *) exit 2;;
    esac
done
[ -z "$HOST" ] && { echo "usage: $0 -H <ip> [-c community] [-u admin -p pass]" >&2; exit 2; }
section() { echo; echo "===== $* ====="; }

section "1. SNMP: identifikace (sysDescr, sysObjectID, sysName)"
for oid in 1.3.6.1.2.1.1.1.0 1.3.6.1.2.1.1.2.0 1.3.6.1.2.1.1.5.0; do
    snmpget -v2c -c "$COMMUNITY" -t 3 -r 1 "$HOST" "$oid" 2>&1
done

section "2. SNMP: ktere vetve Ruckus MIB (25053) zarizeni plni"
snmpbulkwalk -v2c -c "$COMMUNITY" -t 5 -r 1 -Cr50 "$HOST" 1.3.6.1.4.1.25053 2>&1 \
    | awk -F'25053.' 'NF>1 {split($2,a,"."); key=a[1]"."a[2]"."a[3]"."a[4]; cnt[key]++}
                      END {for (k in cnt) printf "  25053.%s : %d radku\n", k, cnt[k] | "sort"}'

section "3. SNMP: vzorek prvnich 120 radku"
snmpbulkwalk -v2c -c "$COMMUNITY" -t 5 -r 1 "$HOST" 1.3.6.1.4.1.25053 2>&1 | head -120

section "4. SNMP: kandidatni tabulky (pocty radku)"
for sub in 1.3.6.1.4.1.25053.1.2.2.1 1.3.6.1.4.1.25053.1.2.2.4 1.3.6.1.4.1.25053.1.15; do
    n=$(snmpbulkwalk -v2c -c "$COMMUNITY" -t 5 -r 1 "$HOST" "$sub" 2>/dev/null | wc -l)
    echo "  $sub : $n radku"
done

if [ -n "$USER" ] && [ -n "$PASS" ]; then
    section "5. Unleashed admin API: login + per-klient statistiky"
    CJ=$(mktemp)
    curl -sk -c "$CJ" "https://$HOST/admin/login.jsp" -o /dev/null
    curl -sk -b "$CJ" -c "$CJ" -d "username=$USER&password=$PASS&ok=Log+In" \
         "https://$HOST/admin/login.jsp" -o /dev/null -w "  login HTTP %{http_code}\n"
    curl -sk -b "$CJ" -H "Content-Type: text/xml" \
         --data "<ajax-request action='getstat' comp='stamgr' updater='blabla'><client LEVEL='1'/></ajax-request>" \
         "https://$HOST/admin/_cmdstat.jsp" | head -c 4000; echo
    curl -sk -b "$CJ" -H "Content-Type: text/xml" \
         --data "<ajax-request action='getstat' comp='system'><sysinfo/></ajax-request>" \
         "https://$HOST/admin/_cmdstat.jsp" | head -c 2000; echo
    rm -f "$CJ"
else
    section "5. Unleashed admin API: PRESKOCENO (chybi -u/-p)"
fi

section "HOTOVO - vystup poslat zpet"
```

## 5. Ranní kontroly z vedlejšího projektu (AFD/works), ať se neztratí

1. **POPy po prohození DNS** (primary 8.8.8.8 + secondary 9.9.9.10 cleartext):
   Quad9 jako primár slil 32/35 dep na FRA (neposílá ECS → Azure steering
   nevidí polohu depa). Po prohození čekáme návrat rozptylu FRA/BER/CPH/WAW.
   Kontrola: Works dashboard, resolver tabulka; nebo Claude přes Zabbix MCP.
2. **works HC na FortiGatech:** v Zabbixu zatím jen PRUM + Zlín — buď
   nedoběhla SNMP discovery (~1× za hodinu), nebo install neprošel (FMG
   Install log). Po discovery vyjmenovat a VYPNOUT auto-vzniklé triggery
   „Health Check State Failed to works on member N" (rodí se zapnuté,
   action 3 by pageovala).
3. **pl-cieszyn locale bug:** sonda tam vyrábí `"loss_pct":0,0` (pl_PL
   locale) → nevalidní JSON, depo vypadává z fleet agregace. Fix hotový:
   commit d7d3d02, větev `fix/afd-probe-locale`, worktree
   `~/git/infra/lin_zabbix_proxy-locale` (roles repo lin_zabbix_proxy).
   Zbývá: push + MR + merge + redeploy (`tools/afd-rollout.sh -l "$SKIP" --apply`).
4. **gw-failed:** cz-depot-03 a cz-mikulov — FGT neodpovídá proxyně na DNS
   z LAN (`config system dns-server` na LAN interface). Opravit před
   zapnutím triggeru „AFD: DNS forwarder na FortiGate neodpovida" (109250).
5. **Zapínání AFD triggerů po burn-in:** všech 10 na šabloně 10897 + 3 fleet
   jsou vypnuté; pořadí zapínání: depot verdict (108974, 109202) → fleet
   (109199/109200/109201) → „probe not running" (108782) → FGT HC loss.
   Před každým zapnutím ověřit přes API, že podmínka zrovna neplatí.
6. **Shuttle:** větev `feat/afd-fg-healthcheck` (commit e9e3894, worktree
   `~/github/personal/shuttle-fg-hc`) přidává FG HC graf+tabulku na Works
   stránku — push + PR + dashboard push až po HC rolloutu na flotilu.
