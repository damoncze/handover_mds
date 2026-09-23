# Ruckus One MCP + přesun credentials do Azure Key Vaultu — handover

Stav k 23. 9. 2026, večer. Postaven a nasazen nový MCP server nad RUCKUS One,
credentials obou MCP stacků se stěhují z `.env.local` do Azure Key Vaultu.
Zápisy proběhly do: GitLab (nové repo + commity), Azure Key Vault (secrets).
Na síťových prvcích se **nezapsalo nic** — Ruckus API klíč je Read Only
a ověřeně odmítá zápis (HTTP 403).

---

## 0. Souhrn

1. **`ruckus-mcp` je hotový a běží.** Nové repo
   `infras/networkers/ruckus_one_mcp_server`, 5 commitů, poslední `4acdace`.
   Server na `127.0.0.1:8002`, read-only, 10 nástrojů, 45 testů, ruff čistý.
   Credentials bere **výhradně z Key Vaultu**, `.env.local` byl zrušen úplně.
2. **Key Vault `kv-fortimanager-prod` je tvůj** a je v něm 22 secrets. Doplněny
   `ruckusone-*` (4) a `fortianalyzer-mcp-*` (2). Přístup máš jen ty, model
   jsou access policies, síť je `Deny` s jedinou IP.
3. **`fortinet-mcp` napojený NENÍ a je to blokované** — sdílený read-only účet
   `mcp-readonly` se na FortiManager nepřihlásí. Detail v sekci 3.
4. **Plán přechodu vaultu na RBAC** je hotový v
   `claude-notes/azure-keyvault-rbac-plan.md`. Čeká na rozhodnutí o skupině.
5. **Wi-Fi Štěrboholy**: 2279 skutečných přechodů mezi AP za 24 h u 112 klientů,
   dvě AP dlouhodobě mimo. 802.11r je vypnuté záměrně a roaming neřeší; páky
   jsou jinde. Detail v sekci 5.
6. **Chyba, kterou je dobré znát**: credentials byly ~15 minut v cizím vaultu
   (ITOps). Smazáno a purgnuto, ale zvaž rotaci. Sekce 7.

---

## 1. ruckus-mcp

**Repo:** https://gitlab.zasilkovna.cz/infras/networkers/ruckus_one_mcp_server
**Lokálně:** `~/github/personal/ruckus-mcp`, větev `master`, vše pushnuto.

### Rozjetí na druhém stroji

```bash
git clone git@gitlab.zasilkovna.cz:infras/networkers/ruckus_one_mcp_server.git
cd ruckus_one_mcp_server
az login
./scripts/deploy.sh
./scripts/install-skill.sh
```

Nic se nepřenáší, žádný soubor s heslem neexistuje. **Jediná past: IP.** Vault
má `defaultAction: Deny` a povolenou jedinou adresu `193.179.216.17/32`. Když
bude notebook v jiné síti, `deploy.sh` spadne na čtení secretů. Dočasně:

```bash
az keyvault network-rule add --name kv-fortimanager-prod \
  --subscription a36a3f56-3c48-4a5f-b050-2a883e24859c --ip-address $(curl -s ifconfig.me)
```

Pozor — vault je pod Terraformem, takže tohle příští `terraform apply` smaže.
Trvale patří do `examples/fortimanager/main.tf`.

Pro korelaci s FortiAnalyzerem je potřeba registrace do user scope, jinak je
server vidět jen v tom adresáři:

```bash
claude mcp add --scope user --transport http ruckusone \
  http://127.0.0.1:8002/mcp --header "Authorization: Bearer $(cat secrets/mcp-bearer)"
```

### Co server umí

10 nástrojů: `get_tenant_info`, `list_venues`, `list_aps`, `list_networks`,
`list_clients`, `get_client_history`, `query_events`, `query_incidents`,
`correlate_client`, `r1_request`. Skill `ruckusone-mcp` je v `.claude/skills/`
a popisuje postupy podle typu stížnosti.

Read-only drží dvě vrstvy: gate v `readonly.py` (POST jen na `/query`
a `/reports/`) a Read Only scope API klíče, který `verify-access.sh` před
každým nasazením měří sondou `DELETE /venues/00000000-...` a čeká 403.

### Co je změřené proti živému API (a proč na tom záleží)

Špatný tvar dotazu na RUCKUS One **nevrátí chybu, ale 200 a prázdno**. Tichá
nula se snadno přečte jako „nic se nedělo". Proto je vše zadrátované
v `_query_body` a hlídané v `tests/test_query_shape.py`:

- `fields`, `page` i `pageSize` jsou **povinné**; neznámá pole se tiše zahodí
- hodnoty filtrů musí být **seznamy**, holý řetězec nenajde nic
- `sortOrder` malým písmem, časové okno `fromTime`/`toTime` **uvnitř `filters`**
- MAC s dvojtečkami (velikost písmen nerozhoduje)
- `totalCount` se u širokých dotazů zasekne na 10000 — je to strop, ne počet
- HTTP 500 `EVENT-1000x` je přechodná chyba serveru, klient opakuje

