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
title: 'HashiCorp Vault Enterprise Training – Tag 3: Dynamic Secrets, Kubernetes und Terraform'
description: 'Der letzte Tag des Vault Enterprise Trainings: Dynamic Secrets als Killer-Feature, Vault in Kubernetes mit Agent und Secrets Operator, MFA-Patterns und vollständige Automatisierung via Terraform.'
date: 2025-11-06T00:00:00+0100
modified: 2026-03-26T00:00:00+0100
header:
  image: /assets/images/posts/vault-training-tag3-banner.png
  teaser: /assets/images/posts/vault-training-tag3-teaser.png
---

# HashiCorp Vault Enterprise Training – Tag 3: Dynamic Secrets, Kubernetes und Terraform

Der dritte und letzte Kurstag war für mich der interessanteste – nicht weil das Fundament der ersten zwei Tage weniger wichtig wäre, sondern weil Tag 3 zeigt, wo Vault seinen grössten praktischen Unterschied macht. Dynamic Secrets, Kubernetes-Integration und Infrastructure-as-Code: Das sind die Themen, die im Alltag direkt spürbar werden.

 > 
 > Dieser Post ist Teil einer dreiteiligen Serie. [Tag 1](/vault-enterprise-training-tag-1) behandelt Basics, Tokens und Policies. [Tag 2](/vault-enterprise-training-tag-2) behandelt Architektur, Deployment und Replication.

## Plugins: Vault ist erweiterbar

Vault ist von Grund auf plugin-basiert aufgebaut. Jede Auth Method, jede Secrets Engine, jede Database-Integration ist ein Plugin. Das macht Vault erweiterbar, ohne den Core anfassen zu müssen.

Drei Plugin-Kategorien:

* **Offizielle Plugins** – von HashiCorp gepflegt, im Core ausgeliefert
* **Partner-Plugins** – von Drittanbietern zertifiziert (z. B. Datenbankintegrationen)
* **Community-Plugins** – frei verfügbar, ohne Support-Garantie

Externe Plugins müssen explizit registriert werden, bevor sie aktiviert werden können. Der Upgrade-Ablauf für Plugins ist klar definiert:

1. Neues Plugin registrieren und aktivieren
1. Als "pinned version" im Cluster setzen
1. Globalen Plugin-Reload triggern – ohne Neustart des Vault-Prozesses

Das Plugin-Modell erklärt auch, warum Vault so viele Integrationen hat: Jede neue Datenbank, jeder neue Cloud-Provider, jedes neue Auth-System ist potenziell ein Plugin.

## Dynamic Secrets: Das Killer-Feature

Wenn ich Leuten erklären will, warum Vault mehr als ein sicherer Key-Value-Store ist, ist das mein Argument: **Dynamic Secrets**.

Das Konzept: Statt statische Credentials irgendwo zu speichern und zu rotieren, generiert Vault **on-demand kurzlebige Credentials** – direkt bei Bedarf, mit definiertem TTL, automatisch revoked wenn nicht mehr gebraucht.

Ein konkretes Beispiel für Datenbank-Credentials:

1. Eine Applikation authentifiziert sich bei Vault (z. B. via Kubernetes Auth)
1. Vault erzeugt in der Zieldatenbank einen temporären User mit zufälligem Passwort
1. Die Applikation bekommt diese Credentials – gültig für z. B. 1 Stunde
1. Nach Ablauf des TTL revoked Vault die Credentials – der User in der Datenbank wird gelöscht

Was das bedeutet: **Es gibt keine langlebigen Datenbank-Passwörter mehr.** Kein Rotationsscript, das irgendwo vergessen wird. Kein statisches Passwort, das in einer `.env`-Datei landet. Kein "wer hat nochmals Zugriff auf die Produktionsdatenbank?"

Zusätzlich gibt es **Static Roles** mit Auto-Rotate: Vault übernimmt die Rotation eines bestehenden Datenbankusers nach einem definierten Zeitplan. Das ist der pragmatische Weg für Legacy-Systeme, die keinen User-pro-Session unterstützen.

Der Lifecycle von Dynamic Secrets ist vollständig automatisiert:

* **Erstellen** beim Request
* **Verwenden** durch die Applikation
* **Revoken** nach TTL oder explizit

## Vault in Kubernetes: Zwei Patterns

Vault lässt sich auf zwei grundlegend unterschiedliche Arten in Kubernetes integrieren. Die Wahl hängt davon ab, ob man Kubernetes Secrets als Zwischenstufe akzeptiert oder vermeiden will.

