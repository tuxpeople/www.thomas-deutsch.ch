---
aliases: []
tags:
- Vault
- HashiCorp
- SecretsManagement
- Training
- Kubernetes
- audience-practitioner
- audience-architect
- type-experience-report
language: de
title: 'HashiCorp Vault Enterprise Training – Tag 2: Architektur, Deployment und Replication'
description: 'Tag 2 des Vault Enterprise Trainings: Wie Vault produktionsreif aufgebaut wird – Referenzarchitektur, Raft-Storage, Replication-Modi und was beim Monitoring wirklich zählt.'
date: 2025-11-05T00:00:00+0100
modified: 2026-03-26T00:00:00+0100
header:
  image: /assets/images/posts/vault-training-tag2-banner.png
  teaser: /assets/images/posts/vault-training-tag2-teaser.png
---

# HashiCorp Vault Enterprise Training – Tag 2: Architektur, Deployment und Replication

Der zweite Trainingstag hatte einen klaren Schwerpunkt: Wie sieht ein produktionsreifes Vault-Setup aus? Nach dem Konzeptfundament vom Vortag ging es nun um konkrete Architektur-Entscheide – und um die Frage, was passiert, wenn etwas schiefläuft. Das sind die Themen, bei denen sich viele Teams erst zu spät Gedanken machen.

 > 
 > Dieser Post ist Teil einer dreiteiligen Serie. [Tag 1](/vault-enterprise-training-tag-1) behandelt Basics, Tokens und Policies. Tag 3 behandelt Dynamic Secrets und Kubernetes-Integration.

---

## Secrets Access Patterns: Wie Applikationen an ihre Secrets kommen

Bevor wir zur Infrastruktur kamen, stand ein konzeptuell wichtiges Thema auf dem Programm – denn die beste Vault-Architektur nützt wenig, wenn unklar ist, wie Applikationen überhaupt auf ihre Secrets zugreifen sollen.

Vault kennt drei grundlegende Patterns, die sich nach Applikationstyp und Migrationssituation unterscheiden:

**1. Direkte API-Integration**
Die Applikation spricht selbst mit der Vault API, authentifiziert sich und holt ihre Credentials. Das ist der sauberste Ansatz – setzt aber voraus, dass die Applikation "Vault-aware" ist. Bei Eigenentwicklungen gut umsetzbar, bei Fremdsoftware oft nicht.

**2. Vault Agent (Sidecar)**
Ein Vault Agent läuft als Sidecar-Prozess neben der Applikation, übernimmt die Authentifizierung und legt Secrets als Dateien in ein gemeinsames Volume. Die Applikation liest einfach eine Konfigurationsdatei – sie weiss nichts von Vault. Das ist der pragmatischste Weg für Legacy-Applikationen oder Szenarien, in denen man keinen Code anpassen kann oder will.

**3. Platform Integration**
Kubernetes-native Patterns wie der Vault Secrets Operator synchronisieren Vault-Secrets direkt als Kubernetes Secrets. Mehr dazu in Tag 3.

Die Wahl des richtigen Patterns hängt stark vom Kontext ab – es gibt kein universell richtiges Muster.

---

## Namespaces: Multitenancy auf Vault-Ebene (Enterprise only)

Wer Vault für mehrere Teams oder Kunden betreiben will, stösst früh auf die Frage der Isolation. Genau dafür gibt es **Vault Namespaces** – ein Feature, das nur in der Enterprise Edition verfügbar ist.

Das Konzept ist elegant: Jeder Namespace verhält sich wie eine eigene Vault-Instanz – mit eigenen Auth Methods, Secrets Engines, Policies und Tokens. Teams oder Kunden verwalten ihren Namespace selbst, ohne Sichtbarkeit in andere Namespaces. Kleinerer Blast Radius, saubere Delegation, keine gegenseitigen Abhängigkeiten.

Namespaces können verschachtelt werden. Policies sind dabei standardmässig auf den Namespace beschränkt, in dem sie erstellt wurden – es gibt keine automatische Policy-Vererbung von Parent zu Child. Was jedoch möglich ist: Ein Parent-Namespace kann Policies schreiben, die explizit auf Child-Pfade verweisen. Umgekehrt können Policies in einem Child-Namespace auf Entities oder Gruppen aus dem Parent referenzieren, und ein Parent kann Policies auf Identitäten in Child-Namespaces durchsetzen. Dieses Cross-Namespace-Verhalten muss aktiv konfiguriert werden (via `group-policy-application`-Endpunkt) – es passiert nicht automatisch.