Autoritativní zdroj cest je konsolidované OpenAPI schéma tenantu
(Administration > Account Management > Settings > **Download API Schema**,
962 cest). Veřejná dokumentace se s ním rozchází — např.
`/reports/clients/sessionHistories` z oficiálního SDK v API vůbec neexistuje,
správně je `POST /historicalClients/query`.

### Co nefunguje

`query_incidents` — incidenty nejsou v API schématu tenantu, jsou v oddělené
RUCKUS AI API. Nástroj dostane 200 z `/reports/incidents/query`, ale vrací
prázdno a přidá `warning`. **Z nuly nedělat závěr, že incidenty nejsou.**

---

## 2. Azure Key Vault

| | |
|---|---|
| Vault | `kv-fortimanager-prod` |
| URI | `https://kv-fortimanager-prod.vault.azure.net/` |
| RG | `tmobile-hub-fortigate-prod-we` (subscription **T-Mobile-HUB**) |
| Tvoje role na RG | **Owner** |
| Model | access policies, přístup **jen ty** |
| Síť | `Deny` + jediná IP `193.179.216.17/32` |
| Purge protection | zapnutá |

Obsahuje 22 secrets: `fortimanager-*` (5), `fortianalyzer-*` (3),
`fortiauthenticator-*` (3), `ruckusone-*` (4), `netbox-*` (2), `soti-*` (4),
`zabbix-token`.

**Pozor na dvojče:** `kv-infrastructure-1wrxlh` v `rg-itops-o11y-k8s-prod-we`
patří ITOps (o11y/k8s). Máš na něm Secrets Officer, ale **není tvůj** —
viz sekce 7.

Shuttle i `netbox-fortimanager-automation` na `kv-fortimanager-prod` už míří
(role `azure_kv_secrets`, `config.py`), takže po zprovoznění účtů poběží
i bez lokálních `.env`.

---

## 3. fortinet-mcp — BLOKOVANÉ, tady se pokračuje

Repo `infras/networkers/fmg_faz_mcp_server` je **sdílené pro tým a výhradně
read-only**. Read-write FMG práce má vlastní repo pod `denis.dolicek`.

Zkoušel jsem přihlášení na `fortimanager.packeta.com`:

| Účet | Zdroj | Výsledek |
|---|---|---|
| `mcp-readonly` | vault `fortimanager-mcp-username/password` | **-22 Login fail** |
| `mcp-dolicek` | lokální `.env.local` | 0 OK |
| `packeta-terraform` | vault `fortimanager-username/password` | 0 OK |

Na FortiAnalyzeru `mcp-readonly` z vaultu **funguje** (0 OK).

**Proč nešlo pokračovat:** obě fungující varianty jsou pro sdílený read-only
nástroj špatně. `packeta-terraform` je read-write účet pro Terraform — obešel
by smysl read-only vrstvy a `verify-rbac.sh` by nasazení stejně zastavil,
protože čeká odmítnutí zápisu. `mcp-dolicek` je osobní účet, pod kterým by se
v auditu FMG objevilo čtení celého týmu.

**Další krok:** na FortiManageru zprovoznit `mcp-readonly` (ověřit že existuje,
má read-only profil, srovnat heslo s vaultem). Když se heslo změní:

```bash
az keyvault secret set --vault-name kv-fortimanager-prod \
  --subscription a36a3f56-3c48-4a5f-b050-2a883e24859c \
  --name fortimanager-mcp-password --value '<nove heslo>'
```

Ověření bez zásahu do repa:

```bash
cd ~/github/personal/fortinet-mcp
V=kv-fortimanager-prod; S=a36a3f56-3c48-4a5f-b050-2a883e24859c
kv() { az keyvault secret show --vault-name $V --subscription $S --name "$1" --query value -o tsv; }
curl -sS --cacert ca/trust-bundle.crt -H 'Content-Type: application/json' \
  -d "{\"id\":1,\"method\":\"exec\",\"params\":[{\"url\":\"sys/login/user\",\"data\":{\"user\":\"$(kv fortimanager-mcp-username)\",\"passwd\":\"$(kv fortimanager-mcp-password)\"}}]}" \
  https://$(kv fortimanager-hostname)/jsonrpc
```

Až vrátí `"code": 0`, napojení je stejný zásah jako u ruckus-mcp:
`kv_get` do `_lib.sh`, přepsat sekci credentials v `bootstrap.sh`,
`FMG_EXPECTED_ACCESS`/`FAZ_EXPECTED_ACCESS` z `.env.local` do proměnných
prostředí, smazat `.env.local.example`, srovnat dokumentaci. Pak deploy bez
jakéhokoli lokálního souboru včetně RBAC gate.

---

## 4. Přechod vaultu na RBAC

Plán je v `claude-notes/azure-keyvault-rbac-plan.md`. Podstatné:

