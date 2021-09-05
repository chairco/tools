## sql 討論

就短短寫一下好了：
１）
從來不喜歡這種juicy的標題……
２）
所謂的Optimization，是指結果相同下用了更少的resource（像CPU time, memory / developer effort……）
原文中２個query，看起來相同，但是跑出來結果是不同的。
這種因為developer bug引起的side effect，叫不上optimization。
（下文第３＋４點會補充）
３）
學SQL的都會知道：
用上limit（和offset）時，一定會配上order by clause的。
Oracle, MySQL, Postgresql（其他自己查證別問我）
在沒配上order-by時，其排序是根據底層data storage的排序
所以，如果你用limit但是不配上order by時。
除非你只想知道「有沒有resultset滿足我的要求？」，或是「我就是只想隨便拿Ｎ個來工作」，否則這是bug。
而原文query是配上offset的，這99.9487%是bug
４）
為何會跑出這樣的結果？
首先，這個table是有對user.age加上index的。
（不然是跑不出第二個query會比較快的）
Query1：
MySQL optimizer無視user.Age column，直接做full table scan找出所有滿足age = 10的record，然後再作limit
Query2：
這個sub-query是有limit的，所以不管oracle / pg / mysql都是沒法做subquery unnesting要先行跑的。
然後，這個subquery只用上id和age，在mysql的世界下：
age這個index其實是 user.age --> user.PrimaryKey的 B+ tree
所以單純用上user.age這B+ tree，Ｏ(log N)下找到age = 10的leaf node然後再range scan，整個過程用上的data block很可能是10個以下
這當然很快
然後拿到id，再對user table跑10次的get record by PK，這也很快
５）
有人會問：
為何Query1時MySQL不會用上query 2的execution plan來跑？
我的實務回答：
理論上，ＭySQL的optimizer是cost-based的。
而現實上，ＭySQL的optimizer是出了名的爛，跟pg / oracle不是個一個層次的。
（就是optimizer爛，才需要有那麼多的SQL Hint......）
實務經驗告訴我：
Mysql的optimizer似乎不會把所有execution plan都拿出來計分，所以明明Mysql的optimizer是cost-based的，現實上跑起來行為卻像rule-based。
而另一個很大的可能性：
MySQL optimizer基於錯誤的column statistics去做cost estimation，然後就自己用了sub-optimium的execution plan。

ref:
1. https://allaboutdataanalysis.medium.com/%E4%B8%80%E9%81%93sql%E9%9D%A2%E8%A9%A6%E9%A1%8C-%E6%B7%98%E6%B1%B0494%E4%BD%8D%E5%80%99%E9%81%B8%E4%BA%BA-377cdcfb5201?fbclid=IwAR3X3HOY0qnNb4qblVBVKpy26p32k3_d3kf9BR9cDLp0aE9THJBPYYoWC7s
2. https://www.facebook.com/groups/616369245163622/posts/2462101053923756/