**Wann macht ein Namespace Sinn?**

* Unterschiedliche Audit-Anforderungen pro Team oder Mandant
* Chargeback- oder Self-Service-Modelle
* Verschiedene Secrets-Engine-Anforderungen pro Bereich
* Organisationsstruktur erfordert klare Trennung

**Antipattern:** Zu viele Namespace-Ebenen. Wie bei Kubernetes-Namespaces gilt: Struktur ist gut, aber Überstruktur erzeugt Verwaltungsaufwand ohne echten Sicherheitsgewinn.

---

## Referenzarchitektur: 6 Nodes mit Redundancy Zones

Wenn man versteht, wie Vault intern mit State umgeht, erschliesst sich auch, warum es in einer bestimmten Konfiguration deployed werden sollte. Die Empfehlung aus dem Training: **6 Nodes, verteilt auf 3 Availability Zones**.

````
AZ-1              AZ-2              AZ-3
┌──────────┐      ┌──────────┐      ┌──────────┐
│ Voter    │      │ Voter    │      │ Voter    │  ← 3 Voting Nodes
└──────────┘      └──────────┘      └──────────┘
┌──────────┐      ┌──────────┐      ┌──────────┐
│ Non-Voter│      │ Non-Voter│      │ Non-Voter│  ← 3 Non-Voting Nodes
└──────────┘      └──────────┘      └──────────┘
````

Pro AZ läuft jeweils ein **Voting Node** (aktiv am Raft-Quorum beteiligt) und ein **Non-Voting Node** (Stand-by). Das Schlüsselfeature: **Vault Autopilot** kann bei Ausfall eines Voting Nodes automatisch den Non-Voter in derselben AZ zum Voter promoten – das Quorum bleibt stabil, auch bei einem vollständigen AZ-Ausfall.

Ein kurzer Hinweis zur Einordnung: Die offizielle HashiCorp-Basisdokumentation zeigt häufig ein einfacheres **5-Node-Setup** mit ausschliesslich Voting Nodes als Minimalreferenz. Beide Ansätze sind technisch korrekt – sie verfolgen aber unterschiedliche Ziele:

|Setup|Nodes|Modell|Stärke|
|-----|-----|------|------|
|Basis-HA|5|Alle Voting|Einfach, weniger Overhead|
|Enterprise Pattern|6|3 Voter + 3 Non-Voter|AZ-stabile HA, Autopilot-Promotion|

Das 6-Node-Setup mit Redundancy Zones ist der konservativere, produktionsreifere Ansatz – besonders dort, wo ein AZ-Ausfall keine manuelle Intervention erfordern soll.

**Storage: Integrierter Raft**

Um zu verstehen, warum Vault typischerweise mit mehreren Nodes betrieben wird, lohnt ein kurzer Blick auf das zugrunde liegende Speicher-Konzept. Vault verwendet bevorzugt den **integrierten Raft-Storage**. Raft ist ein Konsensalgorithmus, der sicherstellt, dass alle Nodes denselben State sehen – auch wenn einzelne Nodes ausfallen oder kurz offline gehen. Ein Leader Node nimmt alle Writes an, Follower Nodes replizieren und können Reads bedienen.

