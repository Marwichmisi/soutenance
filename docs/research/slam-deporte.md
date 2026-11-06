# Quel SLAM open-source mature pour cerveau PC déporté + ESP32-CAM / ultrasons / LiDAR low-cost ?

> Recherche pour l'issue #2 (part of #1, map wayfinder). Contexte : binôme 3e année GE-Info, 10 h/sem × 2, 4 mois,
> budget serré, **pas de Raspberry Pi 4/5**. Base acquise : châssis 4 roues, Arduino UNO, ESP32/ESP8266, ESP32-CAM,
> HC-SR04, L298N. Cerveau déporté sur PC portable en WiFi (Python + MQTT/HTTP, option ROS 2 sans Pi).
> Exigence Q7 : **vrai SLAM open-source mature**, pas de grille d'occupation maison seule. Salle test ~4×4 m, zones A/B/C.

## Verdict en 30 secondes

- **Choix principal : Hector SLAM + RPLidar A1 (~100 €).** Seul SLAM LiDAR 2D mature qui **fonctionne sans
  odométrie** — donc compatible avec la base actuelle (moteurs sans encodeurs). Léger sur PC, grille d'occupation
  directement réutilisable par le planificateur.
- **Fallback (0 €, si le LiDAR est hors budget) : ORB-SLAM3 monoculaire sur flux ESP32-CAM**, échelle récupérée via
  ultrasons/odométrie estimée. Dégradé assumé : dérive d'échelle, exige texture + lumière.
- **Point dur : aucun SLAM mature n'accepte des ultrasons HC-SR04 comme source principale.** Faisceau ~15°, 1 mesure
  par capteur à ~20 Hz max, réflexions spéculaires : trop creux pour alimenter un scan-matcher. Les ultrasons restent
  utiles en **complément** (anti-collision proche, levée d'ambiguïté d'échelle), pas comme entrée SLAM.

## 1. Comparatif (sources primaires)

