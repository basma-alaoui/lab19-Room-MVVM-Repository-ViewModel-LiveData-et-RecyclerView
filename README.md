LAB 19 – ROOM, MVVM, REPOSITORY, VIEWMODEL, LIVEDATA ET RECYCLERVIEW

Cours : Programmation Mobile – Android avec Java


COMPÉTENCES VISÉES

À la fin de ce laboratoire, vous saurez :

- Différencier une application codée “directement dans l’Activity” d’une application structurée selon MVVM.
- Comprendre le rôle exact de chaque couche :
  * Entity : structure des données persistées
  * DAO : interface d’accès aux données (requêtes SQL)
  * RoomDatabase : point central de la base SQLite
  * Repository : couche intermédiaire entre les sources de données (Room, réseau, cache) et le ViewModel
  * ViewModel : logique de présentation, conservation de l’état de l’écran (survit aux rotations)
  * LiveData : observable respectant le cycle de vie (met à jour l’UI uniquement si l’Activity est active)
  * RecyclerView : affichage performant d’une liste
- Comprendre pourquoi les opérations Room ne doivent pas s’exécuter sur le thread principal (blocage UI).
- Tester la persistance locale, la rotation d’écran et la suppression.


ARCHITECTURE GLOBALE DU PROJET

Activity (UI) → ViewModel → Repository → DAO → Room Database → SQLite

Flux :
- L’Activity observe des LiveData via le ViewModel.
- Le ViewModel appelle le Repository.
- Le Repository exécute des opérations asynchrones (coroutines ou threads) via le DAO.
- Room gère la base SQLite.
- Toute modification de la base est automatiquement remontée via LiveData jusqu’à l’UI.


ÉTAPE 1 – CRÉATION DU PROJET ET DÉPENDANCES

1. Android Studio → New Project → Empty Activity.
   Nom : NotesApp
   Langage : Java
   Minimum SDK : API 24
   Finish.

2. Ouvrir build.gradle (Module : app). Ajouter les dépendances suivantes (versions stables) :

   def room_version = "2.6.1"
   def lifecycle_version = "2.7.0"

   implementation "androidx.room:room-runtime:$room_version"
   annotationProcessor "androidx.room:room-compiler:$room_version"
   implementation "androidx.room:room-ktx:$room_version"

   implementation "androidx.lifecycle:lifecycle-viewmodel:$lifecycle_version"
   implementation "androidx.lifecycle:lifecycle-livedata:$lifecycle_version"
   implementation "androidx.lifecycle:lifecycle-viewmodel-ktx:$lifecycle_version"

   implementation "androidx.recyclerview:recyclerview:1.3.2"
   implementation "com.google.android.material:material:1.11.0"

3. Synchroniser le projet (Sync Now).


ÉTAPE 2 – ORGANISATION PROPRE DES PACKAGES

Créer les packages suivants :

- data (ou model)
- db (ou database)
- repository
- viewmodel
- adapter
- ui (pour MainActivity et layout)


ÉTAPE 3 – CRÉER L’ENTITY

Dans le package data, créer une classe Note.java.

public class Note {
    @PrimaryKey(autoGenerate = true)
    private int id;
    private String title;
    private String description;

    // Constructeur sans id (pour l'insertion)
    public Note(String title, String description) {
        this.title = title;
        this.description = description;
    }

    // Getters et setters pour tous les champs
}


ÉTAPE 4 – CRÉER LE DAO

Dans le package db, créer une interface NoteDao.java.

@Dao
public interface NoteDao {
    @Insert
    void insert(Note note);

    @Delete
    void delete(Note note);

    @Query("SELECT * FROM note ORDER BY id DESC")
    LiveData<List<Note>> getAllNotes();
}


ÉTAPE 5 – CRÉER LA BASE ROOM

Dans le package db, créer une classe NoteDatabase.java abstraite héritant de RoomDatabase.

@Database(entities = {Note.class}, version = 1)
public abstract class NoteDatabase extends RoomDatabase {
    public abstract NoteDao noteDao();

    // Singleton pour éviter plusieurs instances
    private static volatile NoteDatabase INSTANCE;

    public static NoteDatabase getInstance(Context context) {
        if (INSTANCE == null) {
            synchronized (NoteDatabase.class) {
                if (INSTANCE == null) {
                    INSTANCE = Room.databaseBuilder(context.getApplicationContext(),
                            NoteDatabase.class, "note_database")
                            .build();
                }
            }
        }
        return INSTANCE;
    }
}


