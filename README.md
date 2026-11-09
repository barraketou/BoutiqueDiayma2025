# Boutique Diayma
**Étudiante  :** Barraketou NIASS
## 1. Récupération et exécution du code

```
git clone https://github.com/devehod/BoutiqueDiayma2025.git
cd BoutiqueDiayma2025
dotnet build
dotnet run --project P2FixAnAppDotNetCode
```

- `dotnet build` réussit, mais affiche des avertissements :
  - `NU1902` / `NU1903` : packages NuGet présentant des vulnérabilités connues (`Microsoft.AspNetCore.All 2.0.6`, `Microsoft.NETCore.App 2.0.0`). C'est l'équivalent de la fenêtre d'avertissement ⚠ de Visual Studio, ignorée comme demandé.
  - `NETSDK1138` : la version cible `netcoreapp2.0` n'est plus prise en charge.
- Le premier `dotnet run` échoue : le runtime `Microsoft.NETCore.App 2.0.0` est introuvable (seul .NET 10 était installé).
- Après installation du SDK adéquat (voir étape 4), l'application démarre et la boutique s'affiche dans le navigateur.

## 2. Projets de la solution

Commande : `dotnet sln Diayma.sln list`

La solution `Diayma.sln` contient **un seul projet** :

| Projet | Chemin | Type |
|---|---|---|
| `Diayma` | `P2FixAnAppDotNetCode\Diayma.csproj` | Application web ASP.NET Core MVC |

## 3. Version du SDK .NET utilisée

Le fichier `Diayma.csproj` indique `<TargetFramework>netcoreapp2.0</TargetFramework>` : le projet cible **.NET Core 2.0** (version sortie en 2017, en fin de support).

## 4. Installation du SDK

Installation du **SDK .NET Core 2.1.202**, le dernier SDK de la branche 2.0, qui inclut le runtime **.NET Core 2.0.9**.

Vérification :

```
> dotnet --list-sdks
2.1.202 [C:\Program Files\dotnet\sdk]
10.0.403 [C:\Program Files\dotnet\sdk]

> dotnet --list-runtimes
Microsoft.NETCore.App 2.0.9 [...]
Microsoft.NETCore.App 10.0.14 [...]
```

Plusieurs versions de .NET cohabitent sur la même machine. L'application demande la version `2.0.0` et utilise automatiquement le dernier correctif installé (`2.0.9`).

## 5. Dépôt GitHub personnel

```
git remote rename origin upstream
git remote add origin https://github.com/barraketou/BoutiqueDiayma2025.git
git push -u origin master
```
## 6. Bugs trouvés

### Bug n°1 : le stock ne diminue pas après une commande

- **Étapes :** noter le stock d'un produit (ex. 20) → l'ajouter au panier → valider la commande → revenir à l'accueil.
- **Attendu :** le stock diminue de la quantité commandée (ex. 19).
- **Obtenu :** le stock reste à 20.
- **Conséquence :** la boutique peut vendre des produits qui ne sont plus disponibles.

### 🐛 Bug n°2 : le passage à l'espagnol ne fonctionne pas

- **Étapes :** choisir **Español** dans le sélecteur de langue.
- **Attendu :** l'interface s'affiche en espagnol.
- **Obtenu :** les textes restent en français.
- **Cause identifiée pendant le débogage :** dans `Startup.ConfigureServices`, la liste `supportedCultures` ne déclare que `en-GB`, `en-US`, `en`, `fr-FR` et `fr`. La culture espagnole (`es`) est absente, donc elle est refusée par l'application.

## 7. Points d'arrêt

| Fichier | Ligne | Code | Rôle |
|---|---|---|---|
| `Startup.cs` | 20 | `Configuration = configuration;` | Stocke la configuration au démarrage |
| `ProductController.cs` | 15 | `_productService = productService;` | Reçoit le service des produits |
| `CartSummaryViewComponent.cs` | 12 | `_cart = cart as Cart;` | Reçoit le panier |
| `CartController.cs` | 15 | `_cart = pCart;` | Reçoit le panier |
| `OrderController.cs` | 17 | `_orderService = service;` | Reçoit le service des commandes |

Les 5 points d'arrêt se trouvent dans des **constructeurs**. Aucun code de l'application n'appelle ces constructeurs : c'est ASP.NET Core qui crée les objets et leur **injecte** les services enregistrés dans `Startup.ConfigureServices` (`AddSingleton`, `AddTransient`). C'est l'**injection de dépendances**.

Au chargement de la page d'accueil, seuls les points d'arrêt de `Startup`, `ProductController` et `CartSummaryViewComponent` sont atteints. Ceux de `CartController` et `OrderController` ne le sont qu'en ouvrant le panier ou la page de commande.

