# Le Besoin:

- Nexus Repository Manager est un outil de gestion de référentiels qui permet de stocker et de distribuer des artefacts logiciels tels que des bibliothèques, des dépendances et des packages. Il est utilisé pour faciliter la gestion des dépendances dans les projets de développement logiciel, en fournissant un emplacement centralisé pour stocker et partager les artefacts.
- Nexus Repository Manager prend en charge plusieurs formats de référentiels, tels que Maven, npm, Docker, et bien d'autres. Il permet aux équipes de développement de publier, de partager et de gérer les artefacts de manière efficace, tout en assurant la sécurité et la traçabilité des artefacts. En utilisant Nexus Repository Manager, les équipes peuvent améliorer la collaboration, réduire les temps de construction et garantir la cohérence des dépendances dans leurs projets logiciels.

# Les Concepts:

- **Repository**: Un référentiel est un emplacement de stockage pour les artefacts logiciels.
- **Type de Repository**: Il existe différents 3 types de repositories dans Nexus Repository Manager:
    - **Hosted Repository**: Un référentiel hébergé est un référentiel qui est géré et maintenu par Nexus Repository Manager. Il peut être utilisé pour stocker des artefacts internes ou des artefacts tiers.
    - **Proxy Repository**: Un référentiel proxy est un référentiel qui agit comme un intermédiaire entre Nexus Repository Manager et un référentiel distant. Il permet de mettre en cache les artefacts du référentiel distant pour une utilisation ultérieure.
    - **Group Repository**: Un référentiel de groupe est un référentiel qui regroupe plusieurs référentiels en un seul point d'accès. Il permet de simplifier la gestion des dépendances en regroupant plusieurs référentiels sous une seule URL.
- **Group**: Un groupe est une collection de référentiels qui peuvent être utilisés pour regrouper des artefacts similaires.
- **Artifact**: Un artefact est un fichier binaire ou un package qui est stocké dans un référentiel. Il peut s'agir de bibliothèques.
- **Component**: Un composant est une unité de travail dans Nexus Repository Manager. Il peut être un artefact ou un ensemble d'artefacts liés.
  
# Installation de Nexus:

- Télécharger Nexus Repository Manager https://help.sonatype.com/en/download.html
- Décompresser le fichier téléchargé et exécuter le script `nexus`.
- executer le script ./nexus start pour démarrer le serveur Nexus et apres ./nexus run pour le garder en marche.
- NOTE: si tu est en train d'utilise nexus dans une machine virtuelle est vous avez un erreur de manque de ram , vous pouvez changer combien du RAM java va allocer dans le fichier `nexus.vmoptions`
bash '''
    xmx1024m -> xmx512m (512m de RAM pour Nexus)
    xms1024m -> xms512m (512m de RAM pour Nexus)
'''

# Sources Utilies:
- https://dev.to/jkosla/a-complete-guide-to-setting-up-nexus-2-ways-how-to-connect-nexus-to-jenkins-34c9
- https://medium.com/@Raghvendra_Tyagi/all-about-nexus-and-how-to-setup-nexus-sonatype-repository-e67548bf8356
- stackoverflow.com/questions/61105368/how-to-use-github-personal-access-token-in-jenkins
- https://medium.com/@developerwakeling/setting-up-github-webhooks-jenkins-and-ngrok-for-local-development-f4b2c1ab5b6
