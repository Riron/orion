---
title: "Ecrivons notre propre RxJS"
date: 2023-04-21
summary: Pour bien comprendre une librairie, quoi de mieux que de la réécrire soi-même ? Démystifions RxJS en écrivant notre propre version !
keywords: rxjs, javascript, reactive, build-your-own
draft: false
---

Quand on souhaite comprendre le fonctionnement d'une librairie, le meilleur moyen est souvent d'essayer de la réécrire soi-même. C'est ce que je vous propose de faire aujourd'hui avec moi pour RxJS.

## Définition

On suppose ici que tout le monde connait déjà RxJS. Si ce n'est pas le cas, je vous invite à consulter [leur documentation officielle](). Cependant, prenons quand même le temps de rappeler certains principes pour voir comment ceux-ci se retrouvent ensuite dans le code.

RxJS est une librairie de **programmation réactive** basée sur les **Observables**. Ces Observables représentent un flux de données que j'appelerai des **collections**. Ces collections peuvent représenter une suite d'événements, des requêtes HTTP, des streams, des tableaux, etc. L'avantage de les représenter ainsi, c'est que l'on peut les manipuler facilement, avec des opérateurs tels que `map`, `filter`, `reduce`...

Les quelques lignes suivantes, tirées de la documentation officielle, résument plusieurs principes importants:

> Observables are lazy Push collections of multiple values. They fill the missing spot in the following table:
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

<iframe src="https://codesandbox.io/embed/vigilant-hooks-k3c8cl?fontsize=14&hidenavigation=1&module=%2Fsrc%2Frxjs.js&theme=dark"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="vigilant-hooks-k3c8cl"
     allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
     sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
   ></iframe>

> **DISCLAIMER**
> Je ne suis en aucun cas un _expert RxJS_ et cet article n'a pas vocation à reproduire le fonctionnement exact de la librairie. L'idée est simplement de démistifier certains concepts.
> Si certains points vous semblent confus ou erronnés, n'hésitez pas à venir en discuter dans les commentaires !

## De callback à Observer

Ouvrons la documentation RxJS et regardons le premier snippet de code que l'on peut y trouver:

```js
import { fromEvent } from "rxjs";

fromEvent(document, "click").subscribe(() => console.log("Clicked!"));
```

À première vue, cela peut sembler compliqué pour pas grand-chose. Rappelons qu'un JS plus classique ressemblerait à ça:

```js
document.addEventListener("click", () => console.log("Clicked"));
```

On a un élément du DOM, `document`, on y attache un event handler au clic, et à chaque clic sur le document notre callback est appelé.
Avec RxJS, on change notre manière de voir les choses. On passe d'une vision _callback_ à une vision _d'Observer_.

RxJS c'est ça. C'est changer de paradigme, quitter les callbacks et considérer tout comme des collections. On prend notre gestionnaire d'événements qui émettait ponctuellement des valeurs indépendantes et on le transforme en une collection manipulable. Cette colletion grandit avec le temps. Grâce à cette vision de collection, on peut traiter une série d'évènements comme on traiterait un tableau avec lodash et lui appliquer tout un tas d'opérations comme on le verra plus tard.

D'ailleurs, les tableaux peuvent eux aussi être considérés comme des collections. Transformons un tableau classique en Observable.

```js
const array = [1, 2, 3, 4, 5];
array.forEach((value) => console.log(value));
```

En RxJS, le code devient

```js
const array = [1, 2, 3, 4, 5];
const observer = (value) => console.log(value);
from(array).subscribe(observer);
```

Le principe est le même. On transforme une API _à callback_ classique en un _Observable_ qui expose une méthode `subscribe()`.

Ce `subscribe()` c'est le moyen donné par l'Observable pour permettre à un **Observer** de s'abonner au flux de données qu'il va produire. L'Observer a la possibilité de passer une fonction qui est appelée par l'Observable à chaque fois qu'une valeur est émise.

Vous noterez au passage que le parcours d'un tableau exécute notre callback de manière synchrone, alors que dans le cas de l'event listener, nos callbacks étaient exécutés de manière asynchrone à chaque clic. Avec les Observables en fait peut importe. On a un flux de données qui est poussé (_Push_) à l'Observer. On traite donc de la même maniètre synchrone et asynchrone. On sait juste que la collection qu'on observe peut avoir des valeurs qui changent au cours du temps.

D'ailleurs, on n'a pas toujours une valeur avec les callbacks.

