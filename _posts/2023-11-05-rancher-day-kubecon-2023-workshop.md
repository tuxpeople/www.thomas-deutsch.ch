---
aliases: []
tags:
- Rancher
- Kubernetes
- KubeCon2023
- Security
- NeuVector
- Fleet
- audience-practitioner
- audience-community
- type-experience-report
- blog/publiziert
language: de
title: Rancher Day an der KubeCon NA 2023 – Ein Rückblick
description: Meine Notizen und Eindrücke vom Rancher Day Workshop an der KubeCon North America 2023 in Chicago – von der Installation bis zur Container-Security mit NeuVector.
date: 2023-11-06T00:00:00+0100
modified: 2026-03-20T00:00:00+0100
header:
  image: /assets/images/posts/kubecon-na-2023-banner.jpg
  teaser: /assets/images/posts/kubecon-na-2023-teaser.jpg
---


Die KubeCon North America 2023 fand im November in Chicago statt. Einen Tag vor der eigentlichen Konferenz gab es – wie jedes Jahr – verschiedene Co-located Events. Als jemand, der täglich mit dem SUSE Rancher-Ökosystem arbeitet, war für mich der Rancher Day die naheliegende Wahl. Was folgt, sind meine Notizen und Eindrücke aus diesem Workshop.

 > 
 > **Hinweis:** Dieser Post basiert auf dem Stand von November 2023. Versionsnummern und UI-Details können sich seitdem geändert haben.

## Aufbau des Workshops

Der Workshop war in drei Module gegliedert:

|Modul|Thema|
|-----|-----|
|1|Setup & Installation|
|2|Day-2-Operations|
|3|Application Deployment|

Das Ziel war ambitioniert: Innerhalb eines Tages eine vollständige Rancher-Umgebung aufbauen, eine Demo-Applikation deployen und diese mit integrierten Security-Tools absichern. Klingt machbar – war es auch, aber es war ein straffes Programm.

## Modul 1: Setup & Installation

### K3s und Rancher aufsetzen

Als Basis diente K3s, die leichtgewichtige aber vollständig CNCF-zertifizierte Kubernetes-Distribution von Rancher. Sie ist für ressourcenbeschränkte Umgebungen konzipiert, eignet sich aber auch für schnelle Lab-Setups wie diesen Workshop.

Die Installation ist bewusst simpel gehalten:

````bash
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.26.9+k3s1 sh -

# Berechtigungen anpassen
sudo chmod 600 /etc/rancher/k3s/k3s.yaml
sudo chown ec2-user:root /etc/rancher/k3s/k3s.yaml

# KUBECONFIG setzen
echo "export KUBECONFIG=/etc/rancher/k3s/k3s.yaml" >> ~/.bash_profile
source ~/.bash_profile
````

Danach folgte die Installation von Helm und cert-manager. Letzteres ist eine Voraussetzung für Rancher, da es das TLS-Zertifikatsmanagement übernimmt:

````bash
helm repo add jetstack https://charts.jetstack.io

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.11.0 \
  --set installCRDs=true \
  --create-namespace

# Rollout-Status überwachen
kubectl -n cert-manager rollout status deploy/cert-manager
kubectl -n cert-manager rollout status deploy/cert-manager-webhook
````

Rancher selbst wird ebenfalls per Helm installiert:

````bash
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest

helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.example.sslip.io \
  --set replicas=1 \
  --version 2.7.8 \
  --create-namespace
````

Das initiale Bootstrap-Passwort holt man sich anschliessend so:

````bash
kubectl get secret --namespace cattle-system bootstrap-secret \
  -o go-template='{{.data.bootstrapPassword|base64decode}}{{"\n"}}'
````

#### Dynamic Listener – ein häufiger Stolperstein

Eine Kleinigkeit, die mich beim ersten Mal immer wieder erwischt: Rancher nutzt beim ersten Start einen sogenannten "Dynamic Listener" für TLS. Das bedeutet, das Zertifikat wird erst nach Bestätigung der Server-URL korrekt ausgestellt. Während dieser Umschaltung kann die UI kurz merkwürdig reagieren – Seiten die nicht nachladen, Status der ewig auf "Waiting" bleiben. Ein harter Browser-Refresh (Cmd+Shift+R / Ctrl+Shift+R) nach der ersten Anmeldung ist fast immer nötig.

### Downstream-Cluster und RBAC

Im Workshop haben wir zwei Downstream-Cluster registriert – einen für Development, einen für Production. Die Registrierung erfolgt über die "Custom Cluster"-Option in Rancher: Man konfiguriert den Cluster in der UI, erhält einen Registrierungsbefehl und führt diesen auf dem Ziel-Node aus. Wichtig dabei: Public IP und Private IP des Nodes müssen korrekt gesetzt werden.

