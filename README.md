## Erlang, podstawy programowania równoległego (concurrent) i rozproszonego (distributed).

dokumentacja: https://www.erlang.org/doc/system/conc_prog.html

### Procesy i komunikacja:

Proces w Erlangu = lekki, niezależny wątek w maszynie wirtualnej.
Możemy zobaczyć listę obecnie aktywnych procesów (tylko PIDy) używając funkcji:
`processes().`
Do tworzenia procesu używamy `spawn(fun)`:

`spawn(fun() -> io:format("hello~n") end ).`

lub `spawn(moduł, funkcja, lista_argumentów)`:

`spawn(server, loop, []).`

spawn zwraca ID procesu (PID)

```
Przykładowy moduł:
-module(spawndemo).
-export([start/0, say/2]).

say(What, 0) ->
    done;
say(What, Times) ->
    io:format("~p~n", [What]),
    say(What, Times - 1).

start() ->
    spawn(spawndemo, say, [hello, 3]),
    spawn(spawndemo, say, [goodbye, 3]).
```


Cała komunikacja między procesami odbywa się przez asynchroniczne sygnały. Najczęściej występującymi sygnałami są wiadomości (message signals).

Do otrzymania wiadomości używamy wyrażenia
```
receive
   Format_wiadomości1 ->
       Reakcja1;
   Format_wiadomości2 ->
       Reakcja2;
   ....
   Format_wiadomościN ->
       ReakcjaN
end.
```
`self()` zwraca ID procesu (PID).

Do wysyłania wiadomości używamy operatora `!` w formacie `Pid ! Wiadomość`
np.
`receive {Client, {Str, echo}} -> Client ! {self(), Str} end.`

Przykładowy serwer:
```
-module(server).
-export([start/0, loop/0]).

start() ->
% tworzy proces modułu server, uruchamia funkcję loop
	spawn(server, loop, []).

loop() ->
% gdy otrzyma wiadomość
	receive
% w formacie {ID, {jakiś_tekst, echo}}
		{Client, {Str, echo}} ->
% to wysyła do klienta {ID_serwera, jakiś_tekst}
			Client ! {self(), Str};
% analogicznie zamiana na małe lub wielkie litery
		{Client, {Str, uppercase}} ->
			Client ! {self(), string:to_upper(Str)};
		{Client, {Str, lowercase}} ->
			Client ! {self(), string:to_lower(Str)}
% koniec klauzuli receive
	end,
% uruchom funkcję jeszcze raz
	loop().
```

Przykładowe wysłanie i odebranie wiadomości w konsoli:
```
% kompilacja serwera
c(server).
% PID serwera przypisany do zmiennej
Server = server:start().
% wysłanie wiadomości do serwera
Server ! {self(), {"Hello", uppercase}}.
% odebranie wiadomości z kolejki
receive X -> X end.
```

Przykładowy klient:
```
-module(client).
-export([run/3]).

run(Server, Str, Command) ->
	Server ! {self(), {Str, Command}},
	receive
		{Server, ResultString} ->
			ResultString
% jeśli serwer nie odpowiada przez 2 sekundy, zakończ z komunikatem
	after 2000 ->
		io:format("Timeout~n")
	end.
```
Przykładowe uruchomienie klienta:
`client:run(self(), "Hello", uppercase).`

---------------------

Zadanie:
Moduł będzie miał 3 funkcje:
start(), ping(N, Pong_PID), pong().
Po wywołaniu start():
Ping wysyła wiadomość do pong, pong odbiera wiadomość i wyświetla `Pong received ping`.
Pong odsyła wiadomość do ping. Ping odbiera wiadomość i wyświetla `Ping received pong`.
Powtarza się jeszcze 2 (N-1) razy (można ustawić na sztywno liczbę razy w start lub zrobić start z jednym argumentem).
Na koniec oba procesy kończą działanie.

przykład:
```
1> pingpong:start().
<0.98.0>
Pong received ping
Ping received pong
Pong received ping
Ping received pong
Pong received ping
Ping received pong
ping finished
Pong finished
```
---------------------

`register(atom, Pid)` pozwala przypisać wartość Pid do atomu wewnątrz modułu. Przydatne, gdy procesy są uruchamiane w jednym module niezależnie od siebie, szczególnie na różnych komputerach.

`registered()` zwraca listę zarejestrowanych nazw.

--------------------

Zadanie:
Przeróbcie zadanie z ping i pong tak, żeby ping był wywoływany z jednym argumentem N, bez Pong_PID (użyć funkcji register).

