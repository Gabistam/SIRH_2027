# Exemple 2 — Mon premier e-mail automatique avec n8n (prise en main)

Un workflow de **3 nœuds** : on saisit un prénom, une adresse e-mail et un poste dans un formulaire,
et n8n envoie automatiquement un message de bienvenue.

```
[1. Formulaire]  →  [2. Préparer le message]  →  [3. Envoyer l'e-mail]
```

**Ce que les débutants apprennent**

| Notion | Où la voir |
|---|---|
| Un **déclencheur** démarre le workflow | Nœud 1 |
| Une **expression** `{{ ... }}` insère une donnée dans un texte | Nœud 2 (`Bonjour {{ $json['Prénom'] }}`) |
| Un **credential** = les identifiants d'un service, saisis une seule fois | Nœud 3 |
| Une **action** fait quelque chose dans le monde réel | Nœud 3 |
| Un destinataire (**À**) et une copie (**Cc**) | Nœud 3 |

Les e-mails ne partent pas sur internet : ils arrivent dans une **boîte mail factice en local**.
Pas de compte Gmail, pas de risque d'écrire à une vraie personne.

---

## Étape 0 — Préparer la boîte mail factice (Mailpit)

Mailpit reçoit les e-mails envoyés par n8n et les affiche dans le navigateur.

```bash
docker run -d --rm --name mailpit -p 1025:1025 -p 8025:8025 axllent/mailpit
```

- Port **1025** : le serveur SMTP auquel n8n se connecte.
- Port **8025** : la boîte de réception, à ouvrir sur <http://localhost:8025>.

Pour l'arrêter : `docker stop mailpit` (le conteneur disparaît tout seul grâce à `--rm`).

> Cette commande vient de la documentation de Mailpit ; je ne l'ai pas exécutée moi-même
> (voir « Ce qui a été vérifié »).

## Étape 1 — Importer le workflow

Comme pour l'exemple 1 : **Create workflow** → menu **⋯** → **Import from file…** →
`habiteo-accueil-email.json`.

Le nœud **3. Envoyer l'e-mail** affiche un avertissement : *il manque les identifiants*. C'est normal, c'est l'étape suivante.

## Étape 2 — Créer le credential SMTP (une seule fois)

1. Double-cliquer sur le nœud **3. Envoyer l'e-mail**.
2. Dans *Credential to connect with* → **Create new credential** → type **SMTP**.
3. Remplir :

| Champ | Valeur |
|---|---|
| User | *(vide)* |
| Password | *(vide)* |
| Host | `localhost` |
| Port | `1025` |
| SSL/TLS | **désactivé** |

4. **Save**, puis donner un nom, par exemple « Mailpit local ».

> Si `localhost` ne fonctionne pas sur votre poste, essayer `127.0.0.1`.

## Étape 3 — Tester

1. **Execute workflow** : le formulaire de test s'ouvre dans un nouvel onglet.
2. Saisir un prénom, une adresse (par exemple `camille.durand@exemple.fr`) et un poste, puis envoyer.
3. Ouvrir <http://localhost:8025> : l'e-mail est là.

**Ce que vous devez voir**

- **De** : Service RH Habitéo
- **À** : l'adresse saisie dans le formulaire
- **Cc** : `rh@habiteo.example`
- **Objet** : `Bienvenue chez Habitéo, Camille !`
- **Texte** : `Bonjour Camille, … au poste de <poste> …`

Dans n8n, cliquer sur le nœud 2 : on voit en sortie les trois champs `email`, `objet` et `message`
que le nœud 3 va utiliser.

---

## Comprendre chaque nœud

**1. Formulaire** — Le champ « Adresse e-mail » est de type *email* : le navigateur refuse une saisie qui n'a pas la forme d'une adresse.

**2. Préparer le message** — Il compose l'objet et le texte en insérant le prénom et le poste. On sépare ainsi
*fabriquer le message* et *l'envoyer* : on peut vérifier le texte avant tout envoi.

**3. Envoyer l'e-mail** — Les champs *To*, *Subject* et *Text* contiennent des expressions
(`{{ $json.email }}`, `{{ $json.objet }}`, `{{ $json.message }}`) qui récupèrent ce que le nœud 2 a préparé.
Le service RH est en **Cc** (menu *Options*).

## Un piège à connaître

Depuis la version 2.1 du nœud, n8n ajoute par défaut à chaque message la phrase
*« This email was sent automatically with n8n »* suivie d'un lien. Pour un e-mail RH ce n'est pas souhaitable :
dans le fichier fourni, cette mention est **désactivée** (menu *Options* du nœud 3).

## Passer à une vraie messagerie (à faire avec précaution)

Il suffit de créer un autre credential SMTP avec les paramètres de votre fournisseur (hôte, port, identifiants).
Avant tout envoi réel :
- **tester d'abord avec votre propre adresse** ;
- ne jamais mettre vos identifiants dans le fichier JSON (le credential reste dans n8n) ;
- rappeler que des adresses e-mail sont des **données personnelles** (RGPD) : à qui écrit-on, pourquoi, et qui peut lire le journal des exécutions ?

---

## Ce qui a été vérifié

Le workflow a été importé et exécuté sur une instance n8n 2.10.4 isolée, en soumettant le formulaire par son URL de production.
Le nœud d'envoi a transmis l'e-mail à un **serveur SMTP de test minimal** (écrit pour l'occasion, port 1025, sans authentification) :
destinataire en **À**, `rh@habiteo.example` en **Cc**, objet et corps avec accents corrects (« Hélène », « Électricien »),
et aucune mention d'attribution n8n. Le credential était alors sans identifiants, comme indiqué à l'étape 2.

**Non vérifié :** l'image Docker Mailpit et son interface web (<http://localhost:8025>) n'ont pas été lancées lors de mes tests,
ni l'interface graphique de n8n (libellés de menus d'après la documentation), ni le message affiché dans le navigateur après l'envoi
du formulaire (« Formulaire reçu : le message de bienvenue est en cours d'envoi. »).

## Pour aller plus loin

1. Ajouter au formulaire un champ **Date d'arrivée** et l'inclure dans le message.
2. Passer le message en **HTML** (*Email Format* → HTML) avec un peu de mise en forme.
3. Changer le destinataire en **Cci** (*BCC*) plutôt qu'en **Cc**, et constater la différence dans Mailpit.
4. **Combiner avec l'exemple 1** : envoyer un e-mail d'alerte au service RH uniquement sur la branche **URGENT**
   (brancher un nœud *Send Email* derrière « 4a. Alerte RH »).
