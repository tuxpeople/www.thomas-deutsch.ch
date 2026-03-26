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
  - blog/publiziert
language: de
title: 'HashiCorp Vault Enterprise Training – Tag 1: Basics, Tokens und Policies'
description: >-
  Meine Notizen vom ersten Tag des HashiCorp Vault Enterprise Trainings:
  Seal/Unseal, Token-Typen, Authentication und das Policy-Modell – kompakt
  aufbereitet für alle, die Vault besser verstehen wollen.
date: 2025-11-04T00:00:00+0100
modified: 2026-03-26T00:00:00+0100
header:
  image: /assets/images/posts/vault-training-tag1-banner.png
  teaser: /assets/images/posts/vault-training-tag1-teaser.png
---

# HashiCorp Vault Enterprise Training – Tag 1: Basics, Tokens und Policies

Im November 2025 hatte ich die Gelegenheit, den dreitägigen Kurs **8H600GCH Vault Enterprise** bei LearnQuest/IBM zu besuchen. Vault begegnet mir immer häufiger in Kundenprojekten – sei es für Secrets-Management in Kubernetes-Umgebungen, PKI-Workflows oder dynamische Datenbank-Credentials. Höchste Zeit, das Thema systematisch anzugehen statt es nur aus der Praxis zu kennen.

Was folgt, sind meine aufbereiteten Notizen vom ersten Tag. Kein Tutorial, kein Deep Dive in die Doku – sondern das, was mir nach einem langen Kurstag als wichtigste Erkenntnisse hängen geblieben ist. Der Fokus liegt auf den Konzepten, die man verstanden haben muss, bevor Vault-Architektur-Entscheide Sinn ergeben.

> Dieser Post ist Teil einer dreiteiligen Serie. Tag 2 behandelt Architektur und Deployment, Tag 3 Dynamic Secrets und Kubernetes-Integration.

---

## Was ist Vault eigentlich?

Eine Frage, die man sich nach Jahren im Kubernetes-Umfeld irgendwann stellen muss: Warum Vault, wenn ich auch einfach Kubernetes Secrets verwenden kann?

Die Antwort liegt im Ansatz. Vault ist ein **identitätsbasiertes Secrets- und Verschlüsselungs-Management-System**. Es geht nicht nur darum, Secrets zu speichern – sondern darum, kontrollierten, authentifizierten und auditierten Zugang zu diesen Secrets zu gewähren. Vault ist damit viel näher an einem Sicherheitssystem als an einer Datenbank.

Konkret kann Vault:
- **Secrets speichern und versionieren** (KV Secrets Engine)
- **On-Demand-Credentials generieren** – für Public Clouds, Kubernetes, Datenbanken
- **PKI-Workflows** abbilden – Zertifikate ausstellen und rotieren
- **Verschlüsselung als Service** bereitstellen – ohne dass Applikationen Kryptographie selbst implementieren müssen

In der Praxis bedeutet das: Vault ist kein Drop-in-Ersatz für Kubernetes Secrets, sondern eine eigene Schicht im Security-Stack. Das Multitool-Prinzip macht es mächtig, aber auch komplex. Erste Erkenntnis des Tages: Wer Vault ernsthaft einsetzen will, braucht ein solides Grundverständnis der Konzepte – sonst konfiguriert man es falsch und merkt es erst unter Last.

---

## Sealed und Unsealed – warum dieser Unterschied grundlegend ist

Um zu verstehen, wie Vault mit Secrets umgeht, muss man zuerst verstehen, wie Vault mit sich selbst umgeht. Vault kennt zwei Betriebszustände: **Sealed** und **Unsealed**.

Im Sealed-Zustand ist Vault vollständig verschlüsselt und unbrauchbar – kein einziger API-Call funktioniert. Das ist der sichere Ruhezustand nach einem Neustart oder nach einem erkannten Sicherheitsvorfall. Für den Betrieb ist dieses Konzept wichtig, weil es bedeutet: Vault kann nicht einfach "aus Versehen" laufen. Es muss aktiv entsperrt werden.

Für das Unsealen ist es wichtig, den Schlüsselbaum zu verstehen. Vault verwendet drei Schlüsselebenen:

| Schlüssel | Funktion |
|---|---|
| **Key Shares** | Einzelne Fragmente, die zusammengeführt den Unseal Key ergeben |
| **Unseal Key** | Entschlüsselt den Root Key |
| **Root Key** | Schützt den eigentlichen Encryption Key, der die Vault-Daten verschlüsselt |

Beim Unsealen gibt man also nicht direkt den Root Key ein – sondern eine ausreichende Anzahl **Key Shares**, die gemeinsam den Unseal Key rekonstruieren, welcher wiederum den Root Key entschlüsselt. Hier kommt **Shamir's Secret Sharing** ins Spiel: Der Unseal Key wird in mehrere Shares aufgeteilt. Erst wenn eine definierte Mindestanzahl zusammengeführt wird, entsteht der vollständige Unseal Key. Der Standard: 5 Shares, davon werden 3 benötigt.

