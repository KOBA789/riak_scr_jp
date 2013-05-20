=====================
Vector clocks と CRDT
=====================

メモ

- vector clocks 使用例
- pn-counter が PR あがってるのはなに?
- {default_bucket_props, [{allow_mult, true}]},
- client で同じ vclock で書き込みに行ったらどうなる?
- レプリカ/read-repair との関係
- statebox との絡み

話の流れ

- 読んで書く、並列更新、並列であることがどうやったら分かるか
  値に順序をメモっておく => 複数ノード(vnode) では vc になる
  TODO: レプリカとの関係
- vc で出来ること、siblings、ユーザが知っているべきこと
  - サーバ視点で見ると、書き込まれた "順序" を保持している
    レプリカ間の自動マージが出来る(ことがある)
  - クライアント視点で見ると、更新が衝突したら複数返ってくる(metadata, value)
    解決して書き戻せば良い、vclock の中身は気にしない
  - 読んで書く、いきなり書いたら siblings
  -
- vc マージ
- siblings1: single node にも siblings はあるか? どう格納されているか?
- sibligns2: multi nodes での siblings
- crdt 超ざっくり
- PN counter
- おまけ: PN counter を ruby から使ってみる
- おまけ: client ID での PN counter は可能か? (現実的に)

Riak Source Code Reading @Tokyo #11

:author: Shunichi Shinohara ( Github: @shino / Twitter: @itawasa )
:date: 2013-05-21
:riak: ``6b6db1d`` "Merge pull request #312"
:PN-counter PR for riak_kv: https://github.com/basho/riak_kv/pull/536
:CRDT for Riak: https://github.com/basho/riak_dt

.. contents:: :depth: 2

概観
====

複数レプリカへの書き込みの課題:

- レプリカ間の整合性を取りたい => 2PC、分散ロックなどを使う => 可用性が下がる
- レプリカそれぞれ個別に書き込み => 違うデータをもつ可能性がある

Riak は可用性を重視:

- レプリカ間の不整合は許容する
- 並列更新時の 2 つのモード

  1. 後勝ち
  2. サーバでは複数の更新を保持しておいて、クライアント側で不整合解消

- 複数保持のため、Riak は vector clocks を利用する
- 出来る範囲で整合性を保つ (Read-Repair, Active Anti-Entropy, Hinted-Handoff)
- アプリ要件によっては、クライアントで不整合解消ができる and/or やらなければいけない

参照/更新要件によってはある程度の自動化、定番化(ライブラリ化)が可能:

