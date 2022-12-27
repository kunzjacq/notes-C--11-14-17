# Concepts C++20

## `requires`

Usage basique de `requires` pour indiquer une contrainte booléenne permettant de filtrer les types admissibles dans un template (fonction ou classe):

~~~C++
// booleen templaté 

template<class T>
constexpr bool b = false; // constexpr ou const nécessaire 

template<>
constexpr bool b<bool> = true;

//instancier f uniquement si b<T> est vrai via "requires"
template <typename T>
requires b<T> // <---------------------------------------------------
void f(T x){}

// autre exemple
template <typename T, int N>
requires (N >= 0) //<------------------------------------------------
constexpr T SumUpToN = N + SumUpToN<T, N - 1>;
 
template <typename T>
constexpr T SumUpToN<T, 0> = 0;
~~~

## Concepts nommés

~~~C++
// créer un 'concept' pour nommer la condition du paragraphe précédent
template <typename T> 
concept Number = b<T>;
~~~

Il est également possible de spécifier une liste d'expressions qui doivent être valides avec le type considéré :
~~~C++
template <typename T>
concept Number = requires(T t) {
    t + t;
    t - t;
    t * t;
    t / t;
    -t;
    +t;
    --t;
    ++t;
    t--;
    t++;
};

// Utilisation d'un template nommé
template <typename TVal, typename TThreshold=TVal>
requires Number<TVal> && Number<TThreshold> // <---------------------
bool IsNearlyZero(TVal val, TThreshold threshold){
    return (val < 0 ? -val : val) < threshold;
}

// autre syntaxe : requires après le prototype de la fonction
template <typename TVal, typename TThreshold=TVal>
bool IsNearlyZero(TVal val, TThreshold threshold)
    requires Number<TVal> && Number<TThreshold> // <-----------------
{
    return (val < 0 ? -val : val) < threshold;
}

// syntaxe simplifiée, utilisable si la condition à imposer est
// l'appartenance à un seul concept (comme ci-dessus)
template <Number T> // <---------------------------------------------
bool IsNearlyZero(T val, T threshold)
{
    return (val < 0 ? -val : val) < threshold;
}

// avec template abrégé utilisant auto, on peut aussi imposer
// l'appartenance au concept comme ci-dessous :
bool IsNearlyZero(Number auto val, Number auto threshold) // <-------
{
    return (val < 0 ? -val : val) < threshold;
}

// on peut aussi fusionner la définition d'un concept avec son utilisation, 
// ce qui aboutit à un double 'requires' :
template <typename T>
requires requires(T t) // <------------------------------------------
{
    t + t;
    // ...
}
bool IsNearlyZero(Number auto val, Number auto threshold)
{
    return (val < 0 ? -val : val) < threshold;
}

// combinaison de concepts : concept nommé et concept
// 'ad hoc' défini en place
template <typename T>
concept Integer = Number<T> && requires(T t) // <--------------------
{
    t << t;
    t % t;
    // ...
};

// concepts appliqué aux membres d'un objet :
template <typename T>
concept Vector2 = requires(T t) {
    //  t.X et t.Y doivent être des 'Number'
    Number<decltype(t.X)>; // <--------------------------------------
    Number<decltype(t.Y)>; // <--------------------------------------
};

//Autre syntaxe ayant le même sens :
template <typename T>
concept Vector2 = requires(T t) {
    {t.X} -> Number; // <--------------------------------------------
    {t.Y} -> Number; // <--------------------------------------------
};
~~~

## Concepts et surcharges

Quand plusieurs surcharges d'une même fonction existent, contraignant chacune les arguments de la fonction à l'aide de concepts, le compilateur choisira la fonction utilisant le contexte *le plus contraint* s'appliquant au contexte d'emploi.

~~~C++
template <typename T>
concept Iterable = requires(T t) {
    t.getBegin();
    t = t->next();
    *t;
};
 
template <typename T>
concept Indexable = Iterable<T> && requires(T t) {
    t[0];     // lecture par index
    t[0] = 0; // écriture par index
};

// Surcharge de l'accès au n-ième élément en O(1) pour 
// les types satisfaisant 'Indexable' 
int getAtIndex(Indexable auto collection, int index)
{
    return collection[index];
}
 
// Surcharge de l'accès au n-ième élément en O(n) pour 
// les types satisfaisant 'Iterable' 
int getAtIndex(Iterable auto collection, int index)
{
    auto cur = collection.getBegin();
    for (int i = 0; i < index; ++i) cur = cur->next;
    return *cur;
}
~~~

Si un élément `x` appartient à un type satisfaisant `Iterable` et `Indexable`, `getAtIndex(x)` appelera la variante la plus spécialisée en O(1).

A l'inverse, si un élément `x` appartient à un type satisfaisant `Iterable` mais pas `Indexable`, `getAtIndex(x)` appelera la surcharge en O(N).

Comment est défini ce critère *le plus contraint* ? Dans l'exemple ci-dessus, c'est simple car l'un des concepts est défini par raffinement de l'autre.

Si l'on remplace le concept `Indexable` par la version ci-dessous, équivalente mais qui ne fait plus référence à `Iterable` :

~~~C++
template <typename T>
concept Indexable = requires(T t) {
    t.getBegin();
    t = t->next();
    *t;
    t[0];
    t[0] = 0;
};
~~~

Le compilateur n'est plus en mesure de comparer les deux concepts et un appel sur un élément à la fois `Indexable` et `Iterable` devient ambigu, ce qui déclenche une erreur à la compilation.

## Concepts et *type traits*

Voir le header `concepts` (<https://en.cppreference.com/w/cpp/header/concepts>) qui définit des concepts classiques comme `integral`. Forte relation avec les *type traits* correspondants :

~~~C++
template <class T>
concept integral = std::is_integral_v<T>;
~~~

Dans ce cas, n'est pas défini comme possédant certaines propriétés mais simplement par la liste des types natifs concernés :

 > Checks whether `T` is an integral type. Provides the member constant value which is equal to `true`, if `T` is the type `bool`, `char`, `char8_t` (since C++20), `char16_t`, `char32_t`, `wchar_t`, `short`, `int`, `long`, `long long`, or any implementation-defined extended integer types, including any signed, unsigned, and cv-qualified variants. Otherwise, value is equal to `false`.

## Compléments

Exemple d'usage de `requires` avec une instantiation ne respectant pas la condition imposée :

~~~~C++
struct Vector2
{
    float X;
    float Y;
};
 
template <typename T>
constexpr bool IsVector2 = false;
 
template <>
constexpr bool IsVector2<Vector2> = true;
 
template <typename TVector, typename TComponent>
requires IsVector2<TVector> 
TComponent Dot(TVector a, TVector b)
{
    return a.X*b.X + a.Y*b.Y;
}
 
// OK:
Vector2 vecA{2, 4};
Vector2 vecB{2, 4};
DebugLog(Dot<Vector2, float>(vecA, vecB));
 
// Compiler error:
DebugLog(Dot<int, int>(2, 4));
// Candidate template ignored: constraints not satisfied
// [with TVector = int, TComponent = int]
// TComponent Dot(TVector a, TVector b)
//            ^
// test.cpp:60:10: note: because 'IsVector2<int>' evaluated to false
// requires IsVector2<TVector>
~~~~
