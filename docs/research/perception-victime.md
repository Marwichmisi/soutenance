# [Recherche] Perception victime multimodale low-cost avec YOLO open-source

> Ticket : issue #3 (part of #1) — « Quelle perception victime multimodale low-cost avec YOLO open-source ? »
> Date : 2026-11-06 · Auteur : sous-agent de recherche AFK · Statut : recommandation, à valider par le binôme
> Contraintes rappelées : pas de Raspberry, budget serré · ESP32-CAM capture → PC portable déporté infère en WiFi ·
> capteurs PIR + ultrasons HC-SR04 (+ option MLX90614 si pas cher) · victime simulée : poupée/mannequin + bouteille
> chaude + QR code / tissu rouge · exigence Q9 : YOLO open-source selon le use case (pas une simple détection
> couleur) · 4 mois, binôme 10 h/sem, salle de démo 4×4 m.

## 1. Question posée

Quel pipeline de détection de victime (personne allongée, simulée par poupée/mannequin) choisir avec :
un ESP32-CAM qui ne fait que capturer/streamer, un PC portable qui infère en WiFi, des capteurs binaires
low-cost (PIR, HC-SR04, option MLX90614), et l'obligation d'utiliser un YOLO open-source adapté au use case ?

## 2. Réponse courte (recommandation)

**Pipeline principal : YOLO11n (détection, classe `person`) en export OpenVINO/ONNX sur le PC portable,
imgsz 416–640, + fusion à règles PIR + HC-SR04 + score de confiance, + QR code / tissu rouge uniquement
en fallback de démo — jamais comme détecteur principal (exigence Q9).**

- Modèle exact : `yolo11n.pt` (Ultralytics, AGPL-3.0 / licence entreprise), fine-tuné si le temps le permet
  sur ~200–400 images maison « poupée allongée en salle 4×4 m » ; sinon poids COCO pré-entraînés.