- Vault **už pod Terraformem je** a state žije
  (`fortimngtfstateprod/fortimanagerstate/azure-keyvault-terraform.tfstate`).
  Co naklikáš v portálu, příští `terraform apply` vrátí. Proto Terraform.
- Modulu chybí `enable_rbac_authorization` a `role_assignments` — HCL je v plánu.
- **Tři applye, ne jeden.** Nejdřív role assignments (při vypnutém RBAC jsou
  neaktivní), pak přepnout flag, pak úklid access policies. Přepnutí okamžitě
  zneplatní access policies, takže obrácené pořadí = ztráta přístupu.
- **Hodnoty secrets do Terraformu nikdy** — skončily by ve state v plaintextu.
- Skupina: `Hotline-L2-Network` vlastníš a má jen tebe, můžeš do ní kolegy
  přidat sám. Jméno ale sedí na support tier, ne na přístup k secrets.

Nevyřešené: egress rozsahy kanceláře a VPN místo té jediné IP.

---

## 5. Wi-Fi Štěrboholy — nálezy

**Rozsah Ruckusu**: venue `PAC-CZ-PRUMYSLOVA` = sklad Štěrboholy, 40 AP,
čtečky Zebra. `PAC-CZ-BALABENKA` má v Ruckusu jen 2 AP a nula klientů.
**Kancelářská Wi-Fi na HQ jede na FortiAP s RADIUS EAP-TLS a Ruckus ji nevidí.**
Jména AP se mezi oběma světy překrývají — vždy ověřit venue.

**Dvě AP dlouhodobě mimo:** AP-44 bez kontaktu od 17. 9., **AP-42 od 28. 5.**

**Roaming:** za 24 h 2575 událostí „connected to another AP or RF band",
z toho **92 % skutečná změna AP** (2279) a jen 7 % změna pásma. Jeden klient
292 přechodů, tedy každých 5 minut. Podle SSID: Zebra 2316, PacketaVD 546.

**802.11r je vypnuté ZÁMĚRNĚ** — zkoušené bylo a způsobovalo auth failures na
čtečkách. Navíc obě Zebra SSID jedou na **WPA2Personal**, kde je roam jen
4-way handshake, takže FT by ušetřil skoro nic (má smysl na WPA2-Enterprise,
kde se přeskakuje EAP). **Roaming se přes 802.11r neřeší.** Zapsáno ve skillu,
ať to nikdo nenavrhuje znovu.

Páky, které se autentizace nedotýkají (stav na `Zebra`):

| Pole | Hodnota |
|---|---|
| `enableTransientClientManagement` | False |
| `agileMultibandEnabled` (802.11k/v) | False |
| `enableJoinRSSIThreshold` | False (práh -85 se nevynucuje) |
| `clientLoadBalancingEnable` | **True** |
| `enableBandBalancing` | True |

První hypotéza k ověření: **client load balancing při 25 klientech na 40 AP
nemá co vyvažovat** a nejspíš klienty přehazuje zbytečně. Vypnutí se nedotkne
autentizace. Měnit vždy jednu věc a změřit proti baseline výše.

---

## 6. Otevřené z dřívějška

**FortiGate shaping config** (WAN shaping, preview z FMG) obsahuje dvě věci,
které se shapingem nesouvisí a jsou nebezpečné:

- `config vpn ssl web portal` + `purge` — smaže **všechny** SSL-VPN portály
- `config webfilter urlfilter` `edit 1` → `edit 103` s `url "*"` a
  `action block` — default-deny na celý web filter

Než to půjde do produkce, projet celé.

**Vlastnictví VPN skupin**: 21 skupin FortiGate SSL VPN, 2469 členství,
**žádná nemá vlastníka**. Požadavek je sepsaný v
`Downloads/pozadavek-ownership-vpn-skupiny.md`. Chtěl jsi si to dát sám —
pozor, `Global Reader` na to nestačí, chce to PIM aktivaci role.
`HR & Legal` má 0 členů a `forticlient vpn test msi install` jednoho,
kandidáti na zrušení.

---

## 7. Chyba, kterou je dobré znát

Při hledání „tvého" vaultu jsem 10 secrets (včetně hesla mcp-readonly pro
FMG i FAZ a Ruckus client secretu) **nejdřív zapsal do `kv-infrastructure-1wrxlh`**,
což je vault ITOps o11y/k8s týmu. Ověřil jsem si, že na něj máš Secrets
Officer, a vzal to jako souhlas — což bylo špatné kritérium. Právo zapsat
neznamená, že vault patří tobě.

Smazáno a **purgnuto** do 15 minut, vault je prázdný jako předtím. Po tu dobu
je mohli číst: tomas.nocar, jan.kejr, vojtech.krejci (Secrets Officer)
a service principal `f5f62cdf-b622-4874-a96b-77ff457f25c0` (Secrets User).

**Zvážit rotaci** Ruckus client secretu — otáčí se nejsnáz (nový API key v UI,
starý zrušit) a je pak jen na dvou místech.
