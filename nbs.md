# Nichtblockierende Synchr.

"Optimismus und Toleranz gegenüber Wettläufen von Prozessen" (anstatt Ausschluss derjenigen), "Wettläufe als etwas Seltenes (optimistisch) oder nur Normales ansehen anstatt sie mit dem Sledgehammer (Locks) auszuschließen"

"Tolerance is the suspicion that the other person just might be right." ⇒ nichtblockierende Algorithmen teilweise sogar kooperativ: wenn sie merken, dass ein anderer Prozess auch in demselben Algorithmus schon am Durchlaufen ist, treten sie freiwillig zurück.

- konstruktionale Achse: "state machine outgrowth"
  - Wie sehen Programmarchitektur, Datenstrukturen etc. aus? Sind diese als Zustandsmaschine fassbar, die klein genug ist zur Handhabe? Fassbar i.S.v.
    - konzeptionell
    - technisch (z.B. mittels eines CAS Zustandsübergang ausführbar)
  - Im Vergleich: bei block. Synchronisation oft nur zwei einfache Zustände: außerhalb und innerhalb des Mutex und in beiden ist klar, was zu tun ist:
    - außerhalb des Mutex: Mutex acquirieren
    - innerhalb des Mutex: die gewünschte Operation ungestört ausführen
  - ⇒ bei nichtblock. Synchr: Zustandsexplosion, Zustandspluralismus
    - Listen kurzzeitig "inkonsistent" (d.h. von head aus ist tail kurzzeitig nicht erreichbar)
    - - z. B: von Prozesszuständen:
      - laufender Prozess, der bald blockiert: `running | blocked | pending`

        `pending` könnte after_switch signalisieren, dass Prozess in eine wait queue eingefügt werden sollte
      - laufender, blockierender Prozess, der der letzte im System ist; danach `running | blocked | idle`
      - ... und dessen Wartebedingung aufgehoben wurde: `running | ready | idle`
  - Können Abstraktionsebenen gebrochen werden ("wait-action unfolding"), um Race Conditions zu entgehen?
- transaktionale Achse

  - CAS-basiert

    ```c
    word_t any;                     // shared data

    do {
      word_t old = any;             // 1. make local copy
      word_t new = compute(old);    // 2. compute local result
    } while (!CAS(&any, old, new)); // 3. validate copy against global state, commit
    // 4. repeat if local state was outdated or committing failed
    ```

    sperrfrei (nicht wartefrei), aber ABA-Problem

  - LL/SC-basiert

    ```c
    do {
      word_t old = LL(&any);        // 1. copy and make reservation
      word_t new = compute(old);    // 2. compute local result
    } while (!SC(&any, new));       // 3. store only if reservation still sealed
    ```

    Kein ABA-problem (jedes `LL` vernichtet vorherige Reservierung), aber dadurch auch nicht sperrfrei!

    ```c
    // CAS in terms of LL/SC
    void CAS(word_t *p, word_t old, word_t new) {
      while (LL(p) == old) {
        if (SC(p, new)) return true;
      }
      return false;
    }
    ```

Vergleich block. & nichtblock. Synchr. hinsichtlich Programmarchitektur:

|        Dyn. Datenstruktur  |   Geteilte Daten, auf die nichtblock. Synchr. achten/CAS-sen muss |
| ------------------------------- | ------------------- |
| Stack (LIFO, `enq ‖ deq`)  | Listenkopf
| Queue (FIFO, `enq ‖ enq`) | Listenkopf
| Queue (FIFO, `deq ‖ deq`) | Listenende
| Queue (FIFO, `enq ‖ deq`) | zusätzlich zu oben: Listenkopf und -ende, wenn Liste Singleton war/wird 
| Liste (arbiträres Zugriffsmuster) | alle Listenelemente, Nachbarn von Listenelementen

<br>

**Fazit:** Bei block. Synchr. ist Komplexität womöglich konstant in Anzahl in Zugriffsmustern. Bei nichtblock. Synchr. wächst Komplexität wahnsinnig.

## Stack

Untenstehende Implementierung sperrfrei, aber *nicht* wartefrei.

