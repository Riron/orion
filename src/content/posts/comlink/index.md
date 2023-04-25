---
title: "Enfin des web-workers developer-friendly"
date: 2023-04-25
summary: Démystifions Comlink en écrivant notre propre version
keywords: javascript, web-worker, worker, async build-your-own
_build:
  list: never
---

Après avoir réécrit [RxJS]({{< ref "/demystifions-rxjs" >}}) dans un précédent article, je vous propose de réécrire une nouvelle librairie JavaScript. J'ai nommé [Comlink](https://github.com/GoogleChromeLabs/comlink).

## Les web workers

Les web workers sont une fonctionnalité JavaScript qui permet aux développeurs d'exécuter du code dans des threads séparés du thread principal de l'application web. Voici quelques-uns des avantages à utiliser les web workers :

- Amélioration des performances : Les web workers permettent de répartir la charge de travail sur plusieurs threads, ce qui peut améliorer les performances de l'application en libérant le thread principal pour d'autres tâches.
- Exploitation du matériel : Les web workers permettent de tirer parti des processeurs multi-cœurs et de l'accélération matérielle pour effectuer des tâches intensives en ressources de manière plus efficace.
- Gestion des tâches longues : Les web workers sont particulièrement utiles pour effectuer des tâches longues et intensives en ressources, telles que l'analyse de données, la conversion de formats de fichiers, la génération de graphiques, etc.

En somme, ils nous aident à améliorer les performances de nos applications en exploitant le multithreading.

Mais les web workers sont une fonctionnalité mal aimée des développeurs web et très peu utilisée. Cela est dû, selon moi, en grande partie aux difficultées à communiquer avec eux.

### Communication

Les web workers communiquent avec le reste du monde par message. Ils peuvent recevoir et envoyer des messages avec les threads environnants. Ces messages sont envoyés via la méthode `postMessage()` et reçus via l'*event listener* `onmessage`. Les messages peuvent être de n'importe quel type de données sérialisable, tels que des chaînes de caractères, des tableaux, des objets, etc.

{{< mermaid >}}
graph TD
A[Main thread] -->|postMessage| B(Worker)
B -->|postMessage| A
{{< /mermaid >}}

Voici un exemple de code pour envoyer un message à un web worker :

**app.js (main thread)**
```JavaScript
// Création d'un worker
const worker = new Worker('worker.js');
// Envoi d'un message au worker
worker.postMessage([1, 2, 3, 4, 5]);
// Réception d'une réponse du worker
worker.onmessage = function(event) {
  console.log('Résultat  : ' + event.data);
};
```

**worker.js (thread séparé)**
```JavaScript
// Réception du message du main thread
onmessage = (event) => {
  // Calcul de la somme
  const result = event.data.reduce((sum, i) => sum + i, 0);
  // Envoi du résultat au main thread
  postMessage(result);
}
```

Pour une opération simple comme ici, l'utilisation d'un *message bus* n'est pas vraiment un problème. Mais la complexité augmente rapidement dès que l'on a plusieurs types de message à gérer.
Imaginons un compteur que l'on veut déporter sur le worker. On veut avec ce compteur pouvoir:
- Accéder à la valeur courante
- L'incrémenter
- Le décrémenter
- Le remettre à zéro

Immédiatement, notre worker va devoir se mettre à différencier l'opération qu'il réalise en fonction du type de message reçu.

**worker.js (thread séparé)**
```JavaScript
let counter = 0;
onmessage = (e) => {
  const { type } = event.data;
  switch (type) {
    case "GET":
      break;
    case "INCREMENT":
      counter++;
      break;
    case "DECREMENT":
      counter--;
      break;
    case "RESET":
      counter = 0;
      break;
  }
  postMessage(counter);
}
```

On observe que rapidement, même si ce n'est rien d'insurmontable, communiquer par message uniquement est une contrainte importante. Ecrire du code est plus pénible et moins lisible que lorsqu'on développe dans un thread unique ou l'on peut facilement appeler des fonctions et lire des variables.


### Comlink