Prenons les promesses. Elles supportent le succès ou l'erreur.

```js
fetch(URL).then(onFulfilled, onRejected);

// qui est équivalent à
fetch(URL).then(onFulfilled).catch(onRejected);
```

Ou encore les streams qui ont également la notion de "fin".

```js
const readerStream = fs.createReadStream("file.txt");
readerStream.on("data", cbOnNext);
readerStream.on("error", cbOnError);
readerStream.on("end", cbOnComplete);
```

Comme on a compris que l'on pouvait convertir toute API à callback en Observable, notre méthode `.subscribe()` peut, au lieu d'accepter un simple callback en paramètre, prendre un objet qui en spécifie 3.

```js
const observer = {
  next: (data) => console.log(data),
  error: (err) => console.error("Something bad happened"),
  complete: () => console.info("Stream ended"),
};
from(myStream).subscribe(observer);
```

Sans s'en rendre compte, on vient de terminer l'implémentation d'un Observer. C'est simplement un nom que l'on donne à une ou plusieurs fonctions qui sont appelées par notre Observable lorsqu'une nouvelle valeur est émise.

Pour rester concis dans cet article, nous utiliserons la version simplifiée des Observer: `const observer = (data) => {}`. Nous ne travaillerons qu'avec des APIs à callback unique. Cela simplifie légèrement le code, mais les principes d'implémentation restent les mêmes.

## Dessine moi un Observable

Bon, maintenant que nous avons compris ce qu'est un Observer, nous avons besoin d'un Observable à observer.

Reprenons notre exemple initial et écrivons le code pas à pas.

```js
fromEvent(document, "click").subscribe(() => console.log("Clicked!"));
```

La première fonction à écrire c'est `fromEvent`. Nous avons besoin d'une fonction qui prend deux paramètres : un élément du DOM auquel attacher notre listener, et le type d'événement que nous souhaitons écouter. Cette fonction doit nous retourner un Observable.

```js
function fromEvent(target, type) {
  return new Observable((observer) => {
    target.addEventListener(type, observer);
  });
}
```

Notons plusieurs choses:

- Il faut retourner un Observable, c'est le postulat de base
- Nous donnons au constructeur de cet Observable sa _définition_, le moyen d'émettre des valeurs au cours du temps. A chaque évènement émis, nous appelons notre Observer avec la nouvelle valeur.

> Notez bien que la définition de l'Observable est **une closure** ! `target.addEventListener` n'est donc pas immédiatement invoqué. C'est l'Observer qui, en souscrivant à l'Observable, déclenchera l'event emitter. C'est pourquoi on dit des Observables qu'ils sont **lazy**.

Implémentons maintenant notre classe `Observable`. Nous avons vu qu'il faut un constructeur pour définir le comportement de notre Observable, et que la souscription au flux de données se fait via une méthode `subscribe`.

```js
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

```js
const array = [1, 2, 3, 4, 5];
// Mode callback
array.forEach((value) => console.log(value));

// Mode Observable
from(array).subscribe((value) => console.log(value));
```

Pour l'implémentation de `from` rien de compliqué.

```js
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

```js
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

```js
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

```js
pipe(...args) {
    return args.reduce((prev, op) => op(prev), this)
}
```

Si nous éliminons le `.reduce()`, cela donne:

```js
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

```js
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

- Nous acceptons une fonction de projection `project` pour transformer la donnée ==(1)==.
- Nous retournons une closure ==(2)==. Cette closure capture la fonction de projection pour pouvoir l'appliquer.
- Notre closure prend un Observable de départ en paramètre ==(3)==.
- Notre closure retourne un nouvel Observable ==(4)==.
- Dans la définition du nouvel Observable, on `subscribe` à l'Observable initial pour récupérer sa valeur. On applique la projection sur cette valeur. Puis on renvoie un nouvel Observable, défini avec la valeur projetée que nous venons de calculer.

Un Observer en sortie a désormais accès à la valeur projetée.

Et c'est pareil pour `filter`:

```js
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

On pourrait s'amuser à implémenter encore de nombreux opérateurs, mais vous aurez compris le principe. J'espère que ce "deep dive" dans le fonctionnement de RxJS vous aura permis de démistifier les Observables et de mieux comprendre leur fonctionnement.
Pour en savoir plus sur la librairie et consulter le code source rendez-vous sur [leur github](https://github.com/ReactiveX/rxjs) !
