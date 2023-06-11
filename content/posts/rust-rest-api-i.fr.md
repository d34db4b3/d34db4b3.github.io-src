+++
title = "Un service web REST moderne, écrit en Rust, Partie I"
date = 2022-03-06 23:51:21
updated = 2023-06-11 19:48:24
authors = ["d34db4b3"]
description = "Nous allons plonger dans la conception de back-end web avec une API REST simple, en utilisant le langage de programmation fiable et performant Rust"
[taxonomies]
tags = ["rust", "web", "backend", "REST"]
+++

# TLDR

> Nous preparons notre projet avec [`axum`](https://docs.rs/axum/0.6/axum) et [`tokio`](https://docs.rs/tokio/1/tokio) et nous le testons avec une simple route de bienvenue.

# Introduction

L'idée de ce projet est de mettre en place une API REST simple utilisant un certain nombre de fonctionnalités courantes. Il sera découpé en plusieurs parties afin de se pouvoir se focaliser sur chaque aspect.

Parmi elles nous verrons dans un premier temps la mise en place du serveur web, l'installation de routes et la gestion des requêtes et des réponses. Nous verrons par la suite les contraintes d'une journalisation correcte, de restriction d'accès, de traduction et de gestion de base de données.

Nous allons donc utiliser le langage [Rust](https://www.rust-lang.org/fr), le framework[axum](https://github.com/tokio-rs/axum) pour gérer la partie serveur de notre API ainsi que diverses bibliothèques (crates) détaillées plus tard. Les particularités et la syntaxe de ce langage ne seront pas abordés ici.

La version `1.70.0` de Rust sera utilisé et il est supposé que les outils de compilation sont installés au préalable ([installation de Rust](https://www.rust-lang.org/fr/tools/install)).

[Visual Studio Code](https://code.visualstudio.com/) sera utilisé comme IDE tout au long du projet.

Afin de debugger et de tester notre API, le logiciel [HOPPSCOTCH](https://hoppscotch.io/) pourra être utilisé.
Les fichiers de collection seront fournis pour faciliter les tests.

Pour les débutants en Rust, un tutoriel sera disponible prochainement sur ce blog. En attendant, il est conseillé de lire l'excellent [livre du Rust (en anglais)](https://doc.rust-lang.org/book/). ([version française par la communauté](https://jimskapt.github.io/rust-book-fr/))

Les particularités de sérialisation/désérialisation via [serde](https://serde.rs/) ne seront pas étudiées en détail dans cet article.

Au cours de l'article, des instantanés du projet d'exemple seront disponibles et référenceront des commits du dépôt [d34db4b3/rest-api-tutorial](https://github.com/d34db4b3/rest-api-tutorial)

# Initialisation du projet

## Création du projet

Comme pour tout projet Rust, nous allons passer par l'utilitaire [cargo](https://doc.rust-lang.org/cargo/index.html) pour initialiser la structure de nos sources. Une fois placé dans le dossier où le projet va être créé, la commande suivante va générer le dossier `rest-api-tutorial`.

```sh
cargo new rest-api-tutorial
```

Le dossier nouvellement créé peut ensuite être ouvert dans Visual Studio Code.

Les deux fichiers qui vont nous intéresser sont `Cargo.toml` et `src/main.rs` qui sont respectivement le [manifeste de l'application](https://doc.rust-lang.org/cargo/reference/manifest.html) et le point d'entrée du programme.

```toml
# Cargo.toml
[package]
name = "rest-api-tutorial"
version = "0.1.0"
edition = "2021"

[dependencies]
```

```rs
// src/main.rs
fn main() {
    println!("Hello, world!");
}
```

Il est désormais possible de compiler et de lancer le programme via la commande suivante depuis le répertoire du projet.

```sh
cargo run
```

Un superbe "Hello, world!" nous est présenté, mais ce n'est pas vraiment le propos de cet article.

## Ajout des librairies

Nous aurons besoin de quelques dépendances, notamment [`axum`](https://docs.rs/axum/0.6/axum) et [`tokio`](https://docs.rs/tokio/1/tokio). Axum fournira la plupart des fonctionnalités requises pour un serveur HTTP (routage, implémentation du protocole HTTP, middlewares, etc.) Comme il tire parti des fonctionnalités [`async`](https://rust-lang.github.io/async-book/) de Rust, nous aurons également besoin de `tokio`, qui est un moteur d'exécution pour les applications asynchrones.

Ajoutens les dans le fichier de manifeste `Cargo.toml`.
Pour ce faire, il faut ajouter les lignes  `axum = "0.6"` and `tokio = { version = "1", features = ["full"] }` dans la catégorie `[dependencies]`. Le fichier ressemble donc à

```toml
# Cargo.toml
[package]
name = "rest-api-tutorial"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = "0.6"
tokio = { version = "1", features = ["full"] }
```

Le "0.6" représente la version de `axum` à utiliser. Pour plus de détails sur les dépendances et le fichier manifest, voir [cette page] (https://doc.rust-lang.org/cargo/reference/manifest.html).

Les dépendances peuvent aussi (et devraient probablement pour des raisons de simplicité) être ajoutées en utilisant

```sh
cargo add axum@0.6
cargo add tokio@1 -F full
```

## Programme de test

Afin de vérifier la bonne configuration du manifeste et d'entamer notre aventure avec `axum`, le programme suivant peut être injecté dans le fichier `src/main.rs` à la place du "helloworld" de départ. Notre fichier ressemble donc à 

```rs
use axum::{routing::get, Router};
use std::net::SocketAddr;

#[tokio::main]
async fn main() {
    // register a "helloworld" handler to the `/` route
    let app = Router::new().route("/", get(|| async { "Hello, World!" }));

    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));
    println!("listening on http://{}", addr);
    // start the HTTP server and serve our newly created service
    axum::Server::bind(&addr)
        .serve(app.into_make_service())
        .await
        .unwrap();
}
```

Ce programme ressemble dans sa structure au programme précédant avec sa fonction `main`, mais on observe cependant l'apparition de nouveaux mots clés tels que `async` et `await`. Ces derniers décrivent le fonctionnement asynchrone de `axum` et nous reviendrons sur ces détails plus tard. Pour l'instant nous allons nous focaliser sur le comportement de l'application.

### Application

Afin que le serveur puisse faire son travail, on doit lui décrire son fonctionnement. C'est le rôle du router `Router`, sur lequel on va définir des routes, des services et ce dont il va avoir besoin pour produire le fonctionnement attendu. Ici, on lui demande d'enregistrer une route `/` (`.route("/", ...)`) sur laquelle va être greffé une fonction qui va répondre à la méthode HTTP `GET` (`get(...)`). Cette fonction décrite par un autre lambda (`|| async { "Hello, world! }`) s'occupera simplement de retourner la chaine de caractère `"Hello, world!"`.

### Serveur HTTP

Ensuite, nous aurons besoin de servir notre application avec un serveur HTTP. Un `axum::Server` est lié à l'interface réseau et écoutera sur le port TCP `3000` (`::bind(&addr)`). Nous fournissons l'`app` que le serveur doit servir. Le `.await` va finalement permettre à `tokio` de "gérer" le cycle de vie de notre application.


## Lancement du programme

Pour faciliter le développement, l'utilitaire `cargo-watch` sera utilisé tout au long de cet article. Ce dernier permet une recompilation automatique à chaque changement du code source. Pour l'installer il suffit de faire

```sh
cargo install cargo-watch
```

Ensuite le processus de compilation se lance via

```sh
cargo watch -x run
```

Celui-ci relancera la commande `cargo run` à chaque changement de code source.

### Vérification du fonctionnement

Le fonctionnement attendu du programme est un serveur HTTP écoutant sur le port `3000` et répondant sur la route `/` avec le message `"Hello, world!"`. À l'aide d'un navigateur tel que Firefox, une requête sur la page <http://127.0.0.1:3000/> devrait afficher le message

```
Hello, world!
```

[Instantané du code](https://github.com/d34db4b3/rest-api-tutorial/tree/snapshots/init)

# Analyse d'une route

## Routage

Chaque route est associée à une fonction permettant de traiter la requête et de répondre de manière adaptée. Ces fonctions appelées [handlers](https://docs.rs/axum/0.6/axum/handler/index.html) sont simplement des fonctions asynchrones qui reçoivent des extracteurs comme arguments (voir [`axum::extract`](https://docs.rs/axum/0.6/axum/extract/index.html)) et renvoient un objet qui doit être converti en une réponse (voir [`axum::response`](https://docs.rs/axum/0.6/axum/response/index.html)).

## Analyse de la requête

La plupart des données de la requête sont obtenues via des "extracteurs" (type implémentant le trait [`FromRequest`](https://docs.rs/axum/0.6/axum/extract/trait.FromRequest.html) ou [`FromRequestParts`](https://docs.rs/axum/0.6/axum/extract/trait.FromRequestParts.html)). Plusieurs extracteurs sont fournis par `axum` afin de faciliter l'implémentation, mais il est possible d'en ajouter d'autres si nécessaire (ce que nous verrons avec les restrictions d'accès dans une prochaine partie). Parmi eux, nous pouvons mentionner [`Path`](https://docs.rs/axum/0.6/axum/extract/struct.Path.html) qui permet l'extraction de paramètres dans le chemin de la ressource, [`Query`](https://docs.rs/axum/0.6/axum/extract/struct.Query.html) qui de la même manière permet l'extraction des paramètres de la requête ([plus d'informations sur les URLs](https://developer.mozilla.org/fr/docs/Learn/Common_questions/What_is_a_URL)), [`Json`](https://docs.rs/axum/0.6/axum/struct.Json.html) qui permet la désérialisation du corps de la requête à partir du format [JSON](https://www.json.org/json-fr.html) et enfin [`Form`](https://docs.rs/axum/0.6/axum/struct.Form.html) qui a un comportement similaire à celui du format de formulaire classique encodé dans l'URL.

Ces extracteurs sont donnés en paramètre à nos handlers et nous permettent d'obtenir une extraction de données avec un typage statique et fiable. Les erreurs de désérialisation peuvent ainsi être correctement gérées et il sera possible d'avoir une certaine garantie sur la présence et l'exactitude des paramètres avant le traitement de la requête. Cela permet au code d'être plus robuste, clair et concis.

## Analyse de la réponse

Afin de répondre aux requêtes, les fonctions associées aux routes doivent retourner un objet qui implémente le type [`IntoResponse`] (https://docs.rs/axum/0.6/axum/response/trait.IntoResponse.html).

`axum` fournit par exemple le type [`Json`](https://docs.rs/axum/0.6/axum/struct.Json.html) qui permet de transformer n'importe quel objet sérialisable en une réponse.

## Exemple

### Implémentation

Afin d'illustrer les différents éléments cités plus tôt, nous allons imaginer une route `/greet` qui via la méthode `GET` accepte en paramètres de requête un nom `name` et retournera un objet JSON contenant un message `greeting` qui contiendra la chaine formatée `"Hello, {}!"` avec le nom présent en paramètre.

Nous allons donc dans un premier temps définir nos types de paramètre et de réponse, implémentant respectivement désérialisation et sérialisation.

```rs
#[derive(Deserialize)]
struct GreetQuery {
    name: String,
}

#[derive(Serialize)]
struct GreetResponse {
    greeting: String,
}
```

Le type `GreetQuery` décrit donc un objet possédant un champ `name` de type `String`. Le type `GreetResponse` décrit quant à lui un objet possédant un champ `greeting` de type `String`.

We use the `Query` extractor on the `GreetQuery` type and the `Json` wrapper on the `GreetResponse` type. The handler thus looks like 

```rs
async fn greet(greet_query: Query<GreetQuery>) -> Json<GreetResponse> {
    Json(GreetResponse {
        greeting: format!("Hello, {}!", greet_query.name),
    })
}
```

Il suffit ensuite d'enregistrer la route sur notre application et notre fonction `main` ressemble désormais à

```rs
async fn main() {
    // register our "greeting" handler to the `/greet` route
    let app = Router::new().route("/greet", get(greet));

    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));
    println!("listening on http://{}", addr);
    // start the HTTP server and serve our newly created service
    axum::Server::bind(&addr)
        .serve(app.into_make_service())
        .await
        .unwrap();
}
```

### Test du service

La route `/greet` devrait donc nous répondre avec un joli message sous la forme d'un document JSON.

#### Test valide

Une requête sur la ressource <http://127.0.0.1:3000/greet?name=John%20Doe> devrait nous afficher la réponse suivante :

```json
{
  "greeting": "Hello, John Doe!"
}
```

#### Test invalide

Si on essaye une requête sans le paramètre `name` <http://127.0.0.1:3000/greet>, on observe en revanche le message d'erreur suivant :
```
Failed to deserialize query string: missing field `name`
```

On voit donc que la requête n'est pas traitée, car les conditions ne sont pas remplies. Nous verrons plus tard comment gérer ce type d'erreur afin d'afficher un message personnalisé.

#### Champ optionnel

Il aurait aussi été possible de rendre le nom optionnel en utilisant la structure :

```rs
#[derive(Deserialize)]
struct GreetQuery {
    name: Option<String>,
}
```

De cette manière, il aurait été faisable d'afficher un message par défaut, par exemple :

```rs
async fn greet(greet_query: Query<GreetQuery>) -> Json<GreetResponse> {
    Json(GreetResponse {
        greeting: format!(
            "Hello, {}!",
            greet_query
                .name
                .as_ref()
                .map(String::as_str)
                .unwrap_or("Anonymous")
        ),
    })
}
```

Au lieu de l'erreur lors de la requête sur <http://127.0.0.1:3000/greet> nous aurions alors eu la réponse suivante :
```json
{
  "greeting": "Hello, Anonymous!"
}
```

[Instantané du code](https://github.com/d34db4b3/rest-api-tutorial/tree/snapshots/greetings)

> Félicitation, l'introduction est terminée, la procahine partie est en cours de rédaction.

<!-- Now that we are done with the introduction, let's have a deeper look at error handling in the [next part](@/posts/rust-rest-api-ii.fr.md). -->