--------------------

Link, awarie, monitor:

`link(PID)` pozwala połączyć się obecnemu procesowi z drugim procesem w taki sposób, że gdy jeden z nich zakończy działanie nieoczekiwanie (błąd/exit()/nieobsłużony wyjątek), to wysyła komunikat `{'EXIT', PID, powód}` do drugiego procesu (wtedy drugi domyślnie też wymusi zatrzymanie).

`unlink(PID)` usuwa link

`spawn_link(fun)` lub `spawn_link(moduł, funkcja, lista_argumentów)` = spawn + link

`exit(powód)` zatrzymuje obecny proces
`exit(PID, powód)` zatrzymuje wybrany proces

Przykład działania:
```
% przodek tworzy potomka z linkiem, który po 2 sekundach powinien wypisać komunikat
Parent = self(),
Child = spawn_link(fun() ->
	io:format("Child started work~n"),
	timer:sleep(2000),   % potomek "pracuje"
	io:format("Child finished work~n") end),

% po sekundzie wymuszamy zakończenie dziecka:
timer:sleep(1000),
exit(Child, kill),
io:format("Parent: exit(Child, kill)~n"),
timer:sleep(1000),
io:format("Parent finished work~n").
```
-----------------

`process_flag(trap_exit, true)` zamienia sygnały EXIT w zwykłe wiadomości. Dzięki temu możemy je obsłużyć w receive.
```
% ustawiamy flagę trap_exit na true
process_flag(trap_exit, true),
% przodek tworzy potomka z linkiem, który po sekundzie wymusza zatrzymanie
Child = spawn_link(fun() ->
    timer:sleep(1000),
    exit(crash)
end),

io:format("Parent: expecting child exit~n"),

receive
    {'EXIT', PID, Reason} ->
        io:format("'EXIT' | PID: ~p | Reason: ~p~n",
                  [PID, Reason])
after 2000 ->
    io:format("Parent: nothing happened~n")
end,

io:format("Parent: end~n").
```
-------------------

`monitor(PID, fun)` lub `spawn_monitor(fun)` lub `spawn_monitor(moduł, funkcja, lista_argumentów)`
pozwala procesowi obserwować wybrany proces, i gdy ten wymusi zatrzymanie, monitorujący dostanie wiadomość w formacie `{'DOWN', Ref, process, Pid2, Reason}`
```
% przodek tworzy potomka z monitorem, który po sekundzie wymusza zatrzymanie
{Child, Ref} = spawn_monitor(fun() ->
    timer:sleep(1000),
    exit(crash)
end),

io:format("Parent: expecting child exit~n"),

receive
    {'DOWN', Ref, process, PID, Reason} ->
		io:format("'DOWN' | Ref: ~p | PID: ~p | Reason: ~p~n",
		[Ref, PID, Reason])
after 2000 ->
    io:format("Parent: nothing happened~n")
end,

io:format("Parent: end~n").
```

---------------

Zadanie:
Połączcie moduł serwera i klienta w jeden moduł. Zróbcie tak, żeby serwer po zabiciu automatycznie się wskrzeszał (zostawcie też jakiś sposób na wyłączenie serwera).

przykład:
```
2> Server = restarter:start().
<0.87.0>
3> restarter:client("Hello", uppercase).
"HELLO"
4> exit(Server, ded).
true
5> restarter:client("Hello", uppercase).
"HELLO"
```
-----------------------

Zadanie dodatkowe:
Napiszcie moduł, który: startuje 5 procesów-workerów, monitoruje ich (każdy ma swój Ref), a kiedy jakiś padnie, to wypisuje jego PID i Reason.

-----------------------

Programowanie rozproszone:

Aby komputery mogły komunikować się ze sobą w Erlangu, muszą mieć taką samą zawartość pliku `.erlang.cookie`
Lokalizacja:
Windows: zmienna środowiskowa `$HOME`, można sprawdzić za pomocą `init:get_argument(home).`
Linux: katalog domowy.
Na Linux'ie muszą być konkretne uprawnienia:
`chmod 400 .erlang.cookie`

W pliku `.erlang.cookie` powinien być jeden wyraz: atom, który będzie identyczny dla wszyskich komputerów, które chcą się ze sobą komunikować.

Można (dla programowania rozproszonego nawet trzeba) uruchomić erlang'a z parametrem `-name erl_name@IP`, gdzie erl_name = wybrana nazwa użytkownika, np.:
`> erl -name lab94@192.168.128.94`

