title_image_path: template.jpg
category: Програма
tags:
  - elixir
  - typespec
  - behaviour
  - type
  - typep
  - spec
  - dialyzer
  - polymorphism

--------

# Тип-спецификации и поведения

Преди да говорим за `OTP` и мета-програмиране е добре да споменем една интересна
нотация, идваща от `Erlang`. Това е дефиницията на тип-спецификациите и защо
са полезни.
Част от тази спецификация са поведенията, които са ползвани доста в някои `OTP`
модули, с които ще се запознаем скоро.

## Задаване на тип-спецификация

В [документацията](https://hexdocs.pm/elixir/typespecs.html#types-and-their-syntax)
можете да видите всички възможни типове. Ние ще ви представим синтаксиса.
Ще използваме модул със структура представляваща състоянието и дъската на
играта 'морски шах':

```elixir
defmodule TicTacToe.Grid do
 @dimensions 3

 defstruct [
    state: :playing,
    winner: nil,
    turn: 0,
    board: (
      for _ <- 1..@dimensions, j <- [nil], do: List.duplicate(j, @dimensions)
    )
  ]
end
```

Структурата държи:
* Състояние - дали играта се играе или е завършила. Използват се атомите `:playing` или `:finished`.
* Победител - `nil`, когато все още няма победител или играта е патова. Иначе `:x` или `:o`
* Ред - колко реда са изиграни до момента, започваме от `0`.
* Самото поле за игра - матрица. В случая списък от списъци с по три елемента. Всеки елемент може да е `nil`, `:x` или `:o`.

Горните четири точки не са видни от самата дефиниция на модула и структурата.
Бихме могли да ги добавим в `@moduledoc` документация.
Но с тип-спецификации можем да ги направим видни от 'пръв поглед'. Ето така:

```elixir
defmodule TicTacToe.Grid do
 @dimensions 3

 defstruct [
    state: :playing,
    winner: nil,
    turn: 0,
    board: (
      for _ <- 1..@dimensions, j <- [nil], do: List.duplicate(j, @dimensions)
    )
  ]

  @type state :: :playing | :finished
  @type player :: :o | :x
  @type player_or_nil :: player | nil

  @type t :: %__MODULE__{
    state: state, winner: player_or_nil, turn: non_neg_integer,
    board: [[player_or_nil]]
  }
end
```

Нека разгледаме какво направихме.

Първо дефинираме типът 'състояние' - `state` и казваме че това е или атомът
`:playing` или атомът `:finished`. Точно както го описахме по-горе.

Втората дефиниция на тип е `player`. Дефинираме типът като или `:o` или `:x`.
Сега ясно се вижда какви са символите, представляващи играч.

В някои случаи, да речем като искаме да опишем стойностите на полето `winner`,
използваме или стойност от `player` типа, или `nil`, затова дефинираме нов тип:
`player_or_nil`, който е `player` или `nil`. Можем да композираме типове както желаем.

Последната тип-спецификация е типът на структурата `TicTacToe.Grid`. Към този
тип ще можем да се обръщаме с `TicTacToe.Grid.t`. Прието е типовете на структури
да ги дефинираме с `t` в модула им. Нека да го разгледаме поле по поле:
1. `state` полето е от тип `state`. Точно както го описахме по-горе : `:playing` или `:finished`.
2. `winner` полето е от тип `player_or_nil`. Когато няма победител е `nil`, когато има е `:x` или `:y`. Отново - това описахме по-горе.
3. `turn` е `non_neg_integer`. Този тип-спецификация идва с `Elixir`. В [документацията](https://hexdocs.pm/elixir/typespecs.html#types-and-their-syntax) може да разгледате всички такива типове.
4. `board` е списък от списъци от `player_or_nil` стойности. `[]` е списък. `[[]]` е списък от списъци. `[[player_or_nil]]` е списък от списъци от `player_or_nil` стойности.

Виждате колко лесно можем да видим всички възможни стойности на дадено поле на структурата сега.
Тип-спецификациите ни помагат за това.
Когато дефинираме структура е добра идея да я опишем с тип-спецификация и да опишем нейните полета с типове.

Често ще искаме да добавим или използваме типове за да опишем типовете на стойностите, които функции приемат или връщат.
Всъщност има начин на опишем самата функция и какви типове приема и връща тя.
Това става със тип-спецификацията за функции.

## Тип-спецификации за функции

Нека да добавим функция `set` към `TicTacToe.Grid`. Тя ще взима `Grid` структура,
символ на играч и две числа - ред и колона. Нещо такова:

```elixir
defmodule TicTacToe.Grid do
  def set(grid, player, i, j)
end
```

Тази функция ще връща наредена-двойка. Ако е възникнал проблем, да речем полето,
сочено от `i` и `j` е вече заето, резултатът ще е `{:error, <съобщение>}`, ако
има успех, резултатът ще е `{:ok, <grid-с-поставен-символ-на-играч-на-дадената-позиция>}`.

Ето как ще опишем това с тип-спецификации:

```elixir
defmodule TicTacToe.Grid do
  @type mod_result :: {:error, String.t} | {:ok, TicTacToe.Grid.t}

  @spec set(t, player, i :: pos_integer, j :: pos_integer) :: mod_result
  def set(grid, player, i, j)
end
```

Първо дефинираме нов тип - `mod_result` : резултат от модификация на `Grid`.
Този тип е или наредена двойка от атома `:error` и низ - съобщение, или
наредена двойка от атома `:ok` и `Grid`. Точно каквото описахме за резултата
от `set` по горе.

Тип-спецификации на функции задаваме със `@spec`, следван от името на функцията,
аргументи в скоби, `::` и тип на резултат.

В случая за функцията `set` казваме че приема `4` аргумента:
1. Структура от тип `TicTacToe.Grid.t`, можем да го реферираме просто като `t` вътре в модула.
2. Стойност на типа `player`, това значи `:x` или `:o`.
3. Ред - представен от `i`, който е положително цяло число.
4. Колона - представена от `j`, която е положително цяло число.

Като резултат на функцията задаваме тип `mod_result`.

Сега знаем какво приема и какво връща тази функция.

## Още модулни атрибути свързани с тип-спецификации

Освен с `@type`, можем да задаваме тип спецификации с `@typep`. Това са скрити
типове, които не могат да се реферират от други модули, но могат да се реферират
от модулът в който са дефинирани.

Има още един начин за задаване на тип-спецификация. Говорим за типове, които
могат да се реферират публично, но структурата им не е видима. Те се дефинират
с `@opaque`.

Какво се има предвид? Да речем в `iex`, можем да прегледаме даден тип или типове:

```elixir
iex> t TicTacToe.Grid.t
@type t() :: %TicTacToe.Grid{board: [[player_or_nil()]], state: state(), turn: non_neg_integer(), winner: player_or_nil()}
```

Ако бяхме дефинирали този тип като `@opaque`, щяхме да видим:

```elixir
iex> t TicTacToe.Grid.t
@opaque t()
```

Скритите типове, дефинирани с `@typep` не се виждат. Ако искаме да видим всички
видими типове дефинирани в модул го правим с `t <име-на-модул>`:

```elixir
iex> t TicTacToe.Grid  
@type player() :: :o | :x
@type player_or_nil() :: player() | nil
@type state() :: :playing | :finished
@type t() :: %TicTacToe.Grid{board: [[player_or_nil()]], state: state(), turn: non_neg_integer(), winner: player_or_nil()}
@type mod_result() :: {:error, String.t()} | {:ok, TicTacToe.Grid.t()}
```

Още за типове можете да прочетете в [документацията](https://hexdocs.pm/elixir/typespecs.html).
Нека сега разгледаме инструмента който прави проверки за типовете - `dialyzer`.

## Dialyzer

`Dialyzer` е инструмент за анализиране на `BEAM bytecode` или `Erlang source code`.
Способен е да намери грешно ползване/подаване на типове, когато работи върху компилиран
код с тип-спецификации. Освен това намира недостижим/мъртъв код и ненужни тестове.

Името на `Dialyzer` означава `DIscrepancy AnalYZer for ERlang` или анализатор за несъответствия в `Erlang` код.
Хубавата новина е, че това, че `Dialyzer` работи върху `BEAM bytecode`, значи че можем да го ползваме да анализира `Elixir`.
Тъй като инструментът е малко неудобен за работа, има `Elixir` интерфейс - `mix` задача/библиотека, наречена `Dialyxir`.

Можем лесно да си добавим `dialyxir` като зависимост с:

```elixir
defp deps do
  [{:dialyxir, "~> <x>.<y>", only: [:dev]}]
end
```

И ще имаме на разположение `mix` задачата `mix dialyzer`.
Първият път когато го пуснем (с `mix dialyzert`) ще отнеме доста време.
Това е така, защото инструментът ще си построи `PLT (Persistent Lookup Table)` която
ще използва в последствие. Тя включва стандартната библиотека и затова отнема толкова време.
За щастие следващите пъти, когато го използваме в проекта ще минава бързо и ще проверява
само новите промени по кода.

Нека извикаме функцията `set` от предишните примери с аргументи, които не са правилни типове.
Да речем за `player` да подадем `:y`:

```elixir
grid = %TicTacToe.Grid{}
TicTacToe.Grid.set(grid, :y, 1, 1)
```

Нека сега да пуснем `mix dialyzer`. Ще видим следният проблем:

```
The call 'Elixir.TicTacToe.Grid':set(
  grid@1::#{
    '__struct__':='Elixir.TicTacToe.Grid',
    'board':=[['nil',...],...],
    'state':='playing',
    'turn':=0,
    'winner':='nil'
  },
  'y',
  1,
  1
)
breaks the contract
(t(), player(), i::pos_integer(), j::pos_integer()) -> mod_result()
```

Това е добре, виждаме че не всички параметри, които подаваме на функцията са от типовете,
които са декларирани. Ако променим `:y` на `:x`, `dialyzer` ще приключи с успех.

Ако използваме тип-спецификации, не само че правим кода си `self-documented` и
по-разбираем, но когато ги съчетаем и с `dialyzer` е по-лесно да засечем проблеми, които
биха възникнали `runtime`.
Статично-типизираните езици имат проверка на типовете по време на компилация.
Ние можем да постигнем нещо такова с тип-спецификациите и `dialyzer`.

## Поведения

Поведенията дефинират множество от спецификации за функции, които даден модул,
ако иска да изпълни даденото поведение, трябва да дефинира и имплементира.

Те са нещо като интерфейси. Компилаторът ще ни нотифицира с `warning` ако даден
модул е дефинирал, че ще изпълни поведението, но някои от функциите не са
дефинирани и имплементирани.

Най-добре дадени концепции се разбират и овладяват с примери, затова ние ще
дефинираме поведение `SynchronousCallHandler`. Това е поведение на процес,
който получава и отговаря на съобщения синхронно.

```elixir
defmodule SynchronousCallHandler do
  @type state :: term

  @callback init(args :: term) ::
    {:ok, state} |
    {:stop, reason :: term}

  @callback handle_call(request :: term, pid, state) ::
    {:reply, reply, new_state} |
    {:no_reply, new_state} |
    {:stop, reason} when reply: term, new_state: term, reason: term
end
```

Както виждате дефинираме нормален модул, но в него няма имплементирани функции,
само тип-спецификации за функции.

Когато използваме `@callback`, ние декларираме функции на поведение. Тези
функции трябва да бъдат дефинирани от модули, които изпълняват поведението.

Първата функция на поведението е `init`. Идеята ѝ е да бъде нещо като конструктор
на процеса. Тя се извиква, когато процесът е създаден с аргументи от какъвто и да е тип.
Ако се върне `{:ok, state}`, процесът е успешно създаден и държи състоянието `state`.
Ако пък имплементацията върне `{:stop, reason}`, процесът не може да бъде създаден правилно и ще
бъде 'убит'.

Втората функция, `handle_call` приема `request`, което е съобщение, изпратено от друг процес,
`pid`-a на процесът, който изпраща съобщението и текущото състояние.
Тя може да реагира по три начина:
1. Да върне `{:reply, reply, new_state}`. `reply` е съобщението което трябва да бъде изпратено на дадения `pid`. Точно това прави това поведение, поведение на комуникиращ синхронно процес - процесът, който 'извиква' `handle_call`, чрез съобщение, ще трябва да чака за отговор. Също това извикване може да промени състоянието, новото състояние е последният елемент на наредената тройка, върната от функцията.
2. Да върне `{:no_reply, :new_state}` - просто може да се промени състоянието. Отговор към процеса-изпращач би могъл да е атомът `:ok`.
3. Да върне `{:stop, reason}`, което би трябвало да спре текущия процес с дадената причина.

Разбира се това поведение може да бъде имплементирано по различни начини. Състоянието може да бъде специфичен тип и `handle_call` ще може да приема различни типове `request`.
Самата логика за работа с една имплементация на `SynchronousCallHandler` трябва да дефинираме ние.
Ето една дефиниция:

```elixir
defmodule Caller do
  @type on_start :: {:ok, pid} | {:error, term}

  @spec start(module, any) :: on_start
  def start(module, args) when is_atom(module) do
    starter_pid = self()

    pid = spawn(fn ->
      case module.init(args) do
        {:ok, state} ->
          send(starter_pid, {:ok, self()})
          run(module, state)
        {:stop, reason} ->
          send(starter_pid, {:error, reason, self()})
          exit(reason)
      end
    end)

    receive do
      {:ok, ^pid} -> {:ok, pid}
      {:error, reason, ^pid} -> {:error, reason}
    end
  end
end
```

В модула `Caller` ще сложим няколко 'клиентски' функции за работа с `SynchronousCallHandler`-и.
Първата дефинираме по-горе, тя стартира един процес използвайки модул, имплементиращ `SynchronousCallHandler`.

Функцията `start` приема модула и аргументи, които да подаде на `init` и връща `on_start` резултат.
* `{:ok, pid}` означава успех и имаме `pid`-а на `SynchronousCallHandler`-а.
* `{:error, term}` означава че процесът не може да функционира правилно с тези аргументи и е терминиран.

Говорим си за синхронни извиквания, затова всички функции в `Caller` работят синхронно - чакат отговор от `SynchronousCallHandler` имплементацията.

Как работи `start`?
1. Пуска нов процес и в него извиква `init` функцията на имплементацията на `SynchronousCallHandler` с подадените аргументи.
2. Ако `init` върне `{:ok, state}` - изпращаме отговор на процеса, извикал `start` - `{:ok, self()}`. Извикваме `run` - това е `run loop`-ът на процеса.
3. Ако `init` върне `{:stop, reason}` - изпращаме отговор на процеса, извикал `start` - `{:error, reason, self()}` и излизаме с дадената причина.
4. Извикващият процес чака за отговор. Ако отговорът е, че всичко е наред връщаме `{:ok, pid}`, aко има проблем връщаме `{:error, <причина>}`.

Използваме `pid`-а на процеса `SynchronousCallHandler` в `pattern matching`-а за да сме сигурни че получаваме отговори точно от него.

Нищо сложно. Нека сега да дефинираме `run`:

```elixir
defmodule Caller do
  defp run(module, state) do
    receive do
      {pid, msg} when is_pid(pid) ->
        on_call_result(module.handle_call(msg, pid, state), pid, module)
      _ -> :ok
    end
  end

  defp on_call_result({:reply, reply, new_state}, pid, module) do
    send(pid, {self(), reply})
    run(module, new_state)
  end

  defp on_call_result({:no_reply, new_state}, pid, module) do
    send(pid, {self(), :no_reply})
    run(module, new_state)
  end

  defp on_call_result({:stop, reason}, pid, _) do
    send(pid, {self(), :stop})
    exit(reason)
  end
end
```

Функцията `run` не е публична, тя е имплементация на трансформирането на изпратено съобщение
в `handle_call` извикване.

Това е едно извикване на `receive`, което пропуска непознати съобщения, за да не ни задръстват опашката от съобщения и прихваща `{pid, <съобщение>}`.
Когато едно такова съобщение е прихванато се използва модула, имплементиращ `SynchronousCallHandler` поведението. Извикваме му `handle_call` със съобщението, `pid`-а който получихме и текущото състояние.
Използваме `on_call_result` за да върнем отговор - подаваме му резултата от `handle_call`, `pid`-а и модула, имплементиращ поведението.

Функцията `on_call_result` има три случая:
1. Да 'отговори' на извикващия процес с дадено съобщение, при което състоянието може да се смени. Разбира се `run` се извиква рекурсивно.
2. Да не отговаря - пак може да се смени състоянието, и трябва да се нотифицира процеса, че това е така с `:no_reply`. Отново се извиква `run`.
3. Да прекрати изпълнението на процеса. Изпраща отговор, че ще го направи и излиза с причината.

Това е цикълът на изпълнение на един `SynchronousCallHandler`.

Последната функция, която ще дефинираме в `Caller` е клиентска функция за лесно синхронно извикване на `handle_call`:

```elixir
defmodule Caller do

  @spec call(pid, msg :: any) :: :ok | :stopped | response :: any
  def call(pid, msg) do
    send(pid, {self(), msg})

    receive do
      {^pid, :no_reply} -> :ok
      {^pid, :stop} -> :stopped
      {^pid, response} -> response
    end
  end
end
```

Просто праща съобщение във формата `{pid, <съобщение>}` и чака за отговор - синхронно извикване.

## Имплементация на поведение

Можем да иплементираме `Wrapper` от предната статия като `SynchronousCallHandler`.
Ето така:

```elixir
defmodule Wrapper do
  @behaviour SynchronousCallHandler

  def init(state) do
    {:ok, state}
  end

  def handle_call(:get, _, state) do
    {:reply, state, state}
  end

  def handle_call({:set, new_state}, _, state) do
    {:no_reply, new_state}
  end

  def handle_call({:get_and_update, f}, _, state) when is_function(f) do
    {:no_reply, f.(state)}
  end

  def handle_call({:stop, reason}, _, _) do
    {:stop, reason}
  end
end
```

С директивата `@behaviour` дефинираме, че даден модул имплементира дадено поведение.
Както знаем трябва да имаме `init` и `handle_call` с правилните типове/резултати.

`init` просто взима състояние и го връща в наредена-двойка за успех. Все пак това е процес, опаковащ състояние. Състоянието ни е аргумента на `init`.

`handle_call` имплементира съобщенията ни за четене и модификация на състояние, както и за спиране на `Wrapper`.

Ето пример как го използваме:

```elixir
{:ok, pid} = Caller.start(Wrapper, 5)
# {:ok, #PID<0.111.0>}

Caller.call(pid, :get)
# 5

Caller.call(pid, {:set, 6})
# :ok
Caller.call(pid, :get)
# 6

Caller.call(pid, {:get_and_update, &Kernel.*(&1, 2)})
# :ok
Caller.call(pid, :get)
# 12

Caller.call(pid, {:stop, :tired})
# :stopped
Process.alive?(pid)
# false
```

Лесно можем да имлементираме някаква структура от данни, да речем дърво, която живее в процес и операциите над нея.

## Полиморфизъм

В [статията за протоколи](/posts/protocols.md) видяхме как можем да имплементираме функция, която
може да работи с безкрайно много типове дата. Даже с такива, за които още не знаем - някой друг ще я имплементира.
Друг начин да постигнем полиморфизъм е с поведения. Всъщност има и трети начин, който вече няколко статии ползваме.

Намерихме много интересна [тема](https://groups.google.com/forum/#!msg/elixir-lang-talk/Vq--xwBMmAI/WRjH3NKlZVYJ), дискутирана в `mailing` листата на `Elixir`.
_Жозе_ казва, че можем да разделим кода си на три групи: процеси, модули и данни, които са взаимно-свързани.
Процесите изпълняват код, който е дефиниран и структуриран в модули, които работят с дадени типове данни.

Всяка от тези три категории има своя начин да постигне полиморфизъм:
1. Можем да пратим едно и също съобщение на множество процеси. В зависимост от кода в процеса, ще имаме различно поведение. Не се интересуваме от процесите, важно е дали слушат за точно този тип съобщение.
2. Когато викаме точно определена функция на модул, както по горе `module.handle_call`, не се интересуваме какъв е модулът, важно е да имплементира тази функция.
3. Когато извикваме функция от протокол с аргумент даден тип - не се интересуваме от типа, важното е, че функцията е имплементирана за него.

Разликата между протоколите и поведенията е около какво се върти имплементацията - дали около тип данни или около модул.

Пишем протоколи, когато искаме някой да имплементира логика с определена форма за свой тип данни. Код, който искаме да бъде `extend`-нат за нов тип данни.

Пишем поведения, когато имаме код работещ с набор от функции и искаме някой друг да може да имплементира тези функции.
Когато пишем система, която може да се разширява с `plugin`-и, когато идеята ни е да пишем код, който може да сменя имплементации лесно,
когато самият ни код има места, които могат да се разширят от някой друг.

## Заключение

В следващите статии ще говорим за някои поведения идващи с `OTP` платформата, които са написани с идеята да работят с клиентски код.
Досещате се че става въпрос за нещо, в което са намесени поведения и процеси.