```c
typedef struct chain_t {
  chain_t *link;
} chain_t;

void push(chain_t *this, chain_t *item) {
  do {
    item->link = this->link;            // (1-2) copy & compute locally
  } while (!CAS(&this->link, c, item)); // (3) comp.-and-swap local comp. result globally
                                        // (4) repeat until commit successful
}

chain_t *pop(chain_t *this) {
  chain_t *item;
  do {
    if (!(item = this->link)) { // copy locally
      break;                    // with shortcut for empty stacks
    }
  } while (!CAS(&this->link, item, item->link));
  //        ^^^^^^^^^^^^^^^^^^^^^
  // In case `this->link == item` evaluates to true, this _local_ state is *wrongly* used to extrapolate to what the global state is.
  // => ABA-Problem

  return item;
}
```

### ABA-Problem

Wenn lokale Kopie, die in Validierung benutzt wird, keine "faithful representation" des globalen Zustands ist.

- `A -> B -> C`

**Lösung mit CAS auf getaggten Pointern (Generationenzählern):**

Tagging etwa via

- Pointertagging Tricks
  - untere Bits, die via Alignment `0` sind: bei 64-bit Systemen und 8-bit Pointern sind das 3 freie Bits
  - obere Bits, abhängig von Adressraummodell
    - bei [user-space Linux Prozessen etwa obere 4 Bits](https://www.kernel.org/doc/html/v5.12-rc5/x86/x86_64/mm.html) (immer `0`)
    - bei [kernel-space Linux Prozessen etwa obere 4 Bits](https://www.kernel.org/doc/html/v5.12-rc5/x86/x86_64/mm.html) (immer `f`)

  ⇒ 2⁷ = 128 Generationen möglich (könnte noch Generationenüberläufe erlauben)
- extra wortbreiter Zähler (mit DCAS), auf 64-bit Systemen also 64-bit

  ⇒ Generationenüberlaufe praktisch ausgeschlossen

**Vorteil:** portabler als LL/SC, welches nur auf RISC existiert; CAS gibt es bei CISC (nativ) und bei RISC (emuliert via LL/SC).

**Nachteile:** leaking abstraction, getaggte Pointer werden Aufrufer von push/pop zurückgegeben und werden auch so von Aufrufer erwartet (um Generationen hochzählen zu können).

```c
chain_t *raw(chain_t *item);

void push(chain_t *this, chain_t *item) {
  do {
    raw(item)->link = this->link;
  } while (!CAS(&this->link, raw(item)->link, tag(item)));
  //                                          ^^^^^^^^^
  //                                    evolve generation
}

chain_t *pop(chain_t *this) {
  chain_t *item;
  do {
    if (!(item = this->link)) break;
  } while (!CAS(&this->link, item, raw(item)->link));
  //            ^^^^^^^^^^^^^^^^^
  // involves check of generations
  return item;
}
```


**Lösung mit LL/SC:**

```c
void push(chain_t *this, chain_t *item) {
  do item->link = LL(&this->link);
  while (!SC(&this->link, item));
}

chain_t *pop(chain_t *this) {
  chain_t *item;

  do if (!(item = LL(&this->link))) break;
  while (!SC(&this->link, item->link));

  return item;
}
```

**Nachteil: nicht einmal sperrfrei** (im Gegensatz zur CAS-basierten Lösung!)

## Queue (`enq ‖ enq`)

```c
void enqueue_lfs(queue_t *this, chain_t *item) {
  chain_t *last;
  item->link = 0;
  do {
    last = this->tail;
  } while (!CAS(&this->tail, last, item));
  last->link = item;
}

// my idea: even waitfree!
void enqueue_wfs(queue_t *this chain_t *item) {
  item->link = 0;
  atomic_exchange(&this->tail, item)->link = item;
}
```

## Queue (`deq ‖ deq`)

```c
chain_t *dequeue_lfs(queue_t *this) {
  chain_t *node;
  do if ((node = this->head.link) == 0) return 0;
  while (!CAS(&this->head.link, node, node->link));

  if (node->link == 0)
    this->tail = &this->head;
}
```

## Queue (`enq ‖ deq`)

Neuralgischer Punkt, wenn `enq` und `deq` parallel auf einelementiger Liste ausgeführt werden.<br>
Probleme:

- P1: `enq` läuft gerade, `deq` nimmt das Kopfelement heraus, `enq` verlinkt `[herausgenommenes Element] -> [neues Element]`!
- P2: `deq` läuft gerade, `enq` fügt Element ein, `deq` setzt `tail`-Zeiger zurück auf Kopfelement

**Konservative Idee:** verbleibe bei `struct queue_t { chain_t head; chain_t *tail; }`

- Zustandspluralismus einführen: `item->link` `== item` für das letzte Elemente, `== 0` für als "invalid" gekennzeichnete und sonstig für innere Elemente.
- konservativ in konstruktionaler Achse: Datenstruktur bleibt annäherend gleich!
  - `enqueue`: zuerst tail umsetzen, dann Verlinkung vom ehemalig letzten Element aus herstellen
  - `dequeue`: zuerst aushängen; bei letztem Element: `enqueue` warnen und wenn erfolgrech: tail umsetzen, sonst: head nach `enqueue` korrigieren.

```c
// my idea: even waitfree!
void enqueue_wfs(queue_t *this, chain_t *item) {
  item->link = 0;
  chain_t *last = atomic_exchange(&this->tail, item);
  if (!CAS(&last->link, last, item)) {
    this->head.link = item;
  }
}

void dequeue_lfs(queue_t *this, chain_t *item) { // from slides: chapter 11, slide 32
  chain_t *node, *next;
  do {
    if ((node = this->head.link) == 0) return 0;
    next = node->link;
  while (!CAS(&this->head.link, node, next == node ? 0 : next));

  if (next == node) { // dequeued item was last, neuralgic point!
    // try to warn enqueue operation
    if (CAS(&node->link, node, 0)) { // warned successfully
      CAS(&this->tail, node, &this->head);
    } else { // otherwise, clean up mess
      this->head.link = node->link;
    }
  }
}

// my idea: probably has a race condition with dequeue_lfs
chain_t *deplete_wfs(queue_t *this) {
  chain_t *list = atomic_exchange(&this->head.link, 0);
  this->tail = &this->head;
}
```

**Datenstruktur-ändernde Idee ([Michael-Scott-Queue](https://www.cs.rochester.edu/research/synchronization/pseudocode/queues.html)):** nutze `struct queue_t { chain_t *head; chain_t *tail }`, d.h. head ist Pointer genau wie tail und wir lassen `head` immer auf ein Dummyelement zeigen

- Problem P1:
  angenommen
  ```
  h ---> [dummy] ---> [X]
  t -------------------^
  ```
  wobei `X` das einziges Listenelement ist.
  
  Die (vorher problematische) Verlinkung `[X] ---> [neues Element]` ist immer korrekt, *auch* wenn `deq` kommt: denn dann ist `[X]` einfach das neue Dummyelement:

  ```
  h ---> [X] ---> [neues Element]
  t --------------------^
  ```

```c
// Miachel-Scott queue, pseudocode as adapted from https://www.cs.rochester.edu/research/synchronization/pseudocode/queues.html

void enqueue(queue_t *this, word_t value) {
  item_t *node = make_node(value), *last;
  // zuerst last rausfinden
  while(1) {
    last = this->tail;
    item_t *next = last->next;
    if (next == NULL) {
      if (CAS(&last->next, NULL, node)) break;
      else continue;
    }
    else CAS(&this->tail, last, last->next);  // tail falling behind, try advancing
  }
  CAS(&this->tail, last, node);
}
word_t *dequeue(queue_t *this) {
  item_t *head, *tail, *next;
  while (1) {
    head = this->head;
    tail = this->tail;
    next = head->next;
    // Is queue empty or tail falling behind?
    if (head == tail) {
      if (next == NULL) return NULL;      // queue was empty
      else CAS(&this->tail, tail, next);  // tail falling behind, try advancing
    } else {                              // no need to deal with tail
      word_t value = next->value;
      if (CAS(&this->head, head, next)) { // try to swing head to the next node
        // dequeue is done
        // now, next is serving as the "dummy node" to which head points to
        free(head);
        return value;
      }
    }
  }
}
```

## Liste (arbiträre Zugriffsmuster)

Siehe (ggf. verbuggten) Foliensatz 11.