Das Prinzip dahinter ist klug: Kein einzelner Administrator kann Vault alleine unsealen. Man braucht immer mehrere vertrauenswürdige Personen – ein technisch erzwungenes Vier-Augen-Prinzip.

In der Praxis ist manuelles Unsealen allerdings unhandlich. Für Produktionsumgebungen gibt es daher **Auto-Unseal**: Die Entschlüsselung wird an einen vertrauenswürdigen externen Dienst delegiert – etwa ein Hardware Security Module (HSM) oder einen Cloud KMS (AWS KMS, Azure Key Vault, GCP Cloud KMS). Vault unsealt sich dann beim Start automatisch, ohne dass jemand eingreifen muss.

Wichtiger Unterschied bei Auto-Unseal: Statt Unseal Keys kommen hier **Recovery Keys** zum Einsatz. Diese sind ein reiner Autorisierungsmechanismus – sie entschlüsseln den Root Key *nicht* direkt. Recovery Keys werden für administrative Operationen benötigt, die andernfalls einen Quorum an Unseal-Key-Inhabern voraussetzen würden.

Eine Randnotiz: Vault setzt `mlock()` ein, um zu verhindern, dass sensible Daten aus dem RAM in den Swap-Speicher ausgelagert werden. Details wie diese zeigen, wie konsequent das Sicherheitsmodell von Vault durchgezogen ist – aber sie bedeuten auch, dass Vault etwas sorgfältiger deployed werden will als ein gewöhnlicher Dienst.

---

## Tokens – wie Vault Identität und Zugriff verbindet

Nachdem klar ist, wie Vault sich selbst schützt, stellt sich die Frage: Wie kommuniziert man mit Vault? Die Antwort ist konsequent einheitlich: über **Tokens**.

Egal ob ein Mensch sich via LDAP anmeldet oder eine Kubernetes-Workload eine Datenbank-Credential anfragt – am Ende bekommt jeder einen Token, mit dem er mit der Vault API interagiert. In der Praxis bedeutet das, dass nahezu jede Vault-Interaktion über kurzlebige, kontrollierbare Credentials erfolgt – ein zentraler Unterschied zu klassischen Secret Stores, die oft mit statischen API-Keys oder langlebigen Passwörtern arbeiten.

Die drei eigentlichen Token-Typen:

| Typ | Beschreibung |
|---|---|
| **Service Token** | Das "normale" Token. Wird im Storage Backend persistiert, kann Child-Tokens erstellen, unterstützt alle Features. |
| **Batch Token** | Leichtgewichtiger, kleineres Feature-Set. Sinnvoll für Hochlast-Szenarien mit sehr vielen kurzlebigen Tokens. |
| **Recovery Token** | Einmalig, nur für den Recovery Mode verfügbar. |

Daneben gibt es **speziell benannte Tokens**: Der wichtigste ist der **Root Token** – kein eigener Token-Typ, sondern ein Service Token mit der `root`-Policy. Root Tokens können so konfiguriert werden, dass sie kein TTL haben und nie ablaufen. Genau das macht sie zum Risiko: Ein vergessener, aktiver Root Token ist ein dauerhaftes Sicherheitsproblem. Die etablierte Best Practice – und aus meiner Sicht in produktiven Umgebungen konsequent umzusetzen – ist, Root Tokens direkt nach dem initialen Setup oder einer Notfall-Operation zu revoken.

Fast alle regulären Tokens haben ein **TTL** (Time To Live). Ist das TTL abgelaufen, ist der Token wertlos – das ist kein Bug, sondern Absicht. Kurzlebige Credentials reduzieren das Schadenspotenzial bei einer Kompromittierung erheblich.

---

## Wer darf rein? Auth Methods und Secure Introduction

Bevor ein Token ausgestellt werden kann, muss sich jemand authentifizieren. Vault kennt dafür eine breite Palette an **Auth Methods** – und trennt dabei sauber zwischen menschlichen und maschinellen Identitäten.

**Für Menschen:**
- **LDAP** – Integration in bestehende Verzeichnisdienste
- **OIDC** – SSO via Identity Provider (z. B. Entra ID, Okta)
- **Username/Password** – für Break-Glass-Szenarien; ein Notfallzugang, der nicht über externe Systeme abhängt

**Für Maschinen und Workloads:**
- **Kubernetes Auth** – Pods authentifizieren sich mit ihrem ServiceAccount-Token gegenüber Vault
- **AppRole** – Role ID + Secret ID, typisch für automatisierte Pipelines
- **Cloud-Plattform-Auth** – AWS, Azure, GCP bringen native Identitäten mit

Separat davon gibt es das Konzept der **Secure Introduction** – also wie ein System überhaupt an seine erste Vault-Credential kommt (das sogenannte "Secret Zero Problem"): Wie gibt man einem frisch gestarteten System eine erste Credential, ohne sie irgendwo hartzukodieren? Das Kurs-Material unterscheidet dabei zwei Patterns:

