# OTP w Erlang – budowanie serwera krok po kroku

W tym repozytorium pokazujemy, jak od prostej, ręcznej implementacji serwera dojść do prawdziwego OTP `gen_server` oraz supervisorów.

Proces składa się z czterech kroków:

1. **Naiwny, ręczny serwer (`kitty_server`)**
2. **Wyodrębnienie części wspólnej – tworzymy mini-framework (`my_server`)**
3. **Implementacja konkretnego serwera w oparciu o framework (`kitty_server2`)**
4. **Prawdziwy serwer OTP (`gen_server`) + włączenie go pod supervisora**

Każdy krok jest opisany w takim samym stylu:  
👉 *co mówisz na prezentacji*,  
👉 *co robi kod*,  
👉 *jak działają funkcje i moduły*.

---

# Krok 1 – ręczny serwer `kitty_server`

W pierwszym kroku piszemy serwer „sklepu z kotami” **bez żadnych udogodnień OTP**.

To znaczy:

- sami musimy pisać `spawn/spawn_link`,
- sami implementujemy pętlę `receive`,
- sami obsługujemy monitory,
- sami robimy time-outy, komunikaty i wysyłanie odpowiedzi.

## Jak to tłumaczyć na prezentacji

Tutaj mówisz:

> To jest najprostsza, „naiwna” wersja serwera.  
> Działa, ale ma jedną poważną wadę: **za każdym razem musimy pisać tę samą infrastrukturę od zera**.  
> Dlatego w kroku drugim wyciągniemy tę logikę do generycznego modułu.

### Co robi ten moduł?

- `start_link/0` uruchamia proces serwera.
- `order_cat/4` implementuje synchroniczne wywołanie (monitor, wysłanie wiadomości i czekanie na odpowiedź).
- `return_cat/2` to asynchroniczne powiadomienie.
- `close_shop/1` to synchroniczne zakończenie działania serwera.
- Cała logika przetwarzania wiadomości jest w pętli `loop/1`.

**Problem:** pętla `receive`, monitory, time-outy i wysyłanie odpowiedzi trzeba pisać ręcznie.

---

# Krok 2 – wyodrębnienie części wspólnej – `my_server`

Tworzymy moduł **`my_server`**, który staje się małym odpowiednikiem OTP `gen_server`.  
Ten moduł nie zna logiki aplikacji — obsługuje tylko:

- uruchamianie procesu,
- przechowywanie stanu,
- pętlę `receive`,
- obsługę wywołań synchronicznych (`call/2`) i asynchronicznych (`cast/2`),
- wysyłanie odpowiedzi (`reply/2`),
- wywoływanie callbacków modułu użytkownika:
  - `init/1`,
  - `handle_call/3`,
  - `handle_cast/2`.

## Jak wytłumaczyć Krok 2 na prezentacji

Możesz powiedzieć:

> W kroku drugim tworzymy własny mini-framework serwerowy — moduł `my_server`.  
> On przejmuje całą logikę „infrastrukturalną”: pętlę receive, obsługę monitorów, call/cast, zarządzanie stanem procesu.  
> Dzięki temu przyszłe serwery będą musiały implementować tylko swoją logikę biznesową, a nie mechanikę komunikacji międzyprocesowej.

---

## Wyjaśnienie funkcji `my_server` (w stylu slajd-prezentacji)

### start/2 oraz start_link/2

> Obie funkcje uruchamiają nowy proces serwera.  
> Różnią się tym, że `start_link/2` linkuje proces z rodzicem.  
> Uruchamiany proces od razu wywołuje `my_server:init(Module, InitialState)`.

### call/2 – synchroniczne RPC

> `call/2` to synchroniczne wywołanie do serwera.  
> Ustawia monitor na proces, wysyła wiadomość `{sync, PidKlienta, Ref, Msg}`  
> i czeka na odpowiedź `{Ref, Reply}`.  
> Jeśli serwer padnie — klient dostanie `'DOWN'`.  
> Jeśli minie timeout — zostanie rzucony `timeout`.

### cast/2 – asynchroniczne powiadomienie

> `cast/2` wysyła `{async, Msg}` i zwraca `ok` bez czekania.  
> Idealne do prostych aktualizacji stanu.

### reply/2 – odpowiedź z serwera

> `reply({Pid, Ref}, Reply)` wysyła do klienta wiadomość `{Ref, Reply}` — dokładnie takiej odpowiedzi oczekuje `call/2`.

### init/2 i loop/2 – wnętrze serwera

> `init/2` wywołuje `Module:init/1`, a potem przechodzi do głównej pętli `loop/2`.  
>  
> `loop/2` rozdziela wiadomości na dwa typy:
>
> - `async` → wywołuje `Module:handle_cast/2`,  
> - `sync` → wywołuje `Module:handle_call/3`,  
>
> aktualizuje stan i wraca do siebie.

**Jedno zdanie podsumowania:**  
> `my_server` to mały framework serwerowy, który ukrywa całą mechanikę procesów i komunikacji — przyszłe serwery będą dzięki temu znacznie prostsze.

---

# Krok 3 – konkretny serwer na bazie frameworka – `kitty_server2`

W tym kroku pokazujemy, że mając `my_server` możemy napisać serwer kotów dużo prościej.  
Moduł `kitty_server2`:

- nie obsługuje pętli receive,
- nie używa monitorów,
- nie wysyła odpowiedzi ręcznie,
- nie buduje protokołu wiadomości.

Pisze tylko:

- logikę zamawiania kota,
- logikę zwracania kota,
- logikę zamykania sklepu.

---

## Jak wytłumaczyć Krok 3 – kitty_server2 (ten styl, o który prosiłeś)

Tutaj mówisz:

> W kroku trzecim mamy konkretną implementację serwera – `kitty_server2`.  
> Ten moduł zna już domenę, czyli „sklep z kotami”, ale nie zajmuje się infrastrukturą — od tego jest `my_server`.

---

# Publiczne API (to, co woła klient)

### start_link/0

> `start_link/0` wywołuje:
> ```erlang
> my_server:start_link(?MODULE, []).
> ```
> To znaczy:
> - uruchom serwer,
> - jako moduł callback użyj `kitty_server2`,
> - stan początkowy to `[]`, czyli pusta lista kotów w magazynie.

---

### order_cat/4

> `order_cat/4` robi synchroniczne zamówienie kota:
> ```erlang
> my_server:call(Pid, {order, Name, Color, Description}).
> ```
> Klient mówi: „daj mi kota”.  
> To, co dokładnie serwer zrobi w odpowiedzi, jest opisane w `handle_call/3`.

---

### return_cat/2

> `return_cat/2` to asynchroniczne zwrócenie kota:
> ```erlang
> my_server:cast(Pid, {return, Cat}).
> ```
> Klient niczego nie oczekuje — po prostu informuje serwer, że kot wrócił do sklepu.

---

### close_shop/1

> `close_shop/1` robi synchroniczne zamknięcie serwera:
> ```erlang
> my_server:call(Pid, terminate).
> ```
> Serwer odsyła `ok`, wypisuje komunikaty o kotach i kończy proces.

---

# Callbacki – logika biznesowa

### init/1

> `init([]) -> [].`  
> Stan początkowy to pusta lista kotów.

---

### handle_call/3 – obsługa synchronicznych zapytań

Pierwsza klauzula:

```erlang
handle_call({order, Name, Color, Description}, From, Cats) ->
    case Cats of
        [] ->
            my_server:reply(From, make_cat(Name, Color, Description)),
            Cats;
        [Cat | Rest] ->
            my_server:reply(From, Cat),
            Rest
    end;
