#情報ネットワーク学演習II 10/12 レポート課題
===========
学籍番号 33E16011
提出者 田中 達也

## 課題 (Cbench のボトルネック調査)

Ruby のプロファイラで Cbench のボトルネックを解析しよう。

以下に挙げた Ruby のプロファイラのどれかを使い、Cbench や Trema のボトルネック部分を発見し遅い理由を解説してください。

##解答
###実行
ruby-profを用いて解析を行った。
ruby-profの実行方法はまず以下のコマンドを入力してruby-profをインストールする。
```
gem install ruby-prof
```
続いて端末上のcbench-Tatsu-Tanakaディレクトリにおいて
```
ruby-prof ./bin/trema run ./lib/cbench.rb > cbench-profile.txt
```
を実行する。
これによりプロファイルの結果がcbench-profile.txtに出力されるようになる。
続いて別の端末を立ち上げ、cbench-Tatsu-Tanakaディレクトリにおいて、
```
./bin/cbench --port 6653 --switches 1 --loops 10 --ms-per-test 10000 --delay 1000 --throughput
```
を実行しベンチマークを開始する。
ベンチマーク終了後、最初に実行した端末において`Ctrl+C`を入力しcbenchを終了する。
同ディレクトリ上にプロファイル結果が記載された[cbench-profile.txt](https://github.com/handai-trema/cbench-Tatsu-Tanaka/blob/master/cbench-profile.txt)が出力される。

###解説
cbenchの実行時間は以下である。
```
RESULT: 1 switches 9 tests min/max/avg/stdev = 119.35/130.07/126.67/3.13 responses/s
```
cbench.rbを実行した時のプロファイル結果は[cbench-profile.txt](https://github.com/handai-trema/cbench-Tatsu-Tanaka/blob/master/cbench-profile.txt)である。
その中で、実行時間が長い上位10件を以下に示す。
```
 %self      total      self      wait     child     calls  name
  2.44     18.766     2.599     0.000    16.167   290316   BinData::Struct#instantiate_obj_at
  2.38     10.006     2.532     0.000     7.474   418775  *BinData::BasePrimitive#_value
  2.00      2.133     2.133     0.000     0.000   180669   String#sub
  1.96      2.217     2.088     0.000     0.129   260428   BinData::Base#get_parameter
  1.95      6.917     2.072     0.000     4.845   180661   BinData::Struct#base_field_name
  1.91     17.666     2.027     0.000    15.639   180661   BinData::Struct#find_obj_for_name
  1.84      3.253     1.962     0.000     1.291   253452   Kernel#define_singleton_method
  1.67      5.077     1.772     0.000     3.304   116084   Kernel#dup
  1.64      1.741     1.741     0.000     0.000   198363   Hash#[]=
  1.60      1.705     1.705     0.000     0.000   180661   Array#index
```
各項目の見方は以下である。
|%self|total|self|wait|child|calls|name|
|:--|:--|:--|:--|:--|:--|:--|
|全体時間の%|そのメソッドとそのメソッドから呼び出されたメソッドの合計時間|メソッドの呼び出し時間|スレッドの待機時間|メソッドから更に呼び出されたメソッドの呼び出し時間|呼び出された回数|メソッド名|

上位2番目の`*BinData::BasePrimitive#_value`が呼び出し回数が最も多いことから、ボトルネックであると考えられる。
これはBinDataの生成に時間がかかっていることを示している。
cbench.rbのpacket inハンドラの中で呼び出されている`send_flow_mod_add`について[ドキュメント](http://www.rubydoc.info/github/trema/trema/master/Trema/Controller:send_flow_mod_add)を参照したところ、[FlowMod](http://www.rubydoc.info/github/trema/pio/master/Pio%2FFlowMod%3Ainitialize)オブジェクトが生成されており、さらにFlowMod内ではBinDataを継承した[Format](http://www.rubydoc.info/github/trema/pio/master/Pio/FlowMod/Format#method_missing-instance_method)オブジェクトが生成されている。
そのため、`send_flow_mod_add`がボトルネックになっていると考えられる。

続いて用意されているcbenchを高速化したもの(fast_cbench.rb)を実行した。
1秒当たりの実行回数は以下である。
```
RESULT: 1 switches 9 tests min/max/avg/stdev = 325.23/385.14/363.00/19.54 responses/s
```
1秒当たりの実行回数が増加し、処理速度が向上していることが分かる。
fast_cbench.rbを実行した時のプロファイル結果は[fast_cbench_profile.txt](https://github.com/handai-trema/cbench-Tatsu-Tanaka/blob/master/fast_cbench_profile.txt)である。
```
 %self      total      self      wait     child     calls  name
  3.78     23.604     4.023     0.000    19.581   275621   BinData::Struct#instantiate_obj_at
  3.33      3.675     3.540     0.000     0.136   290410   BinData::Base#get_parameter
  2.54      4.552     2.705     0.000     1.847   283128   Kernel#define_singleton_method
  2.43     15.593     2.590     0.000    13.003   178298   BinData::Struct#find_obj_for_name
  2.42      2.578     2.578     0.000     0.000   207354   Hash#[]=
  2.36      7.966     2.507     0.000     5.459   178298   BinData::Struct#base_field_name
  2.27      3.743     2.410     0.000     1.333   241251  *BinData::BasePrimitive#_value
  2.26      4.703     2.402     0.000     2.302   172489   BinData::Struct::Snapshot#[]=
  2.19      2.328     2.328     0.000     0.000   178298   Array#index
  2.11      2.246     2.246     0.000     0.000   178306   String#sub
```
もとのcbench.rbでは上位2番目の`*BinData::BasePrimitive#_value`が呼び出し回数が減少しているため、実行時間が削減できていると考えられる。
[fast_cbench.rbのソースコード](https://github.com/handai-trema/cbench-Tatsu-Tanaka/blob/master/lib/fast_cbench.rb)では、ボトルネックとなっている`send_flow_mod_add`の記述をせず、
`@flow_mod ||= create_flow_mod_binary(packet_in)`というコードにしている。
`||=`という演算子は左辺の要素がnullまたは偽のときに代入を行うという記述である。
これにより、最初にPacket Inが生じた時は代入を行い、flow_modの処理を行うが、二度目以降のpacket inに対しては`@flow_mod`の内容は変えずにすぐFlow Modを返すことで、処理を高速化していると考えられる。

##参考文献
- デビッド・トーマス+アンドリュー・ハント(2001)「プログラミング Ruby」ピアソン・エデュケーション.  
- [テキスト: 7章 "すべての基本、ラーニングスイッチ"](http://yasuhito.github.io/trema-book/#learning_switch)  
- [rubydoc](http://www.rubydoc.info/github/trema/trema/master/Trema)