Das RBAC-Modell von Rancher kennt drei Ebenen:

|Ebene|Beschreibung|Kubernetes-Analogie|
|-----|------------|-------------------|
|Global|Rollen für Rancher selbst (Cluster erstellen, Auth-Einstellungen)|—|
|Cluster|Rollen auf Cluster-Ebene|ClusterRole|
|Project/Namespaces|Rollen innerhalb von Namespaces|Role (namespaced)|

Ein Rancher-Projekt ist dabei eine logische Sammlung mehrerer Namespaces, über die sich Ressourcenlimits, Quotas und Zugriffsrechte zentral verwalten lassen. Im Workshop-Beispiel bekam der `dev`-User im Development-Cluster Owner-Rechte auf das `widget-factory`-Projekt, im Production-Cluster hingegen nur Lesezugriff:

|Cluster|Projekt|Benutzer|Rechte|
|-------|-------|--------|------|
|Development|widget-factory|dev|Owner|
|Production|widget-factory|dev|Read-only|

Ein simples, aber sehr treffendes Beispiel für den Alltag vieler Plattform-Teams.

## Modul 2: Day-2-Operations

### Monitoring mit Prometheus und Grafana

Rancher bringt einen eigenen Helm-Chart für Monitoring mit, der direkt aus der UI unter "Cluster Tools" installiert werden kann. Er deployt einen vollständigen Observability-Stack:

* **Prometheus** für Metriken-Sammlung
* **Grafana** für Dashboards und Visualisierung
* **Alertmanager** für Alerting

Ohne zusätzliche Konfiguration überwacht der Stack sofort CPU, Memory, Netzwerk und alle Kubernetes-Ressourcen. Für den Lab-Betrieb mussten wir die Prometheus-Ressourcen etwas reduzieren:

````yaml
prometheus:
  resources:
    requests:
      cpu: 250m      # Lab: reduziert von 750m
      memory: 250Mi  # Lab: reduziert von 1750Mi
````

In Produktion würde man hier natürlich entsprechend grosszügiger dimensionieren.

### Logging mit dem Logging Operator

Auch das Logging-Setup lässt sich über den Rancher App-Katalog installieren. Im Hintergrund setzt Rancher auf den Logging Operator mit FluentBit und Fluentd:

* **FluentBit** läuft als DaemonSet und sammelt Logs direkt von den Pods
* **Fluentd** aggregiert die gesammelten Logs und leitet sie weiter

Die Konfiguration erfolgt deklarativ über Kubernetes-Custom-Resources (`Output`, `Flow`), was eine saubere GitOps-Integration ermöglicht.

### NeuVector – Container Security aus einer Hand

Das Highlight des Tages war für mich NeuVector. Die Container-Security-Plattform (ebenfalls von SUSE) setzt auf einen verhaltensbasierten Ansatz:

1. **Learning Mode**: NeuVector beobachtet das Kommunikationsverhalten der Applikation – welche Services miteinander reden, welche Protokolle verwendet werden, welche Ports offen sind.
1. **Policy-Generierung**: Aus dem beobachteten Verhalten werden automatisch Netzwerk- und Security-Policies generiert.
1. **Protect Mode**: Die generierten Policies werden durchgesetzt. Abweichendes Verhalten wird geblockt.

Die Installation erfolgt ebenfalls über die Rancher App-Kataloge. Wichtig beim Einsatz auf K3s: Docker Runtime muss abgewählt werden, stattdessen verwendet man den k3s Containerd Runtime Path:

````
/run/k3s/containerd/containerd.sock
````

Ausserdem empfiehlt es sich im Lab, die Replica-Anzahl für Controller und CVE-Scanner auf 1 zu reduzieren:

````yaml
controller:
  replicas: 1
cve:
  replicas: 1
````

### Longhorn für persistenten Storage

Für persistenten Storage kam Longhorn zum Einsatz – ebenfalls ein Projekt aus dem SUSE/Rancher-Ökosystem. Longhorn stellt softwaredefiniertes, verteiltes Block-Storage für Kubernetes bereit und transformiert Standard-Hardware in eine resiliente Storage-Lösung.

Die Installation ist über den App-Katalog unkompliziert. Für den Lab-Betrieb haben wir die StorageClass-Replica-Anzahl auf 1 gesetzt:

````yaml
defaultSettings:
  defaultReplicaCount: 1
````

Nach der Installation erscheint Longhorn als neue StorageClass und kann sofort für PersistentVolumeClaims verwendet werden.

## Modul 3: Application Deployment mit Fleet

### Die Widget Factory

Als Demo-Applikation diente die "Widget Factory" – eine klassische Drei-Tier-Webanwendung:

* **Vue** Frontend
* **Go** Backend / App Server
* **MySQL** Datenbank

Der Code ist öffentlich auf GitHub verfügbar und gut strukturiert:

