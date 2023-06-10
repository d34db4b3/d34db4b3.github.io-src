+++
title = "Un service web REST moderne, écrit en Rust, Partie I"
date = 2022-03-06 23:51:21
updated = 2023-06-06 22:33:35
authors = ["d34db4b3"]
description = "Nous allons plonger dans la conception de back-end web avec une API REST simple, en utilisant le langage de programmation fiable et performant Rust"
draft = true
[taxonomies]
tags = ["rust", "web", "backend", "REST"]
+++

# TLDR

> EN COURS DE RÉDACTION 

# Introduction

L'idée de ce projet est de mettre en place une API REST simple utilisant un certain nombre de fonctionnalités courantes. Il sera découpé en plusieurs parties afin de se pouvoir se focaliser sur chaque aspect.

Parmi elles nous verrons dans un premier temps la mise en place du serveur web, l'installation de routes et la gestion des requêtes et des réponses. Nous verrons par la suite les contraintes de restriction d'accès, de traduction et de gestion de base de données.

Nous allons donc utiliser le langage [Rust](https://www.rust-lang.org/fr), le framework [Actix](https://actix.rs/) pour gérer la partie serveur de notre API ainsi que diverses bibliothèques (crates) détaillées plus tard. Les particularités et la syntaxe de ce langage ne seront pas abordés ici.

La version `1.59.0` de Rust sera utilisé et il est supposé que les outils de compilation sont installés au préalable ([installation de Rust](https://www.rust-lang.org/fr/tools/install)).

[Visual Studio Code](https://code.visualstudio.com/) sera utilisé comme IDE tout au long du projet.

Afin de debugger et de tester notre API, le logiciel [HOPPSCOTCH](https://hoppscotch.io/) pourra être utilisé.

Pour les débutants en Rust, un tutoriel sera disponible prochainement sur ce blog. En attendant, il est conseillé de lire l'excellent [livre du Rust (en anglais)](https://doc.rust-lang.org/book/). ([version française par la communauté](https://jimskapt.github.io/rust-book-fr/))

Les particularités de sérialisation/désérialisation via [serde](https://serde.rs/) ne seront pas étudiées en détail dans cet article.

Au cours de l'article, des instantanés du projet d'exemple seront disponibles et référenceront des commits du dépôt [d34db4b3/rest-api-tutorial](https://github.com/d34db4b3/rest-api-tutorial)

# Initialisation du projet

## Création du projet

Comme pour tout projet Rust, nous allons passer par l'utilitaire [cargo](https://doc.rust-lang.org/cargo/index.html) pour initialiser les sources. Une fois placé dans le dossier où le projet va être créé, la commande suivante va générer le projet `rest-api-tutorial`.

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

Un superbe "Hello, world!" nous est présenté, mais ce n'est pas vraiment le propos de cet article !

## Ajout des librairies

Dans le fichier de manifeste `Cargo.toml`, il suffit d'ajouter la dépendance à [`actix-web`](https://docs.rs/actix-web/4/actix_web/index.html).
Pour ce faire, il faut ajouter la ligne `actix-web = "4"` dans la catégorie homonyme `[dependencies]`. Le fichier ressemble donc à

```toml
# Cargo.toml
[package]
name = "rest-api-tutorial"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4"
```

Le `"4"` représente la version de `actix-web` à utiliser. Pour plus de détails sur les dépendances et le fichier de manifeste, voir [cette page](https://doc.rust-lang.org/cargo/reference/manifest.html).

## Programme de test

Afin de vérifier la bonne configuration du manifeste et d'entamer notre aventure avec actix, le programme suivant peut être injecté dans le fichier `src/main.rs` à la place du "helloworld" de départ. Notre fichier ressemble donc à 

```rs
use actix_web::{web, App, HttpServer};

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| App::new().route("/", web::get().to(|| async { "Hello, world!" })))
        .bind(("127.0.0.1", 8080))?
        .run()
        .await
}
```

Ce programme ressemble dans sa structure au programme précédant avec sa fonction `main`, mais on observe cependant l'apparition de nouveaux mots clés tels que `async` et `await`. Ces derniers décrivent le fonctionnement asynchrone d'actix et nous reviendrons sur ces détails plus tard. Pour l'instant nous allons nous focaliser sur le comportement de l'application.

### Serveur HTTP

Tout d'abord, un serveur HTTP `HttpServer` est créé et on lui donne comme argument ce que l'on va appeler une usine à application, représentée sous la forme du lambda `|| App::...`. Un lambda est une fonction sans nom, utilisé ici afin de limiter la taille du code et la quantité d'informations présente dans un objectif de clarté. Ce dernier sera appelé lorsque le serveur sera chargé d'instancier notre application.

### Application

Afin que le serveur puisse faire son travail, on doit lui décrire son fonctionnement. C'est le rôle de l'application `App`, sur laquelle on va définir des routes, des services et ce dont elle va avoir besoin pour avoir le fonctionnement attendu. Ici, on lui demande d'enregistrer une route `/` (`App::new().route("/", ...)`) sur laquelle va être greffé une fonction qui va répondre à la méthode HTTP `GET` (`web::get().to(...)`). Cette fonction décrite par un autre lambda (`|| async { "Hello, world! }`) s'occupera simplement de retourner la chaine de caractère `"Hello, world!"`.

### Lancement du serveur

Une fois le serveur HTTP initialisé, il faut le démarrer. On spécifie donc premièrement le port et l'IP sur laquelle le serveur écoutera les requêtes (`.bind(("127.0.0.1", 8080))`), dans notre cas `127.0.0.1:8080`, et on démarre ensuite le serveur (`.run()`).

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

Le fonctionnement attendu du programme est un serveur HTTP écoutant sur le port `8080` et répondant sur la route `/` avec le message `"Hello, world!"`. À l'aide d'un navigateur tel que Firefox, une requête sur la page <http://127.0.0.1:8080/> devrait afficher le message

```
Hello, world!
```

[Instantané du code](https://github.com/d34db4b3/rest-api-tutorial/tree/6a6b9ea4e517d0c2b4bb6ed02eaa4f785cd16961)
<!-- 
<iframe src="https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&code=fn%20main()%20{%0A%20%20%20%20println!(%22Hello%2C%20world!%22)%3B%0A}%0A" style="width:100%; height:400px;">
</iframe> -->

# Analyse d'une route

## Fonction de gestion des routes

Chaque route est associée à une fonction permettant de traiter la requête et de répondre de manière adaptée. Ces fonctions appelées [`Handler`](https://docs.rs/actix-web/4/actix_web/trait.Handler.html) sont simplement des fonctions asynchrones auxquelles sont donné des arguments basés sur une requête (implémentent le trait [`FromRequest`](https://docs.rs/actix-web/4/actix_web/trait.FromRequest.html)) et renvoie un objet qui doit être convertible en réponse (implémente le trait [`Responder`](https://docs.rs/actix-web/4/actix_web/trait.Responder.html))

### Analyse de la requête

La plupart des données de la requête sont obtenues via des "extracteurs" (type implémentant le trait `FromRequest`). Plusieurs extracteurs sont fournis par actix afin de faciliter l'implémentation, mais il est possible d'en ajouter en fonction du besoin (ce que nous verrons avec les restrictions d'accès dans une prochaine partie). Parmi eux on peut citer [`Path`](https://docs.rs/actix-web/4/actix_web/dev/struct.Path.html) permettant l'extraction de paramètres dans le chemin de la ressource, [`Query`](https://docs.rs/actix-web/4/actix_web/web/struct.Query.html) qui de la même manière permet l'extraction des paramètres de la requête ([plus d'information sur les URL](https://developer.mozilla.org/fr/docs/Learn/Common_questions/What_is_a_URL)), [`Json`](https://docs.rs/actix-web/4/actix_web/web/struct.Json.html) qui permet la désérialisation du corps de requête depuis le format [JSON](https://www.json.org/json-fr.html) et enfin [`Form`](https://docs.rs/actix-web/4/actix_web/web/struct.Form.html) qui a un comportement similaire depuis le format classique de formulaire.

Ces extracteurs sont donnés en paramètre de nos fonctions de gestion des routes et nous permettent d'obtenir une extraction de donnée avec un typage statique et sûr. Les erreurs de désérialisation pourront donc être gérées convenablement et il sera donc possible d'avoir une garantie sur la présence des paramètres au bon format au préalable du traitement de la requête.

Il est aussi possible d'interagir avec les données brutes via les extracteurs [`HttpRequest`](https://docs.rs/actix-web/4/actix_web/struct.HttpRequest.html) et [`Payload`](https://docs.rs/actix-web/4/actix_web/web/struct.Payload.html), mais ce processus est fastidieux et il est préférable de passer par des extracteurs dédiés à une tache. Nous ne verrons pas ces fonctionnalités dans cet article.

### Analyse de la réponse

Afin de répondre aux requêtes, les fonctions associées aux routes doivent retourner un objet convertissable en [`HttpResponse`](https://docs.rs/actix-web/4/actix_web/struct.HttpResponse.html) (implémente le trait `Responder`). On appellera ce genre d'objet un "responder".

Il est donc possible de construire manuellement l'objet `HttpResponse`, mais il est préférable d'utiliser des types plus spécifiques afin de pouvoir, de la même manière qu'avec les extracteurs, garantir un typage statique et sûr.

Actix fournis par exemple le type [`Json`](https://docs.rs/actix-web/4/actix_web/web/struct.Json.html) qui permet de transformer tout objet sérialisable en réponse.

## Création de la route

Une fois que le handler (`handler` dans les exemples) est défini, il faut "l'attacher" à une route (`/route` dans les exemples). Deux méthodes permettent d'y parvenir.

Comme vu dans le programme de test, il est possible d'enregistrer la route directement dans la configuration de l'application. Il faut ainsi déclarer la route (`App::route("/route", ...)`) avec en premier argument le chemin correspondant, et ensuite décrire l'appel au handler. Par exemple `web::get().to(handler)` permettra d'appeler le handler dans le cas où la méthode `GET` est utilisée. Le `get` fait ici office de [`guard`](https://docs.rs/actix-web/4.0.1/actix_web/guard/index.html), sujet que nous aborderons plus tard.

Il est aussi possible d'utiliser une macro afin d'alléger le code et de transformer automatiquement le handler en service. [Ces macros](https://docs.rs/actix-web/4.0.1/actix_web/index.html#attributes) s'utilisent de la manière suivante :

```rs
#[get("/route")]
async fn handler() -> impl Responder {
    ...
}
```

Ce service sera ensuite enregistré dans l'application via `App::service(handler)`.

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

On utilise la macro `get` afin de définir le service correspondant à la route `GET /greet` de manière automatique. On utilise aussi l'extracteur `Query` sur le type `GreetQuery` et le responder `Json` sur le type `GreetResponse`. Le handler ressemble donc à 

```rs
#[get("/greet")]
async fn greet(greet_query: web::Query<GreetQuery>) -> web::Json<GreetResponse> {
    web::Json(GreetResponse {
        greeting: format!("Hello, {}!", greet_query.name),
    })
}
```

Il suffit ensuite d'enregistrer le service sur notre application et notre fonction `main` ressemble désormais à

```rs
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| App::new().service(greet))
        .bind(("127.0.0.1", 8080))?
        .run()
        .await
}
```

### Test du service

La route `/greet` devrait donc nous répondre avec un joli message sous la forme d'un document JSON.

#### Test valide

Une requête sur la ressource <http://localhost:8080/greet?name=John%20Doe> devrait nous afficher la réponse suivante :

```json
{
  "greeting": "Hello, John Doe!"
}
```

#### Test invalide

Si on essaye une requête sans le paramètre `name` <http://localhost:8080/greet>, on observe en revanche le message d'erreur suivant :
```
Query deserialize error: missing field `name`
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
#[get("/greet")]
async fn greet(greet_query: web::Query<GreetQuery>) -> web::Json<GreetResponse> {
    web::Json(GreetResponse {
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

Au lieu de l'erreur lors de la requête sur <http://localhost:8080/greet> nous aurions alors eu la réponse suivante :
```json
{
  "greeting": "Hello, Anonymous!"
}
```

[Instantané du code](https://github.com/d34db4b3/rest-api-tutorial/tree/0d827cbbea85581833085803904c93d872e9585c)


> EN COURS DE RÉDACTION