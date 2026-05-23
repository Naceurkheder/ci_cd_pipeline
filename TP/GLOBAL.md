<img width="914" height="438" alt="WhatsApp Image 2026-05-23 at 01 12 46" src="https://github.com/user-attachments/assets/3b9e7634-1c5e-4973-8b2b-5ddc247eb216" />
C'est le schéma complet du pipeline que le prof a conçu. Il résout un problème précis : Jenkins est dans un réseau privé (VMs locales), mais GitHub est sur internet public — comment faire communiquer les deux ?

Le problème central — "The Challenge"
GitHub a besoin d'envoyer un webhook à Jenkins à chaque git push. Mais Jenkins tourne sur une VM locale inaccessible depuis internet. C'est là qu'intervient Ngrok.

Les 8 étapes du flux
ÉtapeAction① Git PushLe développeur pousse du code sur GitHub② WebhookGitHub envoie une notification à l'URL publique Ngrok③ Webhook Triggered & RoutedNgrok reçoit la requête et la redirige via un tunnel sécurisé vers Jenkins④ Git PullJenkins récupère le code source depuis GitHub⑤ BuildJenkins compile et construit l'application⑥ Sonar AnalysisJenkins envoie le code à SonarQube pour analyse qualité⑦ Analysis ResultsSonarQube renvoie le verdict (Quality Gate OK ou ERROR)⑧ Artifact UploadSi tout est OK, Jenkins publie l'artefact sur Nexus

Les deux zones réseau

Public Internet — GitHub + Ngrok Cloud : accessibles depuis partout
Private Local Network — 3 VMs Vagrant isolées :

Jenkins VM : orchestrateur du pipeline
SonarQube VM : analyse statique du code
Nexus VM : stockage des artefacts




Le rôle clé de Ngrok
Ngrok crée un tunnel sortant (outbound) depuis la VM Jenkins vers son cloud public. Il génère une URL publique temporaire comme https://xyz.ngrok-free.app/github-webhook/ que GitHub peut joindre. Jenkins n'a pas besoin d'être exposé directement sur internet.