- Alternative de repli si le PC est très faible : `yolov5n.pt` (même API Ultralytics).
- Chaque étage tourne où : ESP32-CAM = capture MJPEG uniquement · PC portable = réception HTTP,
  inférence YOLO, fusion capteurs (via liaison série/WiFi depuis l'ESP32 ou un 2ᵉ ESP32), décision et IHM.
- Latence WiFi estimée en LAN salle 4×4 m : 150–350 ms de bout en bout (détail §6).

## 3. Comparatif modèles (PC CPU, détection personne allongée)

Chiffres constructeur = pages modèles Ultralytics officielles (COCO val, imgsz 640) ; les FPS CPU sont des
**ordres de grandeur** sur CPU portable type i5 (ONNX/OpenVINO) — à re-mesurer sur VOTRE PC avec
`yolo benchmark` (§7), car le FPS réel dépend du CPU, de l'export et de l'imgsz.

| Modèle (variante nano / légère) | Params / FLOPs (source) | mAP COCO 50-95 (source) | CPU portable, ordre de grandeur (640) | CPU portable (416) | RAM / poids | Verdict use case |
|---|---|---|---|---|---|---|
| **YOLO11n** (`yolo11n.pt`) | 2,6 M / 6,5 GFLOPs | **39,5** | ~8–15 FPS | ~15–30 FPS | ~5–10 Mo, ~0,5–1 Go RAM inférence | **Choix principal : meilleur ratio précision/légèreté, API Ultralytics actuelle, export OpenVINO facile** |
| YOLOv8n (`yolov8n.pt`) | 3,2 M / 8,7 GFLOPs | 37,3 | ~7–12 FPS | ~12–25 FPS | ~6–12 Mo, ~0,5–1 Go RAM | Choix principal bis : écosystème le plus documenté, quasi-équivalent ; prendre v8n si un tuto/ints existant l'impose |
| YOLOv5n (`yolov5n.pt`) | 1,9 M / 4,5 GFLOPs | 28,0 | ~15–25 FPS | ~25–45 FPS | ~4 Mo, RAM la plus faible | Repli si PC très faible ou besoin PyTorch ancien ; précision en retrait (~−10 pts mAP vs 11n) |
| YOLO-World (ex. `yolov8s-worldv2`) | ~12,6 M (+ encodeur texte CLIP) | 35,4 AP **LVIS zéro-shot** (papier, sur V100, pas COCO) | ~2–5 FPS CPU + surcoût prompt | déconseillé CPU | plusieurs centaines Mo | **Écarté en principal** : open-vocabulary séduisant (« person lying on the floor » sans ré-entraînement) mais plus lourd, plus lent sur CPU, chaîne de déploiement moins mature ; garder en option de démonstration « sans ré-entraînement » si temps excédentaire |
| MobileNet-SSD (ex. SSDLite-MobileNetV2, TF OD API / ONNX) | ~4–7 M | ~20–23 (selon variante) | ~15–30 FPS | ~25–50 FPS | léger | **Écarté** : plus rapide mais nettement moins précis, tooling d'entraînement vieillissant (TF1/TF2), pas de Q9 « YOLO » ; utile seulement en base de comparaison vitesse |

Lecture du tableau pour notre use case :
- La cible « personne allongée » est un **cas dur** : la classe COCO `person` est dominée par des personnes
  debout ; un corps allongé vu de haut/3-4 m donne une bbox très allongée, souvent coupée, parfois confondue
  avec un objet. D'où l'écart entre mAP COCO et performance réelle salle — prévoir le fine-tune maison (§7).
- YOLO11n domine YOLOv5n de ~11 pts de mAP pour un coût CPU à peine supérieur : à budget FPS égal,
  mieux vaut YOLO11n en imgsz 416 que YOLOv5n en 640.
- YOLO-World répondrait à l'objection « et si la victime ne ressemble pas à une personne COCO ? » par du
  prompting (« a doll lying on floor », « person lying down »), mais son coût CPU et sa dépendance à
  l'encodeur texte le disqualifient sur PC portable modeste en temps réel ; le papier annonce 52 FPS mais
  **sur V100**, pas sur CPU.
- YOLO11n-pose (`yolo11n-pose.pt`) est une variante à garder en tête en phase 2 : les keypoints permettent un
  test « corps horizontal » (épaules-hanches quasi alignées horizontalement) qui détecte « allongé » mieux
  qu'une bbox seule. Coût : annotation keypoints plus lourde — option, pas le plan A.

## 4. Fusion multimodale : PIR + HC-SR04 (+ MLX90614 optionnel) + score YOLO

Principe : YOLO reste le seul détecteur « victime », les capteurs apportent un **faisceau de présomption**
indépendant (mouvement/chaleur/distance) qui monte ou baisse la confiance et filtre les faux positifs.
Règle de fusion simple, explicable en soutenance (pas de boîte noire) :

1. À chaque frame : `c_yolo` = confiance max des bbox `person` (0 si aucune), `pir` ∈ {0,1},
   `d_us` = distance ultrasons (cm, None si hors portée > 400 cm).
2. Score fusionné : `c = c_yolo × (1 + 0,15×pir + 0,15×cohérence_us) × facteur_chaleur`,
   avec `cohérence_us = 1` si `d_us` dans [30, 400] ET une bbox occupe la zone centrale de l'image,
   sinon 0 ; `facteur_chaleur = 1,1` si MLX90614 présent et T > seuil ambiant + 3 °C, sinon 1,0.
3. Décision : `c ≥ 0,5` → « victime probable » ; `0,3 ≤ c < 0,5` → « doute, tangentialiser / zoomer » ;
   `< 0,3` → rien. Les seuils se calibrent sur le dataset maison (§7).
4. Garde-fous : le PIR seul ne déclenche jamais (courants d'air, passage jury) ; l'ultrason seul ne
   déclenche jamais (mur, chaise) ; **YOLO seul sous 0,5 ne déclenche pas l'alerte finale** — il faut au
   moins un capteur convergent. Enregistrer les 3 signaux horodatés pour la démo et le rapport.

Rôle de chaque capteur dans la démo :
- PIR (ex. HC-SR501, ~1–2 €) : détecte un mouvement/une présence chaude dans son cône ; bon marché,
  binaire, portée ~5–7 m suffisante en 4×4 m. Limite : aveugle à une victime parfaitement immobile après
  ~quelques secondes (à dire au jury, c'est pour ça qu'on le couple à YOLO).
- HC-SR04 (~2–3 €) : distance 2–400 cm, précision ~3 mm en théorie, ~±1–2 cm en pratique salle ;
  monté sur servo ou fixe vers la zone suspecte, il confirme « quelque chose à N cm dans l'axe ».
- MLX90614 (~5–10 €, optionnel) : thermomètre IR sans contact ; la « bouteille chaude » simule la chaleur
  corporelle et donne un 3ᵉ signal indépendant de la vision. N'acheter que si le budget le permet —
  le PIR couvre déjà une partie du besoin.
- QR code / tissu rouge : **fallbacks de robustesse démo**, pas des détecteurs : si YOLO décroche
  (contre-jour total), le QR posé sur la poupée ou le tissu rouge (détection HSV seuillée, 20 lignes de
  code) permet de finir la démo. Le rapport et la soutenance doivent présenter YOLO comme le détecteur
  (Q9), ces artifices comme aides de secours assumées.

## 5. Métriques : précision / rappel + robustesse éclairage

À mesurer sur le dataset d'évaluation (§7), jamais seulement sur COCO :
- Détection : **précision, rappel, F1 @ IoU 0,5**, mAP@50, + courbe précision-rappel en faisant varier le
  seuil YOLO (0,25 / 0,5 / 0,7). Cible démo réaliste après fine-tune maison : précision ≥ 0,8 et
  rappel ≥ 0,7 en éclairage normal salle.
- Système fusionné : taux de fausses alertes / heure en salle vide, temps de détection médian (pose de la
  poupée → alerte), matrice de confusion par scénario.
- Robustesse éclairage : protocole à 3 conditions — néons normaux / pénombre (stores baissés) / contre-jour
  (projecteur face caméra). Attendu honnête : YOLO chute en pénombre (bruit ESP32-CAM) ; le PIR et le
  MLX90614 y sont quasi insensibles — c'est l'argument de vente de la fusion. Prévoir dans la démo une
  lampe d'appoint et l'exposition fixe de l'ESP32-CAM plutôt que l'auto-exposition.
- Coût : FPS médian et latence bout-en-bout mesurés (cf. §6), RAM pic du process Python.

## 6. Où tourne chaque étage + latence WiFi (salle 4×4 m, LAN local)

```
[ESP32-CAM ~7 €] --WiFi MJPEG--> [PC portable : capture HTTP -> YOLO11n OpenVINO -> fusion -> IHM]
[ESP32 (CAM ou 2e board) : PIR + HC-SR04 (+MLX90614)] --série USB ou WiFi/MQTT--> [PC : fusion]
```

- ESP32-CAM : **capture uniquement** (firmware CameraWebServer, résolution 640×480 / SVGA, ~10–15 img/s).
  Ne pas inférer sur l'ESP32 : 520 Ko de SRAM et absence de FPU correcte l'interdisent pour YOLO
  (le framework ESP-DL/ESP-WHO ne propose que des micro-modèles, pas YOLO).
- PC portable : `capture.py` (requête HTTP MJPEG) → `infer.py` (Ultralytics `model.predict`, export
  OpenVINO FP32 ou INT8) → `fuse.py` (règle §4) → IHM (affichage bbox + score + états capteurs).
- Budget latence typique en LAN local : capture+JPEG ESP32 ~50–120 ms, transport WiFi ~20–80 ms,
  décodage ~10–20 ms, inférence YOLO11n CPU (416) ~35–70 ms / (640) ~70–130 ms, fusion+affichage ~5–10 ms,
  soit **~150–350 ms bout-en-bout**, ~5–10 alertes/s effectives — largement suffisant pour une victime
  immobile. Mesurer avec `time.time()` autour de chaque étage et afficher le breakdown en soutenance.
- Conseil réseau : routeur/point d'accès dédié ou hotspot du PC (pas le WiFi de l'école), IP fixe ESP32,
 imgsz 416 par défaut, ne monter à 640 que si le rappel est insuffisant.

## 7. Dataset d'évaluation + protocole de test démo

Dataset maison (à construire semaine 2–3, ~300–500 images) :
- Fond : salle 4×4 m réelle de démo, 3–4 positions de poupée (allongée dos/ventre/côté, semi-occultée par
  chaise), 3 hauteurs de caméra (sol robot ~20 cm, table ~80 cm, plongée ~1,5 m), 3 éclairages (§5),
  avec et sans tissu rouge / QR / bouteille chaude.
- Annotation : Label Studio ou Roboflow, 1 classe `victime` (+ `person` COCO en fine-tune mixte pour ne pas
  oublier la classe d'origine) ; split 70/15/15 train/val/test, seed fixé.
- Compléments gratuits : sous-ensemble COCO `person` en poses assises/allongées pour le pré-entraînement,
  + images négatives (salle vide, sac, veste au sol → faux positifs à tester).

Protocole de test démo (reproductible, 30 min) :
1. Quadrillage 3×3 de la salle, poupée allongée posée successivement sur 5 cases, robot/caméra fixe puis
   balayage : noter détection/fusion par case (tableau 5 cases × 3 éclairages).
2. Faux positifs : salle vide 5 min (compter alertes), veste/sac au sol, passage d'une personne debout
   (doit être détectée `person` mais classée « non allongée » si la règle d'horizontalité est activée).
3. Robustesse : pénombre + contre-jour, avec et sans lampe d'appoint ; noter à quel seuil YOLO décroche et
   si la fusion compense.
4. Latence : script `bench.py` — 100 frames, FPS médian, latence médiane par étage, `yolo benchmark`
   (Ultralytics) pour le tableau officiel du rapport.
5. Critère de succès démo : 4/5 cases détectées en éclairage normal, 0 fausse alerte en 5 min de salle
   vide, latence < 500 ms affichée en direct.

## 8. Plan de charge (binôme, 4 mois, 10 h/sem)

- Mois 1 : stream ESP32-CAM → PC OK, YOLO11n COCO qui détecte la poupée debout/allongée en direct,
  câblage PIR + HC-SR04 avec remontée série, 1ʳᵉ mesure FPS/latence.
- Mois 2 : dataset maison + annotation, fine-tune `yolo train model=yolo11n.pt` (CPU ou Colab gratuit),
  règle de fusion + IHM, 1ᵉʳ protocole complet.
- Mois 3 : robustesse éclairage, export OpenVINO/ONNX + imgsz 416, fallback QR/tissu rouge, MLX90614 si
  budget, répétitions démo.
- Mois 4 : rédaction, courbes précision-rappel, vidéo démo de secours (si le live échoue le jour J),
  préparation Q9 (pourquoi YOLO11n, pourquoi pas la couleur seule).

## 9. Risques et réponses au jury (Q9)

- « Pourquoi pas une simple détection couleur ? » → fragile à l'éclairage et aux leurres (tissu rouge
  ailleurs), aucune sémantique « personne » ; YOLO apporte la classe `person` + bbox, les artifices couleur
  ne sont que secours.
- « YOLO sur CPU portable, n'est-ce pas trop lent ? » → nano + imgsz 416 + OpenVINO ≈ temps réel utile
  pour une cible immobile ; chiffres `yolo benchmark` du PC réel à l'appui.
- « La poupée n'est pas une vraie victime » → assumé : simulation (bouteille chaude = chaleur, PIR =
  présence, YOLO fine-tuné sur la poupée) ; le pipeline est identique avec une vraie personne.
- « Et sans WiFi ? » → hotspot local du PC, aucune dépendance Internet le jour J ; vidéo de secours.

## 10. Sources (primaires)

- Docs modèles Ultralytics — YOLO11 : <https://docs.ultralytics.com/models/yolo11/> (YOLO11n : 2,6 M params,
  6,5 GFLOPs, mAP50-95 COCO 39,5).
