# Polimorfizm

Polimorfizm (z gr. wielopostaciowość) to w odniesieniu do funkcji możliwość działania na obiektach różnych typów.
W tym miejscu zajmiemy się podstawową formą polimorfizmu - polimorfizmem parametrycznym.

Przypomnijmy funkcję `interactionOf`:

```haskell
interactionOf :: world ->
                 (Double -> world -> world) ->
		 (Event -> world -> world) ->
		 (world -> Picture) ->
		 IO ()
```

Funkcja ta jest polimorficzna: możemy ją zastosować używając w miejscu zmiennej `world` dowolnego typu.

## Parametryczność

Ważne aby pamiętać, ze możliwosc wyboru typu lezy po stronie **wywołującego**. 
Oznacza to, że implementacja funkcji musi działać dla **wszystkich** typów.
Co więcej, musi działać **dla wszystkich typów tak samo**.

Dlaczego parametrycznośc jest ważna?

1. Umożliwia *wycieranie typów*. Skoro funkcja wywoływana działa dla każdego typu tak samo, nie potrzebuje informacji 
jakiego typu są jej faktyczne parametry. W związku z tym informacja typowa nie jest potrzebna w trakcie wykonania, a jedynie w trakcie kompilacji.

2. Ograniczając sposoby działania funkcji polimorficznych daje nam **twierdzenia za darmo** (*theorems for free*, termin ukuty przez Phila Wadlera a zarazem tytuł jego [słynnej pracy](https://people.mpi-sws.org/~dreyer/tor/papers/wadler.pdf)).

:pencil: Rozważmy na przykład funkcje o następujących sygnaturach

```haskell
zagadka1 :: a -> a
zagadka2 :: a -> b -> a
```

ile różnych implementacji potrafisz napisać?

:pencil: Trochę trudniejsze, być może do domu: co można powiedzieć o rodzinie funkcji typu

```haskell
(a -> a) -> a -> a
```

## Polimorficzne typy danych

Polimorficzne, mogą być nie tylko funkcje, ale i typy danych. W poprzednim tygodniu pisaliśmy wariant funkcji `interactionOf`
pozwalający na cofnięcie poziomu do stanu początkowego. Spróbujmy teraz rozszerzyć tę funkcję o wyświetlanie ekranu startowego 
i rozpoczynanie właściwej gry po naciśnięciu spacji. Na początek możemy stworzyć bardzo prosty ekran startowy:

```haskell
startScreen :: Picture
startScreen = scaled 3 3 (text "Sokoban!")
```

Musimy wiedzieć czy jestesmy na ekranie startowym czy też gra już się toczy.  Najprościej zapamiętać tę informację w stanie

```haskell
data SSState = StartScreen | Running world
```

Niestety przy takim kodzie, dostaniemy komunikat o błędzie `Not in scope: type variable ‘world’`. Istotnie, co to jest `world`? 
Mozemy uczynic go *parametrem typu*:

```haskell
data SSState world = StartScreen | Running world
```

Teraz możemy zaimplementować:

```haskell
startScreenInteractionOf ::
    world -> (Double -> world -> world) ->
    (Event -> world -> world) -> (world -> Picture) ->
    IO ()
startScreenInteractionOf state0 step handle draw
  = interactionOf state0' step' handle' draw'
  where
    state0' = StartScreen

    step' _ StartScreen = StartScreen
    step' t (Running s) = Running (step t s)

    handle' (KeyPress key) StartScreen
         | key == " "                  = Running state0
    handle' _              StartScreen = StartScreen
    handle' e              (Running s) = Running (handle e s)

    draw' StartScreen = startScreen
    draw' (Running s) = draw s
```

:pencil: Dodaj ekran startowy do swojej gry.

## Interakcje całościowe

Chcielibysmy teraz połączyć funkcjonalność  `startScreenInteractionOf` z funkcjonalnością `resettableInteractionOf`

```haskell
resettableInteractionOf ::
    world ->
    (Double -> world -> world) ->
    (Event -> world -> world) ->
    (world -> Picture) ->
    IO ()
```

tak, aby nacisnięcie `ESC` wracało do ekranu startowego. Ale nie mozemy - obie te funkcje daja wynik typu `IO()` a nie biora argumentów takiego typu. Musimy spróbowac innego podejścia.

Gdybyśmy mieli typ `Interaction`, opisujący interakcje, oraz funkcje

```haskell
resettable :: Interaction -> Interaction
withStartScreen :: Interaction -> Interaction
```

moglibyśmy uzyskać pożądany efekt przy pomocy ich złożenia. Potrzebowalibyśmy jeszcze funkcji

```haskell
runInteraction :: Interaction -> IO ()
```

Jak możemy zdefiniować taki typ `Interaction` ? 

Na razie obrazek, zeby zasugerować rozwiązanie, ale jeszcze go nie zdradzać:

![Cat in a box with a cat in a box](https://i.redd.it/k5mjhyewkxdz.jpg)

Musimy opakować argumenty funkcji `interactionOf` wewnątrz typu `Interaction`:

```haskell
data Interaction world = Interaction
        world
	(Double -> world -> world)
	(Event -> world -> world)
	(world -> Picture)
```
Zwróćmy uwagę, że dla pełnej ogólności typ świata `world` musi być parametrem typu `Interaction`.

Implementacja funkcji `resettable` nie przedstawia większych trudności - musimy po prostu wypakowac potrzebne wartości
przy pomocy dopasowania wzorca:

```haskell
resettable :: Interaction s -> Interaction s
resettable (Interaction state0 step handle draw)
  = Interaction state0 step handle' draw
  where handle' (KeyPress key) _ | key == "Esc" = state0
        handle' e s = handle e s
```

Implementacja (a conajmniej zapisanie typu) funkcji `withStartScreen` wymaga chwili namysłu. Zauważmy, że funkcjonalność tę osiagliśmy przez rozszerzenie stanu świata:

```haskell
data SSState world = StartScreen | Running world
```

Sygnatura naszej funkcji moze wyglądać tak:

```haskell
withStartScreen :: Interaction s -> Interaction (SSState s)
```

a implementacja np. tak:

```haskell
withStartScreen (Interaction state0 step handle draw)
  = Interaction state0' step' handle' draw'
  where
    state0' = StartScreen

    step' _ StartScreen = StartScreen
    step' t (Running s) = Running (step t s)

    handle' (KeyPress key) StartScreen
         | key == " "                  = Running state0
    handle' _              StartScreen = StartScreen
    handle' e              (Running s) = Running (handle e s)

    draw' StartScreen = startScreen
    draw' (Running s) = draw s
 ```
 
 Do kompletu potrzebujemy jeszcze funkcji `runInteraction`.
 
 :pencil: Napisz funkcję `runInteraction :: Interaction s -> IO ()`
 
 :pencil: Przepisz funkcje `walk2` i `walk3` ze swojego rozwiązania tak aby używały funkcji `runInteraction` i `resettable`.
 
 :pencil: Napisz funkcję `walk4 :: IO ()` rozszerzającą `walk3` o ekran startowy.

# :pencil: Zadanie - Sokoban 3

https://github.com/jnp3-haskell-2018/sokoban-3

Oddawanie przez https://classroom.github.com/a/-SaOJf3w

Termin: 01.12.2017 06:00 UTC+0100
