# Chokepoint

**Détecter les fuites de données au moment où elles quittent l'entreprise, quel que soit le chemin emprunté.**

Chokepoint est une solution de prévention des fuites de données (DLP) qui contrôle les fichiers sortant d'une organisation par le web, le mail et les supports amovibles. Plutôt que de s'appuyer sur de simples listes de mots-clés, elle analyse le contenu réel des fichiers, puis applique les règles définies par l'organisation.

> Projet réalisé dans le cadre du Master Cybersécurité de l'ISEN Méditerranée (Toulon).
> L'architecture décrite ci-dessous est une proposition, en attente de validation par l'encadrant.

---

## Le problème

La plupart des fuites de données ne sont pas des attaques sophistiquées. Ce sont des erreurs du quotidien : une mauvaise pièce jointe, un fichier déposé sur le mauvais site, une base clients copiée sur une clé personnelle.

Les outils DLP classiques tentent de les repérer avec des règles fixes et des listes de mots-clés. Le résultat est connu :

- des fuites réelles passent inaperçues lorsqu'aucun mot-clé n'apparaît ;
- des fichiers parfaitement légitimes sont bloqués, ce qui pousse les utilisateurs à contourner l'outil.

Chokepoint cherche à bloquer les vraies fuites sans gêner le travail normal des utilisateurs.

## Notre approche

Chaque fichier qui tente de sortir est retenu le temps d'une courte vérification :

1. **Interception** — le fichier est capté sur l'un des trois canaux : web, mail ou USB.
2. **Extraction** — le texte est extrait du document (PDF, Word, Excel, archives…).
3. **Analyse** — des règles simples repèrent les données évidentes (IBAN, numéros de carte…), puis un modèle de classification détermine la nature du contenu et un niveau de confiance.
4. **Décision** — le moteur de politiques tient compte du contenu et du contexte (expéditeur, destination, canal) pour **autoriser**, **avertir** ou **bloquer**.
5. **Traçabilité** — chaque décision est journalisée, et les événements sensibles remontent dans un tableau de bord.

L'analyse de contenu s'appuie sur un modèle de classification rapide, capable de répondre en quelques millisecondes, afin de décider en temps réel sans ralentir les utilisateurs.

## Architecture (proposition)

```
 Poste utilisateur                Serveur Chokepoint                    Destinations
┌──────────────────┐         ┌──────────────────────────┐         ┌──────────────────┐
│ Client mail ─────┼──SMTP──▶│ Relais mail              │         │                  │
│ Navigateur ──────┼──HTTP──▶│ Proxy web                │────────▶│ Serveur mail     │
│ Agent USB ───────┼──API───▶│ API agents               │────────▶│ Sites web        │
│      │           │         │          │               │         │                  │
│   Clé USB        │         │ Extraction → Analyse ────┼──▶ modèle de classification
└──────────────────┘         │          │               │
                             │ Moteur de politiques     │
                             │          │               │
                             │ Journal + tableau de bord│
                             └──────────────────────────┘
```

Le schéma détaillé est disponible dans [`docs/architecture.drawio`](docs/architecture.drawio).

## Composants

| Composant | Rôle |
|---|---|
| **Proxy web** | Intercepte les uploads HTTP/HTTPS et bloque les transferts non autorisés |
| **Relais mail** | Inspecte les mails sortants et leurs pièces jointes avant leur envoi |
| **Agent USB** | Surveille les copies vers les supports amovibles, y compris hors connexion |
| **Analyse** | Extrait le contenu, applique les règles de détection et interroge le modèle de classification |
| **Moteur de politiques** | Évalue les règles de l'organisation, écrites sous forme de fichiers versionnés |
| **Plateforme** | Journalisation des décisions, alertes et tableau de bord |

L'ensemble est développé en **Rust**, pour la performance et la sûreté mémoire attendues d'un composant placé sur le chemin des données.

## Exemple de politique

```yaml
- nom: salaires-vers-exterieur
  si:
    categorie: salaires
    confiance_min: 0.9
    destination: exterieur
  alors: bloquer

- nom: donnees-clients-sur-usb
  si:
    categorie: clients
    canal: usb
  alors: avertir
```

## Structure du dépôt

```
chokepoint/
├── crates/
│   ├── proxy-web/       # Interception web
│   ├── relais-mail/     # Interception mail
│   ├── agent-usb/       # Agent installé sur les postes
│   ├── analyse/         # Extraction, détection, client de classification
│   ├── politiques/      # Moteur de règles
│   └── plateforme/      # Journal et tableau de bord
├── policies/            # Règles d'exemple
├── lab/                 # Mise en place de l'environnement de test
├── tests/               # Jeux de données fictifs et tests d'intégration
└── docs/                # Architecture et documentation technique
```

## Environnement de test

Le projet est validé dans un laboratoire virtualisé composé de trois machines :

| Machine | Rôle |
|---|---|
| **Poste utilisateur** | Client mail, navigateur et agent USB |
| **Serveur Chokepoint** | Interception, analyse et décision |
| **Extérieur** | Destinataires simulés : serveur mail de test et site d'upload |

Le poste utilisateur n'a aucun accès direct à l'extérieur : tout le trafic transite par le serveur Chokepoint. Les jeux de données utilisés sont entièrement fictifs.

Le guide d'installation sera disponible dans [`lab/`](lab/).

## Feuille de route

- [ ] Environnement de test et intégration continue
- [ ] Proxy web et relais mail en mode transparent
- [ ] Extraction de contenu et règles de détection simples
- [ ] Intégration du modèle de classification et moteur de politiques
- [ ] Agent USB avec mode hors connexion
- [ ] Support HTTPS
- [ ] Tableau de bord et alertes
- [ ] Évaluation : taux de détection, faux positifs, latence

## Méthode

Le projet suit une démarche **DevSecOps** : développement en sprints, revue de code, intégration continue et analyse de sécurité automatisée (SAST et SCA) à chaque modification.

## Équipe

| Membre | Périmètre |
|---|---|
| — | Interception web |
| — | Interception mail |
| — | Agent USB |
| — | Analyse et politiques |
| — | Plateforme et intégration continue |
