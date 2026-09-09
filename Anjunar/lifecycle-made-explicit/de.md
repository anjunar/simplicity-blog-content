Es gibt eine Klasse von Fehlern, die in fast jeder Webanwendung vorkommt und die man selten als solche benennt. Jemand tippt in ein Suchfeld, klickt weiter, und eine Sekunde später schreibt eine verspätete Antwort ihr Ergebnis in eine Seite, die es nicht mehr gibt. Oder es kommt zuerst die Antwort auf die dritte Eingabe und danach die auf die zweite, und die Liste zeigt am Ende das Falsche.

React löst das mit Cleanup-Funktionen in `useEffect` — und das ist genau der Punkt, an dem viele Fehler entstehen, weil das Aufräumen optional aussieht.

In JFX ist der Lebenszyklus keine Option. Er ist ein Objekt.

## Was eine Seite besitzt

```ts
export function asyncLifecycle(): AsyncLifecycle {
  const restore = capture();
  let active = true;
  let timer: ReturnType<typeof setTimeout> | undefined;
  let generation = 0;
  let generationController: AbortController | undefined;
  const operations = new Set<AbortController>();

  disposeWith({
    dispose(): void {
      active = false;
      generation += 1;
      if (timer !== undefined) clearTimeout(timer);
      generationController?.abort();
      for (const controller of operations) controller.abort();
      operations.clear();
    },
  });
  // ...
}
```

Der Kommentar über der Funktion sagt in einem Satz, worum es geht: *Owns timers and requests created by one mounted page/component.*

Eine Seite hat Timer und Anfragen, und beide gehören ihr. Wird sie abgebaut, verschwinden sie.

![Der Lebenszyklus einer Seite, ausgeschrieben](assets/de-image-01.png "Was passiert, wenn jemand wegklickt"){width=720}

`capture()` in der ersten Zeile ist der subtilste Teil. Es merkt sich den Kontext, in dem diese Seite gebaut wurde — das Element, unter dem sie hängt, ihre Sprache, ihre Umgebung. `commit` stellt diesen Kontext später wieder her:

```ts
const commit = (action: () => void): void => {
  if (active) restore(() => { if (active) action(); });
};
```

Zweimal `active` geprüft: einmal vor dem Wiederherstellen, einmal danach. Das ist kein Versehen. Zwischen beiden Prüfungen kann die Seite abgebaut worden sein.

Ohne `capture` würde eine Antwort, die eine Sekunde später eintrifft, in einer Welt landen, in der niemand mehr weiß, wohin sie gehört. Mit `capture` läuft der Callback im richtigen Kontext — oder gar nicht.

## Lesen und Schreiben sind verschieden

Das ist die Unterscheidung, die diese Datei interessant macht.

```ts
nextGeneration(): RequestGeneration {
  generationController?.abort();
  generationController = createAbortController();
  const ownGeneration = ++generation;
  return {
    signal: generationController.signal,
    isCurrent: () => active && ownGeneration === generation,
  };
}
```

Eine neue Lese-Generation bricht die vorige ab. Wer schnell tippt, löst viele Suchen aus; nur die letzte darf ihr Ergebnis zeigen. Und weil `isCurrent()` zusätzlich prüft, kann eine Antwort, die trotz Abbruch noch eintrifft, sich selbst als veraltet erkennen.

```ts
run<T>(load, success, failure): void {
  if (!active) return;
  const controller = createAbortController();
  operations.add(controller);
  void load(controller.signal).then(
    (value) => commit(() => success(value)),
    (error) => { if (!isAbortError(error)) commit(() => failure(error)); },
  ).finally(() => operations.delete(controller));
}
```

Schreibvorgänge laufen anders. Sie brechen einander nicht ab — sie werden gesammelt und beim Abbau der Seite abgebrochen.

Und dazu steht ein Satz im Kommentar, der eine echte fachliche Entscheidung ist:

