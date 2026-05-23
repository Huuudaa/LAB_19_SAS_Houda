# LAB 19 : Room, MVVM, Repository, ViewModel, LiveData et RecyclerView

Ce projet d'application Android illustre l'architecture moderne recommandée par Google pour la persistance locale et la structuration de code. Il implémente le modèle **MVVM** (Model-View-ViewModel) en combinant **Room Database** (SQLite), le patron de conception **Repository**, des streams réactifs via **LiveData** et un affichage performant à l'aide de **RecyclerView**.

<img width="360" height="786" alt="image" src="https://github.com/user-attachments/assets/77fc792f-f71c-4ced-8f66-7ee0f32ddc81" />

---

## Compétences Visées & Concepts Clés

### 1. Pourquoi utiliser MVVM plutôt que de tout coder "directement dans l'Activity" ?
- **Lisibilité & Séparation des responsabilités** : L'Activity ne s'occupe que de l'affichage graphique et de la capture des actions de l'utilisateur. Toute la logique métier, la manipulation des états et l'accès aux données sont externalisés.
- **Testabilité** : Isoler la logique de données et de présentation de l'interface Android (Activity) facilite l'écriture de tests unitaires (sur le ViewModel ou le Repository).
- **Maintenance** : Plus simple d'ajouter des fonctionnalités (ex: recherche, synchronisation API) sans surcharger l'interface graphique.

### 2. Le Rôle de chaque Couche
* **Entity** ([Note.java](app/src/main/java/com/example/lab_19_sas_houda/data/local/Note.java)) : Classe Java annotée pour Room représentant la structure d'une table SQL (`notes_table`).
* **DAO** ([NoteDao.java](app/src/main/java/com/example/lab_19_sas_houda/data/local/NoteDao.java)) : Interface d'accès aux données. Elle regroupe les requêtes SQL (Insert, Delete, Select) annotées.
* **RoomDatabase** ([NoteDatabase.java](app/src/main/java/com/example/lab_19_sas_houda/data/local/NoteDatabase.java)) : Point central d'accès de la base de données SQLite sous-jacente. Gérée sous forme de Singleton (fil-sécurisé avec `volatile` et `synchronized`).
* **Repository** ([NoteRepository.java](app/src/main/java/com/example/lab_19_sas_houda/data/NoteRepository.java)) : Couche d'abstraction qui unifie les sources de données (ici locale via Room, mais pourrait s'étendre à une API web externe) et gère l'exécution asynchrone.
* **ViewModel** ([NoteViewModel.java](app/src/main/java/com/example/lab_19_sas_houda/viewmodel/NoteViewModel.java)) : Prépare et maintient les données pour l'interface utilisateur.
* **LiveData** : Conteneur de données observable qui respecte le cycle de vie Android (*lifecycle-aware*). Les données ne sont poussées à l'Activity que si celle-ci est active (évitant ainsi les fuites de mémoire et les plantages).
* **RecyclerView** : Composant de liste très performant qui recycle les vues de lignes (`CardView` définies dans [note_item.xml](app/src/main/res/layout/note_item.xml)) qui défilent hors écran pour réduire la consommation CPU et mémoire.

### 3. Exécution hors du Thread Principal
Room interdit par défaut l'exécution de requêtes d'écriture/modification sur le thread principal UI. Si le thread UI est bloqué par une opération de disque lourde, l'application se fige (ANR - *Application Not Responding*). 
Le Repository résout cela en encapsulant les opérations d'écriture via un pool de threads d'arrière-plan (`ExecutorService`).

### 4. Ce que le ViewModel résout réellement
Le `ViewModel` est lié au cycle de vie logique d'un écran. Lors d'une **rotation de l'écran**, d'un **changement de configuration** ou d'une **recréation de l'Activity** par le système, l'instance du ViewModel est préservée en mémoire. L'Activity recréée s'y reconnecte simplement et récupère instantanément les données observées en LiveData sans avoir à relire la base SQLite.
* *Limite* : ViewModel n'empêche pas le "Process Death" (lorsque l'OS tue l'application en arrière-plan pour libérer de la mémoire). Pour contrer cela, il faut utiliser le module `SavedStateHandle`.

---

## Architecture Globale du Projet

```
com.example.lab_19_sas_houda
│
├── data
│   ├── local
│   │   ├── Note.java         (Entity - Table SQLite)
│   │   ├── NoteDao.java      (DAO - Requêtes SQL)
│   │   └── NoteDatabase.java (RoomDatabase - Singleton de base)
│   └── NoteRepository.java   (Repository - Liaison données & threads)
│
├── ui
│   ├── MainActivity.java     (View - Écran principal)
│   └── NoteAdapter.java      (Adapter - Rendu RecyclerView)
│
└── viewmodel
    └── NoteViewModel.java    (ViewModel - Logique présentation)
```

---

## Dépendances Utilisées

Les dépendances configurées dans [build.gradle.kts](app/build.gradle.kts) :
```kotlin
val roomVersion = "2.8.4"
val lifecycleVersion = "2.9.3"

// Room (Runtime et Compilateur d'annotations)
implementation("androidx.room:room-runtime:$roomVersion")
annotationProcessor("androidx.room:room-compiler:$roomVersion")

// Architecture Components (ViewModel & LiveData)
implementation("androidx.lifecycle:lifecycle-viewmodel:$lifecycleVersion")
implementation("androidx.lifecycle:lifecycle-livedata:$lifecycleVersion")

// UI Elements
implementation("androidx.recyclerview:recyclerview:1.4.0")
implementation("androidx.cardview:cardview:1.0.0")
```

---

## Scénarios de Test à Réaliser

1. **Test d'insertion simple** :
   - Saisir un titre et une description puis cliquer sur **AJOUTER UNE NOTE**.
   - *Attendu* : La note apparaît instantanément en haut de la liste.
2. **Test de suppression** :
   - Effectuer un **clic long** sur une note.
   - *Attendu* : La note est supprimée de la base et disparaît de l'écran avec un message toast.
   - 
     <img width="178" height="101" alt="image" src="https://github.com/user-attachments/assets/f696097c-7aed-417a-b713-dd33e2079466" />

3. **Test de persistance** :
   - Fermer complètement l'application (tuer la tâche) puis la relancer.
   - *Attendu* : Toutes les notes créées précédemment réapparaissent (sauvegarde SQLite active).
4. **Test de rotation d'écran** :
   - Commencer à saisir ou afficher des notes puis tourner l'appareil.
   - *Attendu* : La liste reste cohérente, le clavier/l'état d'écran se recharge proprement sans perdre les données observées.
5. **Test de suppression globale** :
   - Cliquer sur le bouton **SUPPRIMER TOUTES LES NOTES**.
   - *Attendu* : La base est vidée et la liste devient blanche instantanément.
