---
title: "RxJS en moins de 50 lignes de code"
date: 2023-04-21
summary: Démystifions RxJS en écrivant notre propre version
keywords: rxjs, javascript, reactive, build-your-own
---

Quand on souhaite comprendre le fonctionnement d'une librairie, le meilleur moyen est souvent d'essayer de la réécrire soi-même. C'est ce que je vous propose de faire aujourd'hui avec RxJS.

## Définition

On suppose ici que tout le monde connait déjà RxJS. Si ce n'est pas le cas, je vous invite à consulter [leur documentation officielle](). Cependant, prenons quand même le temps de rappeler certains principes pour voir comment ceux-ci se retrouvent ensuite dans le code.

RxJS est une librairie de **programmation réactive** basée sur les **Observables**. Ces Observables représentent un flux de données que j'appelerai des **collections**. Ces collections peuvent représenter une suite d'événements, des requêtes HTTP, des streams, des tableaux, etc. L'avantage de les représenter ainsi, c'est que l'on peut les manipuler facilement, avec des opérateurs tels que `map`, `filter`, `reduce`...

Les quelques lignes suivantes, tirées de la documentation officielle, résument plusieurs principes importants:

> Observables are lazy Push collections of multiple values. They fill the missing spot in the following table:
>
> | |SINGLE |MULTIPLE |
> |-----|--------|---------|
> |Pull |Function|Iterator |
> |Push |Promise |Observable|

On résume dans la langue de Molière:

- Le concept central est l'_Observable_
- Les Observables sont des collections, il peut y avoir **plusieurs valeurs**
- Les données sont transmises en mode **Push**: l'_Observer_ reçoit la donnée au fil du temps. L'_Observable_ envoie les données
- Les Observables sont **lazy**: la présence d'un Observer déclenche l'émission du flux. Un observable auquel personne ne souscrit ne "démarrera" jamais

Comme un exemple vaut mille mots, terminons cet introduction par un exemple d'implémentation d'un "type ahead", en RxJS et en "vanilla JS". Prêtez bien attention à la différence d'approche, impérative en Vanilla et déclarative en RxJS, entre ces deux codes qui font (presque, cf commentaires) la même chose.

```javascript
// ---------
// Code RxJs
// ---------
fromEvent(input, "keyup")
  .pipe(
    debounceTime(200),
    map((e) => e.target.value),
    distinctUntilChanged(),
    switchMap(fakeContinentsRequest),
    tap((c) => {
      document.getElementById("output-rxjs").innerText = c.join("\n");
    })
  )
  .subscribe();

// ------------
// Code Vanilla
// ------------
const input = document.getElementById("typeahead-vanilla");

// Equivalent à debounceTime()
const debounceOnKeyUp = debounce(onKeyUp, 200);
input.addEventListener("keyup", debounceOnKeyUp);

let latestValue;
function onKeyUp(e) {
  // Equivalent à map()
  const value = e.target.value;

  // Equivalent à distinctUntilChanged()
  if (value === latestValue) {
    return;
  }
  latestValue = value;

  // /!\ Plus ou moins équivalent au switchMap()
  // Ne gère pas le "cancel" de promesses à chaque keyUp.
  // Peut causer des bugs si l'appel réseau N se résout après l'appel N+1.
  getContinents(value).then((continents) => {
    // Equivalent à tap()
    document.getElementById("output-vanilla").innerText = continents.join("\n");
  });
}
```

<br />
<iframe src="https://codesandbox.io/embed/vigilant-hooks-k3c8cl?fontsize=14&hidenavigation=1&module=%2Fsrc%2Frxjs.js&theme=dark"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="vigilant-hooks-k3c8cl"
     allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
     sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
   ></iframe>
<br />

{{< alert "triangle-exclamation" >}}
Je ne suis en aucun cas un _expert RxJS_ et cet article n'a pas vocation à reproduire le fonctionnement exact de la librairie. L'idée est simplement de démystifier certains concepts.
Si certains points vous semblent confus ou erronnés, n'hésitez pas à venir en discuter !
{{< /alert >}}

## Callback ou Observer ?

Ouvrons la documentation RxJS et regardons le premier snippet de code que l'on peut y trouver:

```javascript
import { fromEvent } from "rxjs";

fromEvent(document, "click").subscribe(() => console.log("Clicked!"));
```

Partons de son équivalent "Vanilla", et essayons de comprendre comment on en arrive là.