Najłatwiej uruchomić erlang'a z już ustawioną nazwą i cookie:
`> erl -name erl_name@IP -setcookie <cookie>`

----------------------
Jeśli wysyłamy wiadomości między systemami i używamy atomów zamiast Pid'ów z register/2, to zamiast
`registered_name ! Message`
jest
`{registered_name, node_name} ! Message`, czyli atom uzyskany z register/2 i nazwa komputera z -name

Maszyna wirtualna Erlang'a podłączona do sieci nazywa się node (węzeł).

----------------------

Przykładowy moduł rozproszony: czat grupowy
Uwaga: przed kompilacją musimy ustawić faktyczną nazwę node'a serwera w kodzie.

----------------------
```
-module(discord).
-export([start/0, server/1, logon/1, logoff/0, send/1, client/2, stop/0]).

%%% Change the function below to return the name of the node where the server runs
server_node() ->
    'server_erl_name@server_IP_Address'.

stop() -> exit(self()).
    
%%% Start the server
start() ->
    register(discord, spawn(discord, server, [[]])).

%%% This is the server process for the "discord"
%%% the user list has the format [{ClientPid1, Name1},{ClientPid22, Name2},...]
server(User_List) ->
    receive
    
        {From, logon, Name} ->
            New_User_List = server_logon(From, Name, User_List),
            server(New_User_List);
            
        {From, logoff} ->
            New_User_List = server_logoff(From, User_List),
            server(New_User_List);
            
        % group message
        {From, broadcast, Message} ->
            server_broadcast(From, Message, User_List),
            server(User_List)
            
    end.

%%% Server adds a new user to the user list
server_logon(From, Name, User_List) ->
    % check if logged on anywhere else
    case lists:keymember(Name, 2, User_List) of
        true ->
            From ! {discord, stop, user_exists_at_other_node},  %reject logon
            User_List;
        false ->
            From ! {discord, logged_on},
            % inform others that a new user joined
            broadcast_system_msg(Name, "joined the chat", User_List),
            [{From, Name} | User_List]
    end.

%%% Server deletes a user from the user list
server_logoff(From, User_List) ->
    case lists:keysearch(From, 1, User_List) of
        {value, {From, Name}} ->
            broadcast_system_msg(Name, "left the chat", User_List),
            lists:keydelete(From, 1, User_List);
        false ->
            User_List
    end.

%%% Server broadcasts a message to all users
server_broadcast(From, Message, User_List) ->
    case lists:keysearch(From, 1, User_List) of
        false ->
            From ! {discord, stop, you_are_not_logged_on};
        {value, {From, Name}} ->
            % send message to all users except sender
            lists:foreach(
              fun({Pid, _User}) ->
                      Pid ! {message_from, Name, Message}
              end, User_List),
            From ! {discord, sent}
    end.
    
% broadcast system messages like "User joined the chat"
broadcast_system_msg(FromName, Info, User_List) ->
    lists:foreach(
      fun({Pid, _User}) ->
              Pid ! {system_message, io_lib:format("~s ~s", [FromName, Info])}
      end, User_List).


%%% User Commands
logon(Name) ->
    case whereis(mess_client) of
        undefined ->
            register(mess_client,
                     spawn(discord, client, [server_node(), Name]));
        _ -> already_logged_on
    end.

logoff() ->
    mess_client ! logoff.

send(Message) ->
    case whereis(mess_client) of % Test if the client is running
        undefined ->
            not_logged_on;
        _ -> mess_client ! {broadcast, Message},
             ok
end.


%%% The client process which runs on each server node
client(Server_Node, Name) ->
    {discord, Server_Node} ! {self(), logon, Name},
    await_result(),
    client(Server_Node).

client(Server_Node) ->
    receive
        logoff ->
            {discord, Server_Node} ! {self(), logoff},
            exit(normal);
        {broadcast, Message} ->
            {discord, Server_Node} ! {self(), broadcast, Message},
            await_result();
        {message_from, FromName, Message} ->
            io:format("~s: ~s~n", [FromName, Message]);
        {system_message, Message} ->
            io:format("~s~n", [Message])
    end,
    client(Server_Node).

%%% wait for a response from the server
await_result() ->
    receive
        {discord, stop, Why} -> % Stop the client
            io:format("~p~n", [Why]),
            exit(normal);
        {discord, What} ->  % Normal response
            io:format("~p~n", [What])
    end.
```
