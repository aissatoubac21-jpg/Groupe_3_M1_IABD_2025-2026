# AgriTrac — Application mobile de diagnostic du riz

Application Flutter qui utilise le modèle `agritrac_mobilenetv2_riz.tflite`
(Phases 1 à 4 du notebook) pour diagnostiquer, **hors-ligne, directement sur
le téléphone** :

- feuille saine
- pyriculariose (*Pyricularia oryzae*)
- helminthosporiose (*Cochliobolus miyabeanus*)

et affiche des recommandations en français. Correspond à la Phase 5. Volontairement **pas** de fongicide affiché pour
l'helminthosporiose, ni de synthèse vocale multilingue : voir
`lib/models/disease_info.dart` pour l'explication, et la section
"Prochaines étapes" plus bas.


## Installation (une seule fois)

Prérequis : [Flutter SDK](https://docs.flutter.dev/get-started/install)
installé et `flutter doctor` sans erreur bloquante.

```bash
# 1. Depuis le dossier qui contient CE dossier agritrac_app :
flutter create --platforms=android,ios --org com.agritrac agritrac_scaffold
# 2. Copie les dossiers générés android/ et ios/ dans agritrac_app/
cp -r agritrac_scaffold/android agritrac_app/
cp -r agritrac_scaffold/ios agritrac_app/
rm -rf agritrac_scaffold

cd agritrac_app
flutter pub get
```

## Permissions caméra / galerie (obligatoire)

### Android — `android/app/src/main/AndroidManifest.xml`

Ajoute, juste avant `<application ...>` :

```xml
<uses-permission android:name="android.permission.CAMERA" />
<uses-feature android:name="android.hardware.camera" android:required="false" />
```

### iOS — `ios/Runner/Info.plist`

Ajoute, dans le dictionnaire principal `<dict>` :

```xml
<key>NSCameraUsageDescription</key>
<string>AgriTrac utilise l'appareil photo pour diagnostiquer une feuille de riz.</string>
<key>NSPhotoLibraryUsageDescription</key>
<string>AgriTrac accède à la galerie pour analyser une photo existante.</string>
```

### Android — taille du modèle dans l'APK

`tflite_flutter` embarque le moteur d'inférence sous forme de librairie
native. Si `flutter build apk` se plaint de taille ou de `abiFilters`,
ajoute dans `android/app/build.gradle`, dans le bloc `android { defaultConfig { ... } }` :

```gradle
ndk {
    abiFilters 'armeabi-v7a', 'arm64-v8a'
}
```

## Si `flutter pub get` échoue sur une version

Je n'ai pas eu accès à pub.dev depuis ce sandbox pour vérifier les tout
derniers numéros de version publiés. Les contraintes du `pubspec.yaml`
sont volontairement larges (`>=x.0.0 <y.0.0`) pour laisser pub résoudre
automatiquement. Si une résolution échoue quand même, lance
`flutter pub upgrade --major-versions` puis relis les éventuels
changelogs signalés, en particulier pour `tflite_flutter`.

## Lancer l'application

```bash
flutter run
```

## Structure du projet

```
lib/
  main.dart                     # point d'entrée, charge le modèle une fois
  models/
    diagnosis_result.dart       # résultat d'une classification
    disease_info.dart           # textes FR + recommandations par classe
  services/
    classifier_service.dart     # chargement .tflite + inférence
    history_service.dart        # historique local (hors-ligne)
  screens/
    home_screen.dart            # capture photo / galerie
    result_screen.dart          # affichage du diagnostic
  theme/
    app_theme.dart
assets/
  model/
    agritrac_mobilenetv2_riz.tflite
    labels.txt                  # ordre EXACT des classes en sortie du modèle
```

## Point technique important : l'ordre des classes

`labels.txt` contient :

```
helminthosporiose
pyriculariose
sain
```

C'est l'ordre **alphabétique**, celui que `train_dataset.class_names`
produit automatiquement dans le notebook (Phase 2) et donc l'ordre exact
des 3 sorties du modèle. Si tu ré-entraînes le modèle avec des classes
différentes ou ajoutées, mets à jour `labels.txt` dans le même ordre
alphabétique, sinon les diagnostics affichés seront décalés d'une classe.

## Vérifié dans ce notebook / ce fichier `.tflite`

- Tenseur d'entrée : `[1, 224, 224, 3]`, `float32`, valeurs brutes 0–255
  (le prétraitement `mobilenet_v2.preprocess_input` est intégré dans le
  graphe du modèle, donc **ne pas** re-normaliser côté app).
- Tenseur de sortie : `[1, 3]`, `float32`, probabilités softmax.
- Taille du fichier `.tflite` : environ 2,4 Mo, bien sous l'objectif de
  5 Mo mentionné dans le pitch.

## Prochaines étapes (feuille de route, non incluses ici)

- **Phase 6** — synthèse vocale multilingue (wolof, poular, sérère). La
  structure de `disease_info.dart` sépare déjà le contenu affiché de la
  clé technique pour qu'il soit facile d'ajouter des traductions sans
  toucher au reste de l'app.
- **Phase 7** — module d'alertes/recommandations FAO complet. Le champ
  `recommendedProduct` de `DiseaseInfo` n'est renseigné que pour
  KITAZINE (pyriculariose), seule association confirmée par les sources
  disponibles ici. Fais valider par le phytopathologiste du projet où
  PELT 44 et CORATOP s'insèrent avant de les ajouter, une mauvaise
  association maladie/fongicide dans une appli utilisée sur le terrain
  peut avoir un vrai coût pour un agriculteur.
- **Phases 8–10**, tests techniques, tests terrain, documentation.