- **Platform Integration**: Vault vertraut einer Cloud-Plattform (z. B. Kubernetes), die die Identität einer Workload bestätigt. Der Workload braucht keine vorkonfigurierten Credentials.
- **Trusted Orchestrator**: Ein System (z. B. Ansible), das bereits gegenüber Vault authentifiziert ist, injiziert beim Deployment die notwendigen Credentials für neue Systeme.

Diese Patterns sind keine Auth Methods im technischen Sinne – sie beschreiben die Architektur, wie Systeme initiell an ihre ersten Vault-Tokens gelangen. Für mich ist die Kubernetes Auth Method der relevanteste Einstiegspunkt, da ich täglich mit Kubernetes-Workloads arbeite: Ein Pod beweist seine Identität mit einem von Kubernetes ausgestellten Token, Vault verifiziert das gegen die Kubernetes API und stellt einen Token mit den passenden Berechtigungen aus – kein statisches Passwort, keine langlebige Credential.

---

## Policies – das Herzstück der Zugriffskontrolle

Wenn Auth Methods definieren, *wer* sich anmelden kann, definieren **Policies**, *was* danach erlaubt ist. Das Grundprinzip ist konsequent: **Alles ist verboten, was nicht explizit erlaubt ist.**

Policies werden in HCL (HashiCorp Configuration Language) geschrieben und legen fest, welche Aktionen auf welchen Pfaden erlaubt sind. Die wichtigsten Capabilities sind:

`create`, `read`, `update`, `patch`, `delete`, `list`, `sudo`, `deny`

(Die vollständige Liste enthält noch weitere; für den Alltag sind die genannten die relevanten.)

Zwei Wildcards helfen beim Schreiben flexibler Policies:
- `*` – matcht rekursiv alles unterhalb eines Pfades
- `+` – matcht genau eine Ebene (nützlich für parametrisierte Pfade wie `secret/data/+/config`)

Ein minimales Beispiel, das einem Kubernetes-Namespace Lesezugriff auf seine eigenen Secrets gibt – ohne dass eine separate Policy pro Namespace nötig ist:

```hcl
path "secret/data/{{identity.entity.aliases.auth_kubernetes_XXXX.metadata.service_account_namespace}}/*" {
  capabilities = ["read"]
}
```

Zwei besondere Policies gibt es immer und können nicht gelöscht werden:
- **root**: Zugang zu allem
- **default**: Minimale Grundfunktionalität (z. B. eigenen Token erneuern), wird automatisch jedem Token zugewiesen

In der Praxis empfiehlt es sich, Policies entlang von Rollen in der Organisation zu schreiben – nicht entlang von Systemen. Das macht sie wartbarer, wenn Teams oder Zuständigkeiten sich ändern.

---

## Static Secrets mit KV v2

Zum Abschluss des ersten Tags noch ein praktischer Einstiegspunkt für alle, die Vault zunächst für einfache Anwendungsfälle nutzen wollen: die **KV Secrets Engine (Version 2)**.

Das Prinzip ist einfach: Write before Read – ein Secret wird gespeichert, bevor es gelesen werden kann. Was KV v2 gegenüber einer simplen Datenbank interessant macht, ist die automatische **Versionierung**: Jede Schreiboperation erstellt eine neue Version, standardmässig werden 10 Versionen aufbewahrt. Das ist besonders relevant, wenn Secrets regelmässig rotiert werden oder versehentliche Änderungen rückgängig gemacht werden müssen – Rollbacks auf einen bekannt-guten Stand sind damit kein Problem.

Für einfache Anwendungsfälle wie einen API-Key, den man sicher ablegen und gelegentlich rotieren will, ist KV v2 ein pragmatischer Einstieg. Für dynamische Credentials – also Datenbank-Passwörter oder Cloud-Credentials, die Vault selbst generiert und rotiert – gibt es spezialisierte Secrets Engines, die in Tag 3 Thema sind.

---

## Fazit Tag 1

Der erste Kurstag hat das Fundament gelegt – und dabei ein paar Annahmen korrigiert, die ich aus der Praxis mitgebracht hatte. Meine drei wichtigsten Takeaways:

1. **Vault ist kein Key-Value-Store mit TLS.** Das Identitäts- und Policy-Modell ist der eigentliche Kern – alles andere baut darauf auf.
2. **Root Tokens so kurz wie möglich leben lassen.** Nicht weil es technisch erzwungen wird, sondern weil ein dauerhaft aktiver Root Token ein unkontrolliertes Risiko ist.
3. **Auto-Unseal ist in produktiven Umgebungen kein Nice-to-have.** Manuelles Unsealen mit mehreren Personen mag in der Theorie sicher klingen – in der Praxis ist es ein Availability-Risiko, das man nicht eingehen muss.

Tag 2 geht es um Architektur, Deployment-Patterns und Replication – also den Teil, der für langfristige Infrastruktur-Entscheide relevant ist.