```javascript
const callback = () => console.log("Clicked")
document.addEventListener("click", callback);
```

On a un élément du DOM, `document`, auquel on attache un event handler au clic. A chaque clic sur le document notre callback est appelé.
Puisque notre callback réagit à l'évènement clic, on pourrait dire que notre callback *observe* cet évènement. Renommons notre fonction pour faire émerger ce concept.

```javascript
const observer = () => console.log("Clicked")
document.addEventListener("click", observer);
```

Tout d'un coup, sans s'en rendre compte, on a déjà fait la moitié du chemin vers la compréhension de RxJS.
L'idée est simplement de changer notre manière de voir les choses. De passer d'une vision _callback_ à une vision _Observer_.
Avec RxJS, on change de paradigme, tout est collection que l'on peut oberver. Dit différemment, un Observable émet des valeurs au cours du temps et je peux être informé de ces emissions.

Prenons un autre exemple, le parcours d'un tableau.

```javascript
const array = [1, 2, 3, 4, 5];
const observer = (value) => console.log(value)
array.forEach(observer);
```

Même gymnastique d'esprit: j'observe les valeurs du tableau arriver les unes après les autres.

Le code RxJS équivalent donne donne ceci.
```javascript
const array = [1, 2, 3, 4, 5];
const observer = (value) => console.log(value);
from(array).subscribe(observer);
```

Alors que nous apporte ce changement ? A quoi bon donner un nom différent à des concepts identiques ?
3 choses principales.

1. Traiter synchrone et asynchrone de la même manière

Pour le côté synchrone / asynchrone, les deux exemples donnés ci-dessus l'illustrent. Le parcours d'un tableau exécute notre callback de manière synchrone. Un event listener lui au contraire émet des valeurs de manière asynchrone, à chaque clic de l'utilisateur. Avec les Observables on traite ça exactement de la même manière. On sait juste qu'on a un flux de données qui est poussé (_Push_) à l'Observer. La collection qu'on observe peut donc émettre des valeurs au cours du temps ou tout en même temps, ça ne change rien.

2. Traiter tout type de callback uniformément

On a parlé d'API à callbacks "simples", qui émettent toujours une valeur. Mais on bien d'autres types de callbacks en JavasScript.
Prenons les promesses. Elles supportent un callback au succès et un à l'erreur.

```javascript
fetch(URL).then(onFulfilled, onRejected);

// qui est équivalent à
fetch(URL).then(onFulfilled).catch(onRejected);
```

On a aussi les streams qui ajoutent une notion de "fin" d'émission.

```javascript
const readerStream = fs.createReadStream("file.txt");
readerStream.on("data", cbOnNext);
readerStream.on("error", cbOnError);
readerStream.on("end", cbOnComplete);
```

Bien que tous ces types de callbacks soient différents, RxJS les "wrap" pour les manipuler et les exposer d'une manière identique. C'est ce que nous verrons dans la deuxième partie quand nous définirons le concept d'Observable. Mais grossièrement, RxJS nous expose une fonction `from` qui va accepter n'importe quelle méthode à callback JS et nous retourner un Observable.

En attendant complétons tout de même notre vision de l'observer.
Puisque l'on veut convertir toute API à callback en Observable, il nous faut, au lieu de définir l'Observer comme un simple callback, prendre un objet qui en spécifie 3.

```javascript
const observer = {
  next: (data) => console.log(data),
  error: (err) => console.error("Something bad happened"),
  complete: () => console.info("Stream ended"),
};

from(myStream).subscribe(observer);
```

Avec ce simple objet on gère tous les cas possibles. On a l'Observer dans sa forme la plus complète.
Mais le callback simple est tout de même utilisable dans RxJS lorsque l'on n'a pas à gérer l'erreur ou la complétion. Pour cette raison, c'est la forme simplifiée que nous utiliserons dans la suite de cet article, afin ne pas surcharger le code.

3. Manipuler la donnée facilement

Une API unifiée par dessus les callbacks nous permet d'avoir la donnée toujours sous la même forme. Et cette forme, c'est la collection.
Les collections, il faut les voir un peu comme des tableaux auxquels on ajouterait une notion de temps. Mais les tableaux on a l'habitude de les manipuler. Avec Lodash par exemple on a tout un tas d'opérateurs qu'on utilise bien souvent.
Eh bien, une collection c'est pareil. Elle sera manipulable exactement de la même manière. On pourra donc écrire du code réctif, concis et clair en enchainant l'application d'opérateurs sur un flux de données comme on a l'habitude de faire un `array.map().filter()`.