- CRDT (conflict-free replicated data types)
- クライアント側で出来ること(例: Set 系の型)、サーバサポートも欲しいこと(例: カウンター~

----

**免責** この資料では「並列更新」「同時更新」「更新の衝突」「衝突」という言葉を
ほとんど区別せずに使います。

因果関係と Vector clocks
========================

**因果関係を「メッセージ伝播」** で定義する by L. Lamport

- Lamport, L. (1978). "Time, clocks, and the ordering of events in a distributed system"
  http://research.microsoft.com/users/lamport/pubs/time-clocks.pdf
- メッセージパッシングでつながる2つの点は「過去と未来」である
- そうではないなら「過去でも未来でもない」 (concurrent, independent)

Vector clocks はそのひとつの表現(過去、未来、concurrent の3つの区別を実現する方法)

- http://en.wikipedia.org/wiki/Vector_clock
- プロセスごとにカウンタ(論理クロック)をもつ
- 2つの Vector clocks Va, Vb で

  - Va のすべてカウンタが Vb の対応するカウンタより小さいなら Va < Vb
  - Va のすべてカウンタが Vb の対応するカウンタより大きいなら Va > Vb
  - それ以外なら Va // Vb (concurrent)

Riak での Vector clocks: サーバ側とクライアント側
=================================================

Riak での Vector clocks における処理単位(プロセス)は vnode

- vnode は ID をもつ `riak_kv_vnode:get_vnodeid(Index)`

  - Erlang ノード名の CRC-32 + epoch 32-bit
  - ノード内の vnode ID は同じになることもある

- ファイルに永続化するので再起動でも引き継ぐ ex. `data/kv_vnode/<Index>` ::

クライアント側では...

- Vector clocks の実装方法も操作(increment, merge)も知らなくて良い
- ただし、衝突時には、複数の値が返ってきて、それらを **ひとつにして** (merge)
  書き戻す必要がある
- merge するものは値 (ex. JSON) とメタデータ (ex. 2i)
- 例えば、ショッピングカートなら ::

     {ノート:1冊, ペン:2本} + {ノート:1冊, 消しゴム:1つ}
     ===(merge)==> {ノート:1冊, ペン:2本, 消しゴム:1つ}

Vector clocks ソースコード
==========================

`riak_core/src/vclock.erl`

データ型::

   -opaque vclock() :: [vc_entry()].
   % The timestamp is present but not used, in case a client wishes to inspect it.
   -type vc_entry() :: {vclock_node(), {counter(), timestamp()}}.

`increment/2,3`::

   % @doc Increment VClock at Node.
   -spec increment(Node :: vclock_node(), VClock :: vclock()) -> vclock().
   increment(Node, VClock) ->
       increment(Node, timestamp(), VClock).

   % @doc Increment VClock at Node.
   -spec increment(Node :: vclock_node(), IncTs :: timestamp(),
                   VClock :: vclock()) -> vclock().
   increment(Node, IncTs, VClock) ->
       {{_Ctr, _TS}=C1,NewV} = case lists:keytake(Node, 1, VClock) of
                                   false ->
                                       {{1, IncTs}, VClock};
                                   {value, {_N, {C, _T}}, ModV} ->
                                       {{C + 1, IncTs}, ModV}
                               end,
       [{Node,C1}|NewV].

`merge/2` **ソース見る**

`descends/2` (descend : --の子孫である)::

   % @doc Return true if Va is a direct descendant of Vb, else false
   %      -- remember, a vclock is its own descendant!
   -spec descends(Va :: vclock()|[], Vb :: vclock()|[]) -> boolean().
   descends(_, []) ->
       % all vclocks descend from the empty vclock
       true;
   descends(Va, Vb) ->
       [{NodeB, {CtrB, _T}}|RestB] = Vb,
       case lists:keyfind(NodeB, 1, Va) of
           false ->
               false;
           {_, {CtrA, _TSA}} ->
               (CtrA >= CtrB) andalso descends(Va,RestB)
           end.

siblings : サーバ側
===================

`riak_kv/src/riak_object.erl`

データ型
(`riak-erlang-client/src/riakc_obj.erl` の `riakc_obj()` と順序を除いて同じ) ::

   -record(r_content, {
             metadata :: dict(),
             value :: term()
            }).

   %% Opaque container for Riak objects, a.k.a. riak_object()
   -record(r_object, {
             bucket :: bucket(),
             key :: key(),
             contents :: [#r_content{}],
             vclock = vclock:fresh() :: vclock:vclock(),
             updatemetadata=dict:store(clean, true, dict:new()) :: dict(),
             updatevalue :: term()
            }).
   -opaque riak_object() :: #r_object{}.

`riak_kv_vnode:do_put/7`::

   do_put(Sender, {Bucket,_Key}=BKey, RObj, ReqID, StartTime, Options, State) ->
       case proplists:get_value(bucket_props, Options) of
           undefined ->
               {ok,Ring} = riak_core_ring_manager:get_my_ring(),
               BProps = riak_core_bucket:get_bucket(Bucket, Ring);
           BProps ->
               BProps
       end,
       case proplists:get_value(rr, Options, false) of
           true ->
               PruneTime = undefined;
           false ->
               PruneTime = StartTime
       end,
       Coord = proplists:get_value(coord, Options, false),
       PutArgs = #putargs{returnbody=proplists:get_value(returnbody,Options,false) orelse Coord,
                          coord=Coord,
                          lww=proplists:get_value(last_write_wins, BProps, false),
                          bkey=BKey,
                          robj=RObj,
                          reqid=ReqID,
                          bprops=BProps,
                          starttime=StartTime,
                          prunetime=PruneTime},
       {PrepPutRes, UpdPutArgs} = prepare_put(State, PutArgs),         %% <===========
       {Reply, UpdState} = perform_put(PrepPutRes, State, UpdPutArgs), %% <===========
       riak_core_vnode:reply(Sender, Reply),

       update_index_write_stats(UpdPutArgs#putargs.is_index, UpdPutArgs#putargs.index_specs),
       UpdState.

`riak_kv_vnode:prepare_put/2`::

   prepare_put(State=#state{vnodeid=VId,
                            mod=Mod,
                            modstate=ModState},
               PutArgs=#putargs{bkey={Bucket, _Key},
                                lww=LWW,
                                coord=Coord,
                                robj=RObj,
                                starttime=StartTime}) ->
       %% Can we avoid reading the existing object? If this is not an
       %% index backend, and the bucket is set to last-write-wins, then
       %% no need to incur additional get. Otherwise, we need to read the
       %% old object to know how the indexes have changed.
       {ok, Capabilities} = Mod:capabilities(Bucket, ModState),
       IndexBackend = lists:member(indexes, Capabilities),
       %% LWW かつ Index しない場合は prepare_put/3 を省略
       %% 例: bitcask で LWW なら新しい値を書き込むだけ、最新のものが見える(?):
       %% 例: bitcask で NOT LWW なら prepare_put/3 で一度読む
       %% 例: leveldb なら prepare_put/3 にいく、IndexBackend=true を渡している
       case LWW andalso not IndexBackend of
           true ->
               ObjToStore =
                   case Coord of
                       true ->
                           riak_object:increment_vclock(RObj, VId, StartTime);
                       false ->
                           RObj
                   end,
               {{true, ObjToStore}, PutArgs#putargs{is_index = false}};
           false ->
               prepare_put(State, PutArgs, IndexBackend)
       end.

`riak_kv_vnode:prepare_put/3`::
   prepare_put(#state{idx=Idx,
                      vnodeid=VId,
                      mod=Mod,
                      modstate=ModState},
               PutArgs=#putargs{bkey={Bucket, Key},
                                robj=RObj,
                                bprops=BProps,
                                coord=Coord,
                                lww=LWW,
                                starttime=StartTime,
                                prunetime=PruneTime},
               IndexBackend) ->
       %% まずは backend GET で値を取ってくる
       GetReply =
           case Mod:get(Bucket, Key, ModState) of
               {error, not_found, _UpdModState} ->
                   ok;
               % NOTE: bad_crc is NOT an official backend response. It is
               % specific to bitcask currently and handling it may be changed soon.
               % A standard set of responses will be agreed on
               % https://github.com/basho/riak_kv/issues/496
               {error, bad_crc, _UpdModState} ->
                   lager:info("Bad CRC detected while reading Partition=~p, Bucket=~p, Key=~p", [Idx, Bucket, Key]),
                   ok;
               {ok, GetVal, _UpdModState} ->
                   {ok, GetVal}
           end,
       case GetReply of
           %% not_found or bad_crc のとき
           ok ->
               %% index つかえるか?
               case IndexBackend of
                   true ->
                       IndexSpecs = riak_object:index_specs(RObj);
                   false ->
                       IndexSpecs = []
               end,
               ObjToStore = case Coord of
                                true ->
                                    riak_object:increment_vclock(RObj, VId, StartTime);
                                false ->
                                    RObj
                            end,
               {{true, ObjToStore}, PutArgs#putargs{index_specs=IndexSpecs, is_index=IndexBackend}};
           %% backend に値があった時
           {ok, Val} ->
               OldObj = object_from_binary(Bucket, Key, Val),
               case put_merge(Coord, LWW, OldObj, RObj, VId, StartTime) of %% <====
                   {oldobj, OldObj1} ->
                       {{false, OldObj1}, PutArgs};
                   {newobj, NewObj} ->
                       VC = riak_object:vclock(NewObj),
                       AMObj = enforce_allow_mult(NewObj, BProps),
                       case IndexBackend of
                           true ->
                               IndexSpecs =
                                   riak_object:diff_index_specs(AMObj,
                                                                OldObj);
                           false ->
                               IndexSpecs = []
                       end,
                       case PruneTime of
                           undefined ->
                               ObjToStore = AMObj;
                           _ ->
                               ObjToStore =
                                   riak_object:set_vclock(AMObj,
                                                          vclock:prune(VC,
                                                                       PruneTime,
                                                                       BProps))
                       end,
                       {{true, ObjToStore},
                        PutArgs#putargs{index_specs=IndexSpecs, is_index=IndexBackend}}
               end
       end.

`riak_kv_vnode:put_merge/6`::

   %% いくつか枝があるうち Coord = true, LWW = false
   put_merge(true, false, CurObj, UpdObj, VId, StartTime) ->
       UpdObj1 = riak_object:increment_vclock(UpdObj, VId, StartTime),
       UpdVC = riak_object:vclock(UpdObj1),
       CurVC = riak_object:vclock(CurObj),

       %% Check the coord put will replace the existing object
       case vclock:get_counter(VId, UpdVC) > vclock:get_counter(VId, CurVC) andalso
           vclock:descends(CurVC, UpdVC) == false andalso
           vclock:descends(UpdVC, CurVC) == true of
           true ->
               {newobj, UpdObj1};
           false ->
               %% If not, make sure it does
               {newobj, riak_object:increment_vclock(
                          riak_object:merge(CurObj, UpdObj1), VId, StartTime)}
       end.

`riak_object:merge/2`::

   %% @doc  Merge the contents and vclocks of OldObject and NewObject.
   %%       Note:  This function calls apply_updates on NewObject.
   -spec merge(riak_object(), riak_object()) -> riak_object().
   merge(OldObject, NewObject) ->
       NewObj1 = apply_updates(NewObject),
       OldObject#r_object{contents=lists:umerge(lists:usort(NewObject#r_object.contents),
                                                lists:usort(OldObject#r_object.contents)),
                          vclock=vclock:merge([OldObject#r_object.vclock,
                                               NewObj1#r_object.vclock]),
                          updatemetadata=dict:store(clean, true, dict:new()),
                          updatevalue=undefined}.

衝突した場合

- `#r_object` の contents に メタデータ、値の組みを複数突っ込む。
- vclock は新旧マージして入れておく。

**TODO** もうちょっとちゃんとおいかける

siblings: クライアント側
========================

Riak における vclock により検知された並列更新 JSON データの読み込み時解決

- Ruby: https://gist.github.com/shino/2d700d005ff1e195dc15
- Python: https://gist.github.com/voluntas/a3f0547fcdd2fb5de1fc

CRDT
====

- "A comprehensive study of Convergent and Commutative Replicated Data Types"
  http://hal.upmc.fr/docs/00/55/55/88/PDF/techreport.pdf
- Strong Eventual Consistency and Conflict-free Replicated Data Types
  Microsoft Research
  http://research.microsoft.com/apps/video/dl.aspx?id=153540
- Ruby CRDT library by aphyr: https://github.com/aphyr/meangirls
- Python CRDT library by Eric Moritz: https://github.com/ericmoritz/crdt
- mochi/statebox: https://github.com/mochi/statebox

ものすごく簡単な例 : 追加しか出来ない集合 (Grow-Set, G-Set)

もうちょっと面倒な例 : 増加しか出来ないカウンタ (Grow-Counter, G-Counter)

複合系:
- 増減できるカウンタ(PN-Counter)
- 追加、削除もできる集合 (LWW-Set, Observed-Removed Set)

カウンタはホットデータになるときつい

クライアントサーバで考えると。。。

- 集合はどっちでもおk,クライアントでええやん (と僕は思う)
- カウンタはシリアライズ単位で ID が必要、Riak でサーバ側には ID がある、
  クライアント側でやるならどうする?? (MAC + PID + thread ID?)
  爆発注意! (実は Riak サーバ側でも発生する?)
- どこかで収束させる必要があるのか? (prune てきなもの)

