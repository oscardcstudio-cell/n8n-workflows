# n8n-workflows

Huit workflows n8n signés Oscar de Canecaude, conçus avec Claude Code : publier et suivre un post, produire des images et des vidéos, trier les demandes d'un formulaire de contact. Ils s'adressent aux clients qui cherchent un prestataire en automatisation n8n et agents IA, et aux organismes de formation.

Chaque workflow qui attend une saisie ou une décision vient avec ses pages d'interface : un formulaire pour le lancer, une page de validation pour trancher. Une équipe s'en sert depuis le navigateur, sans ouvrir n8n.

Autres outils : [odcstudio.fr/outils-ia](https://www.odcstudio.fr/outils-ia)
Contact : oscardcstudio@gmail.com

## Les trois familles

| Famille | Ce qu'elle couvre | Workflows |
|---|---|---|
| **W1** | Publication et suivi d'un post | W1a publication · W1b boost Meta Ads · W1c relevé quotidien · W1d rapport de fin de campagne |
| **W2** | Atelier images et vidéo | W2a images · W2b vidéo · W2c historique et export |
| **W3** | Tri des demandes entrantes | W3 tri des demandes |

## Comment ils ont été testés

Les huit fichiers contiennent de vrais nœuds d'API : Instagram Graph (en deux temps), LinkedIn, TikTok, Meta Ads, OpenRouter (image et vidéo), Google Drive, Gmail, Telegram, Postgres, Supabase. Aucune clé ni aucun identifiant de credential ne figure dans les exports.

Les huit ont tourné en local dans n8n 2.42, sur une copie de test. Dans cette copie, chaque appel qui publie, envoie ou dépense était remplacé par une réponse factice. Aucune publication, aucun mail, aucune dépense publicitaire et aucune génération payante n'a donc eu lieu. Seuls deux appels étaient réels : un modèle de langage gratuit sur OpenRouter, et la lecture du catalogue des modèles vidéo d'OpenRouter, qui est gratuite.

Les workflows reposent sur les vraies API, mais aucun n'a été exécuté contre un vrai compte Instagram, LinkedIn, TikTok, Meta Ads, Gmail ou Drive. Aucun n'a servi à un client.

Le principe commun : un workflow qui publie, envoie un mail ou dépense s'arrête d'abord sur une page où un humain valide. Les garde-fous propres à chaque workflow sont listés dans sa fiche.

---

## W1 · Publication et suivi d'un post

Cette famille reproduit en n8n la chaîne de communication de Mecene, un outil de subventions d'Oscar : de la légende au bilan chiffré de la campagne. L'estimation du temps gagné tourne autour de 15 à 25 minutes par post.

### W1a · Publication d'un post

**Problème.** Légende, relecture, publications sur plusieurs réseaux et lien en bio passent par plusieurs outils et laissent des oublis.

1. Un formulaire lance la publication d'un post de la frise.
2. La légende existante est reprise, et l'IA n'en écrit une que si le fichier manque.
3. Les règles du système d'origine relisent la légende.
4. Un humain valide la légende, puis confirme « publier ».
5. Instagram reçoit le post en deux temps (containers, puis media_publish), en carrousel ou en reel. LinkedIn reçoit un document PDF. TikTok reçoit un brouillon.
6. Le lien en bio bascule et une alerte Telegram part.

**Garde-fous**
- Une légende existante n'est jamais écrasée par une légende de l'IA.
- Rien ne part sans la validation de la légende, puis la confirmation « publier ».

**Credentials** : Meta Graph API (token), LinkedIn OAuth2 (page), TikTok (Custom Auth), Supabase, Postgres, Telegram, OpenRouter.

![Formulaire de lancement](captures/W1a-formulaire.png)
![Page de validation de la légende](captures/W1a-validation-legende.png)
![Page de confirmation](captures/W1a-confirmation.png)

### W1b · Boost d'un post Instagram (Meta Ads)

**Problème.** Booster un post organique passe par plusieurs appels Meta Ads, et une erreur de réglage coûte de l'argent.

1. Un formulaire reçoit le post, le budget, la durée et le ciblage.
2. Un préflight vérifie les droits `ads_management`, le compte publicitaire et l'éligibilité du post.
3. Meta n'ayant pas d'endpoint « boost », le workflow crée une créative adossée au post (`source_instagram_media_id`), puis une annonce.
4. Un humain valide la campagne sur une page dédiée.
5. La diffusion démarre seulement avec le mot « diffuser ». Un webhook l'arrête immédiatement.

**Garde-fous**
- Tout est créé en pause (PAUSED) : la création ne dépense pas un euro.
- Le budget reste entre 5 et 200 €, et la durée entre 1 et 14 jours.
- Les intérêts de ciblage s'écrivent à leur nom exact, et Advantage+ est désactivé.
- La déclaration DSA est portée par le workflow.
- Si une étape échoue, la campagne déjà créée est nettoyée.

**Credentials** : Meta Graph API (token avec `ads_management`), Postgres, Telegram.

![Formulaire de campagne](captures/W1b-formulaire.png)
![Page de validation de la campagne](captures/W1b-validation-campagne.png)
![Lancement](captures/W1b-lancement.png)

### W1c · Relevé quotidien

**Problème.** Les chiffres des posts, des comptes et des annonces vivent dans trois outils différents.

1. Chaque matin à 07:00, ou sur appel de webhook, le workflow démarre.
2. Il relève les métriques des posts Instagram (reach, enregistrements, partages, vues) et un instantané du compte.
3. Il relève les vidéos et le compte TikTok, et bascule le lien en bio sur la carte de la dernière vidéo.
4. Il relève la dépense Meta Ads par annonce et par jour, avec 7 jours de recouvrement.
5. Il rend un verdict par post et écrit le tout dans les tables Postgres du système d'origine.

**Garde-fous**
- Une métrique refusée pour un type de média vaut NULL, jamais 0.
- Le workflow refuse de juger un post sous 30 visites.
- Les erreurs sont collectées, jamais avalées : une alerte Telegram part si un relevé est incomplet.

**Credentials** : Meta Graph API (token `ads_read`), TikTok (Custom Auth), Postgres, Telegram.

![Canevas du workflow W1c](captures/W1c-canevas.png)

### W1d · Rapport de fin de campagne

**Problème.** Le bilan d'une campagne dépend de quelqu'un qui remarque qu'elle vient de se terminer.

1. Toutes les heures, le workflow regarde chaque card en diffusion et vérifie si son ad set a atteint `end_time`.
2. Si oui, il pose un verrou en base, avec l'id de l'annonce pour clé.
3. Un bilan part ensuite sur Telegram : dépense Meta, impressions, clics, pages vues et clics sur le site, dossiers complétés, coût par dossier.

**Garde-fous**
- Le verrou garantit une seule alerte, même si deux exécutions se croisent.
- Le verrou reste posé même si Telegram échoue : le bilan ne part jamais en double.

**Credentials** : Meta Graph API (`ads_read`), Postgres, Telegram.
![Canevas du workflow W1d](captures/W1d-canevas.png)

---

## W2 · Atelier images et vidéo

Cette famille reproduit en n8n video-studio, un outil de génération d'images d'Oscar. L'estimation du temps gagné tourne autour de 10 à 15 minutes par série de 4 images.

### W2a · Atelier d'images

**Problème.** Garder le même style sur plusieurs images, et connaître le coût avant de générer.

1. Un formulaire reçoit le projet et le preset choisis.
2. Le workflow assemble la bible du dépôt et celle du projet (la seconde complète la première), puis ajoute le préfixe et le suffixe du preset.
3. Il joint des ancres de style en référence (6 au maximum, avec un rôle : style, composition, both ou subject).
4. L'amélioration du prompt par l'IA est optionnelle.
5. Il estime le coût avant toute dépense, et un humain valide.
6. OpenRouter génère l'image, le coût réel est lu sur la réponse, puis l'image est archivée dans le dossier du projet avec sa fiche de génération.

Le mode édition propose 10 outils (zoom out, détourage, relight...). L'IA rédige l'instruction, un humain la relit, et la nouvelle version se rattache à sa source (`parentId`).

**Garde-fous**
- Le coût s'affiche avant la génération.
- Rien ne se génère sans validation humaine.
- Une instruction d'édition rédigée par l'IA est relue avant exécution.

**Credential** : OpenRouter.

![Formulaire de génération](captures/W2a-formulaire.png)
![Page de validation avec le coût](captures/W2a-validation.png)
![Formulaire d'édition](captures/W2a-edition-formulaire.png)
![Validation de l'édition](captures/W2a-edition-validation.png)

### W2b · Génération vidéo

**Problème.** Un ratio non supporté par le modèle est ignoré en silence, et la vidéo est facturée quand même.

1. Un formulaire reçoit le modèle et les réglages voulus.
2. Le workflow relit en direct les capacités (ratios, durées, résolutions) et les tarifs du modèle sur `GET /api/v1/videos/models`.
3. Il estime le coût avec la formule du fournisseur, qui compte trois familles de tarifs (jetons vidéo, secondes, centimes par seconde).
4. Il écrit une fiche « pending », soumet la demande par `POST /api/v1/videos`, puis sonde toutes les 5 secondes, 15 minutes au plus.
5. Il télécharge le rendu avec authentification (5 essais), l'archive et lit le coût réel sur le rendu terminé.

**Garde-fous**
- Un ratio non supporté est refusé avant de payer.
- Le coût s'affiche avant la génération. Un coût inconnu s'affiche « — », jamais 0 $.
- Un refus du fournisseur (copyright audio, modération) revient sous forme d'un message qui dit quoi corriger.

**Credential** : OpenRouter.

![Formulaire vidéo](captures/W2b-formulaire.png)
![Page de validation avec le coût](captures/W2b-validation.png)

### W2c · Historique, coûts et export

**Problème.** Retrouver les versions d'un projet et ce qu'il a coûté oblige à fouiller les fiches une à une.

1. Un formulaire désigne le projet.
2. Le workflow lit les fiches de génération et reconstruit la chaîne de versions : image source, éditions, vidéos rattachées par `parentId`.
3. Il additionne les coûts réels.
4. Il écrit `historique.md` et `manifest.json` dans le dossier d'export.
5. En option, il envoie tous les fichiers du projet sur Google Drive.

**Garde-fous**
- Un coût inconnu s'affiche « — », jamais 0 $.
- L'envoi sur Google Drive est une option, pas une étape obligatoire.

**Credential** : Google Drive OAuth2 (pour l'envoi).

![Formulaire d'export](captures/W2c-formulaire.png)

---

## W3 · Tri des demandes entrantes

**Problème.** Le formulaire de contact reçoit devis, support, partenariats et spam, et chaque message demande un tri puis une réponse.

1. Un webhook reçoit le message du formulaire de contact.
2. L'IA classe la demande (devis, support, partenariat, spam) et prépare un brouillon.
3. Le relecteur reçoit par Gmail une page de validation, relit le brouillon et le modifie si besoin.
4. Il envoie la réponse ou choisit de ne pas répondre. L'envoi passe par un vrai nœud Gmail.
5. Chaque demande est journalisée.

L'estimation du temps gagné tourne autour de 3 à 5 minutes par demande.

**Garde-fous**
- Aucune réponse ne part sans que le relecteur l'ait validée.
- Le spam est classé sans réponse.

**Credentials** : Gmail OAuth2, OpenRouter. Valeur à remplacer dans le nœud « Réglages » : `relecteur_email`.

![Canevas du workflow](captures/W3-canevas.png)
![Page de validation humaine](captures/W3-validation-humaine.png)

---

## Importer dans n8n

1. Importer les fichiers `.json` voulus. Les exports ne contiennent aucune clé.
2. Créer les credentials indiqués dans la fiche de chaque workflow, puis les brancher sur les nœuds concernés.
3. Ouvrir le nœud « Réglages » de chaque workflow et remplacer les valeurs `A_REMPLACER_...` (identifiants de compte, URL).
4. Définir la variable `SORTIES_DIR`, le dossier d'écriture des fichiers.
5. Pour W1a à W1d, adapter les tables Postgres : elles viennent du système d'origine.

Aucun de ces workflows n'a encore servi à un client.
