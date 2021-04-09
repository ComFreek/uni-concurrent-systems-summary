## Mutexe, Semaphore

- Mutex: Menge an Methoden, um wechselseitigen Ausschluss zu garantieren
- Mutexentität: Objekt mit zwei Methoden `acquire`, `release`, wobei `release` überprüft, ob Aufrufer derselbe ist, der auch vorher `acquire` aufgerufen hatte

> A semaphore can be released by any process

Genauer: siehe Tabelle unten.

> A mutex can be released only by the process having it acquired.

Falsch; wäre richtig mit `mutex entity`.

> A binary semaphore is a valid implementation of one of the many “mutex methods”, but not that restrictive as a “mutex entity” need to be (lecture 7, p. 34)


### Semaphore

| Art | Initialwert | Prozess/Semantik von `P` | Semantik von `V` | Autorisierung für `V` nötig?
|:-|:-:|:-:|:-:|-|
| Binärer Semaphor | 0 oder 1 | -/- | -/- | optional
| Binärer Semaphor für Mutexentität | 1 | Prozess, der Mutex betreten möchte | *derselbe* Prozess, der ihn wieder freigeben möchte | zwingend
| Allgemeiner Semaphor für konsumierbare Ressourcen | 0 | Konsument<br>(auf Nachricht warten) | Produzent<br>(Nachricht signalisieren) | falsch
| Allgemeiner Semaphor für wiederverwendbare Ressourcen | `#Speicherplätze` | Produzent<br>(Platz vor Nutzung belegen) | Konsument<br>(Platz nach Nutzung freigeben) | falsch

Grundsätzliches Problem von Semaphoren:

- Implementierung nicht-trivial (wait action unfolding)
- In welcher Reihenfolge weckt man Prozesse auf? FIFO mit Queue vs. Konformität mit Planerentscheidungen.

  <img src="./lectures/_nice_slide-lec7-sem-waitlist.png" width="500" />

<br>

### Bounded Buffer

> Eine unbegrenze Anzahl konsumierbarer Ressourcen in endlich vielen wiederverwendbaren Ressourcen handhaben.

```c
semaphore_t store = {N}; // for the buffer as a reusable resource
semaphore_t data = {0};  // for the availability of data as a consumable resource (signal)

void producer(void) {
  while(1) {
    P(&store);
    // produce
    V(&data);
  }
}

void consumer(void) {
  while(1) {
    P(&data);
    // consume
    V(&store);
  }
}
```

**Allgemeiner (aber nicht negativ zählender) Semaphor** auf Maschinenprogrammebene ohne Systemaufrufe:

```c
typedef struct sem_t { _Atomic(unsigned int) N; } sem_t;

void P(sem_t *sem) {
  unsigned int N;
  do {
    while ((N = sem->N) == 0); // would benefit from futexes
  } while (!CAS(&sem->N, N, N - 1)); // atomic compare-and-swap
}

void V(sem_t *sem) {
  FAA(&sem->N, +1); // atomic fetch-and-add
}
```

**Allgemeiner Semaphor** mithilfe eines binären Semaphors:

```c
typedef struct sem_t {
  binary_sem_t bsem = {false};
  int value = 0;
};

void P(sem_t *sem) {
  if (FAA(&sem->value, -1) <= 0) {
    binary_P(&sem->bsem);
  }
}

void V(sem_t *sem) {
  if (FAA(&sem->value, +1) < 0) {
    binary_V(&sem->bsem); 
  }
}
```

**Binärer Semaphore** skizziert:

```c
struct sem_t { bool state; waitlist_t wait; }

void P_binary(sem_t *sem) {
  atomic {
    if (sem->state) sem->state = false;
    else {
      queue(&sem->wait, self);
      block();
    }
  }
}

void V_binary(sem_t *sem) {
  atomic {
    if (empty(&sem->wait)) sem->state = true;
    else wakeup(pop(&sem->wait));
    //   ^ as long as processes are waiting, state will remain `false`
  }
}
```

## Monitore

- sprachlich: ein high-level Sprachfeature, Programmierkonvention (höher als Semaphore und Mutexe)
- konzeptionell Mittel zur Ressourcenkoordination (von wiederverwendbaren \[kritische Regionen\] und konsumierbaren \[Signalen\] Ressourcen)


**Multilaterale Synchronisation *implizit*** auf allen Methoden des Monitors (d.h. auf geteilter Variable, z. B. Klasseninstanz)
  - Übersetzer instrumentiert Pro- und Epiloge mit Ein- und Austrittsprotokolle für alle geschützten Methoden
  - komplizierter als nur `acquire()`, `release()` für (transitiv) rekursive Methoden

**Unilaterale Synchronisation** (Bedingungssynchronisation) ***explizit***

- `signal()` und `wait()` werden beide immer nur in Monitoren aufgerufen
- `wait()`: verlässt den Monitor und wartet ("atomar")
- `signal():` verschiedene Semantiken möglich:

  Signal-and- | Signaller | Resume | Signallee does
  |-|-|-|-|
  |continue|stays in monitor|all signalled processes<br>(after return) | `while   (!cond) wait()`
  |wait|leaves and recompetes for monitor|all signalled processes| `while (!cond) wait  ()`
  |urgent wait|leaves monitor while joining preferential queue|one signalled process |   `if (!cond) wait()`
  |return|leaves monitor|one signalled process | `if (!cond) wait()`

  Anmerkungen:
  
  
  - `if (!cond)` toleriert keine falsch-positiven Signalisierungen! (D.h.   Programmierfehler solcher ist kritisch.)
  - "one signalled process" in FIFO-Reihenfolge: Interferenz mit Planer möglich
  - "signal-and-continue" erzeugt weniger Hintergrundrauschen (Kontextwechsel), aber   höhere Signalempfangslatenzen


Signalisierung einer Bedingungsvariable *kann* seiteneffektfrei sein. Im Gegensatz dazu: `V()` eines allgemeinen  Semaphors zählt immer hoch.