| Critère | GMapping | Hector SLAM | Cartographer 2D | RTAB-Map | ORB-SLAM3 (mono) |
|---|---|---|---|---|---|
| Entrées requises | Laser 2D **+ odométrie** ([hardware req.](https://wiki.ros.org/gmapping#Hardware_Requirements)) | **Laser 2D seul**, odométrie inutile ([hardware req.](https://wiki.ros.org/hector_mapping#Hardware_Requirements)) | Laser 2D (+ odom/IMU recommandés, [tuning docs](https://google-cartographer-ros.readthedocs.io/en/latest/tuning.html)) | Caméra RGB-D/stéréo/mono **ou** laser ; tourne **standalone sans ROS** ([repo](https://github.com/introlab/rtabmap)) | Caméra mono (échelle inobservable) / stéréo / RGB-D + IMU optionnelle ([repo](https://github.com/UZ-SLAMLab/ORB_SLAM3)) |
| Algo / sortie | Filtre particulaire Rao-Blackwellized → grille 2D ([openslam.org](http://openslam.org/gmapping.html)) | Scan-to-map multi-résolution à la fréquence du LiDAR ; **pas de fermeture de boucle explicite** ([wiki](https://wiki.ros.org/hector_mapping), [paper SSRR'11](http://www.sim.tu-darmstadt.de/publ/download/2011_SSRR_KohlbrecherMeyerStrykKlingauf_Flexible_SLAM_System.pdf)) | Local SLAM (scan matching) + optimisation globale par graphe de poses ([algo walkthrough](https://google-cartographer-ros.readthedocs.io/en/latest/algo_walkthrough.html)) | Graphe de poses + fermeture de boucle visuelle/sac-de-mots, gestion mémoire long-terme ([repo](https://github.com/introlab/rtabmap)) | Features ORB + BA locale/globale, multi-maps ([repo](https://github.com/UZ-SLAMLab/ORB_SLAM3), Campos et al., TRO 2021) |
| CPU/RAM sur PC portable | Faible (1 cœur, < 500 Mo) | **Faible** (1 thread, estimations de pose au rythme du scan) | Moyen-fort (optimisation globale ; réglage Lua non trivial) | Moyen-fort (boucle visuelle ; ~1–2 Go RAM typique) | Moyen (extracteur ORB temps réel OK sur laptop moderne, GPU inutile) |
| Robustesse salle 4×4 intérieure | Bonne si odom correcte ; diverge vite avec patinage 4 roues sans encodeurs | **Bonne à vitesse lente**, murs/meubles = géométrie suffisante ; dérive en rotation rapide et couloirs pauvres (pas de loop closure) | Très bonne, mais surdimensionné pour 16 m² | Bonne, mais sensible à texture/lumière en pur visuel | Fragile : mono = dérive d'échelle, rolling shutter ESP32-CAM, WiFi jitter |
| Intégration sans Pi | ROS 1/2 (`slam_gmapping`, [repo](https://github.com/ros-perception/slam_gmapping)) ; **exige odom → impose ajout d'encodeurs** | `hector_mapping` ([repo](https://github.com/tu-darmstadt-ros-pkg/hector_slam)) ; un simple pont TCP/UDP ESP32→PC publiant `sensor_msgs/LaserScan` suffit, **ROS 2 optionnel** | `cartographer_ros` ([docs](https://google-cartographer-ros.readthedocs.io/en/latest/)) ; validation sur bag recommandée, courbe d'apprentissage forte | `rtabmap_ros` ou binaire standalone ; accepte flux MJPEG ESP32-CAM | Binaire standalone sur flux vidéo ; calibration caméra obligatoire |
| Coût matériel | RPLidar A1 + **encodeurs roues (~5 €)** obligatoires | **RPLidar A1 seul (~100 €)** | RPLidar A1 + odom/IMU conseillées (~110 €+) | 0 € (ESP32-CAM déjà acquise) + ultrasons appoint | 0 € (ESP32-CAM déjà acquise) |
| Maturité open-source | Très mature, mais **plus maintenu** (statut *unmaintained* depuis Melodic, [wiki](https://wiki.ros.org/gmapping)) | Mature, BSD, éprouvé UGV/USV/drones ; ROS 1 surtout, portages ROS 2 communautaires | Mature (Google, Apache 2.0) mais **complexe à régler** pour un binôme 4 mois | Mature, maintenu (IntRoLab, BSD), grosse commu | Référence académique, **GPLv3** (OK pour soutenance, pas pour produit fermé) |

Notes prix : le [RPLidar A1](https://www.slamtec.com/en/Lidar/A1) (360°, ~12 m, 5,5–10 Hz) est le standard étudiant
~100 €. Alternatives moins chères : LD19/LD06 (~70–90 €, protocole série simple) ouื่อ RPLidar A2M8 en occasion.
**Tout-ultrasons + ESP32-CAM = 0 € mais aucun SLAM mature ne tourne dessus en primaire** (voir verdict).

Écarté du comparatif : SVO / VINS-Mono — excellents en odométrie visuelle légère, mais **pas de cartographie
persistante** (pas de carte à donner au planificateur wayfinder), donc hors exigence Q7.

## 2. Recommandation

### Principal : Hector SLAM + RPLidar A1, cerveau ROS 2 sur PC

**Pourquoi :** c'est la seule combinaison qui respecte *simultanément* les 4 contraintes : (1) vrai SLAM mature
open-source (Q7), (2) zéro modification mécanique/encodeurs, (3) PC portable comme seul calculateur, (4) budget ~100 €.
La doc officielle confirme qu'**aucune odométrie n'est requise** et que les estimations de pose sortent au rythme du
LiDAR — exactement ce qu'un pont WiFi ESP32 peut alimenter.

**Risques et mitigations :**

| Risque | Mitigation |
|---|---|
| Pas de fermeture de boucle → dérive sur longues trajectoires | Salle 4×4 m : trajectoires courtes ; retour zone A + recalage sur carte figée |
| RPLidar A1 à 5,5 Hz (vs 40 Hz du Hokuyo d'origine) → sensible aux rotations rapides | Vitesse angulaire limitée dans le planificateur (< 0,5 rad/s en mapping) |
| WiFi jitter / perte de scans | TCP (pas MQTT) pour les scans ; buffer + timestamp côté ESP32 ; QoS temps réel |
| ROS 1 vs ROS 2 | `hector_slam` est ROS 1 d'origine ; utiliser un portage ROS 2 communautaire **ou** mode sans ROS : bridge Python qui consomme les scans et publie la grille (le scan-matcher d'Hector est portable) |

### Fallback : ORB-SLAM3 monoculaire sur ESP32-CAM (0 €)

Si le LiDAR ne passe pas le budget : flux MJPEG ESP32-CAM → PC → ORB-SLAM3 mono, avec **échelle métrique injectée**
depuis les HC-SR04 (distance mur le plus proche) ou odométrie estimée (consigne vitesse × temps, calibrée au mètre).
**Limites assumées :** dérive d'échelle, besoin d'une salle texturée et éclairée, rolling shutter → rouler lentement.
Prévoir un mode « carte topologique zones A/B/C » plutôt qu'une métrique précise pour le wayfinder.

### Upgrade optionnel (si budget résiduel, ~5–10 €) : encodeurs à fourche optique sur 2 roues → odométrie →
bascule possible vers [slam_toolbox](https://github.com/SteveMacenski/slam_toolbox) (SLAM ROS 2 par défaut de Nav2,
avec fermeture de boucle) sans changer le LiDAR.

## 3. Plan d'intégration (capteurs → carte → planificateur)

```text
[Robot]                                              WiFi (TCP)                [PC portable]
Arduino UNO (L298N, 4 roues, HC-SR04)                                   ┌─────────────────────┐
  ↕ UART                                    ┌────────┐  scans+odom  │  Bridge Python /    │
ESP32 : leurre scans LaserScan (LiDAR UART   │ RPLidar│ ──────────▶ │  micro-ROS agent    │──▶ Hector SLAM ──▶ occupancy_grid ──▶ Planificateur (A*/Nav2) ──▶ consignes cmd_vel
  115200 baud, ~40 ko/s) + MJPEG ESP32-CAM   │ A1     │  + vidéo     │  (timestamp + buffer)│      │                        ╳ waypoints zones A/B/C
  + télémétrie ultrasons                     └────────┘              └─────────────────────┘      ▼                        ╳ retour moteur via ESP32→Arduino
                                                                                          RViz2 / map_server (.pgm+.yaml)
```

1. **S1 (sem. 1–2) :** RPLidar A1 → ESP32 (UART) → trames binaires TCP → script PC qui publie `sensor_msgs/LaserScan`
   (ROS 2 Humble) ou structure équivalente sans ROS. Valider 5,5 Hz stables à 3 m.
2. **S2 (sem. 3–5) :** Hector en mapping lent (< 0,2 m/s, < 0,5 rad/s) dans la salle 4×4 ; figer la carte
   (`map_server`) ; mesurer la dérive sur 5 allers-retours A→B→C.
3. **S3 (sem. 6–10) :** localisation sur carte figée + planificateur wayfinder (A* sur grille) + évitement local
   ultrasons ; fallback ORB-SLAM3 testé en parallèle sur vidéos enregistrées (zéro risque planning).
4. **S4 (sem. 11–16) :** robustification, démo zones A/B/C, rédaction.

## Sources

- GMapping : [ROS Wiki](https://wiki.ros.org/gmapping) · [openslam.org](http://openslam.org/gmapping.html) ·
  [slam_gmapping](https://github.com/ros-perception/slam_gmapping)
- Hector SLAM : [ROS Wiki](https://wiki.ros.org/hector_mapping) ·
  [hector_slam](https://github.com/tu-darmstadt-ros-pkg/hector_slam) ·
  [paper SSRR'11 (PDF)](http://www.sim.tu-darmstadt.de/publ/download/2011_SSRR_KohlbrecherMeyerStrykKlingauf_Flexible_SLAM_System.pdf)
- Cartographer : [docs ROS](https://google-cartographer-ros.readthedocs.io/en/latest/) ·
  [cartographer](https://github.com/cartographer-project/cartographer)
- RTAB-Map : [repo](https://github.com/introlab/rtabmap) · [rtabmap_ros](https://github.com/introlab/rtabmap_ros)
- ORB-SLAM3 : [repo](https://github.com/UZ-SLAMLab/ORB_SLAM3)
- ROS 2 sans Pi : [slam_toolbox](https://github.com/SteveMacenski/slam_toolbox) ·
  [micro-ROS (ESP32)](https://micro.ros.org/) · [Nav2](https://navigation.ros.org/)
- Matériel : [Slamtec RPLidar A1](https://www.slamtec.com/en/Lidar/A1)