## Dessine moi un Observable

Bon, maintenant que nous avons compris ce qu'est un Observer, nous avons besoin d'un Observable à observer.

Reprenons notre exemple initial et écrivons le code pas à pas.

```javascript
fromEvent(document, "click").subscribe(() => console.log("Clicked!"));
```

La première fonction à écrire c'est `fromEvent`. Nous avons besoin d'une fonction qui prend deux paramètres : un élément du DOM auquel attacher notre listener, et le type d'événement que nous souhaitons écouter. Cette fonction doit nous retourner un Observable.

```javascript
function fromEvent(target, type) {
  return new Observable((observer) => {
    target.addEventListener(type, observer);
  });
}
```

Notons plusieurs choses:

- Il faut retourner un Observable, c'est le postulat de base
- Nous donnons au constructeur de cet Observable sa _définition_, le moyen d'émettre des valeurs au cours du temps. A chaque évènement émis, nous appelons notre Observer avec la nouvelle valeur.

{{< alert "triangle-exclamation" >}}
Notez bien que la définition de l'Observable est **une closure** !
`target.addEventListener` n'est donc pas immédiatement invoqué. C'est l'Observer qui, en souscrivant à l'Observable, déclenchera l'event emitter. C'est pourquoi on dit des Observables qu'ils sont **lazy**.
{{< /alert >}}

Implémentons maintenant notre classe `Observable`. Nous avons vu qu'il faut un constructeur pour définir le comportement de notre Observable, et que la souscription au flux de données se fait via une méthode `subscribe`.

```javascript
class Observable {
  constructor(subscribe) {
    this._subscribe = subscribe;
  }

  subscribe(nextHandler) {
    this._subscribe(nextHandler);
  }
}
```

- Nous stockons la fonction `subscribe` dans le constructeur . Sans l'appeler, rappelez vous, un Observable est _lazy_
- Lors de la souscription, nous invoquons la fonction sauvegardée en passant le `nextHandler` en paramètre. `addEventListener` est alors déclenché, démarre le flux de données, et informe l'Observer à chaque clic

Et c'est tout ! En 10 lignes de code nous venons de poser les bases de notre librairie.

Testons notre code pour vérifier qu'il fonctionne comme attendu.

<iframe src="https://codesandbox.io/embed/billowing-frog-fhuprr?expanddevtools=1&fontsize=14&hidenavigation=1&theme=dark"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="billowing-frog-fhuprr"
     allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
     sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
   ></iframe>

Pour vérifier que nous avons bien compris, prenons désormais l'exemple d'un parcours de tableau. L'Observer ne change pas. Nous avons seulement besoin d'une nouvelle méthode pour convertir un tableau en Observable. Avec RxJS, c'est `from` que nous utilisons.

```javascript
const array = [1, 2, 3, 4, 5];
// Mode callback
array.forEach((value) => console.log(value));

// Mode Observable
from(array).subscribe((value) => console.log(value));
```

Pour l'implémentation de `from` rien de compliqué.

```javascript
function from(array) {
  return new Observable((cb) => {
    array.forEach(cb);
  });
}
```

On vient de le voir, passer de l'utilisation de callbacks à l'utilisation d'Observables est techniquement très simple. Bien que notre code soit considérablement simplifié par rapport à RxJS, il est important de retenir que le concept peut être décrit en quelques lignes et n'a rien de magique.

Ce changement de paradigme vers la programmation réactive prend tout son sens lorsqu'on commence à utiliser des opérateurs pour manipuler les collections de manière similaire à la manipulation classique de tableaux.

## Au tour des opérateurs

Nous avons beaucoup parlé d'opérateurs et de manipulation de flux, mais à quoi ressemblent-ils réellement ?

Reprenons un exemple simple:

```javascript
fromEvent(document, "click")
  .pipe(
    map((v) => v.clientX),
    filter((clientX) => clientX > 250)
  )
  .subscribe((v) => console.log(`X du clic: ${v}`));
```

Voici nos premiers opérateurs, les biens connus `map` et `filter`. Tout comme on les applique d'habitude sur un tableau, on les applique ici sur une collection d'évènements. Vous voyez, ça commence à faire sens !

On constate qu'ils sont appliqués via une méthode `pipe`, qui accepte une liste d'opérateurs joués successivement sur la collection.
Implémentons cela.

