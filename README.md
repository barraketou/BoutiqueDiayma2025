# Boutique Diayma 2025

**Étudiante :** Barraketou NIASS

## 1. Récupération et exécution

```
git clone https://github.com/devehod/BoutiqueDiayma2025.git
cd BoutiqueDiayma2025
dotnet build
dotnet run --project P2FixAnAppDotNetCode
```

La compilation réussit avec des avertissements : `NU1902` / `NU1903` signalent des packages NuGet vulnérables (c'est l'avertissement ⚠ de Visual Studio, ignoré comme demandé), et `NETSDK1138` indique que `netcoreapp2.0` n'est plus supporté.
Le premier `dotnet run` échoue, car le runtime .NET Core 2.0 est absent. Après l'installation du SDK (étape 4), la boutique s'affiche.

## 2. Projets de la solution

`dotnet sln Diayma.sln list` → un seul projet : **Diayma** (`P2FixAnAppDotNetCode\Diayma.csproj`), une application web ASP.NET Core MVC.

## 3. Version du SDK

`Diayma.csproj` cible **.NET Core 2.0** (`netcoreapp2.0`).

## 4. Installation du SDK

Installation du **SDK .NET Core 2.1.202**, qui contient le runtime **2.0.9**. Vérification avec `dotnet --list-sdks` : les SDK `2.1.202` et `10.0.403` cohabitent sur la machine.

## 5. Dépôt GitHub

```
git remote rename origin upstream
git remote add origin https://github.com/barraketou/BoutiqueDiayma2025.git
git push -u origin master
```

## 6. Bugs trouvés

**Bug 1 – Le stock ne diminue pas après une commande.**
Un produit en stock à 20 reste à 20 après avoir été commandé. La boutique peut donc vendre des produits indisponibles.

**Bug 2 – L'espagnol ne fonctionne pas.**
En choisissant Español, les textes restent en français.
*Cause :* dans `Startup.ConfigureServices`, la liste `supportedCultures` ne contient que `en` et `fr` : la culture `es` n'y est pas déclarée.

## 7. Points d'arrêt

- `Startup.cs` ligne 20 : `Configuration = configuration;`
- `ProductController.cs` ligne 15 : `_productService = productService;`
- `CartSummaryViewComponent.cs` ligne 12 : `_cart = cart as Cart;`
- `CartController.cs` ligne 15 : `_cart = pCart;`
- `OrderController.cs` ligne 17 : `_orderService = service;`

Ces lignes sont toutes dans des **constructeurs** : ASP.NET Core crée ces objets et leur **injecte** les services enregistrés dans `ConfigureServices` (injection de dépendances).

## 8. Parcours avant l'affichage des produits

**Démarrage de l'application**
1. `P2FixAnAppDotNetCode` › `Program` › `Main`, puis `BuildWebHost`
2. `P2FixAnAppDotNetCode` › `Startup` › constructeur — *pas à pas principal* (simple affectation)
3. `P2FixAnAppDotNetCode` › `Startup` › `ConfigureServices` — enregistre les services
4. `P2FixAnAppDotNetCode` › `Startup` › `Configure` — met en place le pipeline et la route par défaut `{controller=Product}/{action=Index}/{id?}`

**Requête de la page d'accueil**
5. `P2FixAnAppDotNetCode.Controllers` › `ProductController` › constructeur
6. `P2FixAnAppDotNetCode.Controllers` › `ProductController` › `Index()` — *pas à pas détaillé* pour entrer dans `GetAllProducts()`
7. `P2FixAnAppDotNetCode.Models.Services` › `ProductService` › `GetAllProducts()` — *pas à pas détaillé*
8. `P2FixAnAppDotNetCode.Models.Repositories` › `ProductRepository` › `GetAllProducts()` — *pas à pas principal* (code LINQ du framework), puis *pas à pas sortant* pour remonter
9. Retour dans `ProductController.Index()` : `products` contient **9 produits**, envoyés à la vue
10. `P2FixAnAppDotNetCode.Components` › `CartSummaryViewComponent` › constructeur, puis `Invoke()` — affiche le résumé du panier

**Choix des modes**
- *Pas à pas détaillé (F11)* : pour entrer dans une méthode de notre code.
- *Pas à pas principal (F10)* : pour exécuter une ligne sans y entrer (affectation, code .NET).
- *Pas à pas sortant (Maj+F11)* : pour finir la méthode et remonter à l'appelant.

## 9. Publication Windows

```
dotnet publish P2FixAnAppDotNetCode -c Release -r win-x64 --self-contained true -o ..\DiaymaPublish
```

Publication **autonome (SCD)** : le runtime .NET est inclus, donc l'exécutable fonctionne sur n'importe quel PC Windows 64 bits sans installation.

## 10. Lien vers l'exécutable

📦 [Télécharger DiaymaPublish.zip](https://drive.google.com/file/d/1v0pQW9PVIgoWypQALm-wiEVEBDYYUBNl/view?usp=sharing)
