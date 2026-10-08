# Démo Jev : le jugement rapide

Jev (`typesafe/jev`, sur Cloudflare Workers AI) ne rédige pas : il lit un objet (une ligne de log, un ticket, un mail) et répond à des questions typées, en un seul passage, avec des probabilités calibrées. Le code décide ensuite à partir de ces nombres. La démo le montre en six scènes, dans un terminal.

**Le message à faire passer : Jev décrit, le code décide, l'humain tranche l'entre-deux.**

## Les six scènes

| # | Scène | Ce qu'on montre |
|---|---|---|
| 1 | **noul** | Une ligne de log Kafka, une question oui/non. Jev rend P(oui). La même question retournée donne environ 1 − p : les critères écrits des deux côtés rendent la réponse stable. |
| 2 | **choice** | Router un ticket de support entre quatre files, dont « aucune ». Un ticket net part seul (confiance ≥ 0,80), un ticket ambigu va à un humain. Le seuil est dans le code, pas dans le modèle. |
| 3 | **score** | La frustration exprimée dans un message, sur une échelle à trois niveaux. Le score dit où ; la distribution dit avec quelle netteté. |
| 4 | **appel combiné** | Un mail écrit par un agent, avant envoi : trois questions (promesse ? sujet ? risque ?) en un seul appel. C'est le motif à vendre. |
| 5 | **sup7** | Le garde-fou en vrai : trois actions d'un agent (lire un fichier du projet, `rm -rf ~/.ssh`, envoyer des identifiants par mail) jugées par les questions et le code de sup7. Verdicts : approve, deny, deny. |
| 6 | **en direct** | Quelqu'un de la salle tape un ticket ; choice, noul et score partent en un appel. Il faut le réseau et les clés. |

Chaque scène affiche la requête envoyée, la réponse brute puis la lecture (barres de probabilité, verdict). Le texte gris en tête et en fin de scène est ce qu'il faut dire.

## Jouer

Python 3.10 ou plus.

```bash
cd jev
python demo.py              # rejoue l'enregistrement du 7/10 : ni réseau, ni clé
python demo.py 2            # une seule scène (1 à 6)
python demo.py 1 2 3        # plusieurs scènes, dans l'ordre donné
python demo.py --auto       # sans pause, pour vérifier que tout s'affiche
```

**Entrée** fait avancer : requête, puis réponse, puis lecture. En scène 6, une ligne vide termine.

Le mode rejeu est le mode par défaut, et le plus sûr devant une salle. La réponse est marquée « enregistré le 7/10 ». En direct, elle est marquée « EN DIRECT », avec le modèle et la consommation de jetons comme preuve.

### En direct

Il faut un compte Cloudflare avec Workers AI et un jeton API qui l'autorise. Les clés passent par l'environnement, jamais par un fichier du dépôt :

```bash
pip install -r requirements.txt
export CLOUDFLARE_ACCOUNT_ID=...
export CLOUDFLARE_WORKERS_AI_TOKEN=...

python demo.py --live                         # appelle Jev ; si un appel échoue, rejoue l'enregistrement
python demo.py --ticket "Mon prélèvement est passé deux fois"   # un ticket, un appel, sans scènes
```

**sup7 n'est pas nécessaire.** Sans lui, la scène 5 rejoue l'enregistrement, même avec `--live` ; les scènes 1 à 4 et 6 partent en direct. Pour jouer aussi la scène 5 en direct, cloner son dépôt public (`github.com/KTCrisis/flux7-supervisor`) dans `~/flux7-supervisor`, ou ailleurs en exportant `SUP7_SRC=/chemin/vers/flux7-supervisor/src`.

`python demo.py --record` appelle Jev pour les scènes 1 à 5 et **réécrit `recorded.json`**. Ne pas committer un enregistrement refait sans l'avoir relu.

## Bon à savoir

- **Ce n'est pas la config de prod de sup7.** La scène 5 utilise les seuils par défaut du code : injection > 0,5, escalade ; danger ≥ 0,9 hors périmètre, refus ; tous les dangers ≤ 0,2 dans le périmètre, approbation ; sinon, escalade. En production, les seuils sont réglés par configuration.
- **La confiance d'un `choice`** vaut (N·p_max − 1)/(N − 1), N étant le nombre d'options : 0 quand tout est à égalité, 1 quand une option prend tout. Pour un `score`, la formule n'est pas publiée. Retenir seulement qu'une distribution étalée donne une confiance basse.
- **Les durées affichées** sont celles de l'appel réseau depuis la France vers Cloudflare, pas le temps du modèle seul.
- La scène 6 ne fonctionne qu'en direct : prévoir le réseau, ou la sauter.