```javascript
class Observable {
  constructor(subscribe) {
    this._subscribe = subscribe;
  }

  subscribe(nextHandler) {
    this._subscribe(nextHandler);
  }

  pipe(...args) {
    // Comment j'applique mes opérateurs sur cet observable ?
  }
}
```

La méthode `pipe` prend un paramètre `...args`. Ainsi, `args` est un tableau d'opérateurs sur lequel on itére pour les appliquer un par un. Pour ce faire, on utilise traditionnellement `array.reduce()`. Sa valeur de départ est l'Observable courant. Pour chaque opérateur parcouru, on applique la transformation souhaitée sur la valeur courante. Enfin on renvoie un nouvel Observable qui contient la valeur du précédent après transformation.

```javascript
pipe(...args) {
    return args.reduce((prev, op) => op(prev), this)
}
```

Si nous éliminons le `.reduce()`, cela donne:

```javascript
pipe(...args) {
    let currentObservable = this;
    for (const operator in args) {
        currentObservable = operator(currentObservable);
    }
    return currentObservable;
}
```

Charge à nous désormais d'implémenter correctement les opérateurs pour que le fonctionnement que l'on souhaite fonctionne. Mais on connait désormais l'ensemble des contraintes qu'ils doivent respecter.

1. Ce sont des fonctions qui acceptent une fonction de projection en paramètre. Dans l'exemple de l'opérateur `map((x) => x * x)`, notre fonction de projection est `(x) => x * x`.
2. Ce sont des _higher order function_, c'est à dire des fonctions qui retournent une fonction. En effet nous passons `map(...)` à la méthode `pipe()` et `pipe` s'attend à recevoir une fonction. Le résultat de `map()` est ce qui correspond à la variable `operator` dans le code ci-dessus.
3. La fonction retournée par nos opérateurs doit accepter un Observable en entrée puisqu'elle s'applique sur un Observable initial. C'est l'appel `operator(currentObservable)` dans le code ci-dessus.
4. La fonction retournée par nos opérateurs doit retourner un Observable en sortie. C'est l'assignation `currentObservable = operator(currentObservable)` dans le code ci-dessus.

Passons à l'implémentation. Nous l'expliquerons juste après en reprenant chaque numéro de contrainte et en traduisant ou elle se retrouve.

```javascript
function map(project) {
  return (startObservable) => {
    return new Observable((cb) => {
      startObservable.subscribe((value) => {
        const projectedValue = project(value);
        cb(projectedValue);
      });
    });
  };
}
```

- Nous acceptons une fonction de projection `project` pour transformer la donnée <mark>(1)</mark>.
- Nous retournons une closure <mark>(2)</mark>. Cette closure capture la fonction de projection pour pouvoir l'appliquer.
- Notre closure prend un Observable de départ en paramètre <mark>(3)</mark>.
- Notre closure retourne un nouvel Observable <mark>(4)</mark>.
- Dans la définition du nouvel Observable, on `subscribe` à l'Observable initial pour récupérer sa valeur. On applique la projection sur cette valeur. Puis on renvoie un nouvel Observable, défini avec la valeur projetée que nous venons de calculer.

Un Observer en sortie a désormais accès à la valeur projetée.

Et c'est pareil pour `filter`:

```javascript
function filter(project) {
  return (startObservable) => {
    return new Observable((cb) => {
      startObservable.subscribe((value) => {
        // Si la condition est vérifiée,
        // la valeur est passée à l'Observer.
        if (project(value)) {
          cb(value);
        }
        // Sinon, on ne fait rien de la valeur.
        // L'Observer ne la verra pas.
      });
    });
  };
}
```

Encore une fois on n'a rajouté que quelques lignes de code, et pourtant c'est déjà terminé.
Vérifions que tout fonctionne comme souhaité.

<iframe src="https://codesandbox.io/embed/goofy-panka-etoobf?expanddevtools=1&fontsize=14&hidenavigation=1&theme=dark"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="goofy-panka-etoobf"
     allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
     sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
   ></iframe>

## Conclusion

On pourrait s'amuser à implémenter encore de nombreux opérateurs, mais vous aurez compris le principe. J'espère que ce "deep dive" dans le fonctionnement de RxJS vous aura permis de démystifier les Observables et de mieux comprendre leur fonctionnement.
Pour en savoir plus sur la librairie et consulter le code source rendez-vous sur [leur github](https://github.com/ReactiveX/rxjs) !
