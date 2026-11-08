# Boutique Diayma

Application web ASP.NET Core MVC de gestion d'une boutique en ligne.

---

## Tâche 2 — Projets de la solution

La solution `Diayma.sln` contient **un seul projet** :

| Projet | Fichier | Type |
|--------|---------|------|
| **Diayma** | `P2FixAnAppDotNetCode/Diayma.csproj` | Application Web ASP.NET Core MVC |

---

## Tâche 3 — Version SDK .NET

Le projet cible le framework **`netcoreapp2.0`** (**.NET Core 2.0**), défini dans `Diayma.csproj` :

```xml
<TargetFramework>netcoreapp2.0</TargetFramework>
```

---

## Tâche 6 — Bugs identifiés

### Bug 1 — `GetTotalValue()` ignore la quantité (`Cart.cs`, ligne 66)

**Description :** La méthode calcule le total en additionnant uniquement les prix, sans multiplier par la quantité de chaque article.

**Code bugué :**
```csharp
return GetCartLineList().Sum(x => x.Product.Price);
```

**Exemple :** Un panier avec 3 articles à 10 000 F et 1 article à 5 000 F retourne **15 000 F** au lieu de **35 000 F**. Tous les montants affichés à l'utilisateur sont faux.

**Correction :**
```csharp
return GetCartLineList().Sum(x => x.Product.Price * x.Quantity);
```

---

### Bug 2 — Le stock peut devenir négatif (`ProductRepository.cs`, ligne 55)

**Description :** La vérification `product.Stock == 0` retire le produit uniquement quand le stock atteint exactement zéro. Si une commande dépasse le stock disponible, le stock passe en négatif et le produit reste visible et achetable.

**Code bugué :**
```csharp
product.Stock = product.Stock - quantityToRemove;
if (product.Stock == 0)
    _products.Remove(product);
```

**Exemple :** Stock = 2, commande de 5 unités → stock devient **-3**. Le produit reste en vente avec un inventaire négatif.

**Correction :**
```csharp
product.Stock = product.Stock - quantityToRemove;
if (product.Stock <= 0)
    _products.Remove(product);
```

---

## Tâche 7 — Points d'arrêt (Breakpoints)

Points d'arrêt placés sur les lignes suivantes dans VS Code (clic dans la marge gauche) :

| Fichier | Ligne | Code à cette ligne |
|---------|-------|-------------------|
| `Components/CartSummaryViewComponent.cs` | 12 | `_cart = cart as Cart;` |
| `Controllers/ProductController.cs` | 15 | `_productService = productService;` |
| `Controllers/OrderController.cs` | 17 | `_cart = pCart;` |
| `Controllers/CartController.cs` | 15 | `_cart = pCart;` |
| `Startup.cs` | 20 | `Configuration = configuration;` |

---

## Tâche 8 — Chemin d'exécution jusqu'à l'affichage des produits

### Mode de débogage utilisé

| Situation | Mode choisi | Raison |
|-----------|-------------|--------|
| Navigation dans `Startup` | **Pas à pas principal** (`F10`) | Pas besoin de rentrer dans les méthodes du framework |
| Arrivée dans `ProductController.Index()` | **Pas à pas détaillé** (`F11`) | Pour suivre l'appel à `GetAllProducts()` et voir comment les produits sont chargés |
| Sortie de `ProductRepository` | **Pas à pas sortant** (`Shift+F11`) | Pour remonter rapidement vers le contrôleur une fois la liste récupérée |

---

### Ordre d'exécution détaillé

Quand le navigateur charge la page d'accueil (`/`), voici les namespaces, classes et méthodes visités dans l'ordre :

#### 1. Démarrage de l'application

| Ordre | Namespace | Classe | Méthode | Ligne | Description |
|-------|-----------|--------|---------|-------|-------------|
| 1 | `P2FixAnAppDotNetCode` | `Startup` | `Startup(IConfiguration)` | 20 | Constructeur : injecte la configuration (breakpoint ici) |
| 2 | `P2FixAnAppDotNetCode` | `Startup` | `ConfigureServices(IServiceCollection)` | 27 | Enregistre tous les services dans le conteneur DI |
| 3 | `P2FixAnAppDotNetCode` | `Startup` | `Configure(IApplicationBuilder, IHostingEnvironment)` | 63 | Configure le pipeline HTTP (middleware) |

#### 2. Première requête — affichage de la page d'accueil

La route par défaut définie dans `Startup.Configure()` est `{controller=Product}/{action=Index}`, donc `/` pointe vers `ProductController.Index()`.

| Ordre | Namespace | Classe | Méthode | Ligne | Description |
|-------|-----------|--------|---------|-------|-------------|
| 4 | `P2FixAnAppDotNetCode.Controllers` | `ProductController` | `ProductController(IProductService, ILanguageService)` | 15 | Constructeur injecté par le DI (breakpoint ici) |
| 5 | `P2FixAnAppDotNetCode.Controllers` | `ProductController` | `Index()` | 19 | Récupère la liste des produits et retourne la vue |
| 6 | `P2FixAnAppDotNetCode.Models.Services` | `ProductService` | `GetAllProducts()` | — | Appel du service qui délègue au repository |
| 7 | `P2FixAnAppDotNetCode.Models.Repositories` | `ProductRepository` | `ProductRepository()` | — | Constructeur : initialise et génère les produits |
| 8 | `P2FixAnAppDotNetCode.Models.Repositories` | `ProductRepository` | `GetAllProducts()` | — | Retourne les produits filtrés et triés |

#### 3. Rendu de la vue — composant panier

Lors du rendu du layout, le composant `CartSummary` est invoqué pour afficher le résumé du panier en haut de page :

| Ordre | Namespace | Classe | Méthode | Ligne | Description |
|-------|-----------|--------|---------|-------|-------------|
| 9 | `P2FixAnAppDotNetCode.Components` | `CartSummaryViewComponent` | `CartSummaryViewComponent(ICart)` | 12 | Constructeur : reçoit le panier injecté (breakpoint ici) |
| 10 | `P2FixAnAppDotNetCode.Components` | `CartSummaryViewComponent` | `Invoke()` | 15 | Retourne la vue partielle avec les données du panier |

---

### Résumé visuel du flux

```
Navigateur GET /
    └─► Startup.Configure() → route vers ProductController
            └─► ProductController(IProductService) [breakpoint ligne 15]
                    └─► ProductController.Index()
                            └─► ProductService.GetAllProducts()
                                    └─► ProductRepository.GetAllProducts()
                            └─► View(products)
                                    └─► CartSummaryViewComponent(ICart) [breakpoint ligne 12]
                                            └─► CartSummaryViewComponent.Invoke()
                                                    └─► Affichage final dans le navigateur
```

---

## Structure du projet

```
P2FixAnAppDotNetCode/
├── Controllers/
│   ├── CartController.cs       # Gestion du panier (ajout, suppression)
│   ├── OrderController.cs      # Gestion des commandes
│   ├── ProductController.cs    # Affichage des produits
│   └── LanguageController.cs   # Changement de langue
├── Components/
│   ├── CartSummaryViewComponent.cs     # Résumé panier (header)
│   └── LanguageSelectorViewComponent.cs
├── Models/
│   ├── Cart.cs                 # Logique du panier
│   ├── Product.cs              # Modèle produit
│   ├── Order.cs                # Modèle commande
│   ├── Repositories/           # Accès aux données
│   └── Services/               # Logique métier
├── Views/                      # Templates Razor (.cshtml)
├── Resources/                  # Fichiers de traduction (.resx)
└── Startup.cs                  # Configuration de l'application
```
