# pitcp
=====
Just a project for researching, don't used.

# LICENSE
- The [MIT License](./LICENSE)  
- Copyright (c) 2018~2019 [Naupio Z.Y Huang](https://github.com/Naupio) 

# Pre Install
-----
    You must install erlang/otp >= 21.0 and install **rebar3** build tool  

# Build
-----

    $ rebar3 compile

# Example code


```erlang
-module(pitcp_test).

-author("Naupio Z.Y. Huang").

-behaviour(gen_server).
-behaviour(pitcp_protocol).

-define(SERVER, ?MODULE).

%% gen_server callback function
-export([init/1, handle_call/3, handle_cast/2, handle_info/2, terminate/2, code_change/3]).

%% pitcp_protocol callback function
-export([start_tcp/4]).

%% api
-export([tcp_send/2]).

%% test function
-export([run_listener/0,spawn_conn/1]).

run_listener() ->
    Ref = pitcp_test_ref,
    LisOpt = [binary,{port,18080},{packet,0},{active,false},{ip,{127,0,0,1}}],
    ProMod = pitcp_test,
    ProModOpt = [],
    OtherOpt = [],
    pitcp:start_listener(Ref, LisOpt, ProMod, ProModOpt, OtherOpt).

spawn_conn(TestNum) when is_number(TestNum) andalso (TestNum > 0) ->
    lists:foreach( fun(MsgNum) ->
        spawn(fun() ->
            {ok,Socket} = pitcp_util:connect({127,0,0,1},18080,[{active,false}]),
            pitcp_util:send(Socket,<<MsgNum>>),
            error_logger:info_msg("~n send: ~w ~n",[MsgNum])
        end)
    end,
    lists:seq(1,TestNum))
    .

%% main code

start_tcp(Ref, ClientSocket, ProModOpt, OtherOpt) ->
    start_link(Ref, ClientSocket, ProModOpt, OtherOpt).

start_link(Ref, ClientSocket, ProModOpt, OtherOpt) ->
    gen_server:start_link(?MODULE, [Ref, ClientSocket, ProModOpt, OtherOpt], []).

init([Ref, ClientSocket, ProModOpt, OtherOpt]) ->
    self() ! init,
    pitcp_util:setopts(ClientSocket,[{active, once}]),
    State = #{client_socket => ClientSocket
            , ref => Ref
            , pro_mod_opt => ProModOpt
            , other_opt => OtherOpt
            },
    {ok, State}.

handle_call(get_client_socket, _From, #{client_socket := ClientSocket}=State) ->
    {reply,ClientSocket,State};

handle_call(_Msg, _From, _State) ->
    {reply, _Msg, _State}.


handle_cast({send,Data}, #{client_socket := ClientSocket}=State) ->
    pitcp_util:send(ClientSocket,Data),
    {noreply, State};

handle_cast(_Msg, _State) ->
    {noreply, _State}.
    
handle_info(init, _State) ->
    {noreply, _State};

handle_info({tcp,ClientSocket,Data}, #{client_socket := ClientSocket}=State) ->
    pitcp_util:setopts(ClientSocket,[{active, once}]),
    ReplyData = handle_tcp_data(Data),
    tcp_send(self(),ReplyData),
    {noreply, State};

handle_info(_Msg, _State) ->
    {noreply, _State}.

terminate(_Reson, _State) ->
    ok.

code_change(_OldVsn, _State, _Extra) ->
    ok.

handle_tcp_data(Data) ->
    error_logger:info_msg("~n receive: ~w ~n",[Data]),
    Data.

tcp_send(ClientPID,Data) ->
    gen_server:cast(ClientPID,{send,Data}).
```