````
/
  app/          # Vue frontend
  chart/        # Helm chart für Kubernetes
  database/     # DB-Verbindung und Queries
  metrics/      # Prometheus-Metriken
  operations/
    logging/    # Logging Operator Ressourcen
    monitoring/ # ServiceMonitor und Grafana-Dashboards
  pubsub/       # WebSocket-Logik für Live-Updates
  web/          # API- und Web-Handler
````

### Deployment via Fleet

Interessanter als die App selbst war der Deployment-Ansatz: Statt `kubectl apply` oder direktem Helm-Deployment wurde Fleet verwendet – Ranchers integrierte GitOps-Engine.

Fleet zieht die Applikationsdefinitionen direkt aus einem Git-Repository und deployed sie deklarativ auf einen oder mehrere Cluster. Wer FluxCD oder ArgoCD kennt, weiss was das Prinzip ist. Der Vorteil von Fleet ist die enge Integration in Rancher und die Möglichkeit, Deployments über **Cluster-Gruppen** zu steuern.

Cluster-Gruppen werden über Label-Selektoren definiert:

````yaml
# Label auf den Clustern
labels:
  factory-type: widget

# Cluster-Gruppe selektiert alle Cluster mit diesem Label
selector:
  matchExpressions:
    - key: factory-type
      operator: In
      values: [widget]
````

Das Resultat: Ein einziger Git-Push reicht, um die Applikation auf alle passenden Cluster gleichzeitig auszurollen.

### Custom Metrics und Logging-Integration

Die Widget Factory bringt bereits Prometheus-Metriken mit. Über zusätzliche Fleet-Paths lassen sich ServiceMonitor und Grafana-Dashboards deployen:

````
/operations/monitoring  # ServiceMonitor + Grafana Dashboard ConfigMap
/operations/logging     # Logging Output, Flow und Syslog-Receiver
````

Nach dem Deployment tauchen in Grafana automatisch fünf vorkonfigurierte Dashboards auf:

* Current Widgets in Database
* Current Orders in Database
* Orders over time
* Widgets over time
* Total Orders by Widget

Das Logging funktioniert über einen FluentBit-Flow, der Logs der Widget-Factory-Pods sammelt und via Fluentd an einen Syslog-Receiver weiterleitet. Für den Workshop wurde ein simpler Syslog-Receiver deployed, der alles auf stdout ausgibt – in Produktion wäre das natürlich Loki, Elasticsearch oder ein zentrales SIEM.

## NeuVector in Aktion: SQL Injection blockieren

Der abschliessende Demo-Teil war der eindrücklichste. Nachdem NeuVector im Learning Mode das vollständige Verhalten der Widget Factory beobachtet hatte (alle CRUD-Operationen, SQL-Queries, WebSocket-Verbindungen), wurden die generierten Policies exportiert und auf den Production-Cluster angewendet.

Der Export erfolgt aus NeuVector heraus: Policy → Groups → alle `nv.widgetfactory`-Gruppen auswählen → "Export Group Policy" im Protect Mode. Das Resultat ist ein YAML-File mit CRD-Ressourcen, das direkt per `kubectl apply` auf den Zielcluster eingespielt werden kann.

Anschliessend wurde ein klassischer SQL-Injection-Versuch getestet:

````
-1" or 1 order by id desc --
````

Die Anfrage wurde von NeuVector erkannt und geblockt, noch bevor sie die Datenbank erreichte. Im Dashboard erschien das Security Event:

````
Typ:       SQL.Injection
Severity:  Critical
````

Zusätzlich stand ein vollständiger Packet Capture der Anfrage zur Verfügung – nützlich für Forensik und Incident Response.

## Fazit

Der Rancher Day hat gezeigt, was das SUSE-Ökosystem als Ganzes leisten kann. Der Stack umfasst:

* Kubernetes mit K3s / RKE2
* Multi-Cluster-Management mit Rancher
* GitOps mit Fleet
* Monitoring mit Prometheus/Grafana
* Logging mit dem Logging Operator (FluentBit/Fluentd)
* Distributed Storage mit Longhorn
* Container Security mit NeuVector

Alles aus einer Hand, alles über eine einzige Oberfläche verwaltbar. Für Umgebungen, in denen ein konsistentes, integriertes Stack wichtiger ist als maximale Flexibilität bei der Tool-Wahl, ist das ein überzeugendes Argument.

Was mich persönlich am meisten beeindruckt hat, war NeuVector. Die Idee, Sicherheitspolicies nicht manuell zu schreiben, sondern aus beobachtetem Verhalten zu generieren, löst ein echtes Problem: In der Praxis fehlt oft das genaue Wissen darüber, welche Services wirklich miteinander kommunizieren müssen. NeuVector beantwortet genau diese Frage – automatisch.