[Comlink](https://github.com/GoogleChromeLabs/comlink) est une librairie JS écrite par Google qui vise à simplifier les interactions entre main thread et web worker.

> Comlink is a tiny library (1.1kB), that removes the mental barrier of thinking about postMessage and hides the fact that you are working with workers.

L'idée est de masquer entièrement l'API basée sur les messages pour proposer aux développeurs de développer comme si les méthodes et variables exposées par le worker étaient locales au main thread. 

Prenons un exemple tout de suite pour bien comprendre.

**main.js**
```javascript
import * as Comlink from "https://unpkg.com/comlink/dist/esm/comlink.mjs";
async function init() {
  const worker = new Worker("worker.js");
  const obj = Comlink.wrap(worker);
  console.log(`Counter: ${await obj.counter}`); 
  await obj.inc();
  console.log(`Counter: ${await obj.counter}`);
}
init();
```

**worker.js**
```javascript
importScripts("https://unpkg.com/comlink/dist/umd/comlink.js");

const obj = {
  counter: 0,
  inc() {
    this.counter++;
  },
};

Comlink.expose(obj);
```

Dans le fichier `main.js`, le worker est enveloppé pour exposer son contenu via la variable `obj`. Par cette variable on accède à `counter` et `inc()` comme on l'aurait fait pour une fonction classique. Il y a une différence notable, il faut **tout** `await`. Même l'accès à des propriétés ou les méthodes synchrone à première vue. En effet, Comlink *masque* l'envoi de messages asynchrones via `postMessage()`, mais ils existent bel et bien. L'utilisation systématique de promesses est donc incontournable.

Si vous aussi en voyant ça vous vous demander "mais comment cette librairie peut-elle bien fonctionner ?", alors ne bougez pas on va voir ça tout de suite.

## Implémentons

### A la découverte des *Proxy*

Il est temps d'implémenter notre propre version de Comlink. Reprenons le début du code et voyons ce dont nous avons a besoin.

```JavaScript
const worker = new Worker("worker.js");
const obj = wrap(worker);
console.log(`Counter: ${await obj.counter}`);
await obj.inc();
```

On va devoir envelopper le `Worker` pour que d'une manière ou d'une autre il se mette à exposer une propriété `counter` et une méthode `inc()`.
Evidemment un `Worker` n'a pas nativement ces propriétés. Et les lui ajouter n'aurait pas vraiment de sens. On va devoir faire appel à une fonctionnalité JS relativement méconnue que sont les **[Proxys](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy)**.

MDN définit les Proxys comme suit:

> Un objet Proxy permet de créer un intermédiaire pour un autre objet qui peut intercepter et redéfinir certaines opérations fondamentales pour lui.

Notre idée donc ici est de créer un intermédiaire pour notre `Worker` qui soit capable d'intercepter l'appel à `counter` et `inc()`. En interceptant cet appel nous pourrons fournir notre propre implémentation qui sous le capot utilisera l'API `postMessage()`, et retournera la réponse sans que l'utilisateur n'ait conscience de rien.

La création d'un objet Proxy se fait avec deux paramètres. Le *target*, l'objet original devant lequel on veut placer un intermédiaire et un *handler*, un objet qui définit les opérations qui seront interceptées et comment celles-ci seront redéfinies.

Le *target* est utile quand on veut modifier un comportement existant, en ajoutant un log à chaque appel par exemple.
Dans notre cas on part complètement de zéro: le worker qu'on enveloppe **n'a aucune des propriétés et méthodes** que l'on va appeler sur lui. Nous pouvons donc arbitrairement choisir un object vide comme *target*.

Essayons
```JavaScript
function wrap(worker) {
  return Proxy({}, {
    // Handler...
  })
}
```

Pour intercepter les opérations, notre handler va pouvoir définir différents *pièges* ou *trap*. Ces pièges vont intercepter les appels aux *[Object internal methods](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy#object_internal_methods)*, les méthods internes à chaque objet, et nous permettre de modifier leur comportement.

Pour bien comprendre comment ça fonctionne, imaginons l'object suivant.
```JavaScript
const foo = { bar: 1 };
```
Voyons maintenant les chemins possibles lorsqu'on appelle `foo.bar.func()`.
Le chemin théorique d'abord qui tombera en erreur puisque `bar` n'a pas de propriété `func()`. Puis le chemin que l'on va pouvoir générer grâce aux proxys ensuite, qui lui va simuler que tout existe comme prévu et pouvoir avoir le comportement que l'on désire.

{{< mermaid >}}
graph TD
A(foo) -->|"Chemin théorique<br>[[Get]](bar)"| B(foo.bar)
B -->|"[[Get]](func)"| C[undefined]
C-->|"[[Call]]"| D{"Erreur<br>foo.bar.func is<br>not a function"}
A -->|"Chemin grâce aux proxys<br>Trap [[Get]]"| E(Proxy 1)
E -->|"Trap [[Get]]"| F(Proxy 2)
F -->|"Trap [[Call]]"| G(Call worker)
{{< /mermaid >}}

Il faut bien comprendre que notre *trap* est appelé **à la place** des méthodes qu'il piège. Il nous donne donc la possibilité de faire "comme si" `foo.bar.func` existait, même si l'objet `bar` n'a pas de propriété `func`.
C'est exactement le principe que l'on va utiliser quand on eveloppera notre worker: on va faire *comme si* une propriété `counter` et une méthode `inc()` existaient.

### Côté main thread

Reprenons l'implémentation. On vient de le voir, il va falloir créer des proxys. Et pas qu'un seul: si j'appelle `a.b.c()` alors que `a` n'a aucune de ces propriétés, il va falloir qu'à chaque niveau d'appel nous ayons un proxy capable de faire *comme si*. Créons donc un helper `createProxy()` pour nous aider.

```JavaScript
function wrap(worker) {
  return createProxy(worker);
}

function createProxy(worker) {
  return new Proxy({}, {
      // Traps
  })
}
```

Ce proxy va d'abord devoir implémenter les *traps* pour l'accès aux propriétés d'un objet. C'est le *trap* `get()`.

```JavaScript
function createProxy(worker) {
  return new Proxy({}, {
    // + _target est notre objet vide
    // + prop est la propriété à laquelle on accède
    get(_target, prop) {
        // Trap pour l'accès à une propriété
    }
  })
}
```

L'accès à une propriété doit gérer deux cas:
1. Soit c'est la propriété que l'on souhaite obtenir et il faut retourner une valeur

Mais comment savoir dans quel cas nous nous trouvons ? Nous allons nous aider de la contrainte asynchrone mentionnée plus haut. On l'a dit, de par la nature de `postMessage()` nous aurons obligatoirement le résultat via une promesses.
Ainsi, tout appel à `a.b` sera forcément de la forme `a.b.then()`. Et ce même si vous l'écrivez sous la `await a.b`. Ce n'est que du sucre syntaxique qui cache un appel à `then()`.
Et puisqu'on est certain que si l'utilisateur souhaite accéder à `a.b` il écrira `a.b.then()`, on peut intercepter l'accès à `.then` et déduire que la propriété précédente était la propriété à laquelle on souhaite accéder.

2. Soit c'est une étape intermédiaire et on souhaite accéder à une propriété plus profonde ou une fonction. 

Dans ce cas là il faut retourner un nouveau Proxy. Celui-ci sera chargé d'intercepter l'appel du niveau suivant. Une seule subtilité, il faut bien sûr garder le chemin parcouru jusque là.

Voyons comment tout ca se retranscrit dans le code.

```JavaScript
// On introduit un paramètre `path`, le "chemin parcouru". Par défaut il est vide.
function createProxy(worker, path = []) {
  return new Proxy({}, {
    get(_target, prop) {
      // On détecte `.then`. C'est qu'on veut une valeur
      if (prop === 'then') {
        // On va interroger le worker avec le `path` courant et un type `GET` qui indique qu'on souhaite accéder à une propriété
        return requestResponseMessage(worker, { type: "GET", path });
      }

      // Sinon on continue notre chemin dans l'arbre en ajoutant `prop` au chemin parcouru
      return createProxy(worker, [...path, prop]);
    }
  })
}
```

Réfléchissons désormais à quoi pourrait ressembler le fait de récupérer une valeur. J'ai appelé la fonction dans le code ci-dessus `requestResponseMessage`.
Puisque c'est cette fonction qui est chargée de récupérer la valeur, c'est elle qui masque l'envoi d'un message au worker. Elle a les contraintes suivantes:
- C'est là que notre appel à `postMessage()` se fait pour transmettre la demande au worker.
- Il faut démarrer un listener pour attendre la réponse du worker. Cette réponse est asynchrone donc retournée sous la forme d'une promesse.
- Comme le worker peut gérer plusieurs demandes en parallèle, on a besoin d'identifier notre demande. On utilise un uuid.


```JavaScript
function requestResponseMessage(worker, message) {
  // On retourne une promesse puisque postMessage est asynchrone
  return new Promise((resolve) => {
    const uuid = self.crypto.randomUUID(); // On génère un UUID associé à notre requête

    // On commence d'abord par démarrer le listener qui traitera la réponse du worker
    worker.addEventListener('message', function listener(ev) {
      // Si la réponse ne concerne pas notre demande initiale, on l'ignore
      if (!ev.data || ev.data.uuid !== uuid) {
        return;
      }

      // Sinon, on résout notre promesse avec la réponse du worker.
      // Sans oublier de supprimer le listener
      worker.removeEventListener('message', listener);
      resolve(ev.data.value);
    });

    // Enfin, on poste la demande au worker
    // Le listener a été écrit avant pour s'assurer qu'il sera "prêt" à recevoir la réponse du worker
    worker.postMessage({ uuid, ...message });
  });
}
```

On y est presque, mais il reste quand même un problème. Souvenez-vous, dans la méthode `createProxy()`, pour savoir s'il fallait retourner une valeur on a "rusé" en testant si la propriété que l'on lisait était le `then`. Pour être cohérent avec ça, il ne faut pas retourner notre promesse de réponse directement mais la propriété `.then` de notre promesse.
Et il y a une petite subtilité supplémentaire: pour que `then()` fonctionne correctement, il a besoin d'avoir pour contexte la promesse initiale. En effet, en interne, `then()` utilise `this` en partant du principe que ce `this` correspond à la promesse sur laquelle il travaille. Il faut donc utiliser `bind()` pour créer une nouvelle fonction avec ces caractéristiques.

```JavaScript
function createProxy(worker, path = []) {
  return new Proxy({}, {
    get(_target, prop) {
      if (prop === 'then') {
        // Dans le call `a.b.then(cb)` on en est à `a.b.then`
        const promise = requestResponseMessage(worker, { type: "GET", path });
        // Il ne faut donc pas retourner la promesse directement, mais `promise.then`.
        // Sans oublier le bind(). Sinon le `this` serait celui de notre scope courant et `then` ne saurait pas comment s'éxécuter.
        // On aurait alors l'erreur suivante: "TypeError: Method Promise.prototype.then called on incompatible receiver function () {}"
        return promise.then.bind(promise);
      }

      return createProxy(worker, [...path, prop]);
    }
  })
}
```

Pour les accès aux propriétés, c'est bon ! Passons désormais aux appels de fonction. Souvenez-vous, on a besoin de `obj.inc()`.
Pour ça il va falloir faire appel à un nouveau *trap* appelé `apply()`. Ici rien de sorcier. On intercepte l'appel et on renvoie directement la promesse. `then` sera bien appelé par la suite, donc il n'y a aucune manipulation particulière à réaliser.

Une nouvelle petite subtilité tout de même. On réalise ici qu'utiliser un objet vide comme *target* de notre Proxy ne fonctionne pas. Ce n'est pas documenté sur MDN, mais la spec précise bien le suivant:

> A Proxy exotic object only has a [[Call]] internal method if the initial value of its [[ProxyTarget]] internal slot is an object that has a [[Call]] internal method.

C'est à dire qu'un proxy ne peut intercepter un `[[Call]]` que si son *target* a lui même une méthode interne `[[Call]]`. Or un objet vide n'est pas *callable*. Notre *trap* `apply()` ne va donc pas fonctionner. Heureusement, le fix est facile: au lieu d'utiliser un objet vide, nous allons utiliser... une fonction vide ! Qui elle est bien sûr *callable*

```JavaScript
function createProxy(worker, path = []) {
  // On a remplacé `{}` par `function {}` pour le target
  return new Proxy(function {}, {
    get(_target, prop) {
      if (prop === 'then') {
        return requestResponseMessage(worker, { type: "GET", path });
      }
      return createProxy(worker, [...path, prop]);
    },
    // Nouveau trap pour [[Call]], un appel de fonction
    apply(_target, _thisArg, rawArgumentList) {
      // On retourne directement la promesse contenant la réponse
      // On passe en paramètre `type: APPLY` pour que le worker puisse gérer correctement l'appel,
      // et la liste des arguments
      return requestResponseMessage(worker, {
        path,
        type: 'APPLY',
        args: rawArgumentList,
      });
    }
  })
}
```

Notre code côté main thread est terminé et nous sommes presque prêts à tester notre code.
Pour que ça fonctionne et même si on ne regarde pas encore le côté worker, on a tout de même besoin d'un code minimal pour vérifier que nos appels fonctionnent.

**worker.js**
```JavaScript
// Implémentation minimale du worker
// On écoute les messages du main thread
onmessage = (evt) => {
  const { uuid } = evt.data;
  // Et on se contente de renvoyer le UUID de la demande ainsi qu'une fausse valeur à chaque fois
  postMessage({ uuid, value: "Dummy value" });
}
```

<br />
<iframe src="https://stackblitz.com/edit/web-platform-zog2pw?devToolsHeight=33&embed=1&file=index.html"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="web-platform-zog2pw"
     allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
     sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
   ></iframe>

Ca y est, nous sommes désormais capable d'intercepter correctement les appels aux propriétés et aux méthodes et de les transférer au Worker pour demander une réponse.
Il ne nous reste plus qu'à gérer cette réponse correctement côté worker.

### Côté worker

Pas de magie particulière côté worker. Pour rappel, voici le code qu'on essaye de faire fonctionner.

```javascript
const obj = {
  counter: 0,
  inc() {
    this.counter++;
  },
};

expose(obj);
```

La méthode `expose` va écouter les messages du main thread, analyser les paramètres de la demande et les convertir en opérations locales afin d'y répondre. Elle prend bien entendu en paramètre l'objet à exposer, qui sera l'objet sur lequel elle va "travailler".
Il n'y a rien de particulier à expliquer ici, donc je vous mets directement le code commenté.

```javascript
function expose(obj) {
  // La première chose à faire, c'est de démarrer le listener pour écouter les messages du main thread.
  // Notons que `self.addEventListener('message')` est équivalent à `onmessage`. C'est juste une autre manière de l'écrire
  self.addEventListener('message', function (ev) {
    // On va récupérer tout ce dont on a besoin pour traiter la demande dans les données de l'évènement.
    const { uuid, path, type, args } = ev.data;

    // On calcule la valeur correspondante au chemin passé en paramètre.
    // Ex: si `path=[a,b,c]`, alors on calcule `obj.a.b.c`
    const rawValue = path.reduce((prev, cur) => prev[cur], obj);

    // Calculeons désormais la valeur de retour...
    let returnValue;
    switch (type) {
      // Si c'est un appel de type GET, c'est un accès à une propriété.
      // On se contente de retourner la valeur.
      case 'GET':
        returnValue = rawValue;
        break;

      // Si c'est un appel de fonction, un tout petit peu de gymnastique.
      case 'APPLY':
        // On récupère le parent de l'appel. Ex: pour `obj.inc()`, `parent=obj` et `rawValue=obj.inc`, 
        const parent = path.slice(0, -1).reduce((obj, prop) => obj[prop], obj);

        // On appelle ensuite notre fonction en définissant le contexte comme étant le parent (pour que le `this` soit le bon).
        // On passe également les éventuels arguments.
        returnValue = rawValue.apply(parent, args);
        break;
      default:
        throw new Error('Unknown type operation.');
    }

    // On enveloppe le résultat dans un `Promise.resolve()`.
    // - Si ce n'était pas une promesse ça ne change rien
    // - Si c'était une promesse, ça la résout
    return Promise.resolve(returnValue).then((value) => {
      // On a notre résultat final. Il ne nous reste plus qu'à le retourner avec l'UUID correspondant à l'appel
      self.postMessage({ uuid, value });
    });
  });
}
```

Récapitulons tout de même brièvement:
- On démarre le listener pour écouter les messages provenant du main thread.
- Grâce aux paramètres passés par le message, on sait exactement quelle opération il va falloir appliquer.
- Comme le résultat de cette opération peut être une promesse et qu'il nous faut retourner au main thread une valeur sérialisable, on enveloppe le résultat calculé dans un `Promise.resolve()` pour s'assurer qu'on aura bien une valeur disponible.
- Il ne nous reste plus qu'à renvoyer la valeur au main thread. En n'oubliant pas d'ajouter l'UUID à notre réponse pour qu'il sache à quoi nous répondons.

Et c'est terminé ! Nous venons de développer notre Comlink maison.


<iframe src="https://stackblitz.com/edit/web-platform-gjtc6z?devToolsHeight=33&embed=1&file=index.html"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="web-platform-gjtc6z"
     allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
     sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
   ></iframe>

## Conclusion

Alors comme d'habitude, notre implémentation est très incomplète par rapport à Comlink. Nous ne gérons pas les erreurs, nous ne gérons pas les appels à des constructeurs par exemple, ou alors le fait d'assigner des valeurs à des propriétés... Et en plus de ça nous ne gérons que des appels simples à des workers et non pas toute API "postMessage-like" comme le fait Comlink.

Mais l'intérêt n'est pas là. L'intérêt pour nous est de comprendre ce qui rend possible cette API qui peut paraitre presque magique à première vue, et d'en profiter pour se plonger dans le monde merveilleux des Proxys. C'est intéressant car les cas d'usage pour les Proxy sont assez rares, et c'est inspirant de voir comment certains arrivent à pousser les concepts pour inventer de nouveaux usages.