> Writes use `run`: their client request is aborted on disposal, while the server may still commit it; the detached result is intentionally ignored.

Wer einen Kommentar abschickt und sofort wegklickt, hat den Kommentar trotzdem abgeschickt. Der Client hört auf zuzuhören; der Server macht fertig.

Die Alternative wäre gewesen, den Schreibvorgang wirklich abzubrechen. Das klingt sauberer und ist es nicht: Man kann eine HTTP-Anfrage clientseitig abbrechen, aber man kann nicht wissen, ob der Server sie schon verarbeitet hat. Ein „abgebrochener" Schreibvorgang wäre ein Zustand, über den niemand etwas Sicheres sagen kann.

Also die ehrliche Variante: Wir hören auf hinzusehen, und wir schreiben in den Code, dass wir das absichtlich tun.

## Ein Abbruch, der auch dort funktioniert, wo es ihn nicht gibt

```ts
/** GraalJS has no browser AbortController; its request-time fetch ignores the
 * signal, while generation guards still prevent stale commits. */
function createAbortController(): AbortController {
  if (typeof AbortController !== "undefined") return new AbortController();
  // ... eine minimale Nachbildung
}
```

GraalJS ist eine ECMAScript-Laufzeit ohne Web-APIs. `AbortController` gibt es dort nicht.

Man könnte an jeder Stelle prüfen, ob es ihn gibt. Stattdessen gibt es hier eine Nachbildung, die die richtige Form hat und nichts tut — und der Kommentar sagt genau, was dadurch verloren geht: Die Anfrage wird serverseitig nicht wirklich abgebrochen. Was bleibt, sind die Generationsprüfungen, und die reichen, um veraltete Ergebnisse zu verwerfen.

Das ist eine ehrliche Attrappe. Sie behauptet nicht, dasselbe zu tun. Sie hält nur die Form, damit der Code darüber an beiden Orten identisch bleibt.

## Der Debounce gehört auch dazu

```ts
debounce(delayMs, action): void {
  if (timer !== undefined) clearTimeout(timer);
  timer = setTimeout(() => {
    timer = undefined;
    commit(action);
  }, delayMs);
}
```

Fünf Zeilen, und sie sind der Grund, warum eine Suche mit Verzögerung in diesem System keine eigene Fehlerquelle ist. Der Timer gehört der Seite, wird beim Abbau gelöscht, und selbst wenn er noch feuert, läuft er durch `commit` — also nur, wenn die Seite noch da ist.

In den meisten Projekten steht dieselbe Logik in jeder Komponente neu, jedes Mal mit einem `useEffect`, das aufräumt. Und in einer davon fehlt das Aufräumen.

## Der Preis

**Man muss daran denken.** Eine Seite, die `asyncLifecycle()` nicht benutzt und stattdessen direkt `fetch` aufruft, hat all diese Probleme wieder. Das Framework zwingt zu nichts.

**Es ist mehr Code als `useEffect`.** Hundertfünfzig Zeilen für etwas, das ein anderes Framework mitbringt.

**Und es ist ungewohnt.** Wer aus React kommt, sucht die Cleanup-Funktion und findet ein Objekt mit fünf Methoden.

## Warum ich es trotzdem so haben will

Weil ich die Alternative kenne. In einem System mit automatischem Aufräumen weiß man nicht, wann etwas aufgeräumt wird — man vertraut darauf. Und wenn es einmal nicht passiert, sucht man sehr lange.

Hier steht es in einer Datei. `dispose()` ist sieben Zeilen lang, und man kann sie lesen und sagen: Danach läuft nichts mehr.

Das ist derselbe Gedanke wie beim Transaktionsfilter im Backend. Nicht weniger Komplexität — dieselbe Komplexität, an einer Stelle, die man ansehen kann.

Im nächsten Artikel geht es um die Liste, aus der dieses Frontend fast alles ableitet, was es über sich selbst weiß.