ÉTAPE 6 – CRÉER LE REPOSITORY

Dans le package repository, créer une classe NoteRepository.java.

public class NoteRepository {
    private NoteDao noteDao;
    private LiveData<List<Note>> allNotes;

    public NoteRepository(Application application) {
        NoteDatabase db = NoteDatabase.getInstance(application);
        noteDao = db.noteDao();
        allNotes = noteDao.getAllNotes();
    }

    public void insert(Note note) {
        // Exécuter sur un thread séparé (AsyncTask ou Executor)
        new InsertAsyncTask(noteDao).execute(note);
    }

    public void delete(Note note) {
        new DeleteAsyncTask(noteDao).execute(note);
    }

    public LiveData<List<Note>> getAllNotes() {
        return allNotes;
    }

    // AsyncTasks internes (ou utilisation d'ExecutorService)
    private static class InsertAsyncTask extends AsyncTask<Note, Void, Void> {
        private NoteDao noteDao;
        InsertAsyncTask(NoteDao dao) { noteDao = dao; }
        @Override
        protected Void doInBackground(Note... notes) {
            noteDao.insert(notes[0]);
            return null;
        }
    }

    private static class DeleteAsyncTask extends AsyncTask<Note, Void, Void> {
        private NoteDao noteDao;
        DeleteAsyncTask(NoteDao dao) { noteDao = dao; }
        @Override
        protected Void doInBackground(Note... notes) {
            noteDao.delete(notes[0]);
            return null;
        }
    }
}


ÉTAPE 7 – CRÉER LE VIEWMODEL

Dans le package viewmodel, créer NoteViewModel.java qui étend AndroidViewModel.

public class NoteViewModel extends AndroidViewModel {
    private NoteRepository repository;
    private LiveData<List<Note>> allNotes;

    public NoteViewModel(@NonNull Application application) {
        super(application);
        repository = new NoteRepository(application);
        allNotes = repository.getAllNotes();
    }

    public void insert(Note note) { repository.insert(note); }
    public void delete(Note note) { repository.delete(note); }
    public LiveData<List<Note>> getAllNotes() { return allNotes; }
}


ÉTAPE 8 – CRÉER L’ADAPTER RECYCLERVIEW

Dans le package adapter, créer NoteAdapter.java.

public class NoteAdapter extends RecyclerView.Adapter<NoteAdapter.NoteViewHolder> {
    private List<Note> notes = new ArrayList<>();
    private OnItemClickListener listener;

    @NonNull
    @Override
    public NoteViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        View itemView = LayoutInflater.from(parent.getContext())
                .inflate(R.layout.item_note, parent, false);
        return new NoteViewHolder(itemView);
    }

    @Override
    public void onBindViewHolder(@NonNull NoteViewHolder holder, int position) {
        Note currentNote = notes.get(position);
        holder.textViewTitle.setText(currentNote.getTitle());
        holder.textViewDesc.setText(currentNote.getDescription());
    }

    @Override
    public int getItemCount() { return notes.size(); }

    public void setNotes(List<Note> notes) {
        this.notes = notes;
        notifyDataSetChanged();
    }

    public Note getNoteAt(int position) { return notes.get(position); }

    // ViewHolder interne
    class NoteViewHolder extends RecyclerView.ViewHolder {
        private TextView textViewTitle, textViewDesc;
        public NoteViewHolder(@NonNull View itemView) {
            super(itemView);
            textViewTitle = itemView.findViewById(R.id.tv_title);
            textViewDesc = itemView.findViewById(R.id.tv_desc);
            // Gestion du clic long pour suppression
            itemView.setOnLongClickListener(v -> {
                int position = getAdapterPosition();
                if (listener != null && position != RecyclerView.NO_POSITION) {
                    listener.onItemLongClick(notes.get(position));
                }
                return true;
            });
        }
    }

    public interface OnItemClickListener {
        void onItemLongClick(Note note);
    }

    public void setOnItemClickListener(OnItemClickListener listener) {
        this.listener = listener;
    }
}


ÉTAPE 9 – CRÉER LE LAYOUT PRINCIPAL (activity_main.xml)

