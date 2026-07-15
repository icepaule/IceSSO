# SSO-Dienst-Inventar (Authentik-Zentralisierung)

Stand: 2026-07-12. Edge = pfSense-HAProxy (`192.168.178.154:443`, ein SNI-Frontend `joplin_https`).
IdP = Authentik `2026.5.3` @ auth01 `10.10.0.70:9000` (`auth.mpauli.de`).
Split-Horizon-DNS (Phase 1 ✅): alle `*.mpauli.de` intern → `192.168.178.154` (unbound + AdGuard).

## Klassifizierungs-Schema
- **A — OIDC-nativ**: App kann OIDC selbst → echtes SSO, kein Doppel-Login. Bevorzugt.
- **B — Outpost-gated**: kein OIDC → Authentik-Proxy-Outpost davor (302→Login+2FA), dann Dienst-eigenes Login dahinter (ggf. Doppel-Login).
- **C — Guacamole**: Nicht-Web (RDP/SSH/VNC/Konsole).
- **D — Gast-Dashboard**: read-only, ohne Auth, an Authentik vorbei (`guest.mpauli.de`).
- **X — nicht interaktiv gate-bar**: API/Sync/Maschinen-Clients — dürfen NICHT hinter Outpost (bricht den Client). Nur App-OIDC ODER Netz-ACL/Token.
- **KEEP** = bewusst so lassen.

---

## STATUS: bereits erledigt
| Dienst | Host | Klasse | Mechanismus |
|---|---|---|---|
| Guacamole | auth01 :8080 | A | OIDC an Authentik ✅ |
| ITSO-KI (Open-WebUI) | KI02 10.10.0.210:8080 | A | OIDC-nativ ✅ |
| Jella-KI | 10.10.0.210:3001 | B | Proxy-Outpost ✅ |
| Authentik | 10.10.0.70:9000 | — | IdP ✅ |

## KRON-JUWELEN (zuerst, Vorschlag)
| Dienst | Host/Port | Klasse | Neuer Name | Gap / To-do |
|---|---|---|---|---|
| Home Assistant | NUC-HA 192.168.178.108:8123 | B (+D) | ha.mpauli.de | Outpost davor; Companion-App/API = X → Bypass-Pfad für `/api/`,`/auth/token`. Read-only-Dashboard → Gast |
| Immich | Synology 192.168.178.171:2283 | A | immich.mpauli.de | Immich OIDC-nativ (Mobile-App kann OAuth). Shared-Album „Hochzeit" → Gast |
| Grafana (monitoring) | K3s 10.10.0.234:80 | A (+D) | grafana.mpauli.de | Grafana `auth.generic_oauth`; anon-Viewer-Org → Gast |
| Portainer | NUC-HA :9443 | A | portainer.mpauli.de | Portainer OIDC (Business/CE OAuth) |
| Proxmox prox2 | 192.168.178.65:8006 | B + C | prox2.mpauli.de | GUI via Outpost; Node-Konsole via Guacamole (SSH) |

## WEB-DIENSTE — App-OIDC bevorzugt (Klasse A)
| Dienst | Host/Port | Neuer Name | Notiz |
|---|---|---|---|
| ArgoCD | K3s 10.10.0.237 | argocd.mpauli.de | OIDC/Dex nativ |
| Superset | K3s 10.10.0.253:8088 | superset.mpauli.de | OIDC via Flask-AppBuilder |
| XWiki | K3s 10.10.0.240:8085 | xwiki.mpauli.de | OIDC-Extension |
| Paperless-ngx | K3s 10.10.0.243:8010 | paperless.mpauli.de | OIDC (Allauth) |
| Synology DSM | 192.168.178.171:5001 | dsm.mpauli.de | DSM SSO-Client (OIDC) |
| CheckMK | 10.10.0.111 (Port prüfen) | checkmk.mpauli.de | CEE SAML/OIDC |
| Splunk | VM123 (down) | splunk67.mpauli.de | SAML/OIDC; erst Backend hochfahren |
| Confluence | (down) | conf67.mpauli.de | SAML-App; erst hochfahren |
| Beszel | NUC-HA :8090 | beszel.mpauli.de | OIDC-fähig |

