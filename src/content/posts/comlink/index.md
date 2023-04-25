---
title: "Enfin des web-workers developer-friendly"
date: 2023-04-25
summary: Démystifions Comlink en écrivant notre propre version
keywords: javascript, web-worker, worker, async build-your-own
_build:
  list: always
---

Après avoir réécrit [RxJS]({{< ref "/demystifions-rxjs" >}}) dans un précédent article, je vous propose de réécrire une nouvelle librairie JavaScript. J'ai nommé Comlink.

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

On observe que rapidement, même si ce n'est rien d'insurmontable, communiquer par message uniquement est une contrainte importante. Ecrire du code est plus pénible et moins lisible que lorsqu'on code dans un thread unique et que l'on peut facilement appeler des fonctions et lire des variables.


### Comlink

[Comlink](https://github.com/GoogleChromeLabs/comlink) est une librairie JS mise à disposition par l'équipe de Chrome qui vise à simplifier les interactions entre main thread et web worker.

> Comlink is a tiny library (1.1kB), that removes the mental barrier of thinking about postMessage and hides the fact that you are working with workers.

L'idée est de masquer entièrement cette API basée sur les messages pour proposer aux développeurs de simuler le fait que les méthodes et variables du worker sont locales. 

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

Le worker est exposé via une variable `obj`. Sur cette variable, j'accède à la propriété `counter` et je peux appeler la méthode `inc()` comme je l'aurais fait sur le main thread. Avec juste une différence, il faut `await` la réponse. En effet, en sous marin l'appel passe bien par un `postMessage()`, c'est à dire un appel asynchrone. L'utilisation de promesses est donc incontournable.

En voyant ça, peut-être que vous vous aussi vous vous posez la question. Mais comment ça fonctionne ?? Regardons ça ensemble.

## Implémentons

### A la découverte des *Proxy*

Reprenons le début du code et voyons ce dont on a besoin.

```JavaScript
const worker = new Worker("worker.js");
const obj = wrap(worker);
console.log(`Counter: ${await obj.counter}`);
await obj.inc();
```

On va devoir envelopper le `Worker` pour que d'une manière ou d'une autre il se mette à exposer une propriété `counter` et une méthode `inc()`.
Evidemment un `Worker` n'a pas nativement ces propriétés. Et les lui ajouter n'aurait pas vraiment de sens. On va ici faire appel à une fonctionnalité JS que sont les **[Proxy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy)**.

MDN définit les Proxy comme suit:

> Un objet Proxy permet de créer un intermédiaire pour un autre objet et qui peut intercepter et redéfinir certaines opérations fondamentales pour lui.

Notre idée donc ici est de créer un intermédiaire pour notre `Worker` qui soit capable d'intercepter l'appel à `counter` et `inc()`. En interceptant cet appel on pourra fournir notre propre implémentation qui sous le capot utilisera l'API `postMessage()`, et retournera une valeur sans que l'utilisateur n'ait conscience de cette API de messaging.

La création d'un objet Proxy se fait avec deux paramètres. La *cible* ou *target* d'abord, l'objet original devant lequel on veut placer un intermédiaire et un *handler*, un objet qui définit les opérations qui seront interceptées et comment celles-ci seront redéfinies.

Ici on ne sait pas vraiment à quoi peut correspondre la cible. Notre worker initial n'a pas besoin d'un intermédiaire puisque le comportement que l'on souhaite lui greffer est complètement nouveau. Prenons donc un object vide et essayons d'implémenter cela.

Essayons
```JavaScript
function wrap(worker) {
  return Proxy({}, {
    // Handler...
  })
}
```

Pour intercepter les opérations, notre handler va pouvoir définir différents *pièges* ou *trap*. Ces pièges vont intercepter les appels aux *[Object internal methods](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy#object_internal_methods)*, les méthods internes à chaque objet, et vont nous permettre de modifier leur fonctionnement.

Pour bien comprendre comment ça fonctionne, imaginons que notre worker ait une méthode `a.b.c()` et que l'on souhaite l'utiliser dans notre main thread.
Voici le chemin classique que cet appel aurait pris, et le chemin modifié que nous allons pouvoir générer grâce au Proxy.

{{< mermaid >}}
graph TD
A(a) -->|"[[Get]]"| B(a.b)
B -->|"[[Get]]"| C(a.b.c)
C-->|"[[Call]]"| D("a.b.c()")
A -->|"Trap [[Get]]"| E(Proxy 1)
E -->|"Trap [[Get]]"| F(Proxy 2)
F -->|"Trap [[Call]]"| G(Call worker)
{{< /mermaid >}}

Il faut bien comprendre que notre *trap* est appelé **à la place** des méthodes qu'il piège. Il nous donne donc la possibilité de faire "comme si" `a.b` existait, même si l'objet `a` n'a pas de propriété `b`.
Ainsi dans l'exemple initial, on va faire *comme si* une propriété `counter` et une méthode `inc()` existaient sur notre objet `worker`.

### Côté main thread

Reprenons l'implémentation. On vient de voir que pour les propriétés profondes et les appels de fonction, plusieurs niveaux de *trap* étaient nécessaires. Notre proxy devra donc être capable de retourner des Proxys intermerdiaires. Créons un helper `createProxy()` pour nous aider.

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

Ce proxy va d'abord devoir implémenter les *traps* pour l'accès aux propriétés d'un objet. C'est le trap `get()`.

```JavaScript
function createProxy(worker) {
  return new Proxy({}, {
    get(_target, prop) {
        // Trap pour l'accès à une propriété
    }
  })
}
```

L'accès à une propriété doit gérer deux cas:
- Soit c'est la propriété que l'on souhaite obtenir et il faut retourner une valeur
- Soit c'est une étape intermédiaire, et on souhaite accéder à une propriété plus profonde ou une fonction. Dans ce cas là il faut retourner un nouveau Proxy

Mais comment savoir dans quel cas nous nous trouvons ? Ici c'est la contrainte asynchrone qui va nous aider. On l'a dit un peu plus haut, de par la nature de `postMessage()` il nous faudra obligatoirement `await` notre résultat.
Ainsi, tout appel à `a.b` sera forcément de la forme `a.b.then()`. Et ce même si vous l'écrivez sous la `await a.b`. Ce n'est que du sucre syntaxique qui cache un appel à `then()`.
Ca veut dire que si `a.b` est un proxy, il pourra intercepter l'accès à `.then` et comprendre que la propriété précédente était celle à laquelle l'utilisateur souhaitait accéder.

Pour le deuxième cas, comme on l'a dit on va se contenter de retourner un nouverau proxy. Seule subtilité, il faut garder le chemin parcouru en mémoire.

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

Réfléchissons désormais à quoi pourrait ressembler `requestResponseMessage`.
- C'est là que notre appel à `postMessage()` va se faire, pour transmettre la demande au worker.
- Il faudra également mettre en place un listener pour attendre la réponse et la renvoyer sous la forme d'une promesse.
- Comme le worker peut gérer plusieurs demandes en parallèle, on aura besoin d'identifier notre demande. On le fera via un uuid.


```JavaScript
function requestResponseMessage(worker, message) {
  // On retournera une promesse
  return new Promise((resolve) => {
    const uuid = self.crypto.randomUUID(); // On génère un UUID associé à notre requête

    // On commence d'abord par mettre en place le listener qui sera chargé de traiter la réponse du worker
    worker.addEventListener('message', function listener(ev) {
      // Si la réponse ne concerne pas notre demande initiale, on l'ignore
      if (!ev.data || ev.data.uuid !== uuid) {
        return;
      }

      // Sinon, on n'oublie pas de supprimer le listener puis on résout la promesse avec la réponse du worker
      worker.removeEventListener('message', listener);
      resolve(ev.data.value);
    });

    // Enfin, on poste la demande au worker
    worker.postMessage({ uuid, ...message });
  });
}
```

On y est presque, mais il reste quand même un problème. Souvenez-vous, dans la méthode `createProxy()`, pour savoir s'il fallait retourner une valeur on a "rusé" en testant si la propriété que l'on lisait était le `then`. Pour être cohérent avec ça, il ne faut pas retourner notre promesse de réponse directement mais la propriété `.then` de notre promesse. Et il y a une petite subtilité supplémentaire: lorsque `then()` est appelé il a besoin d'avoir pour contexte la promesse initiale. En effet, en interne, `then()` utilise `this` en partant du principe que ce `this` correspond à la promesse sur laquelle il travaille. On doit donc utiliser `bind()` pour créer une nouvelle fonction avec ces caractéristiques.

```JavaScript
function createProxy(worker, path = []) {
  return new Proxy({}, {
    get(_target, prop) {
      if (prop === 'then') {
        // Dans le call `a.b.then(cb)` on en est à `a.b.then`
        const promise = requestResponseMessage(worker, { type: "GET", path });
        // Il ne faut donc pas retourner la promesse directement, mais `promise.then`.
        // Sans oublier le bind(). Sinon le `this` serait celui de notre scope courant et le `then` ne saurait pas comment s'éxécuter.
        // On aurait alors l'erreur suivante: "TypeError: Method Promise.prototype.then called on incompatible receiver function () {}"
        return promise.then.bind(promise);
      }

      return createProxy(worker, [...path, prop]);
    }
  })
}
```

On veut aussi gérer les appels de fonctions pour que `obj.inc()` soit fonctionnel.
Pour ça il va falloir faire appel à un nouveau *trap* appelé `apply()`. Ici rien de sorcier. On intercepte l'appel et on renvoie directement la promesse.

```JavaScript
function createProxy(worker, path = []) {
  return new Proxy({}, {
    get(_target, prop) {
      if (prop === 'then') {
        return requestResponseMessage(worker, { type: "GET", path });
      }
      return createProxy(worker, [...path, prop]);
    },
    // Nouveau trap pour [[Call]], c'est à dire l'appel de fonctions
    apply(_target, _thisArg, rawArgumentList) {
      // C'est un appel de fonction, on peut retourner directement la promesse
      // On passe en paramètre `type: APPLY` pour que le worker puisse gérer correctement l'appel
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

Testons notre code désormais.
Si on ne s'occuppe pas encore de l'implémentation côté worker, on a tout de même besoin d'un code minimal pour vérifier que nos appels fonctionnent.

**worker.js**
```JavaScript
// Implémentation minimale du worker
onmessage = (evt) => {
  const { uuid } = evt.data;
  // On se contente de renvoyer le UUID et une fausse valeur
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

Et voilà, notre implémentation côté main thread est terminée. On est désormais capable d'intercepter correctement les appels aux propriétés et aux méthodes et de les transférer au Worker en attendant une réponse.
Il ne nous reste plus qu'à répondre correctement côté worker.

### Côté worker

Pas de magie de ce côté-ci. Pour rappel, voici le code qu'on essaye de faire fonctionner.

```javascript
const obj = {
  counter: 0,
  inc() {
    this.counter++;
  },
};

expose(obj);
```

La méthode `expose` va écouter les messages du main thread, et les convertir en opérations locales afin d'y répondre. Elle prend en paramètre l'objet sur lequel travailler.
Rien de sorcier ici, donc passons au code directement.

```javascript
function expose(obj) {
  // La première chose à faire, c'est de démarrer le listener pour écouter les messages du main thread
  // Notons que `onmessage` est équivalent à `self.addEventListener('message')`
  self.addEventListener('message', function (ev) {
    // On va récupérer tout ce dont on a besoin pour traiter la demande dans les données de l'évènement
    const { uuid, path, type, args } = ev.data;

    // On calcule la valeur correspondate au chemin passé en paramètre.
    // Ex: si `path=[a,b,c]`, alors on calcule `obj.a.b.c`
    const rawValue = path.reduce((prev, cur) => prev[cur], obj);

    // Puis on va calculer la valeur de retour
    let returnValue;
    switch (type) {
      // Si c'est un appel de type GET, c'est l'accès à une propriété
      // On se contente de retourner la valeur
      case 'GET':
        returnValue = rawValue;
        break;

      // Si c'est un appel de fonction, un tout petit peu de gymnastique
      case 'APPLY':
        // On récupère le parent de l'appel. Ex: pour `obj.inc()`, `parent=obj` et `rawValue=obj.inc`, 
        const parent = path.slice(0, -1).reduce((obj, prop) => obj[prop], obj);

        // On appelle ensuite notre fonction en n'oubliant pas de définir le contecte comme étant le parent
        // On passe à l'appel les éventuels arguments
        returnValue = rawValue.apply(parent, args);
        break;
      default:
        throw new Error('Unknown type operation.');
    }

    // On résout la promesse éventuelle en enveloppant le résultat dans un `Promise.resolve()`
    return Promise.resolve(returnValue).then((value) => {
      // On a désormais le résultat, il ne nous reste plus qu'à le retourner avec l'UUID correspondant à l'appel
      self.postMessage({ uuid, value });
    });
  });
}
```

Récapitulons brièvement:
- On démarre le listener pour écouter les messages provenant du main thread.
- Grâce aux paramètres passés par les messages, on sait exactement quelle opération il va falloir appliquer.
- Comme le résultat de cette opération peut être une promesse et qu'il nous faut retourner au main thread une valeur sérialisable, on enveloppe le résultat calculé dans un `Promise.resolve()`. Si c'est déjà une valeur, ca ne change rien. Si c'est une promesse ça permet d'obtenir la valeur.
- Il ne nous reste plus qu'à renvoyer la valeur au main thread. En n'oubliant pas d'ajouter l'UUID à notre réponse pour qu'il sache à quoi nous répondons.

Et c'est terminé ! Nous avons désormais un code capable de répondre à l'exemple initial.


<iframe src="https://stackblitz.com/edit/web-platform-gjtc6z?devToolsHeight=33&embed=1&file=index.html"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="web-platform-gjtc6z"
     allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
     sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
   ></iframe>

## Conclusion

TODO