Wer Raft besser verstehen will: [thesecretlivesofdata.com/raft](https://thesecretlivesofdata.com/raft/) visualisiert den Algorithmus interaktiv – sehr empfehlenswert.

Consul als alternativer Storage-Backend ist zwar noch supported, wird aber immer seltener eingesetzt. Integrierter Raft ist einfacher zu betreiben und bringt weniger Moving Parts mit.

**Netzwerk:**

* Client Traffic: TCP/8200
* Server-zu-Server (Raft): TCP/8201
* Netzwerklatenz zwischen Nodes sollte unter 8ms liegen
* Nodes sprechen über IP-Adressen, nicht DNS – weniger Abhängigkeiten bei Ausfällen
* TLS überall; ein Loadbalancer mit TLS Pass-through ist möglich

**Audit Devices:**

Ein Detail mit grosser operativer Konsequenz: Wenn alle konfigurierten Audit Devices ausfallen, **verarbeitet Vault keine Requests mehr** – es wird effektiv unavailable. Das ist kein automatisches Sealing, sondern eine bewusste Design-Entscheidung: Vault verweigert den Betrieb ohne funktionierendes Audit-Logging, weil ohne Audit keine Compliance gewährleistet werden kann.

In der Praxis bedeutet das: Mindestens zwei Audit Devices konfigurieren (z. B. File + Syslog), damit der Betrieb bei Ausfall eines Devices weitergeht. Das klingt trivial, wird aber in der Planung oft vergessen – bis es zu spät ist.

---

## Vault Replication: DR und Performance

Enterprise Vault bietet zwei Replication-Modi – mit grundlegend unterschiedlichen Zwecken. Wer beide verwechselt, plant seine Disaster-Recovery-Strategie falsch.

### Disaster Recovery (DR) Replication

DR-Replication ist für den Fall gedacht, dass ein kompletter Cluster ausfällt:

* **Modus:** Warm Standby – der Secondary ist betriebsbereit, bedient aber keine Requests
* **Kein automatischer Failover** – ein Mensch muss den Failover aktiv auslösen
* **Kein Split-Brain** – durch den manuellen Failover wird sichergestellt, dass nie zwei aktive Cluster gleichzeitig existieren

Der DR-Failover-Ablauf ist klar definiert und sollte im Vorfeld geübt werden:

1. Primary prüfen – ist er wirklich ausgefallen?
1. DR Operation Token auf dem Secondary bereitstellen
1. Secondary zu Primary promoten
1. Activation Token für den neuen Secondary generieren
1. Alten Primary (wenn wieder erreichbar) als neuen Secondary registrieren

Root Tokens sollten nach dem Failover-Prozess zeitnah revoked werden – sie haben kein automatisches TTL und müssen aktiv verwaltet werden.

### Performance Replication (PR)

PR-Replication verfolgt ein anderes Ziel: horizontale Skalierung und geografische Verteilung:

* Secondary-Cluster können aktiv Reads bedienen – der Primary hat weniger Last
* **Pfad-Filterung** möglich: Bestimmte Secrets können aus der Replikation ausgeschlossen werden – relevant für Datenschutzanforderungen wie DSGVO oder revDSG, wenn EU-Daten in der EU bleiben sollen
* Sinnvoll, wenn Vault-Clients in verschiedenen Regionen sitzen und niedrige Latenz brauchen

---

## Monitoring: Drei Ebenen, die zusammenspielen müssen

Ein laufendes Vault ist keine Garantie für ein funktionierendes Vault – deshalb ist Monitoring mehr als nur ein Uptime-Check.

**Log-Analyse:** Audit Logs sind das wichtigste Artefakt. Jede Operation in Vault wird geloggt – wer was wann gemacht hat. Diese Logs sollten in ein zentrales SIEM oder zumindest in einen sicheren, unveränderlichen Log-Store fliessen. Im Incident-Fall sind sie oft die einzige verlässliche Quelle.

**Telemetrie:** Vault exponiert Metriken über verschiedene Backends (Prometheus, StatsD etc.). Besonders relevante Metriken: Seal-Status, Token-Ausstellungsrate, Secrets-Zugriffe, Raft-Leader-Wechsel. Ein fertiges Grafana-Dashboard gibt es unter [grafana.com](https://grafana.com/grafana/dashboards/12904-hashicorp-vault/).

**Synthetic Monitoring:** Regelmässige Test-Requests gegen die Vault API, um sicherzustellen, dass Vault nicht nur läuft, sondern auch korrekt antwortet. Das fängt Fälle ab, in denen Vault technisch "up" ist, aber z. B. wegen fehlender Audit Devices keine Requests mehr verarbeitet – ein Szenario, das ein reiner Port-Check nicht erkennen würde.

---

## Fazit Tag 2

Architektur-Entscheide bei Vault haben langfristige Konsequenzen. Was mich am meisten beschäftigt hat:

1. **Audit-Device-Availability ist unterschätzt.** Dass Vault bei fehlendem Audit-Device keine Requests mehr verarbeitet, klingt drakonisch – ist aber konsequent. Wer das nicht in der Planung berücksichtigt, erlebt eine böse Überraschung in Produktion.
1. **Raft ist die richtige Wahl.** Weniger Abhängigkeiten, integriertes Clustering, breite Erfahrung in der Community. Consul als Storage-Backend hat seinen Zenit überschritten.
1. **DR ≠ automatischer Failover.** Das ist eine bewusste Design-Entscheidung, keine Schwäche. Vault legt die Kontrolle in Menschenhände – und das hat gute Gründe.

Tag 3 wird operativer: Dynamic Secrets, Vault in Kubernetes und Terraform-Automatisierung.
