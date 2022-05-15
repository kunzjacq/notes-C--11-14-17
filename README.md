- [Notes C++11/14/17](#notes-c111417)
  - [Catégories de valeurs des expressions](#catégories-de-valeurs-des-expressions)
    - [Exemples d'expressions *lvalue*](#exemples-dexpressions-lvalue)
    - [Exemples d'expressions *rvalue*](#exemples-dexpressions-rvalue)
    - [Catégorie de valeur et `decltype`](#catégorie-de-valeur-et-decltype)
    - [Catégorie de valeur et surcharges de templates](#catégorie-de-valeur-et-surcharges-de-templates)
  - [Déduction de type dans les templates](#déduction-de-type-dans-les-templates)
    - [Références universelles](#références-universelles)
    - [Cas particuliers](#cas-particuliers)
  - [Règles de priorité pour le choix d'un template](#règles-de-priorité-pour-le-choix-dun-template)
  - [Rapport entre déduction de types dans les templates et avec `auto`](#rapport-entre-déduction-de-types-dans-les-templates-et-avec-auto)
  - [*Reference collapsing*](#reference-collapsing)
  - [*rvalue references*, `std::move`, `std::forward`](#rvalue-references-stdmove-stdforward)
    - [`std::move`](#stdmove)
    - [`std::forward`](#stdforward)
  - [Syntaxes alternatives pour le type de retour ou pour les paramètres dans les fonctions](#syntaxes-alternatives-pour-le-type-de-retour-ou-pour-les-paramètres-dans-les-fonctions)
    - [`auto`](#auto)
    - [*trailing return type*, `decltype(auto)`](#trailing-return-type-decltypeauto)
  - [Avantages / désavantages d'`auto`](#avantages--désavantages-dauto)
  - [Constructeurs, `initializer_list<T>`, `universal initialization`](#constructeurs-initializer_listt-universal-initialization)
  - [`nullptr`, `NULL`, `0`](#nullptr-null-0)
  - [`using a = b` vs `typedef b a`](#using-a--b-vs-typedef-b-a)
  - [*type traits*](#type-traits)
  - [`enums` délimités (`scoped enums`)](#enums-délimités-scoped-enums)
  - [Usage de `=delete`](#usage-de-delete)
  - [*reference qualifiers*](#reference-qualifiers)
  - [`override`](#override)
  - [*const_iterators*](#const_iterators)
  - [`noexcept`, `noexcept(bool)`, `noexcept(expr)`](#noexcept-noexceptbool-noexceptexpr)
  - [`constexpr`](#constexpr)
  - [Fonctions / constructeurs générés automatiquement](#fonctions--constructeurs-générés-automatiquement)
  - [`unique_ptr`](#unique_ptr)
  - [`shared_ptr`](#shared_ptr)
  - [`weak_ptr`](#weak_ptr)
  - [`std::make_unique`, `std::make_shared`](#stdmake_unique-stdmake_shared)

# Notes C++11/14/17

Références principales :
  * "Effective Modern C++", Scott Meyers
  * cppreference.com
  * [C++ for C# developers](https://www.jacksondunstan.com/articles/5530)
  * [RValues references explained, Thomas Becker](http://thbecker.net/articles/rvalue_references/section_01.html)
  * posts de blog divers (Herb Sutter / GotW, stackoverflow, etc.)


[//]: # (pour faire un lien interne: un seul # quel que soit le niveau du lien, supprimer la première majuscule.)

## Catégories de valeurs des expressions

Références :

  * [Value Categories in C++17, Barry Revzin](https://medium.com/@barryrevzin/value-categories-in-c-17-f56ae54bccbe)
  * https://en.cppreference.com/w/cpp/language/value_category

Une expression possède à la fois un *type* et une *catégorie de valeur*. Cette catégorie dépend de deux propriétés de l'expression : 
 * définit-elle une entité avec un emplacement mémoire bien défini indépendamment de son usage ? A l'opposé, l'entité représentée peut-elle être créée en place là où elle va être utilisée, comme dans le cas d'un entier littéral ?
 * peut-elle être *déplacée*, c'est à dire réutilisée dans une autre variable, et détruite au passage ? 

 Une valeur qui n'a pas d'emplacement mémoire bien défini peut être déplacée (en fait directement formée en place, mais cela revient au même), donc la combinaison d'attributs "ne possède pas d'emplacement mémoire bien défini" et "ne peut pas être déplacée" n'existe pas. En conséquence, il y a 3 catégories de valeur sur la base de ces deux critères :

 |                      | emplacement bien défini | non       | oui      |          |
 |----------------------|-------------------------|-----------|----------|----------|
 |**peut être déplacé** |                         |           |          |          |
 | **non**              |                         |           | `lvalue` |          |
 | **oui**              |                         | `prvalue` | `xvalue` | `rvalue` |
 |                      |                         |           | `glvalue`|          |

Une expression est donc une *rvalue* ou une *lvalue* (elle peut être déplacée ou non), et également une une *glvalue* ou une *prvalue* (elle définit une entité preexistante en mémoire ou non). Une *glvalue* nécessitera un *move* un *copy* pour être utilisée (dans une affectation, un appel de fonction avec passage par valeur), alors qu'une *prvalue* sera construite en place sans *copy* ni *move* (*guaranteed copy elision*, depuis C++17, qui standardise une comportement qu'avaient déjà les compilateurs en pratique mais qui n'était pas garanti).

Une expression *lvalue* n'est pas, en général, une expression dans laquelle on peut écrire quelque chose. Elle peut être `const`, désigner une fonction... 

### Exemples d'expressions *lvalue* 

Ce qui désigne des entités pouvant être accédées en dehors de l'expression considérée : en règle générale, tout ce qui est nommé (variable, fonction, membre d'une variable, pointeur déréférencé, élément d'un tableau lui-même *lvalue*...). Voir description exacte sur *cppreference.com*.

### Exemples d'expressions *rvalue*

Les littéraux ; les expressions non nommées (résultat de l'appel d'une fonction, de l'application d'un opérateur...) ; n'ayant pas de nom, elles ne peuvent être réutilisées en dehors du contexte où elles apparaissent. Ces exemples sont des *prvalues*.

Une valeur de retour de fonction **qui est une variable locale de celle-ci** (cela inclut les arguments passés par valeur, mais exclut les arguments passés par référence) 
  * est éligible au RVO (*return-value optimization*) : la valeur sera construite si possible directement en place
  * si elle n'est pas construite en place, sera considérée comme une *rvalue*, et pourra donc être déplacée (et non copiée) dans son emplacement final. (c'est pourquoi il ne faut pas appliquer `std::move` à une telle variable, ce qui ne sert à rien et peut empêcher le compilateur d'appliquer la RVO) 

Une *xvalue* (donc *rvalue*) peut être obtenue 
  * soit par conversion d'une *lvalue* en *rvalue reference* par `std::move`, qui revient à affirmer que son argument peut être déplacé sans changer son statut vis-à-vis de son emplacement mémoire, 
  * ou par extraction d'un membre ou d'un élément de tableau d'une *prvalue*. Dans ce cas, l'ensemble de la *prvalue* devant être construit avant de pouvoir en extraire une partie, la valeur extraite ne peut pas être construite en place ; elle peut en revanche être déplacée comme son entité parente, et est donc bien une *xvalue*.

Une variable de type *rvalue reference* ne définit *pas* une expression *rvalue*. Dans

    std::vector<int>&& x = std::vector<int>({});
  ou

    int && x = 2;
  
l'expression `x` est une *lvalue*.

### Catégorie de valeur et `decltype`

Voir [https://en.cppreference.com/w/cpp/language/decltype](https://en.cppreference.com/w/cpp/language/decltype).

Le type d'une expression `e` peut être obtenu par `decltype((e))`, tandis que le type d'une variable `x` peut être obtenu par `decltype(x)`. Dans le cas d'une expression, `&` et `&&` n'ont pas de signification relative aux références dans le retour de `decltype`: 

**Dans le cas d'une expression, le type renvoyé par `decltype` est relié à la catégorie de valeur de l'expression en argument :**
  * *rvalue reference* (&&) pour une `xvalue`;
  * *lvalue reference* (&) pour une `lvalue`;
  * *non-reference* pour une `prvalue`.

En conséquence, pour une variable `x`, `decltype(x)` et `decltype((x))` diffèrent en général.

### Catégorie de valeur et surcharges de templates

La catégorie de valeur d'une expression est associée aux types auxquel elle peut être liée dans un fonction ou une classe templatée :

 |                           | emplacement bien défini | non | oui    |
 |---------------------------|-------------------------|-----|--------|
 |**peut être déplacé**      |                         |     |        |
 | **non**                   |                         |     | `T &`  |
 | **oui**                   |                         | `T` | `T &&` |

Ce lien doit être fait par une instantiation explicite `c<decltype((e))>` et non implicite `f(e)`, sans quoi le choix du template à instancier est ambigu. Voir les [règles de priorité pour le choix d'un template](#règles-de-priorité-pour-le-choix-dun-template).

NB : une fonction `template<class T> f(T&)` ne peut se lier qu'à une *lvalue*, contrairement à une fonction `template<class T> f(const T&)` qui peut se lier à une *lvalue* et à une *rvalue* ([most important const](https://herbsutter.com/2008/01/01/gotw-88-a-candidate-for-the-most-important-const/))

Lorsque 3 surcharges d'une même fonction ou classe existent, en `T`, `T&&` et `T&`, les catégories de valeur déclenchent le choix de la surchage indiquée par le tableau ci-dessus. En cas d'absence d'une surcharge `T &` ou `T &&`, la surchage correspondante `T` (moins spécialisée) sera utilisée à la place.

## Déduction de type dans les templates

Soit une fonction templatée

`template <typename T1, typename T2, ...> f(ParamType param)`

où `ParamType` dépend de `T1`, `T2`... (`const T1&`, `T1*`, `std::vector<T1>&`, `std::tuple<T1,T2>`...)

**Connaissant le type de `expr`, quel est le type déduit pour `ParamType`, et `T1`, `T2`... dans l'expression `f(expr)` ?**

Référence pour ce paragraphe : https://en.cppreference.com/w/cpp/language/template_argument_deduction

Sauf dans le cas où `ParamType` est une référence universelle (voir plus bas), le fait que le type de `expr` soit un type référence ou non ne joue aucun rôle. En effet, c'est `ParamType` qui détermine si un passage par valeur ou par référence va avoir lieu. 

Règles (simplifiées, voir la référence ci-dessus sur cppreference.com) :

  * Si `ParamType` n'est pas un type référence, le passage est par valeur, et les attributs *const* et *volatile* éventuels de `decltype(expr)` sont ignorés puisqu'ils ne s'appliquent pas à la copie ; `T1`, `T2`... sont choisis de sorte que le type de `ParamType` soit identique au type `remove_cv_t<remove_reference_t<decltype(expr)>>`. (voir [Type traits](#type-traits))
  * Si `ParamType` est un référence qui n'est pas une référence universelle, le passage est par référence, `T1`, `T2`... sont choisis de sorte que le type de `remove_reference_t<ParamType>` soit identique à `remove_reference_t<decltype(expr)>`. 
  * Si `ParamType` est une référence universelle,
    * si `expr` est une *lvalue*, `remove_reference_t<ParamType>` est mis en correspondance avec `decltype(expr)&` ;
    * sinon la règle normale s'applique. 
    Ces règles associées au *reference collapsing* font fonctionner le *reference forwarding* dans les références universelles (voir plus bas). 

> La description ci-dessus provient de cppreference.com et diffère de celle de *Effective Modern C++* qui semble fausse dans le cas des références. *Effective Modern C++* traite les références universelles comme un cas complètement à part, alors que les règles ci-dessus fonctionnent également dans ce cas. De plus, *Effective Modern C++* indique que `const` et `volatile` sont ignorés si `ParamType` n'est ni une référence, ni un type pointeur. Mais il s'avère que si `ParamType` est `T*`, et si `expr` est de type `int* const &`, le *const* est bien ignoré, ce qui est logique, puisque le pointeur est lui-même copié. Il y a un *erratum* du livre à ce sujet.

### Références universelles

Les références universelles sont les références déclarées avec l'expression `T&&`, où `T` est un paramètre de template déduit (mais rien d'autre, par exemple `std::vector<T>&&` n'est *pas* une référence universelle; `const T&&` non plus). La présence d'un paramètre de template est nécessaire : par exemple, une référence `int &&` n'est pas une référence universelle et ne pourra se lier qu'à une *rvalue*. 

Une référence universelle peut se lier à une *rvalue* ou à une *lvalue*, et se comporte comme un paramètre `T&` dans le cas d'une *lvalue*. Autrement dit, si `ParamType = T&&` est une référence universelle, et si `expr` est une lvalue de type `S&`, `T=S&` et `ParamType = T&& = S&`. Si `expr` est une *rvalue reference* de type `S&&`, `ParamType = S&&`, `T=S`. si `expr` est une *rvalue*, par exemple un littéral (*prvalue*), de type `S` non référence, on a également `ParamType = S&&` et `T=S`.

On retrouve bien ce comportement en application des règles énoncées plus haut : ici `remove_reference_t<ParamType>=T`. Si `S` est un type non-référence, si l'argument est une *lvalue* de type `S`, `T` doit être choisi de sorte que `remove_reference_t<ParamType>=S&`, donc `T=S&`. Alors `ParamType = T&& = S& && = S&` en vertu du [reference collapsing](#reference-collapsing). Idem si l'argument est `S&`. Si l'argument est `S&&`, on doit assurer `remove_reference_t<ParamType>=S`, donc on a bien `T=S` et `ParamType=S&&`.

### Cas particuliers
  * Si `expr` est de type tableau `S[N]`, et `ParamType=T` n'est pas un type référence, `ParamType` est déduit comme un type pointeur et non comme un type tableau. En revanche, Si `ParamType=T&`, `ParamType` est déduit comme une référence sur un type tableau `S(&)[N]`, et `T=S[N]`. On peut d'ailleurs capturer la taille du tableau, par une déclaration du type `template<class T, std::size_t N> std::size_t f(T(&)[N])`.
  * De façon similaire, si `expr` est une fonction, une capture par valeur conduit à la déduction du type pointeur de fonction correspondant `return-type (*) args` ; une capture par référence conduit à la déduction de la référence sur le type de fonction correspondant `return-type (&) args`.

NB  : depuis C++17 il est possible de déduire automatiquement des types de classe templatée, pas seulement des types de fonctions templatées.

## Règles de priorité pour le choix d'un template

Les bases de l'algorithme sont simples :
  * filtrer les templates (ou fonctions non template) applicables, et produire une erreur si l'ensemble est vide ;
  * les classer selon un ordre partiel, avec déclenchement d'erreur s'il y a plus d'un meilleur élément (ambiguité du meilleur template). L'ordre est un ordre lexicographique issu de plusieurs critères de priorité décroissante définissant chacun un ordre partiel : si le premier critère définit un ordre strict entre deux éléments, ces éléments sont ordonnés suivant cette relation, sinon on passe au second critère, etc.

Pour une discussion plus approfondie, voir 
  * https://stackoverflow.com/questions/31047860/c-template-functions-priority
  * https://stackoverflow.com/questions/17005985/what-is-the-partial-ordering-procedure-in-template-deduction
  * https://blog.tartanllama.xyz/function-template-partial-ordering/ qui explique pourquoi il n'y a pas de relation d'ordre de spécialisation entre `template<class T> f(T)` et `template<class T> f(T&)`. 

En particulier, il est souvent dit que le template choisi et le template le plus spécialisé correspondant à l'appel effectué ; c'est faux dans le sens où le critère de plus grande spécialisation n'est pas le premièr critère appliqué. Viennent avant celui-ci le critère concernant les conversions nécessaires à l'application d'un template : un template nécessitant moins de conversions (notamment en matière de *constness*) sera choisi avant d'arriver à l'examen du niveau de spécialisation de ces templates.

Une expression de type `S&&` peut se lier à un type `T&&` ou `const T&` et se liera préférentiellement au premier en cas de surcharge multiple ; la seconde possibilité correspond au [most important const](https://herbsutter.com/2008/01/01/gotw-88-a-candidate-for-the-most-important-const/). De même une référence `S&` peut se lier à `T&`, `T&&` (référence universelle), ou `const T&`. En cas d'absence de surcharge `const T&`, une référence `const S&` se liera à un surchage `T&` avec `T=const S` (c'est un cas particulier des règles décrites plus haut). Enfin une surchage de type `T` est admissible pour tous ces usages et peut induire ambiguïtés avec `T&&` et `T&` qui feront des erreurs à la compilation.

## Rapport entre déduction de types dans les templates et avec `auto`

La déduction de type en présence d'une définition `auto` est identique à un cas près de celle des templates : `auto x = expr` effectue la même déduction que lorsque `expr` est utilisé pour `x` dans `template<class T> f(T x)`, `auto& x = expr` se comporte comme l'utilisation de `expr` dans `template<class T> f(T& x)`, etc.

Seule différence : quand elle est initialisée avec une `std::initializer_list<T>` fait pour initialiser les conteneurs, (`e.g. std::vector v({0,7,1})`), une variable `auto` obtient ce type, comme dans 

  ~~~
  #include<initializer_list> // nécessaire
  auto x = {0,2,3}; // x est un std::initializer_list<int>
  ~~~

alors qu'appeler un template avec un tel argument n'est pas autorisé.

En revanche, cela fonctionne 
  * si le template demande explicitement un `initializer_list` :
  ~~~
  template<T> void f(std::initializer_list<T>& x);
  f({0,2,3}); // OK
  ~~~
  * ou si le type du paramètre de template est explicité au moment de l'appel : 
  ~~~
  template<class T> f(T&& x);
  f<std::initializer_list<int>>({0,2,3}); // OK
  ~~~

Attention : Il ne faut pas confondre la syntaxe précédente avec `auto` avec les exemples d'*uniform initialization* ci-dessous. Voir [le paragraphe qui est consacré à cette syntaxe](#constructeurs-initializer_listt-universal-initialization)

  ~~~
  auto x{1}; // ici x est un int 
  auto x{1,2}; // erreur de compilation, int n'a pas de constructeur à deux arguments
  ~~~

## *Reference collapsing*
Si `T` n'est pas un type référence, 
  * `T& && = T&& & = T& & = T&` 
  * `T&& && = T&&`

Autrement dit, si une référence désigne une entité comme non *movable*, prendre une *rvalue reference* sur celle-ci ne permettra pas de le considérer *movable*. 

NB: il n'est pas licite de construire explicitement une *rvalue reference* sur une *lvalue*, en revanche les règles ci-dessus sont employées dans la déduction de type des templates, des variables *auto*, dans les typedefs et déclarations d'alias de type (avec `using`), et dans les expressions de type utilisant `decltype` : dans toutes ces situations, un type (désigné par une variable de template, `auto`, un alias de type, ou une expression `decltype`) peut être modifié par `&` ou `&&` pour obtenir un nouveau type selon les règles ci-dessus.

## *rvalue references*, `std::move`, `std::forward`

Références :
  * http://thbecker.net/articles/rvalue_references/section_01.html
  * https://en.cppreference.com/w/cpp/utility/move
  * https://en.cppreference.com/w/cpp/utility/forward
  * *Effective Modern C++*, paragraphes 1, 23, 24, 28, 30.

### `std::move`

Conversion réalisée par `std::move` :
  ~~~
  template<class T>
  decltype(auto) move(T&& param)
  {
    using ReturnType = remove_reference_t<T>&&;
    return static_cast<ReturnType>(param);
  }
  ~~~

L'utilisation de `remove_reference_t<T>` est là pour garantir la conversion *inconditionnelle* en *rvalue reference*. Sans elle, les règles de *reference collapsing* s'appliqueraient lorsque *T* est serait une *lvalue reference*.

### `std::forward`

`std::forward` est fait pour transmettre une *universal reference* à une sous-fonction. Dans ce contexte, `std::forward` convertit conditionnellement son argument qui est une *lvalue* en *rvalue reference*.
 
  ~~~
  // sans std::forward
  template <class T> void f(T&& x){
    g(x); // même si x est une rvalue reference, g sera appelé avec une lvalue
  }
  // avec std::forward
  template <class T> void f(T&& x){
    g(std::forward<T>(x)); // si x est une rvalue reference, g sera appelé avec une rvalue
  }
  ~~~

**`std::forward` requiert que le type du paramètre concerné (ci-dessus `T`) lui soit fourni explicitement.** C'est ce qui permet son fonctionnement correct. En effet, l'expression `x` est une *lvalue* quel que soit le type de `x`, et `std::forward` utilise `T` pour effectuer conditionnellement la conversion nécessaire.

En négligeant ce qui permet de forcer le passage explicite du type de template, `std::forward` est équivalent au code suivant dans les deux cas "normaux" d'utilisation de `std::forward` avec une référence universelle :

  ~~~
  template<class T>
  T&& forward(T&& param)
  {
    return static_cast<T&&>(param);
  }
  ~~~

Ces deux cas normaux sont les suivants : avec `S` type non référence, et `v` de type `S&`,
  * `T = S`  (cas *rvalue*) : `forward<S>  (v)` est `static_cast<S&&> (v)`,
  * `T = S&` (cas *lvalue*) : `forward<S&> (v)` est `static_cast<S&>  (v)`.

Comme on peut le vérifier dans ces deux cas, ce sont les règles de *reference collapsing* qui asssurent le bon fonctionnement de `std::forward`. 
  
En pratique cependant, au lieu de l'implémentation ci-dessus, deux surcharges sont présentes : 

  ~~~
  // la seule surcharge utilisée normalement avec une référence universelle
  template <class T>
  inline T&& forward(typename std::remove_reference<T>::type& t) noexcept
  {
    return static_cast<T&&>(t);
  }

  // une surcharge utilisée si l'argument est une rvalue reference
  // NB l'argument de ce template n'est pas une référence universelle
  template <class T>
  inline T&& forward(typename std::remove_reference<T>::type&& t) noexcept
  {
    static_assert(!std::is_lvalue_reference<T>::value, "Can not forward an rvalue as an lvalue.");
     return static_cast<T&&>(t);
  }
  ~~~

 Ces implémentations se comportent comme la fonction ci-dessus dans les cas normaux mais interdisent les utilisations pathologiques, comme `forward<int&>(5)` ou `forward<std::string&>(std::string{})` qui vont déclencher le `static_assert`. Voir cette [discussion sur stackoverflow](https://stackoverflow.com/questions/27501400/the-implementation-of-stdforward).

 `std::forward` ne permet pas de tout forwarder. Il n'est pas utilisable dans les cas suivants :
   * le passage d'une fonction ayant plusieurs surcharges à une fonction attendant un pointeur de fonction : l'appel à `std::forward` masque le type exact de pointeur attendu, et la bonne surcharge ne peut être sélectionnée ; solution : pré-convertir explicitement la fonction en le type de pointeur de fonction attendu ;
   * les *bit fields*, qui ne peuvent être liés à des références non `const` ; solution : les pré-convertir en entiers ;
   * le passage de 0 / NULL à une fonction attendant une adresse : la valeur sera interprété comme un entier, ce qui ne fonctionnera pas ; solution : utiliser `nullptr` ;
   * le passage d'un membre entier `static const` déclaré mais non défini (ce qui est licite tant que l'on ne prend pas de référence sur cet entier) : le passage par `std::forward` nécessite une prise de référence ce qui oblige à définir un stockage pour la valeur dans un fichier *.cpp* :

  ~~~
  // dans .h
  class x {
    static const int y = 2; // déclaration
  };
  // dans un .cpp incluant le .h ci-dessus
  const int x::y; // définition 
  ~~~

## Syntaxes alternatives pour le type de retour ou pour les paramètres dans les fonctions

### `auto`

A partir de C++14, `auto` peut être utilisé 
  * comme type de retour pour une fonction
  * comme type pour le paramètre d'une fonction lambda. La déduction se comporte alors comme dans le cas des templates (en particulier, les `std::initializer_list` sont interdits).

`auto` comme paramètre pour une lambda a un usage similaire à un paramètre de template : permettre d'adapter le type employé à l'usage du lambda, comme dans 

  ~~~
  
  auto compare =
    [] (const auto& p1, const auto& p2)
    { return *p1 < *p2; }; 

  ~~~

Comme `auto` utilisé comme type de retour applique les règles de déduction de types des templates, il ne permet pas de retourner une référence, puisque le type référence d'une expression est ignoré lors de la déduction de type d'un template.

### *trailing return type*, `decltype(auto)`

Deux autres syntaxes existent pour le type de retour :
  * *trailing return type* (C++11) : type de retour indiqué manuellement, mais pouvant utiliser les types des arguments. Exemple : `template<class T, class I> auto f(T& v, I i) -> decltype(v[i])`. Le type de l'expression est pris tel quel, donc une référence n'est *pas* ignorée dans ce cas.
  * `decltype(auto)` (C++14) au lieu de `auto`, sans *trailing return type* : similaire à `auto`, mais là aussi, c'est le type de l'expression de retour qui est pris en compte, sans modification.

`decltype(auto)` peut également être utilisé pour la déclaration d'une variable, avec les mêmes conséquences.

Rappel : `decltype((x))` est le type de `x` vu comme une expression, e.g. `int&` si `x` est `int`. En conséquence,

  ~~~~~~
  decltype(auto) f()
  {
    int x=0;
    return (x);
  }
  ~~~~~~

a pour type de retour `int &` et retourne une *référence* sur (la variable locale) `x` :-/

## Avantages / désavantages d'`auto`

Avantages : 
  * force l'initalisation des variables
  * évite les conversions non voulues (sources de bugs ou de problèmes de performance: une copie non désirée peut être faite quand une conversion est nécessaire entre le type renvoyé et le type déclaré, y compris quand ils ne diffèrent que par leur *constness*)
  * permet plus de généricité (dans les arguments des lambdas)

Inconvénient :
  * perte de lisibilité si les types manipulés ne sont plus explicites
  * `auto` ne se marie pas bien avec les classes *proxy*, qui se convertissent implicitement à la demande.

  Exemple de problème avec la classe proxy l'opérateur `std::vector<bool>[]`,  qui renvoie non pas un `bool`, mais un `std::vector<bool>::reference` qui peut être converti en `bool` :
  ~~~
  std::vector<bool> f();
  auto v = f()[0]; // renvoie un std::vector<bool>::reference qui fait référence au résultat de f(), temporaire 
  // => l'utilisation de v déclenchera un comportement indéfini
  ~~~
`auto` peut toutefois être utilisé dans ce genre de contexte avec un `static_cast<>` :
  ~~~
  std::vector<bool> f();
  auto v = static_cast<bool>(f()[0]);
  ~~~

## Constructeurs, `initializer_list<T>`, `universal initialization`

Depuis C++11, la syntaxe avec accolades est disponible pour construire un objet :

    MyClass x{1, 2.0, true, "str"};

ou, de façon équivalente,

    MyClass x({1, 2.0, true, "str"});

Cette syntaxe permet
  * soit d'appeler un constructeur prenant des arguments individuels correspondant aux valeurs listées
  * soit d'appeler un constructure prenant en argument un `std::initializer_list<T>`, ce qui est très utile pour initialiser un container (e.g. `std::vector<T>`) avec un nombre variable de valeurs. 

La syntaxe avec accolades est appelée `uniform initialization` (UI) ou `brace initialization`. 

**Avantages de la syntaxe UI**
  * les conversions pouvant résulter en une perte d'information (*narrowing conversions*) sont interdites
  * c'est une syntaxe utilisable dans tous les contextes, notamment sans argument (pas de *most vexing parse*)

**Problèmes de la syntaxe UI**
  * Lorsqu'une classe possède un constructeur prenant un argument de type `std::initializer_list<T>` (avec `T` fixe ou paramètre de template), ce constructeur sera utilisé préférentiellement par un appel UI dès lors que la conversion des arguments en `T` est possible, ce qui peut donner des résultats inattendus (si la conversion est possible mais peut résulter en une perte d'information, une erreur de compilation sera levée). Exemple :

  ~~~
  std::vector<int> v1(10, 20);  // v1 est un vecteur de 10 éléments, tous égaux à 20
  std::vector<int> v2{10, 20};  // v2 est un vecteur de deux éléments, 10 et 20
  ~~~

  * En revanche, une liste vide entre accolades `{}` appelle le constructeur par défaut, et non pas le constructeur `std::initializer_list<...>` avec une liste vide, ce qui n'est pas très cohérent avec le cas avec au moins un argument.
  * Les constructeurs peuvent accepter l'une ou l'autre des syntaxes d'initialisation, mais les deux possibilités ne peuvent être transférées à travers un template variadique qui ne peut déduire un argument de type `std::initializer_list<T>` (voir [ce paragraphe](#rapport-entre-déduction-de-types-dans-les-templates-et-avec-auto)). Les templates variadiques permettent de forwarder de façon générique un appel de fonction vers une construction utilisant soit la syntaxe UI soit la syntaxe historique ; l'appelant ne verra, lui, qu'un appel de fonction avec parenthèses, et le résultat final dépendra de la syntaxe choisie dans la fonction. C'est un problème rencontré en pratique dans la STL dans `std::make_unique` et `std::make_shared`, qui choisissent la syntaxe historique. On peut tout de même accéder depuis ces fonctions aux constructeurs prenant un argument de type `std::initializer_list<T>` en le spécifiant explicitement, comme dans l'exemple ci-dessous :

  ~~~
  #include <vector>
  
  template<class ... Ts> auto f(Ts ...args)
  {
    return std::vector(args...); //(*)
  }

  int main(){
    f(2, 3); // OK : calls regular constructor
    // f(2, 3, 6); // ERROR : constructor with initalizer_list not considered in (*) because it is a regular (non-UI) ctor call
    // f({2,3,6}); // ERROR: cannot deduce Ts={initializer_list<int>}
    f<std::initializer_list<int>>({2,3,6}); // OK
  }
  ~~~

## `nullptr`, `NULL`, `0`
`nullptr` est typé comme un type pointeur générique, peut se convertir en tout type pointeur. 0 ou NULL peut être vu comme un entier et déclencher, par exemple, une mauvaise déduction de type de template ou un mauvaux choix de surcharge.

## `using a = b` vs `typedef b a`
`using` (déclaration d'alias) est plus agréable avec les pointeur de fonctions ou pointeurs sur tableaux:

  ~~~
  typedef void (*FP)(int);      // typedef sur un pointeur de fonction
  using FP = void (*)(int);     

  typedef int (*A)[10]; 
  using A = int (*)[10];
  ~~~

`using` peut être templaté :

  ~~~
template<typename T>
using MyAllocListU = std::list<T, MyAlloc<T>>;  // MyAllocListU<T> est synonyme de std::list<T, MyAlloc<T>>
  ~~~

L'équivalent avec `typedef` consiste à créer un typedef dans une `class` ou un `struct` templaté :
  ~~~
  template<typename T>
  struct MyAllocList {
    typedef std::list<T, MyAlloc<T>> type; // MyAllocList<T>::type est synonyme de std::list<T, MyAlloc<T>>
  };               
  ~~~

on a le `::type` en plus, mais surtout, dans le cas du `typedef`, l'usage dans une classe templatée nécessite de préciser que l'on a affaire à un type avec `typename`, car le compilateur n'est pas en mesure de s'en assurer :
  ~~~
  template<typename T>
  class Widget {
    typename MyAllocList<T>::type list;  
  };
  ~~~
en effet, `MyAllocList<T>::type` pourrait être un membre et non pas un type pour certaines spécialisations de `MyAllocList<T>`, inconnues au moment où le code ci-dessus est rencontré. `MyAllocList<T>::type` est appelé un `dependent type`. Il n'y a pas le même problème avec `MyAllocListU<T>` qui ne peut être qu'un type, et l'on peut donc écrire 

  ~~~
  template<typename T>
  class Widget {
    MyAllocListU<T> list;  
  };
  ~~~
qui est nettement plus simple.

## *type traits*

Le header `type_traits` permet de convertir type ou d'extraire des propriétés de types manipulés. Il est utilisable avec les deux syntaxes indiquées au paragraphe précédent à partir de C++14, par exemple :

  ~~~
  std::remove_const<T>::type           // C++11 : const T → T.
  std::remove_const_t<T>               // Équivalent C++14.
  std::remove_reference<T>::type       // C++11 : T&/T&& → T.
  std::remove_reference_t<T>           // Équivalent C++14.
  std::add_lvalue_reference<T>::type   // C++11 : T → T&.
  std::add_lvalue_reference_t<T>       // Équivalent C++14.
  ~~~

## `enums` délimités (`scoped enums`)

Ancienne syntaxe des enums :

  ~~~
  enum couleur {bleu = 1, rouge, noir}; // avec un exemple de choix de valeur pour un membre de l'enum

  auto x = bleu; // type = couleur

  int x_int = x; // ok
  ~~~

Le type de stockage est déterminé par l'énumération des enums, l'enum ne peut donc être forward-déclaré sauf si l'on précise le type de stockage (`enum couleur : int`).

Nouvelle syntaxe, `enums` délimités :

  ~~~
  enum class couleur {bleu = 1, rouge, noir};

  auto x = couleur::bleu; // type = couleur
  
  //int x_int = x; // Erreur, conversion implicite interdite
  int x_int = static_cast<int>(x); // ok
  ~~~

Contrairement aux cas des *unscoped enums*, les *scoped enums* ne polluent pas le namespace principal.

Le type de stockage par défaut est `int`, peut être changé. Exemple : `enum class couleur:uint32_t {...}`

## Usage de `=delete`

`=delete` peut être utilisé pour interdire l'accès à des constructeurs, mais aussi à des surcharges de fonctions afin par exemple d'empêcher des conversions indésirables. Une fonction comme `bool f(int x);` peut être appelée avec un entier, mais aussi un `bool`, un `double`... Pour interdire des conversions :

  ~~~
  bool f(bool) = delete;
  bool f(double) = delete;
  ~~~

de même avec des fonctions templatées :

  ~~~
  template<class T> void f(T*);
  template<> void f<char>(char*) = delete;
  template<> void f<const char>(const char*) = delete;
  ~~~
  en revanche, il y a d'autres fonctions à interdire avec `delete` si l'on veut être exhaustif : `volatile char*`, `const volative char*`, soit toutes les combinaisons des qualificateurs *cv*.

  Cette syntaxe fonctionne également pour interdire une spécialisation d'une méthode templatée dans une classe :

  ~~~
  class C {
  public:
    template<typename T> void f(T* ptr) { … }
  };
  template<> void C::f<void>(void*) = delete;
  ~~~

## *reference qualifiers*

  ~~~
  class Widget {
  public:
    void doWork() &;       // uniquement utilisé lorsque *this est une lvalue.
    void doWork() &&;      // uniquement utilisé lorsque *this est une rvalue.
  };
  ~~~

## `override`

Qualificateur de méthode garantissant qu'elle est dérivée, donc qu'elle correspond à une méthode dans une classe de base. Permet d'éviter une erreur dans les paramètres / le type de retour / les qualificateurs (`const`, et *reference qualifiers*) / le nom de méthode / l'absence du mot-clé *virtual* dans la classe de base qui ferait qu'il n'y a pas dérivation contrairement à l'intention.

NB le mot clé `virtual` est uniquement nécessaire au niveau de la classe de base, pas devant les méthodes dérivées.

Exemple :

  ~~~
  class Base {
  public:
    virtual void mf1() const;
    virtual void mf2(int x);
    virtual void mf3() &&;
    virtual void mf4() const;
  };

  class Derived: public Base {
  public:
    void mf1() const override;
    void mf2(int x) override;
    void mf3() && override;
    void mf4() const override;
  };
  ~~~

## *const_iterators*

à utiliser quand on ne veut pas modifier les données pointées. peuvent être obtenus par `cbegin()`, `cend()` (méthode du container, n'existe pas toujours, ou fonction non-membre dans `std`, toujours applicable) (C++11, C++14 pour les cas non-membre, comme les arrays), ou par `begin()` et `end()` sur un conteneur `const` à partir de C++14.

## `noexcept`, `noexcept(bool)`, `noexcept(expr)`

Tout ce qui ne peut générer d'exception gagne a être spécifié `noexcept` pour des raisons de performance : fonctions, méthodes, mais aussi constructeurs / destructeurs.

C'est particulièrement important pour les `move ctors` d'objets utilisés dans des containers comme des `vector` ou `deque` : lors de l'insertion d'un élément, il est possible que tout le conteneur soit déplacé. Si le `move ctor` des éléments est déclaré `noexcept`, il sera utilisé pour déplacer les éléments ; sinon des copies seront faites (en raison de la `strong exception guarantee` du container : en cas d'exception en cours de route, le conteneur ne doit pas être modifié, ce qui n'est pas possible si les éléments sont déplacés par un `move ctor` qui peut déclencher une exception en cours de transfert).

On peut déclarer une fonction conditionnellement `noexcept`, si une autre fonction l'est :

  ~~~
  template <class T, size_t N>
  void swap(T (&a)[N], T (&b)[N]) noexcept(noexcept(swap(*a, *b))); 
  ~~~

La syntaxe générale est `noexcept(bool)`. `noexcept(expr)` est un booléen qui vaut 1 si l'évaluation de `expr` est déclarée comme ne produisant pas d'exception.

## `constexpr`

Pour une variable : évaluable statiquement (à la compilation), et donc utilisables pour tailles de tableaux, paramètres numériques de templates, valeurs pour des `enum`...

Pour une fonction (`constexpr int f()`) : elle produit une `constexpr` si ses arguments le sont.

Des objets peuvent être construits statiquement si leur constructeur est `constexpr`. 

Les constructions autorisées pour les fonctions ou méthodes `constexpr` sont considérablement plus étendues en C+14 qu'en C++11.


## Fonctions / constructeurs générés automatiquement

sont public, inline ( = plusieurs instanciations autorisées dans plusieurs *translation units*), *nonvirtual* sauf cas particulier destructeur (voir ci-dessous).

  * toutes les fonctions concernées ne sont générées que si elles sont utilisées.
  * constructeur par défaut généré uniquement si aucun constructeur déclaré
  * destructeur généré automatiquement si pas déclaré (virtuel ssi dans classe fille avec un destructeur virtuel pour la classe de base)
  * `assignment operator` / `copy ctor` : chacun généré s'il n'est pas défini, et si aucune opération *move* opération n'est définie. Pourrait à l'avenir évoluer comme dans le cas des `move operator / ctor` : ne seront générés que si aucune opération par *move* ou *copy* n'est définie, et si le destructeur n'est pas défini.
  * `move assignment operator` / `move ctor` : sont générés tous les deux seulement si aucun des deux n'est défini, si aucune des deux opérations par copie n'est définie, et si le destructeur n'est pas défini.

Cas typiques :
  * classe sans héritage, où les fonctions par défaut conviennent : on peut ne rien écrire
  * classe avec héritage et donc destructeur virtuel où les fonctions par défaut conviennent : tout déclarer avec `=default` (et `virtual` pour le destructeur)
  * et le cas où l'on déclare les opérations manuellement (habituellement, les 4 opérations *copy* / *move* + destructeur)
  * le constructeur par défaut est généralement effacé (`=delete`)

Règle aux effets de bord bizarres : la présence d'un constructeur templaté ne désactive la génération automatique d'aucun constructeur, même si ce template peut être instancié en le constructeur par copie, par exemple.

## `unique_ptr`

`unique_ptr`: modélise la possession. `unique_ptr<T>` pour un élément unique de type `T`, `unique_ptr<T[]>` pour un élément tableau d'éléments de type `T` (mais peut être avantageusement remplacé par un `array<T,N>`, on a alors un `unique_ptr<array<T,N>>`). L'élément ou le tableau pointé est détruit (avec `delete` ou `delete[]` dans le cas d'un tableau) quand le `unique_ptr` est détruit. Le `unique_ptr` ne peut pas être copié, mais peut être déplacé. Le `unique_ptr` d'origine ne "possède" plus l'objet pointé et aucune destruction n'a lieu quand il est lui-même détruit.

`unique_ptr` peut être instancié avec un deuxième type de template et argument de constructeur qui est le *deleter* remplaçant `delete` ou `delete[]`. Cela modifie le type du `unique_ptr` : des `unique_ptr` avec des *deleter* différents ne peuvent être mélangés dans un tableau par exemple. 

  ~~~
  // avec deleter loggingDel
  std::unique_ptr<Widget, decltype(loggingDel)> uptr(new Widget, loggingDel);
  ~~~

Dans le cas d'un type tableau, `unique_ptr<T[]>` dispose d'un opérateur [] permettant d'accéder aux éléments du tableau.

Hors cas d'un *deleter* custom, un `unique_ptr` se résume en mémoire au pointeur sous-jacent.

**L'accès à l'objet via un `unique_ptr` est dans la plupart des cas aussi rapide que via un pointeur nu** (vraisemblablement vrai également avec un *deleter* custom). C'est vrai si le `unique_ptr` est une variable locale, ou passé par valeur, car l'accès se résume dans ce cas à un déréférencement de pointeur à un offset connu à la compilation par rapport à au *frame* de la fonction sur la pile. C'est également vrai si le `unique_ptr` est membre d'un objet accédé par adresse. 

Cela n'est pas vrai en cas de passage d'un `unique_ptr` par pointeur / par référence puisqu'alors l'accès à l'entité pointée nécessite un double déréférencement, plus éventuellement un ajout d'offset en cas de *deleter* custom (mais l'offset est sans doute nul dans la plupart des implémentations). 

Un `unique_ptr` ne doit pas être passé par pointeur / par référence de toute façon : il doit être passé par valeur pour transférer la possession de l'objet pointé.


## `shared_ptr`

`shared_ptr` est un type pointeur qui gère la ressource sous-jacente par *reference counting*.
Pour être utilisé correctement, il ne faut jamais créer plusieurs `shared_pointer` à partir du même pointeur nu : chacun aura son compteur et ne "verra" pas l'autre. Une copie d'un `shared_pointer` incrémente le compteur, une destruction le décrémente. Le compteur est partagé entre `shared_pointer`, il fait partie d'un objet commun, le *control block*. Celui-ci est alloué dynamiquement. `std::make_shared` contruit l'objet pointé et le *control block* de façon contiguë dans un seul bloc mémoire.

Le compteur de référence est incrémenté / décrémenté atomiquement, et peut donc être utilisé en *multithread*.

L'élément pointé est détruit lorsque le compteur de référence atteint 0. `delete` est utilisé par défaut quel que soit le type pointé **qui ne doit pas être un tableau** (on peut bricoler avec un *deleter* custom qui appelle `delete[]`, mais cela pose des problèmes pour les conversions dans une hiérarchie de classe ; c'est une mauvaise idée). Le *deleter* peut être changé via un second argument optionnel. Contrairement au cas du `unique_ptr`, le destructeur n'affecte pas le type du `shared_ptr` (il est mémorisé dans le *control block*, pas dans le `shared_ptr` lui-même).

  ~~~
  // avec deleter loggingDel
  std::shared_ptr<Widget> sptr(new Widget, loggingDel);
  ~~~

**L'accès à l'objet via un `shared_ptr` est aussi rapide que via un pointeur nu** dans les cas courants (sur la pile, membre d'un objet).

Un `unique_ptr` peut être converti en `shared_ptr`.

Un `shared_ptr<T> x` peut être converti implicitement en `share_ptr<U>` si `U` est une classe parente de `T`. La conversion peut être explicite  avec `static_pointer_cast<U>(x)`, qui est un équivalent pour les `shared_pointer` de `static_cast`. De même, pour les conversions d'une classe de base vers une classe dérivée, `dynamic_pointer_cast` est un équivalent pour les `shared_pointer` d'un `dynamic_cast`.

Il existe une classe dont on peut faire dériver un objet pour essentiellement lui ajouter un *control block* : `std::enable_shared_from_this<T>`. Cela ajoute la méthode `shared_from_this()` qui permet d'obtenir un `shared_pointer` sur l'objet. Dans cet idiome, la classe `T` dérivée met son constructeur en privé pour interdire l'accès au pointeur nu sur les objets crées, et propose une *factory* qui renvoie directement des `shared_pointer`.

  ~~~
  class T: public std::enable_shared_from_this<T> {
  public:
  // factory function that perfect-forwards args to a private ctor
  template<typename... Ts>
  static std::shared_ptr<T> create(Ts&&... params){
    T(std::forward<Ts>(params)...);
  }
  //…
  void process(){
     processed.emplace_back(shared_from_this());
  }
  
  private:
    // ctors
  };
  ~~~

## `weak_ptr`

Un `weak_ptr` est construit à partir d'un `shared_ptr`, mais n'incrémente pas le *reference count* de l'objet pointé. Il peut pointer sur un objet qui a été détruit.

Création : `std::weak_ptr<T> wptr(sptr)`

Teste si l'objet pointé a été détruit : `wptr.expired()`

Mais cette information a une durée de vie nulle en contexte multithread.

Tester atomiquement l'existence de l'objet pointé et créer un `shared_ptr` à partir d'un `weak_ptr` :

  ~~~
  std::shared_ptr<T> sptr(wptr);    // if wptr is expired, throw std::bad_weak_ptr
  ~~~

Autre construction qui n'utilise pas les exceptions :

  ~~~
  std::shared_ptr<T> sptr = wptr.lock();  // if wptr is expired, sptr is null
  auto sptr = wpw.lock();                 // same as above, but uses auto
  ~~~

 `weak_ptr` est principalement utile pour construire des structures de cache.

## `std::make_unique`, `std::make_shared`

`std::make_shared` (resp. `std::make_unique`) combine l'allocation d'un objet et la création d'un `std::shared_ptr` (resp. `std::unique_ptr`). Comme vu plus haut, `std::make_shared` alloue dans un seul bloc mémoire le *control block* et l'objet pointé, ce qui est un avantage (une allocation au lieu de 2, moins de fragmentation mémoire) et un inconvénient (dans le cas d'utilisation de `weak_ptr`, le *control block* ne peut être désalloué qu'avec le dernier `weak_ptr`, au contraire de l'élément pointé qui peut être détruit avec le dernier `shared_ptr` ; si les deux sont dans le même bloc mémoire, toute la zone restera allouée.)

`make_shared` et `make_unique` sont incompatibles avec des allocateurs / désaloccateurs custom.

**Avant C++17**, `std::make_shared` (et `std::make_unique`) permettait de garantir l'absence de fuite mémoire dans un appel du type `f(std::make_shared(MyClass), g())`, au contraire de la syntaxe `f(std::shared_ptr<MyClass>(new MyClass), g())` : en effet dans le second cas, l'appel de `g()` pouvait théoriquement être entrelacé entre `new MyClass` et la construction du `shared_ptr`, ce qui pouvait conduire à une fuite mémoire en cas d'exception dans l'appel de `g()`. **C++17 garantit que cela n'arrive pas** : l'ordre d'évaluation des arguments n'est pas spécifié, mais chaque expression correspondant à un argument est complètement évaluée et liée à l'argument avant de passer à un autre argument. **Cet avantage de `std::make_shared` et de `std::make_unique` n'est donc plus valable à partir de C++17**.