### Vault Secrets Operator (empfohlen)

Der Operator synchronisiert Vault-Secrets als Kubernetes Secrets in den Cluster. Applikationen greifen auf gewohnte Kubernetes Secrets zu – sie müssen nichts von Vault wissen.

````yaml
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultStaticSecret
metadata:
  name: my-app-secret
spec:
  vaultAuthRef: my-vault-auth
  mount: kvv2
  path: my-app/config
  destination:
    name: my-app-secret   # → Kubernetes Secret
    create: true
````

**Wichtiger Hinweis:** Der Operator erstellt unverschlüsselte Kubernetes Secrets. Das ist kein Fehler des Operators – Kubernetes Secrets sind standardmässig nur base64-kodiert, nicht verschlüsselt. Wer den Operator nutzt, muss RBAC konsequent anwenden und sollte über Encryption at Rest für etcd nachdenken (z. B. mit KMS-Integration).

Eine erwähnenswerte Alternative: Die **JWT/OIDC Auth Method mit Kubernetes als OIDC Provider**. Statt der TokenReview API verifiziert Vault dabei die JWTs via Public-Key-Kryptografie direkt. Das vereinfacht Multi-Cluster-Setups, weil kein Reviewer-JWT konfiguriert werden muss.

Allerdings gibt es einen wichtigen Trade-off: Da die TokenReview API nicht verwendet wird, erkennt Vault ein von Kubernetes revoziertes Token erst nach Ablauf seines TTL als ungültig. Die Empfehlung der Vault-Doku ist deshalb, bei diesem Ansatz kurze TTLs für ServiceAccount-Tokens zu verwenden.

### Vault Agent Injector

Der Agent läuft als Sidecar-Container neben der Applikation und schreibt Secrets als Dateien in ein gemeinsames Volume. Keine Kubernetes Secrets, kein Zwischenspeicher – die Applikation liest direkt aus dem Filesystem.

Annotation-basierte Konfiguration im Pod:

````yaml
annotations:
  vault.hashicorp.com/agent-inject: "true"
  vault.hashicorp.com/agent-inject-secret-config: "my-app/config"
  vault.hashicorp.com/role: "my-app-role"
````

Der Injector mutiert den Pod automatisch via Admission Webhook und fügt den Sidecar-Container ein.

**Wann welches Pattern?**

* **Operator**: Für neue Setups, wo Kubernetes-native Integration gewünscht ist. Einfacher zu betreiben.
* **Agent Injector**: Wenn Secrets nicht als Kubernetes Secret existieren sollen, oder wenn Legacy-Applikationen Dateipfade statt Umgebungsvariablen erwarten.

### Dev-Mode: Nur fürs Labor

Eine Randnotiz, die aber wichtig ist: Vault unterstützt einen Dev-Mode, der bereits initialisiert und unsealed startet, komplett im Memory ohne persistenten Storage.

````bash
vault server -dev
````

Das ist nützlich in CI/CD-Pipelines zum Testen von Vault-Konfigurationen – und **ausschliesslich** dafür. In Produktion hat der Dev-Mode nichts verloren: keine Persistenz, kein TLS, kein echter Schutz.

## Multi-Factor Authentication

Vault unterstützt MFA – in zwei konzeptuell verschiedenen Varianten:

**Login MFA** fügt dem normalen Login-Workflow einen zweiten Faktor hinzu. Nach der primären Authentifizierung (z. B. LDAP) muss ein TOTP-Code oder ein Okta/Duo/PingID-Push bestätigt werden, bevor ein Token ausgestellt wird.

**Enterprise Step-Up MFA** ist für Non-Login-Usecases gedacht: Eine bereits authentifizierte Session muss für bestimmte sensible Operationen (z. B. PKI-Zertifikate ausstellen, Root-CA-Operationen) nochmals per MFA bestätigt werden. Das ist das "Vault kennt mich, aber für diese Aktion muss ich mich extra ausweisen"-Prinzip.

Zwei Authentifizierungs-Flows:

* **Single Phase**: Zweiter Faktor wird direkt beim Login mitgegeben
* **Two Phase**: Login funktioniert, aber ein MFA-Request wird separat gestellt und muss bestätigt werden, bevor der Token vollständig gültig ist

## Vault Agent und Vault Proxy

Beide sind **Client-Daemons**, die zwischen Applikation und Vault-Server vermitteln – aber mit unterschiedlichem Schwerpunkt.