Pour suivre le parcours complet, des points d'arrêt supplémentaires ont été ajoutés au début de `Startup.Configure`, `ProductController.Index` et `ProductRepository.GetAllProducts`.

## 8. Parcours jusqu'à l'affichage des produits

Débogage lancé avec **F5** dans VS Code (configuration « C#: Launch Startup Project »).

Raccourcis utilisés : **F5** Continuer · **F10** Pas à pas principal · **F11** Pas à pas détaillé · **Maj+F11** Pas à pas sortant.

### A. Démarrage de l'application

| # | Namespace | Classe | Méthode | Mode utilisé |
|---|---|---|---|---|
| 1 | `P2FixAnAppDotNetCode` | `Program` | `Main` puis `BuildWebHost` (visibles dans la pile d'appels) | — |
| 2 | `P2FixAnAppDotNetCode` | `Startup` | constructeur `Startup(IConfiguration)` | Arrêt sur point d'arrêt, puis **pas à pas principal** (simple affectation, rien à détailler) |
| 3 | `P2FixAnAppDotNetCode` | `Startup` | `ConfigureServices(IServiceCollection)` | **Pas à pas principal** : enregistre les services (panier, langues, produits, commandes), la localisation, la session et MVC |
| 4 | `P2FixAnAppDotNetCode` | `Startup` | `Configure(IApplicationBuilder, IHostingEnvironment)` | **Continuer** jusqu'au point d'arrêt : met en place le pipeline (`UseStaticFiles`, `UseRequestLocalization`, `UseSession`, `UseMvc`) |

La route définie dans `Configure` est :

```csharp
template: "{controller=Product}/{action=Index}/{id?}"
```

La page d'accueil (`/`) est donc traitée par la méthode `Index` de `ProductController`.

### B. Requête du navigateur sur la page d'accueil

Pendant l'arrêt, la page reste blanche : le serveur est en pause et n'a pas encore répondu.

| # | Namespace | Classe | Méthode | Mode utilisé |
|---|---|---|---|---|
| 5 | `P2FixAnAppDotNetCode.Controllers` | `ProductController` | constructeur `ProductController(IProductService, ILanguageService)` | **Continuer** jusqu'à `Index` |
| 6 | `P2FixAnAppDotNetCode.Controllers` | `ProductController` | `Index()` | **Pas à pas détaillé** sur `_productService.GetAllProducts()` pour entrer dans notre propre code |
| 7 | `P2FixAnAppDotNetCode.Models.Services` | `ProductService` | `GetAllProducts()` | **Pas à pas détaillé** sur `_productRepository.GetAllProducts()` |
| 8 | `P2FixAnAppDotNetCode.Models.Repositories` | `ProductRepository` | `GetAllProducts()` | **Pas à pas principal** : la ligne n'appelle que des méthodes LINQ de .NET (`Where`, `OrderBy`, `ToList`), inutile d'y entrer. Puis retour automatique vers les méthodes appelantes (**pas à pas sortant** possible avec Maj+F11) |
| 9 | `P2FixAnAppDotNetCode.Controllers` | `ProductController` | `Index()` (retour) | **Pas à pas principal** : la variable `products` contient **9 produits** ; `return View(products)` envoie la liste à la vue |
| 10 | `P2FixAnAppDotNetCode.Components` | `CartSummaryViewComponent` | constructeur, puis `Invoke()` | Arrêt sur point d'arrêt pendant la génération de la page : affiche le résumé du panier dans l'en-tête |

Après un dernier **Continuer**, la page s'affiche avec les produits. Les fichiers statiques (CSS, polices, `favicon.ico`) sont servis directement par `StaticFileMiddleware` (`app.UseStaticFiles()`), sans passer par un contrôleur.

### Architecture observée

```
Navigateur ── GET / ──► ProductController.Index()          (Controllers)
                              │
                              ▼
                        ProductService.GetAllProducts()    (Models.Services)
                              │
                              ▼
                        ProductRepository.GetAllProducts() (Models.Repositories)
                              │  produits avec Stock > 0, triés par nom
                              ▼
                        View(products) + CartSummaryViewComponent ──► page HTML
```

### Choix du mode de pas à pas

- **Pas à pas détaillé (F11)** : quand la ligne appelle une méthode **de notre code** dont on veut voir l'intérieur (`GetAllProducts` du service, puis du repository).
- **Pas à pas principal (F10)** : quand la ligne est une simple affectation ou appelle du code du framework .NET (LINQ, `View()`), pour ne pas se perdre dans du code qui n'est pas le nôtre.
- **Pas à pas sortant (Maj+F11)** : pour terminer la méthode en cours et remonter directement vers la méthode appelante.