- Docs modèles Ultralytics — YOLOv8 : <https://docs.ultralytics.com/models/yolov8/> (YOLOv8n : 3,2 M params,
  8,7 GFLOPs, mAP 37,3).
- Docs modèles Ultralytics — YOLOv5 : <https://docs.ultralytics.com/models/yolov5/> et repo
  <https://github.com/ultralytics/yolov5> (YOLOv5n : 1,9 M params, 4,5 GFLOPs, mAP 28,0).
- Hub de comparaisons Ultralytics : <https://docs.ultralytics.com/compare/> (recommandation YOLO11/YOLO26
  pour nouveaux projets, mise en garde YOLO12/YOLO13 instables sur CPU — d'où leur exclusion ici).
- Repo Ultralytics : <https://github.com/ultralytics/ultralytics> (code, exports ONNX/OpenVINO, `yolo benchmark`).
- YOLO-World (open-vocabulary) : docs <https://docs.ultralytics.com/models/yolo-world/> + papier
  Cheng et al., « YOLO-World: Real-Time Open-Vocabulary Object Detection », arXiv:2401.17270,
  <https://arxiv.org/abs/2401.17270> (35,4 AP LVIS zéro-shot à 52 FPS **sur V100** — pas transposable au CPU).
- COCO dataset (classes `person`, métrique mAP) : Lin et al., « Microsoft COCO: Common Objects in Context »,
  arXiv:1405.0312, <https://arxiv.org/abs/1405.0312>.
- SSD (base de comparaison MobileNet-SSD) : Liu et al., « SSD: Single Shot MultiBox Detector »,
  arXiv:1512.02325, <https://arxiv.org/abs/1512.02325> ; MobileNet : Howard et al., arXiv:1704.04861,
  <https://arxiv.org/abs/1704.04861>.
- ESP32 (pas d'inférence YOLO embarquée, rôle capture) : Espressif, <https://www.espressif.com/en/products/socs/esp32>
  + CameraWebServer (`esp32-camera`) et framework vision micro ESP-WHO <https://github.com/espressif/esp-who>.
- Capteurs : datasheets HC-SR04 (2–400 cm), HC-SR501 PIR, Melexis MLX90614 (thermomètre IR, optionnel
  ~5–10 €) — références fabricants à joindre au dossier d'achat.
