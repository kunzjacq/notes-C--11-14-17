# Coroutines C++20

## Références

- [1] [Lewis Baker introduction to coroutines](https://lewissbaker.github.io/) : légèrement dépassé, mais très clair et complet sur le "coroutines TS". Lewis Baker est l'auteur de la bibliothèque [*cppcoro*](https://github.com/lewissbaker/cppcoro).
  - [1.1] [Coroutine theory](https://lewissbaker.github.io/2017/09/25/coroutine-theory)
  - [1.2] [C++ Coroutines: Understanding operator `co_await`](https://lewissbaker.github.io/2017/11/17/understanding-operator-co-await)
  - [1.3] [C++ Coroutines: Understanding the promise type](https://lewissbaker.github.io/2018/09/05/understanding-the-promise-type)
  - [1.4] [C++ Coroutines: Understanding Symmetric Transfer](https://lewissbaker.github.io/2020/05/11/understanding_symmetric_transfer)
- [2] [Raymond Chen's 3 coroutine series](https://devblogs.microsoft.com/oldnewthing/20210504-01/?p=105178)
- [3] [My tutorial and take on C++20 coroutines, from David Mazières](https://www.scs.stanford.edu/~dm/blog/c++-coroutines.html)
- [4] [C++ for C# developpers Part 36 - Coroutines](https://www.jacksondunstan.com/articles/6311)
- [5] [cppreference.com coroutines page](https://en.cppreference.com/w/cpp/language/coroutines)

## Aperçu

Une coroutine est une fonction résumable, c'est-à-dire une fonction qui peut être suspendue et reprise. Pour que cela soit possible, l'état de la fonction, y compris son point de reprise, doit être conservé dans une structure de données pendant les suspensions. Il y a deux façons de le faire : les coroutines stackful, où une coroutine a sa propre pile et où quitter une coroutine pour la reprendre plus tard signifie changer de pile (voir par exemple les coroutines implémentées avec `setjmp`/`longjmp`) ; et les coroutines stackless, où les états des coroutines sont des structures de données spéciales stockées dans le tas. Les coroutines stackful peuvent être suspendues et reprises à partir d'appels de sous-fonction, alors que les coroutines stackless ne le peuvent pas, puisque la structure de données de taille fixe associée à une coroutine ne peut stocker que l'état de cette fonction et non les données associées à un nombre arbitraire d'appels de fonction.

C++20 implémente les coroutines sans pile. Les états des coroutines sont gérés par le compilateur et ne sont pas directement visibles depuis le code. Une partie de l'état de la coroutine est un objet qui peut néanmoins être personnalisé : l'objet *promise*.

Une fonction régulière devient une coroutine et est compilée comme telle dès qu'elle utilise une opération coroutine : `co_await`, `co_return` ou `co_yield`. Par conséquent, l'impossibilité de suspendre/reprendre une fonction régulière appelée par une coroutine est intégrée dans la syntaxe et la sémantique.

Le comportement d'une coroutine est largement défini par son *type de retour*. Le type de l'objet de retour doit suivre certaines conventions telles que la définition du type de l'objet *promise* comme un sous-type de lui-même :

~~~C++
struct return_object {
  struct promise_type {
    //...
  };
};
~~~

Certaines méthodes standard de type `promise_type` sont appelées lorsque la coroutine invoque `co_await`, `co_yield` ou `co_return` ; une méthode prédéfinie est également invoquée pour la création de l'objet de retour lui-même.

De la même manière qu'une relation *promise/future*, l'objet *promise* est plus ou moins l'interface de la coroutine, alors que l'objet retour est ce qui est vu par le code qui a lancé la coroutine (puisque ce code obtient l'objet retour). L'objet retour peut facilement avoir accès à l'objet promise, et par ce biais, à des informations sur l'état de la coroutine ou sur ce qu'elle a produit.

Dans certains contextes, un *manipulateur de coroutine* peut être obtenu lorsqu'une coroutine est suspendue. Celui-ci agit comme un pointeur sur l'état de la coroutine, et possède une méthode `resume()` qui permet de reprendre la coroutine au point de suspension enregistré dans son état. Un handle de coroutine peut aussi être explicitement construit à partir d'un objet *promise* ou d'un pointeur brut vers l'état de la coroutine.

Il peut être surprenant que, dans la syntaxe des coroutines, une coroutine ne renvoie pas explicitement son objet de retour. Le code qui s'en charge est généré par le compilateur.

### En bref

- type de retour de la coroutine = objet de retour ≠ objet *promise*
- coroutine handle ≈ pointeur vers l'état de la coroutine.
- objet *promise* ⊂ état de la coroutine
- type d'objet *promise* défini dans type d'objet retour, en tant que sous-type `promise_type` de cet objet.

## *coroutine handle*, état d'une coroutine, objet *promise*, objet retour

On peut obtenir un *coroutine handle* à partir d'un objet *promise* `p` parce que l'adresse de l'état de la coroutine peut être déduite de l'adresse de l'objet *promise*, qui est à un offset fixe dans l'état de la coroutine. C'est le travail de la méthode promise `std::coroutine_handle<promise_type>::from_promise()` : `p.from_promise()` produit le handle.
Inversement, à partir d'un handle de coroutine `h` ayant le type `std::coroutine_handle<promise_type>`, `h.promise()` retourne une référence à l'objet *promise*. Son type `promise_type` doit être spécifié pour que cela fonctionne ; `.promise()` ne peut pas être utilisé sur un handle `std::coroutine_handle<>` (qui peut être converti en n'importe quel handle).

L'objet de retour est créé par un appel à la méthode promise `get_return_object()` sur l'objet *promise* inclus dans l'état de la coroutine, lorsque la coroutine est appelée et avant d'entrer dans sa portée.

Bien que le type d'objet de retour définisse le type d'objet *promise* dans son espace de noms, les deux objets ne sont pas liés.

Dans l'exemple 2 de [3], la méthode promise `get_return_object()` met un *coroutine handle* obtenu avec `this->fromise()` dans l'objet de retour. C'est une technique commune avec de nombreuses variations qui permet d'accéder à l'état de la coroutine depuis l'objet de retour. Cet accès peut être utilisé pour reprendre la coroutine (avec `h->resume()`, `h` étant le handle stocké), ou pour accéder aux valeurs qui sont stockées par la coroutine dans l'objet promise.

Une de ces variations consiste à inclure dans l'objet de retour un opérateur de conversion vers le type de handle de la coroutine. De cette façon, le code appelant peut accéder au handle et, à travers lui, à l'objet *promise* :

~~~C++
  // in return object class
  std::coroutine_handle<promise_type> h;
  operator std::coroutine_handle<promise_type>() const { return h; }

  // in calling code:
  // start the coroutine and retrieve a handle on it
  std::coroutine_handle<promise_type> h = my_coroutine();
  // or, using an implicit cast from
  // std::coroutine_handle<promise_type>
  // to std::coroutine_handle<>
  std::coroutine_handle<> h = my_coroutine();
~~~

Une façon de faire alternative, et peut-être une conception plus explicite, laisse le code appelant récupérer l'objet de retour complet, et les méthodes internes de l'objet de retour accéder à l'objet *promise* via le handle `h`.

## Séquence de démarrage d'une coroutine

Citons [5] :
> Lorsqu'une coroutine commence son exécution, elle effectue les opérations suivantes :
>
> - allocation de l'objet d'état de la coroutine en utilisant l'opérateur `new`.
> - copie de tous les paramètres de la fonction dans l'objet d'état de la coroutine : les paramètres par valeur sont déplacés ou copiés, les paramètres par référence restent des références (et peuvent donc devenir invalides si la coroutine est reprise après la fin de vie de l'objet référencé)
> - appel du constructeur de l'objet *promise*. Si celui-ci a un constructeur qui prend tous les paramètres de la coroutine, ce constructeur est appelé, avec des arguments de coroutine post-copie. Sinon, le constructeur par défaut est appelé.
> - appel de `promise.get_return_object()` et stockage du résultat dans une variable locale. Le résultat de cet appel sera retourné à l'appelant lors de la première suspension de la coroutine. Toutes les exceptions lancées jusqu'à cette étape incluse se propagent à l'appelant, elles ne sont pas stockées dans l'objet *promise*.
> - appel de `promise.initial_suspend()` et exécution de `co_await` sur son résultat. Les types *promise* typiques retournent soit un `suspend_always`, pour les coroutines démarrées de façon paresseuse, soit un `suspend_never`, pour les coroutines démarrées immédiatement.
> - lorsque `co_await promise.initial_suspend()` reprend, exécution du corps de la coroutine.

## Methodes de l'objet *promise*

~~~C++
struct return_object
{
  struct promise_type
  {
    std::exception_ptr exception;
    using handle_type = std::coroutine_handle<promise_type>;

    // called upon first coroutine call, before entering in
    //the coroutine code, to create the return object
    // mandatory
    return_object get_return_object();

    // returns an awaiter object that determines whether
    // the coroutine is suspended before start.
    // is mandatory
    std::suspend_always initial_suspend()
    {
      return {};
    }
    // returns an awaiter object that determines whether
    // the coroutine is suspended before exiting (co_return or end of scope).
    // otherwise its state is immediately destroyed.
    // is mandatory
    std::suspend_always final_suspend() noexcept
    {
      return {};
    }
    // called when an exception is thrown inside the coroutine.
    // mandatory only if the corresponding coroutine statement is used
    void unhandled_exception()
    {
      exception = std::current_exception();
    }
    // called when using co_yield from inside the coroutine.
    // mandatory only if the corresponding coroutine statement is used
    template<std::convertible_to<T> From> // C++20 concept
    std::suspend_always yield_value(From &&from)
    {
      value = std::forward<From>(from);
      return {};
    }
    void return_void()
    // called
    // - when using co_return without argument in the coroutine, or
    // - when using co_return <expr> with expr having type void, or
    // - at the end of the coroutine scope
    // mandatory only if the corresponding coroutine statement is used (UB otherwise)
    {}

    template<class T> // called when using co_return v, with v of type T, in the coroutine
    // typically stores the value in the promise and somehow signals that a value is available
    // mandatory only if the corresponding coroutine statement is used
    void return_value(T e)
    {}
  };
};
~~~

Pour `suspend_always` et `suspend_never`, voir le paragraphe sur les awaiters ci-dessous.

## Comportement de la pile au démarrage et à la reprise d'une coroutine

Quand une coroutine est lancée, un *stack frame* est créé sur la pile du thread courant, comme pour un appel de fonction normal. Celui-ci est détruit lorsque l'appel revient, bien que la coroutine puisse continuer à fonctionner. Pour que le comportement soit correct, les arguments de la coroutine sont copiés ou déplacés dans l'état de la coroutine lorsque la coroutine est appelée ; sinon, certains arguments peuvent être indisponibles après une suspension/reprise de la coroutine, voir [1.1]. De même, lorsque la coroutine est reprise depuis son handle `h` en utilisant `h.resume()`, cet appel est un appel de fonction normal qui crée un stack frame dans la pile du thread courant, qui est ensuite détruit quand / si l'appel revient.

## `co_return` et `co_yield`

`co_return` (avec ou sans argument) termine la coroutine : il n'est pas possible de la reprendre ensuite. La coroutine peut ou non être suspendue une dernière fois, en fonction de l'awaiter retourné par `p.final_suspend()`, `p` étant l'objet *promise*. Voir les méthodes `return_void` et `return_value` de l'objet *promise*.

`co_yield` est utilisé pour produire une valeur à l'intérieur de la coroutine, avec ou sans la suspendre, et avec la possibilité de reprendre la coroutine plus tard. La valeur produite peut par exemple être stockée dans l'objet *promise* pour une consommation ultérieure. Voir la méthode `yield_value` de l'objet *promise*.

## `co_await` et objets awaiter

Les objets *awaiter* sont utilisés lorsqu'une coroutine utilise l'opérateur `co_await`. Lorsque l'expression `co_await a` est rencontrée, cela déclenche une logique qui crée un objet *awaiter* `b`. Dans le cas le plus simple, `b` est simplement égal à `a`. Un awaiter doit implémenter les méthodes `await_ready`, `await_suspend` et `await_resume`. Les rôles de ces méthodes sont expliqués ci-dessous.

~~~C++
struct Awaiter {
  bool await_ready() const noexcept
  {
    // called first after co_await.
    // If it returns true, the coroutine is ready,
    // it should not be interrupted.
    // In that case the calling coroutine is immediately
    // resumed; state saving / restoring does not occur.
    // if it returns false, the coroutine state is saved
    // and await_suspend is called (see below)
      return false;
  }
  void/bool/std::coroutine_handle<> await_suspend(std::coroutine_handle<> h)
  {
    // after await_ready, if the coroutine is not ready,
    // the call to co_await saves the coroutine state and calls
    // a.await_suspend(h),
    // where h is a handle on the calling coroutine state.
    // h enables to resume the coroutine that co_awaits.
    // `await_suspend` is responsible to store `h`
    // so that the coroutine can be resumed later.
    // `await_suspen` may return nothing, a bool or
    // a coroutine handle.
    // - if it returns bool:
    //   tells whether the coroutine should be suspended.
    // - if it returns a coroutine_handle:
    //   the coroutine corresponding to this handle is
    //   immediately resumed.
  }
  custom_type await_resume() const noexcept
  {
    // determines the value of the the co_await expression
    // when the coroutine is resumed.
    // the return type may be set to whatever type is returned.
  }
};
~~~

`std::suspend_always` et `std::suspend_never` sont des *awaiters* standards dont les méthodes `await_ready` retournent respectivement `false` et `true`, et dont les méthodes `await_suspend` et `await_resume` ne font rien. Le type de retour de `await_resume` est `void`.

### Logique de dérivation de l'objet *awaiter*

Pour dériver l'objet *awaiter* `b` de `co_await <expr>`, d'abord, un objet *awaitable* est créé, puis l'objet *awaiter* est déterminé à partir de celui-ci :

- si l'objet `promise` de la coroutine qui appelle `co_await` possède une méthode `await_transform` :
 l'objet `awaitable a` est obtenu comme `promise.await_transform(<expr>)`
sinon
 `awaitable a = évaluation de <expr>`
- ensuite, si le `a` possède un opérateur `co_await()`, `b` est obtenu par `a.co_await()`.
Sinon, `b` est égal à `a`.

> Voir l'objet *awaiter* `GetPromise` de [1] pour un exemple d'un awaiter qui ne suspend pas la coroutine (`await_ready` retourne `false`, mais `await_suspend` retourne aussi `false`), et qui est  utilisé pour permettre à la coroutine d'accéder à son propre objet *promise* : `await_suspend` stocke le handle de la coroutine dans l'objet *awaiter*, et `await_resume` le convertit en un pointeur vers l'objet *promise*, qu'il retourne. Ce mécanisme est utilisé pour stocker une valeur issue de la coroutine dans l'objet *promise*, qui peut alors être obtenue par le code qui a lancé la coroutine à travers l'objet de retour, à condition que celui-ci ait une copie du handle de la coroutine comme discuté précédemment. Cette technique n'est cependant pas nécessaire car le résultat est similaire à ce qui est fait par `co_yield`.

## Exemple de générateur

Adapté de [3] (ou [5]). Le type de l'objet *promise* :

~~~C++
template<typename T>
struct generator;

template<class T>
struct generator_promise_type
{
  T value;
  std::exception_ptr exception;
  using handle_type = std::coroutine_handle<generator_promise_type>;
  // defined after generator<T>
  generator<T> get_return_object();
  std::suspend_always initial_suspend() { return {}; }
  std::suspend_always final_suspend() noexcept { return {}; }
  void unhandled_exception() { exception = std::current_exception(); }
  template<std::convertible_to<T> From>
  std::suspend_always yield_value(From &&from)
  {
    value = std::forward<From>(from);
    return {};
  }
  void return_void() {}
};
~~~

The classe *generator* (qui est le type de retour de la coroutine) est définie comme suit :

~~~C++
template<class T> struct generator_promise_type;

template<typename T>
struct generator {
  using promise_type = generator_promise_type<T>;
  using handle_type = std::coroutine_handle<promise_type>;

  generator(handle_type h) : h(h) {}
  ~generator() { h.destroy(); }
  generator(const generator&) = delete;
  generator(generator&&) = default;
  explicit operator bool() {
    fill();
    return !h.done();
    // h.done(): Checks if a suspended coroutine is suspended
    // at its final suspended point.
    // https://en.cppreference.com/w/cpp/coroutine/coroutine_handle/done
  }
  T operator()() {
    fill();
    full = false;
    return std::move(h.promise().value);
  }

private:
  handle_type h;
  bool full = false;

  void fill()
  {
    if (!full)
    {
      h(); // same as h.resume()
      if (h.promise().exception)
      {
        std::rethrow_exception(h.promise().exception);
      }
      full = true;
    }
  }
};

template<class T>
generator<T> generator_promise_type<T>::get_return_object()
{
  return generator<T>(handle_type::from_promise(*this));
}

~~~

Exemple d'emploi :

~~~C++
generator<unsigned> counter()
{
  for (unsigned i = 0; i < 3; i++) co_yield i;
}

void main()
{
  auto gen = counter();
  while(gen) std::cout << "counter: " << gen() << std::endl;
}
~~~