**Vault Agent** ist der Feature-reiche Allrounder:

* Auto-Authentifizierung gegenüber Vault
* **Templating**: Secrets werden in Konfigurationsdateien gerendert und bei Rotation automatisch aktualisiert
* **Process Supervisor Mode**: Secrets werden als Umgebungsvariablen in einen Child-Prozess injiziert
* Löst das "Secret Zero Problem" auf der Infrastrukturebene

**Vault Proxy** ist auf den API-Proxy-Usecase fokussiert:

* Auto-Authentifizierung gegenüber Vault (wie Agent)
* Cached Vault-API-Calls und leitet sie weiter – Applikationen sprechen gegen den Proxy statt direkt gegen Vault
* Cached neu erstellte Tokens, Leases und KV-Secrets
* Kann Clients zwingen, das automatisch authentifizierte Token des Proxy zu verwenden

Der wesentliche Unterschied: Vault Agent rendert und verwaltet Secrets aktiv (Templates, Dateien, Env-Vars). Vault Proxy fungiert primär als intelligenter API-Relay mit Auto-Auth und Caching – er hat kein Templating, aber dafür einen kleineren Feature-Footprint, was ihn für bestimmte Szenarien schlanker macht.

## Vault mit Terraform verwalten

Der letzte grosse Block des Trainings: Vault selbst als Code verwalten.

Der [Vault Terraform Provider](https://registry.terraform.io/providers/hashicorp/vault/latest/docs) erlaubt es, den kompletten Vault-Lifecycle zu automatisieren:

````hcl
resource "vault_auth_backend" "kubernetes" {
  type = "kubernetes"
  path = "kubernetes"
}

resource "vault_kubernetes_auth_backend_config" "config" {
  backend            = vault_auth_backend.kubernetes.path
  kubernetes_host    = "https://kubernetes.default.svc"
}

resource "vault_policy" "my_app" {
  name   = "my-app"
  policy = file("policies/my-app.hcl")
}
````

Was sich sinnvoll als Code verwalten lässt:

* Auth Methods aktivieren und konfigurieren
* Secrets Engines mounten
* Policies pflegen
* Kubernetes Auth Backend konfigurieren

**Caveats:** Einige Vault-Ressourcen sind state-sensitiv – z. B. Unseal-Keys, Recovery-Keys oder Root-Tokens. Diese gehören nicht in Terraform State. Sensible Outputs sollten mit `sensitive = true` markiert werden.

Für mich ist die Terraform-Integration der konsequente letzte Schritt: Vault konfiguriert als Code, versioniert in Git, nachvollziehbar und reproduzierbar.

## Fazit: Was bleibt nach drei Tagen Vault

Der Kurs hat mein Verständnis von Vault grundlegend verschoben – weg vom "sicherer Key-Value-Store" hin zu einem vollständigen Secrets-Lifecycle-System.

Die drei wichtigsten Erkenntnisse über alle drei Tage:

**Dynamic Secrets sind der echte Mehrwert.** Nicht die sichere Speicherung – die kann auch ein gut gesicherter Kubernetes Secret Store. Sondern die Fähigkeit, kurzlebige, automatisch rotierende Credentials on-demand zu generieren. Das eliminiert eine ganze Klasse von Sicherheitsproblemen.

**Vault ist Betriebsaufwand.** Seal/Unseal, Audit Devices, Replication, Token-Lifecycle – das sind alles Themen, die operativ durchdacht und geübt sein müssen, bevor man in Produktion geht. Vault kauft einem keine Sorglosigkeit, sondern gibt einem präzise Kontrolle. Die muss man bereit sein zu nutzen.

**Kubernetes und Vault ergänzen sich gut.** Die Kubernetes Auth Method ist elegant: Pods beweisen ihre Identität mit einem kurzlebigen ServiceAccount-Token, Vault stellt im Gegenzug eine kurzlebige Vault-Credential aus. Kein statisches Passwort, keine Out-of-Band-Konfiguration.

Offene Fragen, die mich noch beschäftigen:

* Macht ein Vault im Homelab Sinn – und wenn ja, mit Yubikey als "HSM"? ([Yubico und Vault](https://www.yubico.com/works-with-yubikey/catalog/hashicorp/))
* Wie würde ein Vault-Setup für ein Kubernetes SDMZ-Cluster konkret aussehen?
* Dynamic Secrets für Datenbanken in einer bestehenden Kundenumgebung einführen – wo fange ich an?