## WEB-DIENSTE — Outpost-gated (Klasse B, kein OIDC)
| Dienst | Host/Port | Neuer Name |
|---|---|---|
| Node-RED | NUC-HA (HA-Addon) | nodered.mpauli.de |
| n8n (deal-hunter) | NUC-HA :5678 | n8n.mpauli.de |
| Bewerbungsassistent | NUC-HA :18080 | bewerbung.mpauli.de |
| NSDAP-Research UI | NUC-HA :8081 | nsdap.mpauli.de |
| Spiderfoot/Maigret/Blackbird | NUC-HA :5001/:5002/:9797 | osint-*.mpauli.de |
| Paperless-AI | K3s 10.10.0.236:3100 | paperless-ai.mpauli.de |
| SearXNG | K3s 10.10.0.244:8888 | searx.mpauli.de |
| Stock-Analyzer / Leak-Mon / Ransomwatch / Open-Archiver / IceSeller | K3s .242/.233/.246/.232/.239 | *.mpauli.de |
| UniFi UCK+ | 192.168.1.7 | unifi.mpauli.de (Outpost + eigenes Login) |

## NICHT-WEB — Guacamole (Klasse C)
| Ziel | Adresse | Protokoll |
|---|---|---|
| Proxmox prox2 (Node-Shell) | 192.168.178.65 | SSH |
| Proxmox prox1 (Hetzner) | 94.130.128.171 | SSH |
| Windows AD-DC BDC2025 | 10.10.0.89 | RDP |
| Synology (Shell) | 192.168.178.171 | SSH |
| Cisco WiFi-AZ/WZ/Office | .193/1.6/1.16 | SSH/Telnet |
| CRS305 / CSS326 | 192.168.178.9/.165/.166 | SSH/Winbox |
| NUC-HA / KI02 (Shell) | 10.10.0.100/.210 | SSH |

## GAST — read-only ohne Auth (Klasse D, `guest.mpauli.de`)
| Kachel | Quelle | Read-only-Mechanik |
|---|---|---|
| System-Status | Grafana anon-Viewer-Org | Grafana `auth.anonymous` role=Viewer |
| Home-Dashboard | Homarr 10.10.0.186:7575 | Homarr read-only Board |
| Haus (HA) | HA dediziertes View | eigener read-only HA-User/Dashboard |
| Hochzeitsfotos | Immich Shared-Album | Immich Public-Share-Link |
| Host-Health | Beszel | öffentliche Status-Seite |

## KEEP / SONDERFÄLLE
| Dienst | Entscheidung | Grund |
|---|---|---|
| **pfSense-GUI** (10.10.0.2 / 192.168.1.1) | LAN-direkt, NICHT gaten; extern nur via Guacamole/VPN | Zirkulär: Authentik läuft HINTER pfSense |
| **Joplin** (joplin67) | KEEP offen (kein Outpost) | Sync-Clients (Mobile/Desktop) können kein interaktives OIDC → X |
| **Pingvin** (pingvin67) | KEEP offen | anonymer Gäste-Upload gewollt |
| **Barbeleg** (invoice) | KEEP eigenes GeoIP-DE + Basic-Auth | THW-Nutzer nicht im Verzeichnis |
| **MQTT** (1883/8883) | KEEP Netz-ACL | Maschinen-Protokoll, nicht Authentik-fähig → X |
| **Ollama** (:11434), **Registry** (.245), **Trino/Hive/MinIO** (.251/.252/.230) | KEEP Netz-ACL/Token | interne APIs/Daten-Plane → X |
| **Cribl/Kibana/ES** (.247/.249) | deprecated | POC beendet 10.07. |

## IDENTITÄT
- Quelle **primär AD `paulis.net`** (LDAP-Source `ad-paulis`, svc-authentik), **Fallback lokal**.
- Lokale Authentik-Konten anlegen: `mpauli` (existiert via AD), **`jella` (neu, AD + lokal)**, `guest` (nur für D nicht nötig — D ist auth-frei).
- 2FA (WebAuthn/TOTP) erzwungen (bestehender Flow).

## DNS / NAMEN
- `*.mpauli.de` = externer + interner Edge (Split-Horizon → .154). Pro neuem Dienst: DDNS-HOST + LE-Zert + HAProxy-ACL/Backend + unbound-Override + AdGuard-Rewrite.
- **Optimierung möglich**: Wildcard `*.mpauli.de → .154` in beiden Resolvern statt Pro-Host (dann entfällt der DNS-Schritt je Dienst).
- **paulis.net** (AD-Domain) verfügbar: optional für rein-interne Direkt-Aliase (`dienst.paulis.net → interne IP`, ohne HAProxy) — aktuell nicht nötig, Split-Horizon deckt alles ab.