Un LinearLayout vertical contenant :

- Deux EditText : edtTitle (hint = "Titre"), edtDesc (hint = "Description")
- Un Button (btnAdd) texte "AJOUTER"
- Un Button (btnDeleteAll) texte "SUPPRIMER TOUTES LES NOTES" (optionnel)
- Un RecyclerView (rvNotes) avec id et layout_height = match_parent

Ajouter également un fichier item_note.xml (LinearLayout horizontal avec deux TextView).


ÉTAPE 10 – CRÉER MAINACTIVITY

Dans le package ui (ou par défaut), écrire MainActivity.java.

public class MainActivity extends AppCompatActivity {
    private NoteViewModel noteViewModel;
    private RecyclerView recyclerView;
    private NoteAdapter adapter;
    private EditText edtTitle, edtDesc;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        edtTitle = findViewById(R.id.edtTitle);
        edtDesc = findViewById(R.id.edtDesc);
        Button btnAdd = findViewById(R.id.btnAdd);
        Button btnDeleteAll = findViewById(R.id.btnDeleteAll);
        recyclerView = findViewById(R.id.rvNotes);
        recyclerView.setLayoutManager(new LinearLayoutManager(this));
        adapter = new NoteAdapter();
        recyclerView.setAdapter(adapter);

        noteViewModel = new ViewModelProvider(this).get(NoteViewModel.class);
        noteViewModel.getAllNotes().observe(this, notes -> {
            adapter.setNotes(notes);
        });

        adapter.setOnItemClickListener(note -> {
            noteViewModel.delete(note);
            Toast.makeText(this, "Note supprimée", Toast.LENGTH_SHORT).show();
        });

        btnAdd.setOnClickListener(v -> {
            String title = edtTitle.getText().toString().trim();
            String desc = edtDesc.getText().toString().trim();
            if (!title.isEmpty() && !desc.isEmpty()) {
                Note note = new Note(title, desc);
                noteViewModel.insert(note);
                edtTitle.setText("");
                edtDesc.setText("");
            } else {
                Toast.makeText(this, "Veuillez remplir les deux champs", Toast.LENGTH_SHORT).show();
            }
        });

        btnDeleteAll.setOnClickListener(v -> {
            // Pour supprimer tout, il faudrait une méthode dans Repository/ViewModel
            // (non obligatoire pour le test 5, mais utile)
        });
    }
}


ÉTAPE 11 – EXPLICATION COMPLÈTE DU FONCTIONNEMENT

- L’Activity observe la liste LiveData via noteViewModel.getAllNotes().
- Dès qu’une note est ajoutée ou supprimée, Room met à jour le LiveData.
- L’Observer est appelé automatiquement (grâce au lifecycle-aware) et l’adapter reçoit la nouvelle liste.
- Le Repository exécute les opérations d’écriture (insert/delete) dans un thread séparé (AsyncTask) pour ne pas bloquer l’UI.
- Le ViewModel conserve les données lors des rotations d’écran (car lié au cycle de vie de l’Activity).


ÉTAPE 12 – TESTS À RÉALISER

Test 1 – Insertion simple :
Ajouter trois notes. Résultat : elles s’affichent immédiatement dans la liste.

Test 2 – Suppression :
Clic long sur une note. Résultat : la note disparaît de la liste.

Test 3 – Persistance :
Fermer complètement l’application puis la rouvrir. Résultat : les notes sont toujours présentes.

Test 4 – Rotation d’écran :
Ajouter une note, tourner l’écran. Résultat : la liste reste cohérente, l’UI se recharge sans perte.

Test 5 – Suppression globale (optionnelle) :
Implémenter un bouton “SUPPRIMER TOUTES LES NOTES” qui appelle une méthode deleteAll dans le DAO et le Repository. Résultat : la liste devient vide.


CE QUI A ÉTÉ APPRIS

- L’architecture MVVM avec Room, Repository, ViewModel, LiveData, RecyclerView.
- La séparation des responsabilités : UI, logique métier, accès aux données.
- L’exécution asynchrone des opérations SQLite.
- L’observation automatique des données grâce à LiveData.
- La survie du ViewModel lors des changements de configuration.

Cette base est réutilisable pour toutes les applications Android nécessitant une gestion locale de données structurées (carnet d’adresses, todo list, gestionnaire de notes).
